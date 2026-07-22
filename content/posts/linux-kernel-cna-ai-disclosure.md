---
title: "The Linux kernel wanted to be a CNA. They got their wish. Then AI came along."
date: 2026-05-16
summary: "The 2024 Linux kernel CNA created downstream consequences nobody fully anticipated — NVD enrichment strain, distribution-CNA friction, and an AI-assisted report flood. On May 15, 2026, the kernel community shipped its first comprehensive synthesized response. A reading of the new docs from the perspective of an AI-assisted researcher who shipped a patch into the same window."
tags: ["linux-kernel", "cna", "cve", "ai-agents", "disclosure", "policy", "security-research"]
---

*Christopher Lusk — North Echo Security Research*

## The wish granted

In 2024, the Linux kernel community lobbied MITRE to become its own CVE Numbering Authority. The argument from Greg Kroah-Hartman and others was straightforward: kernel-bug CVEs were being assigned by external CNAs, often slowly, often with classification errors, often without consultation from the maintainers who actually understood the bugs. The community wanted control over its own security bug process — authoritative assignment by the people closest to the code.

The wish was granted. The Linux kernel CNA went live, and Greg KH deployed bippy: an automated tool that scans `Cc: stable@`-tagged fixes and auto-generates CVE entries. The intent was reasonable — formalize the assignment of CVEs to security-relevant fixes that were going to stable kernels anyway. The result was an order-of-magnitude increase in kernel CVE volume. From roughly a hundred a year through the existing process, to several thousand a year under the new CNA. Within months, the kernel CNA was the most prolific CVE issuer in the entire CVE ecosystem.

The first-order surprise wasn't that the kernel community now had authoritative CVE assignment. It was that authoritative assignment at machine speed broke things downstream that no one had budgeted for.

## What broke

The CVE ecosystem doesn't end at assignment. NIST's National Vulnerability Database (NVD) has historically done the enrichment work that makes CVEs useful for the people who consume them: adding CPE strings that identify affected products and versions, computing CVSS scores, categorizing weaknesses, adding cross-references. Without that enrichment, a CVE is just a record; with it, a CVE is consumable input to vulnerability management tooling, patch prioritization, SBOM analysis, compliance reporting.

NVD's enrichment is a human-labor process. It scaled adequately when CVE volume grew gradually. When the kernel CNA started emitting thousands of CVEs per quarter, NVD's enrichment backlog exploded. By early 2024, the backlog was public news; through 2024, NIST publicly acknowledged it couldn't keep up.

I noticed this from a particular professional vantage: federal compliance work. FedRAMP authorization packages depend on enriched NVD data. Vendor vulnerability management workflows depend on it. Automated patch prioritization tools depend on it. When the enrichment pipe stalls, all of those downstream processes break or have to invent workarounds. The cost was real and distributed across every organization that had built tooling on NVD's promise of timely-enriched CVE records.

The kernel community didn't cause the NVD backlog single-handedly — there were other contributing CNAs and broader volume-growth factors. But the kernel CNA was a significant contributor to the volume shock that the existing system couldn't absorb.

## CNA-vs-CNA friction

The second-order effect was structural friction between the new kernel CNA and the existing distribution CNAs — Red Hat, Canonical, SUSE, Debian's security team, and others. Distribution CNAs assign CVEs to bugs that affect the kernels they ship; the kernel CNA assigns CVEs to bugs that affect upstream mainline. Most of the time these overlap. Sometimes they don't.

Distribution CNAs have a customer-facing job: ship product, tell customers what's affected in versions the customer is actually running. The kernel CNA has an upstream-correctness job: identify security-relevant fixes at the source. The two CNAs can reasonably disagree about whether a given bug is security-relevant, whether a particular downstream backport carries the same impact, or whether something is a CVE-worthy issue at all.

Through 2024-2025, public commentary from kernel maintainers about how distribution CNAs assign kernel CVEs got pointed at times. The kernel CNA's documentation discusses CVE revocation explicitly — a tool partly aimed at the same problem. The friction isn't personal; it's structural, between two reasonable positions that can't always be reconciled by a single common process.

