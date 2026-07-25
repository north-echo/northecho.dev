---
title: "Mechanism-first variant analysis: notes on a methodology"
date: 2026-07-24
draft: true
summary: "Distill the structural shape of a confirmed bug class, build predicates that find sibling instances, then refute most of them. Notes on the class shape behind Dirty Pipe, Copy Fail and Dirty Frag, the impact-scoring filter that decides what to work on, an explicit human/AI role split, the discipline rules that came out of process failures, and the limits of the approach."
tags: ["linux-kernel", "security-research", "ai-agents", "variant-analysis", "disclosure"]
---

*Christopher Lusk, North Echo Security Research*

## Why this post exists

Between 2022 and 2026, the Linux kernel security community has disclosed several CVEs that look structurally related from a distance: Dirty Pipe (2022), Copy Fail and Dirty Frag (early 2026), an IVPU dma-buf protection-propagation bug (May 2026, Yametsu reporting, Wachowski fixing), a kTLS in-place encryption corruption bug in SMB (also May 2026), and others. They were each found by different researchers, in different subsystems, by different methods.

What they share, once you read all the writeups in sequence, is a structural shape. Each is a case where attacker-influenced state is staged in one execution context and consumed in another with different assumptions about whether the state has been mutated, freed, re-encoded, or shared across a trust boundary.

This post documents a methodology that uses that shape as the audit seed. The methodology produced three findings in its first two weeks of execution: one duplicate (someone else got there first), one source-only RFC independently found by another researcher within the same week, and one `[PATCH net]` series that drove a senior netdev maintainer to remove 1,147 lines of unmaintainable code from the kernel's net-next tree. The companion post, [*"The bugs are real. You're just not first."*][rediscovery], covers the duplicate-finding arc and the disclosure-process lessons. This post is the methodology layer underneath it.

I'm going to walk through the class shape, the scoring filter, the human/AI role split, the discipline rules that emerged from process failures, the Public-Fix Gate, the limits of the approach, and what I'm deliberately not publishing.

## The class shape

The bug class I'm describing has a specific structural definition:

> *Attacker-influenced state is staged in one execution context (syscall handler, request preparation, ring-buffer publication), then consumed in another execution context (worker, completion callback, retry/reconnect path, async crypto completion, IPC dispatch) under different assumptions about whether the state has been mutated, freed, encoded, copied, or shared across a trust boundary.*

Six members of this class as of mid-2026:

**Dirty Pipe** (CVE-2022-0847). A `struct pipe_buffer` is acquired from a pipe's ring, partially overwritten by a producer, with the `flags` field un-overwritten and carrying `PIPE_BUF_FLAG_CAN_MERGE` from a prior, semantically distinct owner. The next consumer treats the stale flag as authoritative and merges into a page-cache page that should have been read-only.

**Copy Fail** (CVE-2026-XXXX). The 2017 in-place AEAD optimization combined with the `authencesn` mode's scratch-byte writes lets an unprivileged user write four bytes outside the AEAD's documented output region, into the page cache of a file readable by their user.

**Dirty Frag** (CVE-2026-43284 + CVE-2026-43500). skb fragment chain state is mutated by an in-place crypto operation. Later code uses the fragment metadata under assumptions that no longer hold, corrupting the resulting packet.

**IVPU dma-buf RO propagation** (Wachowski's commit `7dd57d7a6350`, Reported-by Yametsu). A buffer object's read-only protection state is recorded at exporter-side pin time, then lost across a PRIME re-import boundary. The importer attaches with bidirectional DMA; the device gets a writable mapping of pages the kernel pinned without write permission.

**SMB in-place encryption corruption** (CVE-2026-43362). `SMB2_write` places write payload in `iov[1..n]`; `smb3_init_transform_rq` pointer-shares the iov; `crypt_message` encrypts iov[1] in-place. On replayable error, retry sends the already-encrypted iov[1] as if it were the original plaintext.

**kTLS async split-record loss.** `tls_push_record` splits the open record into a part being encrypted and a remainder. The synchronous path reattaches the remainder. The async path returns `-EINPROGRESS` before reattachment. The remainder is orphaned, dropped from the network, and leaks as a `tls_rec`. First reported 2026-05-15; drove the removal of the entire kTLS+sockmap integration from net-next (June 2026). Stable trees (4.20+) remain unpatched.

Six instances. Six different subsystems (fs/pipe, crypto, esp, accel/ivpu, smb client, net/tls). Six different specific mechanisms. One structural pattern.

That pattern is auditable. Specific functions process the same data twice with different rules. Specific code paths retain state across an async or retry transition. These are greppable, and once you know what to grep for, you can audit adopter sites across the kernel in a finite amount of work.

## The methodology in two sentences

The approach is simple to state and harder to execute:

> *Distill the structural pattern from confirmed members of a bug class. Build static-analysis predicates that find sibling instances, then prove or refute each by manual review and dynamic validation.*

That's it. The interesting work is in the distillation (you need real structural rigor, not a vague gestalt), in the predicate implementation (it's easy to write a predicate that flags everything), and in the refutation discipline (most candidates are false positives; you have to be willing to kill them).

