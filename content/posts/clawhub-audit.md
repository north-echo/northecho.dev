---
title: "I Scanned 30,000 AI Agent Skills for Malware. Here's What I Found."
date: 2026-03-20
summary: "A static analysis audit of the entire ClawhHub registry using WAINGRO found 43 confirmed malicious skills — including a coordinated C2 campaign that VirusTotal rates as benign."
tags: ["waingro", "supply-chain", "case-studies"]
---

*A static analysis audit of the ClawhHub registry using WAINGRO*

---

In February, Bitdefender published a technical advisory documenting widespread exploitation of the OpenClaw agent skill ecosystem. They found roughly 900 malicious skills on ClawhHub — something like 17–20% of all published skills at the time — including a coordinated campaign they called "ClawHavoc" that delivered Atomic Stealer via 300+ skills.

I read that report and had a question: what does the ecosystem look like *now*, a month later? ClawhHub has grown fast. The registry has cleared 30,000 published skills. Are the problems Bitdefender found getting better or worse? And could a purpose-built static analysis tool catch things that existing defenses miss?

So I built one, pointed it at the entire registry, and spent a week triaging the results.

## The tool

WAINGRO is a static analysis scanner designed specifically for OpenClaw Agent Skills. The name is a *Heat* reference — Waingro is the insider threat who ruins everything from within. Felt appropriate.

Agent skills aren't executables. They're structured markdown files with YAML frontmatter, prose instructions, code blocks, and sometimes bundled scripts. The malicious intent in a bad skill doesn't live in a binary — it lives in natural language that tells an AI agent to do something harmful. That's a fundamentally different threat model than what traditional security tools are built to handle.

WAINGRO parses the skill format natively — frontmatter, markdown body, fenced code blocks, bundled `.sh`/`.py`/`.js` files — and runs 28 detection rules across eight threat categories: execution, exfiltration, injection, network, obfuscation, persistence, social engineering, and typosquatting.

It's open source: [github.com/north-echo/waingro](https://github.com/north-echo/waingro)

## The scan

I cloned the `openclaw/skills` GitHub archive — the official mirror of every skill published to ClawhHub — and ran WAINGRO against all 30,037 skills. The scan finished in about six minutes on a ThinkCentre M720q with four parallel workers.

The raw numbers: 263,693 total findings across all rules. That sounds dramatic, but most of it was noise. One rule alone (OBFUSC-001, which flags base64-encoded strings) produced 175,000 findings — two-thirds of the total. Tuning that rule is on my list. Strip out the noise and you're left with roughly 88,000 signal findings, of which about 5,000 were rated CRITICAL.

## Triage

Raw findings aren't conclusions. I built an interactive triage CLI that walks through flagged skills one at a time, shows the findings alongside the full skill content, and prompts for a verdict: true positive, false positive, suspicious, or skip.

I split the work into priority tiers. Tier 1 covered the highest-confidence rules — C2 infrastructure references, reverse shell patterns, jailbreak attempts — and I reviewed those exhaustively. Tier 2 covered curl-pipe-shell patterns, filtered by domain reputation. Tier 3 sampled eval/exec and credential access patterns. 589 skills triaged in total across about five hours of analyst time.

The result: **43 confirmed malicious skills** across four attack categories.

That's 0.14% of the registry. Whether you read that as "reassuringly low" or "alarmingly nonzero" depends on your perspective. For context, npm averages roughly 0.01–0.02% confirmed malicious packages in academic studies of its registry. PyPI is in a similar range. ClawhHub at 0.14% is an order of magnitude higher — and this was a first pass with conservative triage. The 401 skills I marked "suspicious" but didn't fully confirm would push the number higher with more review time.

## The headline finding

Twelve of the 43 malicious skills form a coordinated campaign. All twelve reference the same command-and-control IP address — infrastructure previously documented by Bitdefender as part of the ClawHavoc operation. Every one of them disguises itself as a security scanning tool. Names designed to sound trustworthy. Descriptions that promise to audit or protect your skills. What they actually do is instruct the AI agent to beacon to the C2 server and exfiltrate workspace data.

The twelve skills were distributed across ten separate author accounts — a deliberate spread to avoid single-account detection. Two accounts published multiple C2 skills each.

I pulled intelligence on the C2 IP from four sources: WHOIS, VirusTotal, AbuseIPDB, and Shodan. The picture is consistent. The IP sits on a recently allocated /24 netblock registered to an offshore entity created in January 2026 — about a month after the netblock was allocated. It's geolocated to Amsterdam, runs a default Apache install with 54 unpatched CVEs, and has 97 abuse reports on file. Multiple reporters document it as an active distribution point for macOS Nova Stealer. The ClawhHub skills are one vector in what appears to be a multi-platform campaign.

