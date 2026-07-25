---
title: "The bugs are real. You're just not first."
date: 2026-07-24
draft: true
summary: "Three mechanism-first research cycles in 2026, three independent rediscoveries: an Intel NPU dma-buf bug fixed upstream two weeks before I sent my disclosure, an Etnaviv variant someone else patched six days before my RFC, and a freenginx regression its own maintainer fixed mid-cycle. Being repeatedly second is evidence the predicates point at real defects, and the interval that beat me was my own disclosure-prep process."
tags: ["linux-kernel", "security-research", "disclosure", "variant-analysis", "ai-agents"]
---

*Christopher Lusk, North Echo Security Research*

## TL;DR

Over about three months in 2026 I ran a mechanism-first variant analysis against a class of Linux kernel bugs, the family that includes Dirty Pipe, Copy Fail, and Dirty Frag, across three separate targets. The method worked. Every cycle produced a real, confirmable defect.

In three of those cycles, somebody else got there first.

- **Intel NPU (`drivers/accel/ivpu`)**: read-only state lost across a dma-buf re-import boundary, with hardware-validated cross-process page-cache mutation. Already reported by another researcher, and the fix had merged to mainline two weeks before I sent my disclosure.
- **Etnaviv (`drivers/gpu/drm/etnaviv`)**: the same class, a read-only userptr flag ignored by the GPU MMU mapping path. Ziyi Guo of Northwestern posted a patch for it **six days before** my RFC hit the list.
- **freenginx** (a userspace sub-track): a regression of the same structural shape, independently fixed by Maxim Dounin partway through my cycle, caught by a pre-send gate before I mailed anyone.

Three for three. That is not bad luck. At three it stops being an anecdote and starts being a property of the method, and I think it is the most useful thing I have to report.

The naive reading is that the methodology is too slow to matter. I think the correct reading is close to the opposite, and this post is mostly an argument for why: **repeatedly landing on defects that competent, independent people also classify as defects is the strongest available evidence that the predicates point at real bugs rather than at noise.** Novelty and correctness are different axes, and only one of them is about being first.

There is a fourth cycle where I *was* first, and it is instructive about what winning actually pays. A kTLS TX bug (`tls_push_record` losing the split open-record remainder on `-EINPROGRESS`), latent since 2019. Nobody had it. The analysis drove Jakub Kicinski to delete the entire kTLS+sockmap integration from net-next in June 2026, 1,147 lines removed, citing "some known hard to solve bugs" and "no known users." No CVE. No `Reported-by`. No `Link:` back to my thread. Being first bought me a deleted subsystem and no attribution.

I am not claiming a CVE, and on three of these I am not claiming priority. I am writing it up because the security community publishes the wins and quietly shelves everything else, and that asymmetry makes the field look far more linear from the outside than it is.

## The bug, briefly

The Intel NPU driver supports user-pointer GEM buffer objects: a userspace process can ask the kernel to wrap a region of its own memory as a device buffer for NPU compute. The driver supports a read-only flag, intended to communicate that the kernel may pin the underlying pages without write permission and that the device must not be allowed to write through them.