The downstream impact, for those of us doing the actual vulnerability-management work, was advisory divergence: the upstream CVE record and the distribution advisory for the same underlying bug might say different things about classification, affected versions, or severity. Compliance reporting then has to reconcile both. More workaround tooling. More divergence to explain to auditors.

## Then AI showed up

In late 2024 and through 2025, AI-assisted security research became mainstream. The shape of "AI slop" reports started emerging across multiple open-source projects' security mailing lists. The basic pattern: an AI tool flags something that looks plausible at first glance, the human submitter forwards it without verification, the maintainer has to spend real time disproving a non-finding. At scale, this is corrosive to maintainer attention.

Daniel Stenberg shut down curl's HackerOne bug bounty in early 2026 in explicit response to AI-slop volume. *"In six years of monitoring AI-assisted submissions, not one has discovered a genuine vulnerability,"* he wrote at the time. Django's security team made similar pushback. The oss-security mailing list — the open-source security community's coordination forum — saw the same shape arrive.

For the Linux kernel security team, AI tools landed on top of an input pipe that was already stressed from two years of CNA-volume work. The team's public commentary through early 2026 distinguished sharply between "AI slop" and the smaller number of legitimately-discovered findings that happened to be AI-assisted. The distinction was operational: some reports were worth triaging, most were not, and the cost of separating the two had become a meaningful fraction of the team's bandwidth.

## May 15, 2026: What Linus merged

Yesterday — 2026-05-15 — Linus Torvalds merged `docs-7.1-fixes`, a documentation tag from Jonathan Corbet that brought in Willy Tarreau's set of security-bug documentation updates. Commit hash for the merge: `36d49bba19f2`. Five sub-commits, three files touched: `Documentation/process/index.rst`, `Documentation/process/security-bugs.rst` (+106 lines), and the new `Documentation/process/threat-model.rst` (+235 lines).

The new docs are substantive. `threat-model.rst` codifies the kernel's security boundaries — user-based isolation, capability-based protection, the assumption that hardware behaves to spec, the explicit list of categories that *don't* constitute a security bug (probabilistic attacks, crafted filesystem images, physical access without specific protections, hardening failures with no exploit path, random information leaks, and others). `security-bugs.rst` gains a "What qualifies as a security bug" section that contextualizes when the security team should be involved versus when normal channels are sufficient, and explicit guidelines on AI-assisted findings.

The AI-assistance section is what the kernel-community readership will focus on. Direct quotes:

> *"If you resorted to AI assistance to identify a bug, you must treat it as public. While you may have valid reasons to believe it is not, the security team's experience shows that bugs discovered this way systematically surface simultaneously across multiple researchers, often on the same day."*

> *"A significant fraction of bug reports submitted to the security team are actually the result of code reviews assisted by AI tools. While this can be an efficient means to find bugs in rarely explored areas, it causes an overload on maintainers, who are sometimes forced to ignore such reports due to their poor quality or accuracy."*

Five concrete guidelines follow: length (concise), formatting (plain text, no Markdown), impact evaluation (verifiable facts grounded in the threat model), reproducer (tested; share only on maintainer request), proposed fix (tested, with `Fixes:` tag). Failure to comply *"exposes your report to the risk of being ignored."*

The exception clause for when public-list disclosure is appropriate was updated to explicitly name AI tools: *"trivial to discover (e.g. result of a widely available automated vulnerability scanning tool that can be repeated by anyone, **or use of AI-based tools**)."* That parenthetical addition is the literal codification of the rule that the kernel security team had been operating under informally for months.

## Reading the tone

The defensiveness in the new docs is unmistakable if you read carefully. *"Resorted to AI assistance."* *"Actually the result of code reviews assisted by AI tools."* *"The validity of the report should be seriously questioned."* *"Failure to consider these points exposes your report to the risk of being ignored."*

This is not the language of welcome. It's the language of an overloaded team putting up guardrails specifically because a particular failure mode has cost them too much already. The tone is calibrated against two years of input-pipe stress — CNA volume first, then AI volume on top — and against the specific shape of report that has consumed the team's bandwidth unproductively.