Two contrasts worth naming:

**Versus fuzzing.** Fuzzing finds bugs by triggering them at runtime via mutated inputs. It's spectacular at memory-safety violations in heavily-tested entry points. It's worse at semantic invariant violations across module boundaries, because those require multi-step state setup that mutation can't synthesize cheaply. The mechanism-first approach finds bugs by structural pattern match against source, before the bug is necessarily reachable by a mutator. The two methodologies complement each other; they don't compete.

**Versus subsystem audit.** Traditional security audits go file-by-file or driver-by-driver within a specific subsystem. That's effective when you have deep subsystem knowledge and the target has been under-audited. Mechanism-first goes class-by-class across all subsystems, looking for the same structural pattern wherever it appears. Best when you have a published class member to seed from and you're willing to do shallower per-subsystem audits.

## The scoring filter

Worth naming up front: this methodology was seeded by the Dirty Pipe / Copy Fail / Dirty Frag CVEs not just because they were technically interesting but because they had reach. Each got coverage proportional to how dramatic the primitive was and how memorable the name was. A bug that doesn't get coverage doesn't get patched broadly in practice. The fix lands upstream, but the stable backports lag, the distro pushes lag, the customer awareness lags, the operators who actually have the vulnerable config in production never hear about it. Press-tier framing is therefore not vanity; it's the mechanism by which a real fix actually reaches the install base. The scoring filter below is what makes that bet explicit, and the choice of class members to seed from (the "Dirty" lineage) is deliberate continuity with the research family that has consistently produced fixes that travel.