When such a buffer is exported via PRIME, imported into a second DRM file (a separate process or even a separate user), and re-bound for NPU submission, the read-only attribute is lost on the re-imported side. The device gets a writable mapping of pages the kernel pinned without write permission. Because the same physical pages may be backing a read-only file mapping in another process (including pages in the kernel's page cache for a file the attacker only has read access to), an NPU write submission against the re-imported buffer can mutate bytes observable to other readers of the same file.

I confirmed the impact end-to-end on hardware I bought specifically for this validation. I am deliberately not publishing the reproduction chain here. The upstream patch is in mainline as of 2026-04-30, but as I write this it has not yet propagated to the v6.19.x or v7.0.x stable kernel trees. Distribution kernels following those stable branches are still exposed, and a detailed reproducer would primarily serve to arm attackers against unpatched systems.

The upstream commit message describes the bug in terms that match what I observed: *"Re-exporting imported GEM buffers causes loss of buffer flags settings, leading to incorrect device access and data corruption."* That sentence is the disclosure-safe description. If you maintain a kernel and you want to know whether your tree is affected, that's the symptom; the patch hash is `7dd57d7a6350`, with a `Fixes:` trailer pointing at the introducing commit `57557964b582` (the userptr-buffer-object feature that landed in v6.19).

## How I got there: variant-analysis methodology

Dirty Pipe, Copy Fail, and Dirty Frag share a structural shape that none of the four post-disclosure writeups elevated to a kernel-wide design statement. In each case, a layer of the kernel made a decision about read-only or write-restricted access to memory; a different layer, in a different subsystem, undid that decision without realizing it had been made. The bug-finding pattern that produced those four findings was not "audit a subsystem." It was "find a piece of inherited state that one layer trusts and another layer overwrites."

Worth being explicit about the motivation alongside the technical observation: those four CVEs got the press coverage they did because their bug shapes were dramatic and the names stuck. I noticed the pattern and asked whether there were undisclosed siblings worth finding, partly because the underlying problem generalizes (and a generalizable defect deserves a generalized audit), and partly because press-tier framing is the mechanism by which a real fix actually reaches the install base. A bug that gets a CVE number and stays at a number lands upstream and backports in a trickle; a bug with a name and a writeup gets operators looking at their configs. The press-tier targeting is security-outcome-optimizing, not vanity.

The natural next move after four data points is subsystem enumeration: audit the next likely subsystem with the same sink shape. That's where most of the public follow-up energy went after Dirty Frag, and it's where I expected to spend most of mine.

I tried a different angle. I treated the four confirmed instances as a class and extracted structural lessons that don't depend on which subsystem the bug lives in. The most actionable one was this: **bugs in this class tend to live at the boundary between layers that each look correct in isolation.** That's a different audit target than "any subsystem with a sink shape". It points specifically at the kernel's framework code that lets one layer hand state to another.

The dma-buf framework is the kernel's standard mechanism for sharing memory buffers between drivers. It carries DMA-direction information but doesn't carry exporter-side access-protection state. That gap made it the natural target for the variant analysis: any exporter that pins pages with restricted permissions (say, no write permission), then wraps those pages in a dma-buf, has dropped the protection state at the framework boundary. Any importer that re-attaches with bidirectional DMA recovers a writable mapping of pages the original exporter never agreed to allow writes through.

The IVPU driver was one of the candidates I identified. It was relatively new (the userptr feature landed in v6.19, October 2025) and supported a documented read-only flag. The hypothesis was that cross-file PRIME re-import would lose the read-only attribute and that the resulting bidirectional attach would let the NPU write to pages pinned without write permission.

I ordered hardware, validated the hypothesis on a Beelink mini-PC running Fedora 43, confirmed the cross-process page-cache mutation shape, and developed a patch that applied cleanly against `v7.1-rc2`.

The methodology worked. The methodology produced a true positive, in a previously un-audited driver, on a class of bug I had argued should generalize beyond the four known instances. If you stop the story there, this is a successful research project.

## The miss: disclosure-prep sourcing failure

Here is where the story stops being successful.

The kernel disclosure path I had planned, and the one I executed, was: report to `security@kernel.org` with an inline patch and a minimal reproducer, observe the standard 7-business-day acknowledgment window, coordinate with distro security teams under a 14-day embargo, then publish coordinated with the upstream patch landing and CVE assignment.

I sent the Stage 1 report on 2026-05-13.

On 2026-05-14, while preparing the linux-distros notice (a separate mail that is supposed to be sent only after the kernel security team has acknowledged), I ran a sourcing check that I should have run two weeks earlier. The sourcing check was a one-line operation: pull the upstream IVPU tree and grep the recent commit log for changes touching the same code path I was reporting on.

The commit that fixes the bug is `7dd57d7a6350`. Author date: 2026-04-30. Committer: Karol Wachowski, IVPU maintainer. `Reported-by: Yametsu <yam3tsu@gmail.com>`. By the time I'd run my hardware validation on 2026-05-12 and 2026-05-13, the fix had been upstream for nearly two weeks.

I had not checked.

The lesson is small and unsatisfying and exactly the kind of lesson that's worth writing down. **Before drafting a disclosure, verify that the bug is still unfixed upstream.** Not "I read the recent release notes." Not "I'd have noticed if there was a fix." A specific grep against the maintainer's tree, on the day of the report, against the canonical source.

The corollary is that the verification has to be on the actual canonical source, not on the version you happened to be testing against. I was running my reproducer on Fedora 43's kernel `7.0.4-100.fc43.x86_64`, which is a stable backport branch. The upstream patch had merged to mainline but not yet propagated to v7.0.x stable. My local kernel reflected the world as it was three weeks earlier, and so did my mental model of upstream state.

I have made this rule a durable part of my process. It's now part of the disclosure-prep checklist I use, and the discipline applies to anything I send externally: verify recipient lists, verify version claims, verify embargo policies, verify `MAINTAINERS` entries, against canonical source, on the day of send, not from memory.

There is nothing especially clever about this lesson. That's the point. The clever-sounding lessons in security research tend to be about novel attack primitives. The lessons that actually keep process-quality high are usually small and embarrassing.

## What the upstream fix does differently

The upstream fix and the patch I drafted close the gate at different layers of the buffer's lifecycle.

My patch rejected the attach: when an importer called `dma_buf_map_attachment()` on a userptr dma-buf whose source was read-only, the map callback returned `-EACCES` if the requested direction was anything other than `DMA_TO_DEVICE`. The advantage of this layer is that the rejection is exporter-driven and would generalize to cross-driver imports. Any importer attaching to the read-only buffer would hit the same gate, regardless of which driver the importer belongs to.

The upstream fix closes the gate earlier, at the export side: the IVPU driver now refuses to re-export an imported GEM object at the `prime_handle_to_fd` callback, returning `-EOPNOTSUPP` for objects that originated as imports. The advantage of this layer is structural simplicity. The buffer never reaches the second DRM file in the first place. There's no second attachment to reject because there's no second handle to attach with.

The upstream fix is better. The vulnerability requires the re-import to happen, and refusing to re-export at all is a cleaner chokepoint than refusing the resulting attach. My patch would have worked, but it solved a harder version of the problem than the upstream solution needed to solve.

There's a small generalization lesson here too: when you find a fix that works, ask whether there's a fix one layer further out that's structurally simpler. I didn't, and the maintainer did, and their fix is the one that landed.

## The class question that survives

The IVPU instance is closed. The class question is not.

The dma-buf framework is used by dozens of in-tree drivers. The protection-propagation question is this: should the framework carry exporter-side access restrictions across attach boundaries as part of the interface contract, rather than relying on each driver to remember to encode and decode them? That question is independent of whether any specific driver happens to handle it correctly today.

The IVPU fix solves the IVPU instance. It does not solve the case of the next driver that grows a userptr-with-restriction feature and forgets to add an equivalent guard. The next instance of this class lives wherever a future maintainer doesn't know to look for the gap.

The same methodology that produced the IVPU candidate, applied to the rest of the in-tree dma-buf exporter set, surfaced a second in-tree instance: `drivers/gpu/drm/etnaviv` loses read-only userptr protection state when mapping the buffer's scatter-gather table into the GPU IOMMU. The mechanism is structurally identical to the IVPU shape, a flag set at the GEM-object layer that the device-mapping layer doesn't consult, but with a simpler trigger path: no PRIME re-import is needed, the bug is reachable directly from a single render-group user's create-userptr + submit lifecycle. I sent the Etnaviv finding as an `[RFC PATCH]` to dri-devel with explicit AI-assistance disclosure and an offer to hardware-validate if the maintainers prefer that ordering before merge. See "Open status at publish time" below for the link.

Two confirmed in-tree class members (one already fixed, one with a patch in review) is harder to wave away as a per-driver oversight than one. I am separately preparing an RFC for the dri-devel mailing list proposing that the dma-buf interface acquire an explicit protection-state field at attach time, so that exporters can communicate access restrictions to importers as part of the framework contract rather than as a per-driver convention. That RFC stands on its own merits regardless of how either driver instance was resolved. If you have thoughts on the interface evolution, the dri-devel thread is the right place to discuss them.

## Three for three: what independent rediscovery actually measures

After the IVPU miss I assumed I had been unlucky. After Etnaviv and freenginx I stopped believing that.

The three cycles are not variations on one event. Different subsystems, different codebases, one of them not even the kernel. Different discoverers: an external researcher credited as `Reported-by` on an Intel commit, a university researcher posting to dri-devel, and the maintainer of a web server fixing his own regression. The only thing they share is that a mechanism-first predicate sweep pointed me at a piece of code, and somebody else's entirely unrelated process pointed *them* at the same code inside the same few weeks.

That convergence is information, and it cuts in a direction most security writeups never have to think about, because they only publish the cycles where convergence didn't happen.

**What it rules out.** The failure mode I was most worried about going in was a predicate sweep that produces a large pile of plausible-looking sites, none of which are really bugs. A methodology that manufactures its own findings and then persuades its author. Independent rediscovery is close to a decisive test against that. When a maintainer or an unrelated researcher independently classifies the same line of code as defective and writes a patch for it, the question of whether the predicate found a *real* thing is settled by someone with no stake in my method.

Three separate arbiters agreed. That is a better validation of the predicates than a CVE would be, because a CVE mostly measures whether a fix reached stable, not whether the analysis was sound.

**What it does not rule out.** It says nothing about whether my predicates find things *nobody else would*. On the evidence so far, the class of bugs I'm finding is exactly the class that careful people find by other means, on roughly the same schedule. If the goal is novel findings, the branded-CVE outcome the methodology was originally aimed at, then three-for-three rediscovery is a straightforward negative result, and I'd rather say so plainly than bury it.

**The timing is the actual lesson.** Six days, in the Etnaviv case. Two weeks on IVPU. Mid-cycle on freenginx. I am not being beaten by months. I am being beaten by the length of my own disclosure-prep process. That reframes the problem from "find better bugs" to "compress the interval between candidate identification and public contact," which is a far more tractable engineering problem, and one I have direct control over. The IVPU cycle spent two weeks on hardware validation for a bug that was already fixed. The check that would have caught it takes about thirty seconds.

**And it reframes what a cycle is for.** If independent rediscovery is the common case rather than the exception, then the value of a cycle cannot be "did I get a CVE," because most cycles won't. The things that survive a rediscovered cycle are the predicates, the discipline rules, and the writeup. Those compound. A CVE doesn't.

I want to be careful not to over-claim in the other direction. Three is three. It is enough to stop calling it luck and not nearly enough to call it a rate. Ask me again after ten.

## What I would do differently

Five things, in roughly increasing order of how durable I think the lesson is.

**Verify upstream HEAD on the day you draft a disclosure, and again on the day you send it.** This is the small, embarrassing, durable process-quality lesson. I now check both the maintainer's tree and mainline, with a specific grep against the file paths I'm reporting on, on every disclosure I send.

**Run hardware validation only after the upstream-HEAD check is clean.** I had the validation hardware in hand on 2026-05-12. I should have re-run the upstream check that morning, before booting the test kernel. The cost of the check is two minutes; the cost of not running it was two days of validation work on a problem that was already solved.

**Score the target before committing hardware spend.** The class methodology I was running is genuinely useful, but it scored the IVPU finding at mid-tier on a headline-impact filter the moment the target was identified: hardware-gated install base, lower press familiarity than fs/, modern subsystem with short audit history. That score was visible before any of the engineering work started. For the Etnaviv follow-up I tried the lighter touch the IVPU experience taught me: source-only `[RFC PATCH]`, explicit offer to hardware-validate only if the maintainer requested it before merge. Zero hardware spend, full methodology contribution. The cost of the hardware-first IVPU validation was real (the dev board plus the time to set it up); the Etnaviv send proved that for a clear source-level mechanism on an unfixed upstream tree, source-only disclosure is enough register for the kernel community.

**Don't conflate "real bug" with "publishable finding."** The IVPU bug is a real bug. The primitive is real. The patch I wrote was real. None of that adds up to a publishable finding in the press-tier sense, and the methodology should have surfaced that gap earlier. A real-bug-but-not-press-tier outcome is not a failure; it just needs to be recognized for what it is and handled proportionally, with an internal note and a methodology refinement, no embargo machinery.

**Publish the misses.** The asymmetry between the security research that gets published (wins) and the security research that happens (a mix of wins and misses) makes the field look more linear from the outside than it is. I am publishing this post in part because the methodology details and the disclosure-prep lesson are useful regardless of which way the IVPU instance fell, and in part because reading other researchers' misses has been more useful to my own practice than reading their wins.

The evidence that these lessons are durable, not aspirational: I applied all of them to the Etnaviv follow-up over the same week. Upstream HEAD re-verified against `linux-next` on day-of-send. `Assisted-by:` trailers for both Codex and Claude included in the first contact, not retroactively. Sent via `git send-email` from a Linux host to bypass any GUI-mailer whitespace mangling. The original IVPU mail had its inline patch mangled in transit and Willy Tarreau flagged it; the Etnaviv mail's tab indentation survived end-to-end through Gmail's relay (verified by saving the received `.eml` and running `git am` against it on a fresh checkout). Public-list-first to dri-devel rather than `security@kernel.org`, per the now-codified `Documentation/process/security-bugs.rst` exception for findings "trivial to discover... or use of AI-based tools" (commit `a03ef333fbd6`, Willy Tarreau, landed 2026-05-12). `checkpatch.pl` clean before send. Dry-run send-email to confirm the recipient set matched `get_maintainer.pl` output exactly. None of those checks caught a fatal mistake the first time around because they were never run; all of them caught issues on the second pass.

## Credit

The IVPU bug was first reported by **Yametsu** (`yam3tsu@gmail.com`), credited in the upstream commit's `Reported-by` trailer.

The fix was authored, committed, and signed off by **Karol Wachowski** of Intel's NPU driver team. **Andrzej Kacprowski** reviewed.

My work on this finding was independent and rediscovery-only. I take no credit for the bug discovery, the fix, or the upstream landing.

### LLM assistance

This work was LLM-assisted at two stages. The static-analysis variant hunt that surfaced the IVPU candidate site (the predicate authoring, the dma-buf class sweep, the audit reports) was driven through **Codex** (gpt-5.5). The disclosure-prep, the patch refinement, the writeups and cover letters (including this post) were driven through **Claude** (`claude-opus-4-6` for the May 2026 disclosure cycles, `claude-opus-5` for the July 2026 revision of this post). Hardware validation on the Beelink, the operator's gate decisions at every external send, and the eventual upstream-HEAD check that caught the duplicate state were human-driven by me.

I'm flagging this explicitly because `Documentation/process/security-bugs.rst` treats LLM-assisted findings as falling under the "automated tool" exception. The exception clause now explicitly names "AI-based tools" alongside automated scanners, added by Willy Tarreau in commit `a03ef333fbd6` (landed 2026-05-12), and a companion commit `4bf85afb9f3e` the same day adds an explicit "Responsible use of AI to find bugs" section. I missed that restatement in the original Stage 1 send to `security@kernel.org`; Willy Tarreau called it out on the IVPU thread before the docs commit landed, and the lesson is the same as the upstream-HEAD miss: disclosure-prep checks belong inside the first send, not in a follow-up.

## Open status at publish time

**CVE:** As of publication, no CVE has been assigned to `7dd57d7a6350`. The kernel CNA generally assigns post-stable-backport, so an assignment is likely once the patch propagates.

**Stable backport:** The patch is in mainline (v7.0 tag) and tagged `Cc: stable v6.19+`. As of publication, it has not landed in the v7.0.x point releases (checked: through v7.0.5). v6.6 and v6.12 LTS trees are **not** affected upstream, since they predate the introducing commit. Distribution kernels may differ; each distribution's security team is the authoritative source for their tree's status.

**Affected install base:** Linux systems running upstream kernel v6.19 or later with an Intel Core Ultra NPU. v6.18 and earlier are not affected. The hardware gate is the dominant constraint on the real-world install base.

**Etnaviv `[RFC PATCH]`:** sent 2026-05-14 to dri-devel + linux-kernel + the etnaviv maintainers (Lucas Stach, Russell King, Christian Gmeiner). Source-only, single-file change to `drivers/gpu/drm/etnaviv/etnaviv_mmu.c`. Public archive: https://lore.kernel.org/all/20260514131401.2660079-1-clusk@northecho.dev/

Ziyi Guo (Northwestern) independently identified the same bug and posted a parallel patch on 2026-05-08, six days before our RFC. Lucas Stach reviewed Ziyi's patch and requested a different approach (using `ETNA_BO_FORCE_MMU` for MMUv1 hardware instead of rejecting the mapping). Neither patch merged. A v2 implementing Lucas's preferred approach went to dri-devel on 2026-07-24: https://lore.kernel.org/all/20260724222124.537101-1-clusk@northecho.dev/

**kTLS async split-record loss (separate class sub-track):** While preparing this post, a parallel application of the methodology to a different bug class (cross-async-boundary state loss, the Copy Fail / Dirty Pipe / Dirty Frag family) produced a confirmed finding in `net/tls/tls_sw.c`. The `tls_push_record()` function splits an open kTLS TX record into the part being encrypted and a remainder; the synchronous path reattaches the remainder, the async path returns `-EINPROGRESS` before reattachment and orphans the remainder. Result: silent data truncation visible to the TLS peer plus a per-trigger `tls_rec` memory leak. `Fixes:` tag points at `d3b18ad31f93` (2018, John Fastabend); `Cc: stable@vger.kernel.org # 4.20+`.

The series went to netdev on 2026-05-15 as `[PATCH net 0/2]`, with a deterministic reproducer (sync provider 17312 bytes peer-read versus async `pcrypt(gcm(aes))` 12916 bytes, an exact 4396-byte truncation) and a TAP-style selftest. An AI review bot (Sashiko, sashiko.dev) found a real UAF in our v1 fix at the 24-hour mark. John Fastabend, the original author of the `Fixes:` tag commit, endorsed the Sashiko finding on the v1 thread.

The patch went through four versions. v2 (drain-on-error fix) drew Jakub Kicinski's response on 2026-05-25: he asked for a surface-narrowing approach that avoided the async+sockmap composition entirely. v3 (setup-time AEAD mask) was reviewed by Jiayuan Chen (who found a separate copy-path aliasing issue) and Kicinski (who proposed removing the integration from net-next altogether). John Fastabend confirmed there were no deployed users of kTLS+sockmap. Sabrina Dubroca concurred with removal.

On 2026-06-14, Kicinski posted a 5-patch series to netdev: `tls: reject the combination of TLS and sockmap`. The series removed 1,147 lines and was reviewed by Jakub Sitnicki, Sabrina Dubroca, and Paolo Abeni. It merged to `netdev/net-next`. The cover letter described the integration as having "no known TLS+ sockmap users" and "some known hard to solve bugs."

The net-next removal closes the bug for future kernel versions. Stable trees (4.20 through current) remain unpatched. The removal series carries no `Cc: stable` tag and no `Fixes:` line. No CVE has been assigned. The practical exposure window is gated by a triple-configuration requirement (kTLS TX + BPF sockmap + async pcrypt loaded) that keeps the real-world attack surface narrower than the version range implies.

Cover Message-ID (v1): `<20260515151556.189841-1-clusk@northecho.dev>`. Public archive: https://lore.kernel.org/all/20260515151556.189841-1-clusk@northecho.dev/

The kTLS bug shape is a separate sub-class from the IVPU dma-buf protection-propagation shape documented above, but both are products of the same broader mechanism-first methodology described in [the companion methodology post][methodology]. Three confirmed in-tree class instances across two sub-tracks within two weeks is what tells me the methodology generalizes, not just that the IVPU rediscovery happened to coincide with the Etnaviv variant.

Distro exposure for kTLS is broad in the version-affected sense (every modern enterprise distro kernel from the 4.18-based enterprise family onward carries the bug-introducing commits, verified against published enterprise-kernel source) and narrower in the real-world sense (triple-config gate: kTLS TX + BPF sockmap + async `pcrypt` loaded). Most realistically affected: Cilium-style service meshes with kTLS-terminated workloads on hosts with `pcrypt` loaded for crypto offload throughput.

**Class RFC:** I will post the dma-buf attach-protection RFC to dri-devel separately, once the Etnaviv patch is in a maintainer tree (or refuted on technical grounds). The RFC will cite both in-tree class members (IVPU fixed, Etnaviv pending) rather than just the IVPU instance. Two confirmed members is a stronger case for a framework-level invariant than one. If the RFC lands before this post publishes, the link will be inserted here.

## Methodology artifacts

The variant-analysis dossier, the static analyzer, the hypothesis-walk reports, and the audit trail for the IVPU instance are kept local to my research environment and are not published. The fixtures and predicates are not generally useful outside the specific class they were built for.

If you are working on similar variant-analysis methodology and would find a methodology-only walkthrough useful (no PoC details, no candidate-driver names), email me at `clusk@northecho.dev` and I'll share what I can.

---

[dp]: https://dirtypipe.cm4all.com/
[cf]: https://copy.fail/
[df]: https://github.com/V4bel/dirtyfrag
[upstream-fix]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7dd57d7a6350
[methodology]: /posts/mechanism-first-variant-analysis/

---
