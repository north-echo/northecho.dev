---
title: "Standards"
description: "How North Echo works and the standard every finding meets before disclosure."
---

North Echo Security Research is an independent security research practice
focused on logic-level vulnerability research in enterprise, platform, and
operating-system software. This page describes how we work and the standard
every finding meets before we disclose it. We publish it because the
credibility of a security report should rest on method, not on assertion.

## What we do

We study how systems actually behave. How privilege is granted and revoked,
how trust is established across component boundaries, how updates and
remediations are applied. We look for the places where a design's stated
guarantees and its real behavior diverge. Our findings are logic and
composition flaws: confused deputies, incomplete revocation, trust that
survives its own removal, enforcement gaps at the seams between components.
This is not memory-corruption fuzzing or scanner output. It is reading,
tracing, and adversarial reasoning about what a system does versus what it
promises.

## How we work

**Intelligence-first, not checklist.** Before we test anything, we build the
picture: the component's architecture and trust boundaries, its published
security model, and the history of what has actually broken in it and in
adjacent code. We hunt from that understanding. We do not run rubrics or
enumerate and classify. A checklist has the appearance of rigor and none of
the signal.

**Adversarial by default.** We assume every control can be bypassed until we
have genuinely tried and failed to bypass it. "Looks enforced" is a
hypothesis to attack, not a conclusion. The goal is a working exploit chain
on lab infrastructure we own, not a rating on a page.

**Reproducible, or it does not exist.** A finding is not a finding until it
reproduces.

## The bar every finding clears before disclosure

Before we report anything to a vendor, it meets all of the following:

- **Reproduced from a clean state, repeatedly.** Every finding is reproduced
  multiple times, typically three of three, from fresh, clean-state snapshots
  of the affected software, on the exact shipped build. Not a development
  branch, not an approximation.
- **Traced to exact source.** We identify the responsible code down to file
  and line in the shipped build, and we confirm whether the behavior is
  shared with the upstream project or is a downstream-specific divergence.
- **Vetted against documentation, prior art, and design intent.** We check
  the finding against the vendor's own documentation, the public CVE and
  advisory record, and the project's issue trackers, and we run an explicit
  "is this by design?" pass. If a behavior is documented, intended, or
  already known, we say so plainly.
- **Scored conservatively.** We assign CVSS with a defensible vector and lead
  with the conservative framing. We would rather understate severity than
  overstate it. Overstatement burns credibility faster than underclaiming
  ever will.
- **Scoped exactly.** We state affected versions, prerequisites, and
  preconditions precisely, and we distinguish what we demonstrated from what
  we infer. We never claim more than the evidence supports.

If a candidate fails any of these, it is not disclosed as a vulnerability. A
large fraction of what we investigate ends here, explained away as intended
behavior, as prior art, or as an incomplete chain. We treat those honest
negatives as a core product of the work, not a failure of it.

## On tools, including AI

We use the best tools available for discovery and analysis, and that includes
AI systems. We consider "was a tool or an AI involved in finding this?" to be
the wrong question, and our process is built to make it irrelevant.

The reason is the bar above. A finding's validity rests on whether it
reproduces from a clean state on the shipped product, traces to real source,
survives a by-design and prior-art review, and holds up under a defensible
severity analysis. None of those tests care how the lead was surfaced. A
hypothesis, however it was generated, is worth nothing until it is validated
against real code and real, repeatable behavior. Once it has been, the method
of discovery adds nothing to and subtracts nothing from the result. We treat
AI as an instrument under human direction, held to exactly the same
evidentiary standard as any other instrument, and every factual claim in a
report is independently verified against primary sources before it is filed.

We hold this line deliberately, because the field is seeing a rise in
unvalidated, machine-generated reports that waste vendor time. The answer to
that problem is not to reject tools. It is to hold every finding, by whatever
means found, to a standard that unvalidated output cannot meet.

## Disclosure

We practice coordinated, vendor-first disclosure. We report to the vendor's
security channel before anyone else, we respect embargoes and coordinate
publication timing, and we do not disclose publicly before a fix or an agreed
date. We do not publish weaponized proof-of-concept code. Our reports carry
teaching-grade reproductions sufficient to confirm and fix an issue, not to
arm an attacker. Our correspondence is factual, dated, and consistent.

## What we do not do

- We do not submit findings we have not reproduced and independently verified.
- We do not overstate severity or blast radius.
- We do not publish exploit code, and we do not disclose before a fix or an
  agreed window.
- We do not claim novelty we have not confirmed against the public record.

---

*Christopher Lusk, North Echo Security Research*