## What VirusTotal doesn't catch

This is the part that surprised me most.

I checked the live C2 skills against VirusTotal. Every single one came back **Benign**. Zero detections out of eight live skills checked. The C2 IP itself is flagged by 23 of 94 VT vendors when you query it directly — but VT doesn't resolve or check IP addresses embedded in the *text content* of files it scans. The IP appears as a string inside a markdown file. It's invisible to signature-based detection.

ClawhHub's own moderation system did better — it flagged five of the eight live C2 skills as "Suspicious." But three were rated Benign with High Confidence, including the most-installed C2 skill in the set with 39 installs.

The detection comparison:

| Method | C2 skills detected | Rate |
|---|---|---|
| WAINGRO | 12/12 | 100% |
| ClawhHub moderation | 9/12 | 75% |
| VirusTotal | 0/12 | 0% |

This isn't a knock on VirusTotal — it's doing exactly what it's designed to do, which is scan for executable malware signatures. The problem is that agent skills represent a new threat surface where the malicious payload is *instructions*, not binaries. Format-aware static analysis fills that gap.

## The other findings

Beyond the C2 campaign, the audit turned up three more categories of confirmed malicious skills. I'm keeping specifics redacted until ClawhHub has had time to act on the disclosure, but in aggregate:

**Reverse shell payloads.** Nine skills containing direct `bash -i >& /dev/tcp/` patterns. Interestingly, an additional fourteen skills with reverse shell patterns turned out to be false positives — legitimate security tools that embed detection signatures. The TP/FP ratio on this rule underscores why automated scanning alone isn't enough; triage matters.

**Jailbreak and safety override attempts.** Nine skills embedding DAN-style prompts or instruction override patterns, attempting to redefine the AI agent's role and bypass safety constraints.

**Malicious install commands.** Thirteen skills piping content from known-malicious or placeholder-malicious domains to shell execution.

## What I learned about building a scanner

A few takeaways from the process that might be useful if you're thinking about supply chain security for agent ecosystems:

**Noisy rules will bury your signal.** OBFUSC-001 (base64 string detection) produced 66% of all findings. Legitimate skills use base64 constantly — for images, encoded configs, API examples. Without aggressive tuning or a minimum-entropy threshold, a rule like this generates so much noise that analysts can't see the real threats. My next WAINGRO release will address this.

**Format awareness is the differentiator.** Generic text scanning would catch some of these patterns, but parsing the skill format — understanding that something in the YAML frontmatter means something different than the same string in a code block example — dramatically reduces false positives and lets you write rules that match behavioral intent rather than string coincidence.

**Author clustering reveals coordination.** Grouping findings by author account exposed the C2 campaign's distribution strategy. A single-skill view would see twelve independent problems; the author view reveals a coordinated operation.

**Triage tooling is as important as detection.** I spent as much time building the interactive triage CLI as I did building the scanner. Being able to see findings in context, alongside the full skill content and bundled files, is what makes the difference between "this looks suspicious" and "this is confirmed malicious."

## Disclosure

I've submitted a detailed disclosure to ClawhHub maintainers with the full list of confirmed malicious skills, organized by threat category and severity. The disclosure includes specific skill names, author accounts, and recommended actions. I'm holding back per-skill details from this post until they've had time to review and act.

The WAINGRO tool and the aggregate audit data (no per-skill details) are public now:

- **Tool:** [github.com/north-echo/waingro](https://github.com/north-echo/waingro)
- **Full audit report:** Published in the repo under `research/clawhub-audit/` once disclosure clears

If you're using ClawhHub skills, you can run WAINGRO against your installed skills today:

```bash
pip install waingro
waingro audit ~/skills/
```

## What's next

The 0.14% confirmed-malicious rate is a floor, not a ceiling. 401 skills are sitting in the "suspicious" bucket waiting for deeper review. OBFUSC-001 needs tuning so the signal-to-noise ratio is workable at scale. And this was purely static analysis — semantic analysis of what a skill *instructs an agent to do* is a much harder problem that I haven't touched yet.

Agent skill registries are the new package managers, and they need the same security infrastructure that npm and PyPI have spent years building. Format-aware static analysis is one piece of that. I hope WAINGRO is useful as a starting point.

---

*Christopher Lusk is a Principal Product Security Engineer and the author of WAINGRO. He writes at [northecho.dev](https://northecho.dev) and publishes tools at [github.com/north-echo](https://github.com/north-echo).*