Reading the new docs as anti-AI misses the larger pattern. The kernel community has been defending against scale-without-quality for two years. AI is the latest amplifier of that problem, not the originating cause. The new rules are calibrated against the median AI-assisted report, not against all AI-assisted research.

## Where this leaves AI-assisted security research

The new rules are achievable. Concise plain-text reports grounded in verifiable facts, with tested reproducers and tested fixes carrying proper trailers, sent to the appropriate public list rather than the security team — these are disciplines that careful researchers were already practicing. The rules formalize a baseline, not a ceiling.

What the rules do change is the default presumption. An AI-assisted report now arrives in a context where the maintainer's default is suspicion — appropriately so, given the flood the community is recovering from. The work of building trust falls on the reporter. The discipline that gets you past the default suspicion is the same discipline the doc spells out: format, verifiable claims, tested reproducer, tested fix. The careful work will survive. The median slop won't.

## A note on lived experience

I shipped a kernel bugfix yesterday morning — a `[PATCH net]` series to the netdev mailing list, about four hours before Linus merged the new docs. The cover letter declared the AI-assistance up front, named the agents and their roles, linked to methodology context, and shipped a TAP-style selftest as the verification artifact. The bug had been reproduced deterministically against a kernel patched with all adjacent fixes from the same area; the proposed patch had been tested against the same reproducer; the `Fixes:` tag had been bisected against the actual introducing commit; the recipient list had been verified against the day-of-send maintainer list.

The cycle to get to that send wasn't clean from the start. An earlier disclosure I'd worked through had six distinct process failures — duplicate-finding state I hadn't checked, stale embargo policy I cited from memory, AI-assistance disclosure omitted from the first send. The discipline rules I now follow came from cataloging those failures and building rules against them. The new docs from yesterday name many of those same rules, codified by the kernel community for the same underlying reasons.

The irony worth naming honestly: the new docs are explicitly skeptical of AI-found bugs ("the validity of the report should be seriously questioned"; "many AI tools are actually better at writing code than evaluating it"). But within ~24 hours of my series landing on the public archive, the first review feedback I received came from an AI review bot running on kernel.org infrastructure. The bot caught a real return-value-masking issue in my fix that I had missed in pre-send adversarial review — a subtle bug where a later iteration's error could mask an earlier iteration's pending-async signal, potentially exposing zero-copy crypto pages to a use-after-free. The bot's finding was correct. The kernel community is comfortable with AI-on-the-review-side and suspicious of AI-on-the-input-side. The docs haven't fully reckoned with that distinction yet. Whatever else you think about the new docs' position, the equilibrium emerging in practice is more nuanced than the docs alone suggest.

I don't write this to claim alignment with the kernel community or to argue that AI-assisted research is being unfairly treated. The new docs describe a reasonable equilibrium for an overloaded community. Careful researchers can meet the bar. Researchers who can't will discover that the doc's closing sentence — *"exposes your report to the risk of being ignored"* — is operational, not rhetorical. And the cases where AI-augmented review catches what AI-augmented research missed are exactly what makes the working equilibrium possible.

## What comes next

The CNA wish from 2024 set the kernel community on a trajectory that nobody fully anticipated. Two years of working through the downstream consequences — NVD strain, distribution friction, AI-input flood — have produced this week's docs. They are the community's first comprehensive synthesized response.

For researchers doing AI-assisted security work, the path forward is clear: ship the kind of work that meets the bar the docs just set. The bar is appropriate. The discipline is achievable. The trust will follow over multiple clean cycles, not in one.

For everyone else who depends on the CVE ecosystem — FedRAMP practitioners, vulnerability management teams, vendor advisory consumers — the equilibrium remains imperfect. The NVD backlog isn't fully cleared. Distribution-CNA / kernel-CNA friction isn't fully reconciled. AI-research flood isn't fully absorbed. But the May 15 docs are the kernel community's clearest signal in two years about how it intends to operate inside that equilibrium.

That's worth knowing.

---

*Christopher Lusk is an independent security researcher under the North Echo Security Research identity. This post reflects professional observations and lived experience; it does not represent the position of any employer or affiliated organization.*