The methodology needs a way to decide which candidates to commit engineering time to. After the IVPU work, which produced a real finding on a hardware-gated, modest-install-base subsystem (the kind of target where the scoring math doesn't favor the spend), I added an explicit impact filter that runs at candidate-selection time, not at candidate-promotion time.

Six dimensions, 0-5 each:

| dimension | low (1) → high (5) |
|---|---|
| Install base | <100K systems → universal |
| Press familiarity | never covered → weekly press coverage |
| Primitive demonstrability | theoretical only → unauth RCE / file overwrite |
| Reachability | requires CAP_SYS_ADMIN → network-default unauth |
| Latency in tree | <1 year → ≥10 years |
| Fix showability | complex patch series → 1-3 line fix |

Anchor calibration against public CVEs: Dirty Pipe scored ~28/30, Copy Fail ~24/30, Dirty Frag ~22/30, the Rift bug in nginx (CVE-2026-42945) ~28/30. These are the headline-tier findings the methodology aspires to find more of.

Promotion threshold: 22+ for dynamic-validation work; 25+ for full disclosure prep. Abandonment threshold: below 18, the candidate doesn't justify the engineering time even if the bug is real.

Worked examples from the recent two-week run:

- **IVPU finding** scored ~19/30 from the moment it was identified. Hardware-gated (Intel Core Ultra NPU), short audit history, lower press familiarity than fs/ subsystem work. Mid-tier. The filter said "viable for methodology validation, not headline." That's what I should have noticed before spending the validation budget.
- **Etnaviv finding** scored ~14-16/30. Small Vivante-GPU install base, similar reachability constraints. Below 18 by strict reading. Pursued anyway because it was the methodology's clean second instance and the upstream contribution is real even at low scoring.
- **kTLS finding** scored ~26-28/30 from identification. Network-reachable (kTLS in HTTPS data path), high press familiarity (every kTLS CVE gets coverage), 7-year latency in tree (`Fixes: d3b18ad31f93`), small fix. Above the 25 disclosure-prep threshold. The scoring justified the engineering time the moment the candidate surfaced. Distro-impact verification (done during the v2 day-of-send pass) confirmed that every modern enterprise distro kernel from the 4.18-based enterprise family onward carries the bug-introducing commits. The version-affected install base is larger than the methodology had initially estimated, though real-world exposure is gated by a triple-config trigger (kTLS TX + BPF sockmap + async pcrypt loaded) that keeps the practical at-risk population narrower than the version range alone implies.

The filter is a discipline tool, not an oracle. It can't tell you if a bug exists, only whether, conditional on finding something, the target's structural properties would land a headline-tier finding. But that conditionality is exactly what's missing from "I'll audit this subsystem because it looks interesting."

## AI tools: explicit role separation

This is the part most readers came here for, and the part I want to describe most carefully.

The methodology uses two LLM-class tools alongside the human operator. The role split is explicit and load-bearing:

**Codex (gpt-5.5)** drives the static-analysis variant hunt. It authors predicates, runs corpus enumeration, surfaces candidate sites, and proposes the first-cut patches. It works fluently across the kernel source tree and produces large structured outputs quickly. Its strength is parallel exploration and pattern recognition.

**Claude (`claude-opus-4-6` through the May 2026 cycles, `claude-opus-5` from July 2026)** drives the disclosure-prep layer. Reads Codex's outputs adversarially, verifies canonical-source claims, runs the Public-Fix Gate at each stage, drafts the actual external-facing prose (cover letters, blog posts, this one), and enforces sourcing discipline. Its strength is structured writing, adversarial review, and consistent application of process rules.

**Human operator (me)** owns hardware validation, every external action, every gate decision, and every "press send" call. Coordinates the two AI agents, decides what to publish vs. withhold, and takes legal responsibility under the kernel's DCO.

Per the kernel's `Documentation/process/coding-assistants.rst` guidance: AI agents do not add `Signed-off-by` tags. Only humans certify the DCO. The patches I sent carry exactly one `Signed-off-by: Christopher Lusk <clusk@northecho.dev>` and two `Assisted-by:` trailers, one per AI agent, chronological by contribution.

Per `Documentation/process/security-bugs.rst` (as of commit `a03ef333fbd6`, Willy Tarreau, landed 2026-05-12): LLM-assisted findings explicitly fall under the exception clause for findings "trivial to discover (e.g. result of a widely available automated vulnerability scanning tool that can be repeated by anyone, **or use of AI-based tools**)." The "or use of AI-based tools" parenthetical was added in that commit. Practically: LLM-assisted findings go to public lists, not `security@kernel.org`. The kTLS series went to `netdev` directly, as the Etnaviv RFC did to `dri-devel`. Neither used the kernel security team's private channel.

The same commit adds a new "Responsible use of AI to find bugs" section that imposes five concrete formatting requirements: concise human-style reports (not the multi-page Markdown-laden output AI tools default to), plain text only, verifiable facts grounded in the kernel's threat model (codified in the new `Documentation/process/threat-model.rst`), thoroughly-tested reproducers, and tested proposed fixes with `Fixes:` tags. The kTLS cover letter and patches comply with all five.

One tension worth surfacing honestly: the new doc says *"since the report will be posted to a public list, the reproducer should only be shared upon maintainers' request."* The kTLS v1 series included a TAP-style selftest as patch 2/2, which is a deterministic reproducer when run against an unpatched kernel. That bundles a reproducer publicly. The choice was guided by netdev's strong convention of "selftests with bugfixes please," without which a maintainer would have to reproduce the bug from scratch to verify our claim. Kicinski's v2 reply resolved this tension directly: "There are already tests for sockmap + TLS in bpf, point your bots at that please." v3 onward dropped the selftest. The two kernel conventions (netdev: bundle selftests; new AI rules: withhold reproducers) are not yet fully reconciled; the methodology errs toward the more specific subsystem convention when the conflict arises, and defers to maintainer direction when given.

The honest part of this layer:

The IVPU first send to `security@kernel.org` did not include the AI-assistance disclosure. I learned about Willy's policy guidance after the send (his reply on the IVPU thread became the catalyst for what the docs commit now codifies). The lesson was simple: AI-assistance disclosure goes in the *first* contact, not the follow-up. By the time of the Etnaviv RFC and the kTLS series, that rule was internalized; both cover letters lead with an explicit "this work is LLM-assisted" paragraph that names each agent, its model version, its role, and links to this methodology context.

A timing irony worth naming: the kTLS series was sent at 11:15 EDT (15:15 UTC) on 2026-05-15. Linus merged the security-docs update at 19:24 UTC the same day, ~4 hours later. The send predated the codification, operating under Willy's email guidance plus the pre-existing "trivial to discover" exception. The codification confirmed that interpretation; it didn't constrain a send already in motion.

The two-agent split is not a marketing choice. It exists because the two agents fail differently. Codex over-claims confidence on patterns; Claude under-claims and asks for verification. Codex's output benefits from a different agent reading it adversarially before it becomes anything externally visible. The human review remains the final gate, but the agent-on-agent adversarial layer catches a lot before it reaches me.

A real-world data point arrived 24 hours after the kTLS v1 send. A third agent, Sashiko, an LKML-attached automated reviewer at sashiko.dev, found a genuine UAF in the v1 patch I'd sent. Specifically: the return-value masking in `bpf_exec_tx_verdict()` could lose a pending `-EINPROGRESS` signal when a later verdict iteration hit a hard error, allowing `MSG_ZEROCOPY` pages to be released while the crypto engine was still reading them. The bot's analysis was sharp enough that it got endorsement from John Fastabend, the original author of the commit our `Fixes:` tag points to, when he engaged on the thread five days later. The series went through four versions in response: v2 addressed Sashiko's UAF with a drain-on-error fix; v3 switched to Kicinski's preferred surface-narrowing approach (AEAD allocation mask at TLS setup time); v4 dropped an unnecessary field latch per Jiayuan Chen's review.

The equilibrium this points at is the one the new kernel AI policy documents codify: AI-assisted research and AI-augmented review both operate in public, under scrutiny, improving the work. An AI bot finding a real bug in an AI-assisted patch is the system working, not a contradiction. That the process ultimately led a senior maintainer to remove 1,147 lines of unmaintainable code, rather than patching the specific bug, validates the methodology's deeper claim: the class-shape analysis found a code surface that was structurally unsound, not just a single instance to patch.

### Two things I got wrong that the tools did not catch for me

The Etnaviv follow-up in July produced two failures worth naming, because both are the kind an AI-assisted workflow makes *more* likely rather than less.

**A maintainer's suggested mechanism is not a claim about that mechanism's semantics.** Lucas Stach, reviewing a parallel patch for the same bug, suggested using `ETNA_BO_FORCE_MMU` on read-only userptr buffers rather than rejecting the mapping outright. That is sound advice and I took it. The draft I built on top of it then claimed the result made read-only userptr genuinely read-only. It does not. On MMUv1 hardware the page table entries carry no writeable bit at all, and `etnaviv_iommuv1_map()` accepts a protection argument and discards it. Forcing the buffer through the page tables keeps the GPU confined to the mapped pages, which is worth having, but it is containment and not write protection.

Nobody lied to me. Lucas answered the question he was asked, about routing. I heard an answer to a question he was not asked, about enforcement. The correction came from reading the two map functions directly, which took about ten minutes and which I should have done before writing the claim rather than after. When a maintainer hands you a mechanism, you still owe the reader a check that the mechanism does the thing your changelog says it does.

**An automated reviewer points at a mechanism, not at its full reachable surface.** Fifteen minutes after that patch went out, Sashiko flagged that read-only userptr buffers are DMA-mapped `DMA_BIDIRECTIONAL`, so under bounce buffering the unmap copies data back over pages the kernel pinned read-only. Correct, and a real second write path that the MMU fix does not close.

The useful part was what the bot did not say. Following the finding meant asking where else that direction comes from, and the answer was `etnaviv_gem_cpu_prep()`, which derives its sync direction from a *userspace-supplied* argument. Ask for `ETNA_PREP_READ` and you get `DMA_FROM_DEVICE`, which bounces back exactly the same way, with no GPU submission involved and no limit on repeats. It is the more accessible of the two paths and the report never mentioned it. A fix written to the report as filed would have closed the harder path and left the easier one open.

So the rule I would give anyone wiring an automated reviewer into their process: treat its findings as leads to investigate, never as finished bug reports. Also check its severity ordering against your own. That run rated the reachable DMA issue "High" and rated two genuinely out-of-scope pre-existing items "Critical," because the labels track pattern severity rather than reachability in the code you actually touched.

## Sourcing discipline: the IVPU lessons

The IVPU disclosure cycle had six distinct sourcing failures. They weren't catastrophic, since no patch shipped externally with the worst of them, but they were embarrassing enough on a public-list thread that the lessons became durable. I'll catalog them, name what each rule prevents, and note how the kTLS cycle's clean execution shows the rules work when applied.

1. **Embargo policy citation was stale.** I cited an outdated Openwall `linux-distros` policy from memory. The actual current policy is 14-day max once the list is notified. *Rule:* fetch the current policy page from canonical source on the day of send; quote the actual text.

2. **Downstream backport status was misframed.** I asserted that a specific enterprise distribution was not affected, without accounting for distro hardware-enablement backports. *Rule:* state "upstream X.Y+ is affected; each distribution security team must verify their tree."

3. **Affected-version range was wrong.** I said v6.18+ when the correct answer was v6.19+. *Rule:* verify by `git show <tag>:<vulnerable-source-file>` against each candidate tag.

4. **Maintainer names were from prior-session memory.** I named maintainers who had moved on; the current `MAINTAINERS` listed different people. *Rule:* always run `scripts/get_maintainer.pl --no-rolestats --no-git-fallback <file>` against a fresh kernel tree on the day of send.

5. **Upstream HEAD was never checked.** The big one. The IVPU bug had been fixed upstream two weeks before my disclosure send. A one-line `git log` against the maintainer's tree on day-of-draft would have caught the duplicate-finding state. *Rule:* the Public-Fix Gate (see next section) runs at every commitment point, not just at candidate-promotion time.

6. **AI-assistance disclosure was omitted from the first send.** Willy Tarreau caught it. His IVPU-thread reply was the catalyst for what is now codified in `Documentation/process/security-bugs.rst` (commits `a03ef333fbd6` and `4bf85afb9f3e`, both landed 2026-05-12). *Rule:* AI-assistance disclosure goes in the cover letter prose and the patch trailers from the first external contact onward.

The kTLS cycle's clean execution traces every one of these rules back to its corresponding failure:

- Embargo policy: not applicable (public-list-first for LLM-assisted)
- Backport scope: explicit `Cc: stable@vger.kernel.org # 4.20+` trailer, with the introducing commit (`d3b18ad31f93`, John Fastabend, 2019) cited as the `Fixes:` tag
- Affected-version range: bisected against the actual introducing commit's merge tag, not a release-date estimate
- Maintainer names: `get_maintainer.pl` run from a fresh shallow clone of `net.git` on the day of send. The recipient set correctly dropped two BPF maintainers that an older list had included.
- Upstream HEAD: re-grepped on day of send against Torvalds master, `net`, `net-next`, and `linux-next`; confirmed the vulnerable shape still present.
- AI-disclosure: visible in the cover letter, visible in both patches' trailers, visible at the public-archive landing.

None of the IVPU failures repeated in the kTLS cycle. That's the proof the rules are durable, not aspirational.

Two additional discipline rules emerged from the kTLS cycle that the IVPU experience couldn't have produced:

7. **Day-of-send Public-Fix Gate runs against every relevant tree, not just one.** The kTLS day-of-send check verified absence of competing fixes across `net.git`, `net-next`, `linux-next`, `bpf`, and `bpf-next`, five trees in total. A check against only the primary maintainer tree would have left blind spots for fixes queued in `bpf-next` or staged in `linux-next`.

8. **KASAN+LOCKDEP-instrumented runtime validation is mandatory for any fix that changes async or error-path composition.** A TAP-passing selftest is necessary but not sufficient. The kTLS v2 cycle's pass-then-drop BPF probe, a reproducer specifically targeting the failure mode the Sashiko bot identified, ran clean under both sanitizers against a known-vulnerable base before send. The lesson: when a fix restructures the way an async path interacts with an error path, sanitizer evidence is what separates "the test passes" from "the test doesn't accidentally pass while the new code path leaks."

## The Public-Fix Gate at multiple stages

The Public-Fix Gate is the single most important discipline rule. Briefly: before drafting, before promoting, before sending, before publishing: verify that the bug you think you found is still open upstream, against canonical source, on the actual day of the action. Never rely on the previous cycle's state.

The gate runs at four distinct points:

**1. Lane selection.** Before committing engineering time to a candidate set, confirm the structural pattern still exists in mainline and that no maintainer tree has a queued fix. This is the gate IVPU failed (we never ran it; the bug was already fixed upstream by the time I started validation).

**2. Candidate promotion.** Before claiming a static finding as something worth dynamic-validating, re-check the specific code path. Subsystem-specific fixes sometimes land in adjacent commits that look unrelated until you read the diff carefully.

**3. Day-of-send.** Before the SMTP connection fires, fresh check against the canonical source. This caught a real near-miss during the kTLS cycle: Codex re-tested the reproducer against a kernel patched with the May 11 SG-chain fixes (commits `285943c6e7ca` and `ff26a0e8377d`) and confirmed our specific bug persists with those fixes applied. The adjacent-fix question was real; the answer was "different bug."

**4. Pre-merge response.** When a maintainer is about to apply your patch, do one more check that nothing has landed in the interim that obviates or conflicts with your fix.

The principle: an upstream-HEAD check is a two-minute operation; the cost of skipping it is publishing a duplicate finding. The math heavily favors running the check every time, even when it feels redundant.

A concrete example of the gate firing usefully: before the kTLS send, a lore search surfaced Jakub Kicinski's "[PATCH net 0/7] net: tls: fix some issues with async encryption" series from February 2024. The series title looked dangerously similar to our target area. Verification: the 2024 series addressed RX-side async issues and the `-EBUSY` backlogging family (which became CVE-2026-31533). Our bug is in the TX `-EINPROGRESS` split-remainder path, a different code flow. The gate result was "adjacent, not duplicate." Without the explicit check, the similarity could have looked like a duplicate.

A second concrete example from the v2 cycle: the day-of-send pass nearly didn't run at all because the build host hit 100% disk utilization from a stale build tree (the linux-v7.1-rc2 source that had been used for the v2 KASAN+LOCKDEP validation, still carrying ~29 GB of compiled artifacts). The git fetch failed silently, with "no space left on device." The discipline forced a `make clean` + retry rather than skipping the gate. The IVPU lesson is direct: "gate skipped because of a tooling problem" is indistinguishable in outcome from "gate ran clean," and the discipline only works if "tooling stopped me" forces resolution, not bypass.

## Limits and anti-patterns

This methodology is not a universal kernel-bug finder. There are real classes of bugs it doesn't address well, and real failure modes I've hit applying it.

**Memory-safety bugs that fuzzers already find.** oss-fuzz has been hammering kernel subsystems for years with sanitizer-instrumented builds. Pure heap-overflow, use-after-free, and double-free bugs in heavily-fuzzed entry points are increasingly hard to find by manual analysis. The mechanism-first approach's edge is in semantic/logic/state-machine violations, not memory safety. If your candidate predicate flags raw memory bugs, expect a hostile baseline.

**Bugs requiring deep subsystem expertise.** Scheduler races, RCU correctness violations, virtual-memory subsystem invariants: these require years of subsystem familiarity to find and reason about. Class-shape audit doesn't substitute. The researchers who land brand-tier findings in these areas consistently (Project Zero, Qualys, V4bel) have multi-year sustained focus. The methodology described here is complementary to that, not a replacement.

**Predicates aren't findings.** A static-analysis predicate that fires on 50 sites doesn't tell you you've found 50 bugs. It tells you you have 50 candidates to manually triage, of which 45+ will be false positives. The discipline is in the refutation. Treating predicates as findings is the fastest way to publish a wrong claim.

**Reachability optimism.** During the retained-iterator campaign, Codex's first prioritization put RxRPC at the top of the audit queue (predicate confidence + alphabetical ordering). RxRPC has real bugs but a tiny install base, the AFS-using population. A correct re-ranking by reachability x install base put kTLS first (which is what eventually surfaced the finding). Sorting by predicate-confidence-tier alphabetical is an anti-pattern.

**Skipping discipline steps after a clean cycle.** The first post-IVPU send (Etnaviv) was clean. The temptation after a clean cycle is to skip the practice round, skip the day-of-send gate, trust the gitconfig. Don't. The discipline exists precisely for the cycle where it's harder to enforce. The kTLS cycle hit a local-Postfix near-miss (a `git send-email` run on macOS queued three messages in local Postfix because the Mac's git wasn't configured with the Gmail SMTP path I'd verified on KELSO). The discipline of running from KELSO with explicit `--smtp-server` flags is what prevented the partial-send escalation.

**This is not a quick win.** Each sub-track is 4-8 weeks of focused work. Anyone reading this looking for a methodology that produces findings in days should know that up front. The kTLS finding was the methodology's third sub-track over the course of two weeks of work, and even then, "two weeks" is the calendar window; the actual concentrated audit time was substantially less, spread across several distinct campaigns.

## What this is, what it isn't

To be specific about claims:

**This is:**

- A structured, repeatable methodology for finding sibling bugs to known class members.
- A working example of explicit AI/human role separation in security research, with each role's outputs and gates documented.
- A strict-discipline approach to coordinated disclosure under the kernel's LLM-assisted public-list policy.
- Evidence, in the form of a kTLS finding that drove a 1,147-line code removal from the kernel's networking stack, that the approach produces real upstream impact.

**This is not:**

- An automated bug finder. Every claim requires human validation; predicates surface candidates, not findings.
- A competitor to fuzzing. The two approaches occupy different niches and complement each other.
- A guarantee of headline-tier findings. The scoring filter optimizes for reach *conditional on finding something*, not for guaranteeing a find.
- A replacement for deep subsystem expertise. Researchers who have spent years inside specific subsystems will continue to find bugs the methodology can't reach.
- A claim that AI tools "found the bug." The bug was found by a human-driven research program that used AI tools as force multipliers under specific role constraints.

## What I'm deliberately not publishing

This post describes the frame. It does not describe the implementation. Specifically:

- The static-analysis predicates themselves (the Go code under `pkg/sinks/`) stay local. They are directly executable against any kernel; publishing them helps adversaries as much as it helps defenders.
- The hypothesis sets per sub-track (the Y1-Y8 enumerations in each campaign spec) stay local. They name the specific files and functions to grep for. Publishing them is a roadmap.
- The forward audit target list (what subsystems and shapes I'm going to audit next) stays local. Forward research depends on not telegraphing it.
- The Codex configuration and prompt structure stay local. Reproducibility of the agent setup is competitive advantage.

The principle: publish the frame, keep the target list private. The frame's value comes from being widely understood, cited, and critiqued. The target list's value comes from being privately held, at least until each target's specific finding is published on its own terms.

This is the same model Qualys uses for their kernel work (Sequoia, Looney Tunables, Fragnesia): retrospective writeups publish the methodology and the specific finding after disclosure; forward research stays private.

## A note on distro impact

Once a class member is confirmed in mainline, the next question is the cross-distribution exposure surface. The kTLS finding's distro-impact verification is worth describing because it changes how the methodology's outputs land in conversations with security teams.

The bug-introducing commits are `d3b18ad31f93` (2018-12, the BPF sk_msg integration) and `a42055e8d2c3` (2018-09, the async TLS infrastructure). Both predate mainline v4.20. By default, that means any pre-v4.20 distro kernel is structurally not affected.

However: the 4.18-based enterprise Linux family backported both commits through its normal feature-backport pipeline. Verified against the published source of a 4.18.0 enterprise kernel from that family, frozen at its final 2024 build: `bpf_exec_tx_verdict` present, the `-EINPROGRESS` split-remainder code path intact. That moves the whole enterprise Linux 8 generation, and its rebuilds, from "unaffected by version" to "affected by backport." Every modern enterprise distro kernel in production is in the affected range:

| distro                     | kernel              | affected? |
|----------------------------|---------------------|-----------|
| Enterprise Linux 8 family  | 4.18.0-based        | yes (verified backport) |
| Enterprise Linux 9 family  | 5.14                | yes |
| Enterprise Linux 10 family | 6.12                | yes |
| Ubuntu 22.04 / 24.04 LTS   | 5.15 / 6.8          | yes |
| Debian 11 / 12             | 5.10 / 6.1          | yes |
| SLES 15                    | 5.x                 | yes |
| Amazon Linux 2 / 2023      | 5.x / 6.x           | yes |
| Oracle UEK7+               | 5.15+               | yes |

The triple-configuration gate keeps the real-world exposure narrower than the affected-version range: the bug needs kTLS TX active on a socket, a BPF sk_msg verdict program calling `bpf_msg_apply_bytes()`, and an async-capable AEAD provider (typically `pcrypt` loaded). That conjunction is uncommon in default installs and characteristic of specific high-performance production deployments, most realistically Cilium-style service meshes terminating TLS on kernel sockets with `pcrypt` loaded for crypto offload throughput. Default desktop and default-server installs are not at practical risk; specific cloud production deployments are.

Both numbers, the affected-version-range universal across enterprise distros and the real-world install base narrowed by the config gate, belong in writeup of any methodology member with this profile. Eliding either understates the work in opposite directions: ignoring the broad version range underestimates the backport tail; ignoring the config gate overestimates the live attack surface.

## What's next

As I publish this:

- The kTLS finding is settled. The v1 series (sent 2026-05-15) went through four versions of review on netdev. Sashiko bot found a genuine UAF in v1; John Fastabend endorsed it. v2's drain-on-error approach drew Kicinski's response: surface-narrow rather than patch. v3 (AEAD allocation mask) drew Jiayuan Chen's observation that the copy-path also has an aliasing bug with `bpf_msg_pop_data()`. The endpoint: Kicinski removed the entire kTLS+sockmap integration from net-next on 2026-06-14 in a 5-patch series, citing "no known users" and "hard to solve bugs." Sabrina Dubroca, Jakub Sitnicki, and Paolo Abeni reviewed it. No CVE has been assigned. Stable trees (4.20+) remain unpatched.

- The Etnaviv finding is open. Our RFC (sent 2026-05-14) and a parallel patch from Ziyi Guo (sent 2026-05-08) both address the same bug; neither merged. Lucas Stach reviewed Ziyi's patch and requested a different approach. A v2 implementing his feedback is in preparation. The bug remains in the kernel.

- The IVPU finding was a duplicate. Fixed upstream 2026-04-30. Closed.

- The dma-buf class-level question, whether the framework should carry exporter-side access restrictions across attach boundaries, stands regardless of how any specific driver instance was resolved. Two independent researchers finding the same Etnaviv gap within six days of each other suggests the pattern generalizes. A framework-level RFC is a future item.

- Forward campaigns are scoped but paused. Specific targets stay private.

If you've read this far and you're a kernel security researcher, maintainer, or someone applying similar methodology and wanting to compare notes, I'm reachable at `clusk@northecho.dev`. I'm particularly interested in:

- Critique of the class-shape definition. Does it generalize beyond the six members I've listed?
- Refinement of the scoring filter. The 6-criterion model is my current best guess; sharper formulations would help.
- Counter-examples to the methodology. Bugs that look like they should be findable this way but aren't, or bugs that turned out not to be class members on closer inspection.
- AI/human role-split patterns from other researchers. The Codex/Claude/human triad works for me; I'd like to know what works for others.

This is independent research under the North Echo Security Research identity. I work on this in my own time. The work is not affiliated with my day job; the identity separation is durable and intentional.

## Credits and acknowledgments

The IVPU class instance: **Yametsu** (original reporter), **Karol Wachowski** (Intel NPU driver maintainer, author of the upstream fix), **Andrzej Kacprowski** (reviewer).

The Etnaviv finding: **Ziyi Guo** (Northwestern, independent co-discoverer), **Lucas Stach** (Pengutronix, Etnaviv maintainer), **Russell King**, **Christian Gmeiner**.

The kTLS finding and removal: **Jakub Kicinski** (netdev maintainer, author of the net-next removal series), **John Fastabend** (BPF/kTLS expertise, endorsed the Sashiko UAF finding and confirmed no deployed users), **Jiayuan Chen** (found the copy-path aliasing issue on v3), **Sabrina Dubroca**, **Jakub Sitnicki** (reviewed the removal), **Paolo Abeni**, **Eric Dumazet**, **Shuah Khan** (kselftest).

The Sashiko AI review bot (sashiko.dev) found a real bug in our v1 patch. Credit where earned.

The disclosure-discipline framing: **Willy Tarreau** (kernel security team) for the IVPU-thread guidance and the `security-bugs.rst` / `threat-model.rst` updates (commits `a03ef333fbd6` and `4bf85afb9f3e`, reaching mainline in Linus's merge `36d49bba19f2`) that codified public-list-first for LLM-assisted findings.

Methodological precedents and prior art: **Max Kellermann** (Dirty Pipe writeup as the canonical example of methodology described honestly); **V4bel** (Dirty Frag); the **Copy Fail** authors; the **Project Zero** team for the institutional model of public-frame + private-target-list; the **Qualys** Threat Research Unit (Bharat Jogi and team) for the retrospective-disclosure pattern; **Daniel Stenberg** and the **curl** team for the AI-slop pushback that shaped the discipline rules around human validation.

LLM acknowledgments per the methodology described above:

- **Codex** (OpenAI, model `gpt-5.5`): static-analysis variant hunt, predicate authoring, corpus enumeration, candidate surfacing across the kernel source tree.
- **Claude** (Anthropic, model `claude-opus-4-6` for the May 2026 disclosure cycles, `claude-opus-5` for the July 2026 work including the current revision of this post): disclosure prep, adversarial review of Codex outputs, sourcing-discipline enforcement, patch refinement, writeup of this post and the companion rediscovery post.

All operator decisions, hardware validation, and external sends remain human-driven.

---

[dp]: https://dirtypipe.cm4all.com/
[cf]: https://copy.fail/
[df]: https://github.com/V4bel/dirtyfrag
[rediscovery]: /posts/the-bugs-are-real-youre-just-not-first/
[ivpu-fix]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7dd57d7a6350
[etnaviv-rfc]: https://lore.kernel.org/all/20260514131401.2660079-1-clusk@northecho.dev/
[ktls-v1]: https://lore.kernel.org/all/20260515151556.189841-1-clusk@northecho.dev/
[ktls-v3]: https://lore.kernel.org/all/20260526025154.60607-1-clusk@northecho.dev/
[ktls-removal]: https://lore.kernel.org/all/20260614014102.461064-1-kuba@kernel.org/
[coding-assistants]: https://docs.kernel.org/process/coding-assistants.html
[security-bugs]: https://docs.kernel.org/process/security-bugs.html

---
