---
title: "I Scanned 1,000 of GitHub's Most-Starred Repos for the Vulnerability That Took Down Trivy. Here's What I Got Wrong."
date: 2026-03-21
lastmod: 2026-03-22
summary: "Fluxgate found 12–14 confirmed-exploitable pwn request vulnerabilities across the top 1,000 most-starred repositories on GitHub — and I had to retract several disclosures when my scanner's blind spots led me to overstate the severity."
tags: ["fluxgate", "github-actions", "supply-chain", "disclosure"]
---

*Updated March 22, 2026 with corrections*

---

On March 19, 2026, an autonomous AI agent exploited a single misconfigured GitHub Actions workflow in [Trivy](https://github.com/aquasecurity/trivy), the most popular open-source vulnerability scanner, to steal credentials, publish a malicious release, and poison 75 GitHub Actions version tags. The tool millions of developers trust to *find* malware was *delivering* malware.

The vulnerability class? A `pull_request_target` workflow that checks out attacker-controlled code and executes it with write access to the repository. It's been documented for years. It's still everywhere.

I built [Fluxgate](https://github.com/north-echo/fluxgate) to find out how widespread the problem actually is. Then I pointed it at the top 1,000 most-starred repositories on GitHub.

The results were worse than I expected. And then I discovered my scanner had blind spots that led me to overstate the severity of several findings, which meant going back and correcting disclosures I'd already filed. This post covers both the research and the mistakes.

## The Pattern

The dangerous pattern has three ingredients:

1. A workflow triggered by `pull_request_target` (runs with the *base* repository's secrets and permissions, not the fork's)
2. A checkout of the pull request's head code (`ref: ${{ github.event.pull_request.head.sha }}`)
3. Execution of that code: `npm install`, `pip install`, `make`, or anything that runs build scripts from the checked-out tree

Here's what it looks like in the wild:

```yaml
name: PR Build
on:
  pull_request_target:
    types: [opened, synchronize]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      - run: npm install    # <- executes attacker's package.json
      - run: npm test
```

Anyone who opens a pull request against this repository gets arbitrary code execution with the repository's secrets. No review required. No approval needed. The workflow runs automatically on `opened` and `synchronize`.

This is how Trivy was compromised. This is FG-001 in Fluxgate's rule set.

## The Scan

Fluxgate is a Go-based static analyzer purpose-built for GitHub Actions workflows. It fetches workflow files via the GitHub API, parses the YAML, and runs detection rules against the parsed structure.

For this research, I ran Fluxgate in batch mode against the top 1,000 repositories on GitHub by star count, including projects like React, TensorFlow, Kubernetes, and VS Code. Fluxgate stores results in SQLite with WAL mode for resumability, which matters when you're making thousands of API calls and GitHub's rate limiter has opinions.

```bash
fluxgate batch --top 1000 --db findings.db --resume --delay 1s
```

## The Numbers

| Metric | Count |
|--------|-------|
| Repos scanned | 1,000 |
| Repos with workflows | 777 |
| Repos with at least one finding | 713 (91.8%) |
| Total findings | 24,423 |

The breakdown by rule:

| Rule | What It Detects | Findings | Severity |
|------|----------------|----------|----------|
| FG-001 | Pwn Request (`pull_request_target` + fork checkout) | 44 | Critical / High |
| FG-002 | Script Injection (expression interpolation in `run:`) | 44 | High |
| FG-003 | Tag-Based Action Pinning (mutable references) | 23,850 | Medium / Info |
| FG-004 | Overly Broad Permissions | 455 | Medium |
| FG-005 | Secrets Exposed in Logs | 30 | Low |

FG-003 dominates the count because nearly every repository pins actions by tag (`@v4`) instead of by commit SHA. That's a known supply chain risk, and it's how the second Trivy incident on March 19th worked, with the attacker force-pushing 75 trivy-action tags to point at malicious code. But the FG-001 findings are the ones that keep me up at night.

## 44 Pwn Request Findings: Confirmed, Likely, and Pattern-Only

Not all FG-001 findings are created equal. A `pull_request_target` workflow that checks out PR head code but only runs `diff` or `grep` against it is categorically different from one that runs `npm install`.

Fluxgate classifies findings into confidence tiers:

**Confirmed** (post-checkout code execution detected): The workflow runs a build tool like `npm install`, `pip install`, `make`, or `cargo build` on the checked-out code. An attacker can inject arbitrary commands through `package.json`, `setup.py`, `Makefile`, or equivalent.

**Likely** (setup action + run step after checkout): The workflow installs a language runtime (`actions/setup-node`, `actions/setup-python`) and then runs a command that isn't in the known read-only list. Tools like `eslint` or `prettier` load configuration files from the repo, which can execute arbitrary code.

**Pattern-only** (checkout without detected execution): The workflow checks out PR head code but post-checkout steps appear to be read-only: `diff`, `grep`, `test -f`. Still worth flagging for defense in depth, but not immediately exploitable.

The initial distribution from the v0.1.0 scan:

- 20 critical findings (confirmed or likely execution) across 16 unique repositories
- 24 high findings (pattern-only) across 12 unique repositories

**These numbers changed.** Keep reading.

## What I Found in the Top 1,000

I can't name specific repositories until patches are in place or disclosure windows expire. But I can share the aggregate picture.

Among the initial 20 critical findings:

- 4 repositories had `write-all` permissions (no `permissions:` block, which defaults to full write access) combined with confirmed code execution. This is the worst case: an attacker gets write access to the repository, its packages, its releases, and its secrets.
- 8 findings involved `npm install` or `pnpm install` as the execution vector, the most common pattern by far.
- 5 findings involved `pip install`.
- 3 findings involved `make`.
- 4 findings were "likely": a setup action followed by a non-read-only run step.

Star counts for affected repositories ranged from 31,000 to 94,000. These are not obscure projects.

## What I Got Wrong

After filing the initial round of disclosures, I built Fluxgate v0.2.0 with significantly improved detection, including the ability to parse job-level `if:` conditionals, environment approval gates, trigger type filters, and sparse-checkout configurations. When I rescanned the same repositories with the improved scanner, I discovered that several of my filed disclosures had pre-existing defensive controls that v0.1.0 couldn't see.

**Full withdrawals (3):**

- One repository's workflow used `sparse-checkout` to pull only a `docs/` subdirectory. The fork's `package.json` was never present in the working directory, and the build command ran against the base branch's lockfile. It also had a label gate and all actions were SHA-pinned. I withdrew the advisory entirely.
- One repository had a fork guard (`if: github.event.pull_request.head.repo.full_name == github.repository`) that predated my filing. Forks were already excluded. Full withdrawal.
- A third had the same pattern: an existing fork guard that my scanner missed. Withdrawal sent the same day I discovered it.

**Downgrades (3):**

- One project (4 workflows flagged as critical) had label gates on all four workflows. They wouldn't trigger without a maintainer applying a specific label first. Downgraded from critical to high. Still a risk if an attacker gains collaborator access, but not exploitable by an anonymous fork.
- One project had two flagged workflows; the secondary one had a fork guard and actor check that my scanner missed. Partial correction. The primary workflow remains critical.
- One project's secondary workflow had an environment approval gate. Minor update. The primary workflow at CVSS 9.6 was unaffected.

**Corrected during triage (1):**

- A bounty program's triage team pushed back on a finding, citing `persist-credentials: false` as a mitigation. They were partially right. While `persist-credentials: false` doesn't prevent `GITHUB_TOKEN` access via environment variables, the workflow also had multiple legitimate defenses (label gate, environment approval, maintainer permission check, SHA-pinned actions) on its primary job that my original report didn't account for. I sent a corrected comment acknowledging the defenses while noting that a secondary job still executes attacker-controlled code without those guards.

**False positives caught before filing (2):**

- One repository had a runtime fork check in its workflow logic that Fluxgate couldn't detect statically.
- Another had a conditional restricting execution to same-repo branches matching a specific naming pattern. Forks were excluded.

### The corrected numbers

After all corrections, the real picture is closer to **12–14 confirmed-exploitable findings** rather than the original 20. The aggregate scan statistics (24,423 total findings, 713 repos with findings) remain accurate. The corrections only affect FG-001 critical classifications.

That's still bad. Over a dozen of the most-starred repositories on GitHub, with confirmed code execution on fork PRs and access to repository secrets, is a serious finding. But it's not 20, and I owe that correction to anyone reading this.

## Why This Happened

The root cause was a gap in Fluxgate v0.1.0's detection capabilities. The scanner could identify the dangerous structural pattern (`pull_request_target` + fork checkout + build command) but couldn't see the defensive controls that some maintainers had already put in place:

- **Job-level `if:` conditionals** that gate execution on labels, actor permissions, or fork status
- **Environment approval gates** that require manual approval before a workflow job runs
- **Sparse-checkout configurations** that limit which files from the fork are actually checked out
- **Trigger type filters** that restrict which `pull_request_target` event types activate the workflow

I filed disclosures based on the v0.1.0 scan without manually verifying every defensive control in every workflow. When v0.2.0 revealed the blind spots, I sent corrections within hours.

The lesson: **static analysis finds candidates, not verdicts.** Every finding needs manual verification before it becomes a disclosure.

## The Disclosure Campaign

The campaign has grown beyond the original top-1000 scan. As of this update, I've filed disclosures for 18 repositories across the original top-1000 scan, a targeted code-search scan, and a Red Hat org sweep through GitHub Security Advisories, HackerOne, Google VRP, Kubernetes HackerOne, MSRC, and direct email. Of those, 3 have been fully withdrawn and 3 downgraded after the v0.2.0 rescan revealed pre-existing defenses.

Each disclosure follows Fluxgate's [published disclosure protocol](https://github.com/north-echo/fluxgate/blob/main/DISCLOSURE.md): 30-day coordinated window, aggregate stats only until patches ship, no naming of unpatched repos.

Every disclosure platform has a different format. MSRC wants field-by-field structured input. HackerOne wants Summary/Steps/Impact/References with program-specific CVSS versions (some use 3.1, some use 4.0). Google VRP is free-form with dropdowns. GitHub Security Advisories are markdown. If you're doing bulk disclosure, plan for the formatting overhead, and plan for corrections.

## Lessons Learned

**Your scanner is only as good as its parser.** Fluxgate v0.1.0 could detect the dangerous *pattern* but not the *defenses*. Job-level conditionals, environment gates, sparse-checkout, and runtime fork checks are all things a workflow author might use to mitigate the `pull_request_target` risk. If your scanner can't see them, you'll overcount. v0.2.0 now detects most of these, but sparse-checkout detection is still on the backlog.

**Verify before you disclose.** I spent 20 seconds per finding when I should have spent 15 minutes reading the full workflow YAML on GitHub. The structural pattern match is the *start* of the analysis, not the end. Pull up the actual file. Read every `if:` conditional. Check for environment gates. Look at what the checkout step actually checks out. Only then decide if it's a real finding.

**Send corrections immediately.** When I discovered the errors, I sent corrections within hours, including full withdrawals where warranted. Several maintainers and triage teams responded positively. Retracting a finding you filed in good faith is uncomfortable, but it preserves the trust that makes coordinated disclosure work.

**Severity is contextual.** Early in this research, a maintainer challenged a finding where the post-checkout steps were read-only (grep, git diff). They were right. No code execution, no exploitability. That exchange directly motivated the confidence tiering system. Later, the v0.2.0 rescan proved the same point at scale: a workflow can have the dangerous *structure* without being *exploitable*, if the right guards are in place.

**Broken security contacts are common.** One project's `SECURITY.md` listed an email address that hard-bounced. I had to fall back to filing a GitHub Security Advisory and sending a backup email to an address I found in the project's original research paper. Another project had no security policy at all and most of the repository was in Chinese, making disclosure coordination significantly harder. If you maintain an open-source project, verify your `SECURITY.md` contact actually works.

**Default permissions are the silent killer.** When a workflow doesn't include a `permissions:` block, GitHub defaults vary by configuration, and in many cases the default is `write-all`. Several of the most dangerous findings I identified had no explicit permissions block at all. The maintainers likely didn't realize their CI workflows had write access to their entire repository.

## What Maintainers Should Do

If you maintain a project that uses `pull_request_target`:

**Audit your checkout refs.** If any step uses `ref: ${{ github.event.pull_request.head.sha }}` or `ref: ${{ github.event.pull_request.head.ref }}`, you are checking out fork code.

**Check what runs after checkout.** If anything builds, installs, tests, or lints the checked-out code, an attacker controls what executes. This includes `npm install`, `pip install -e .`, `make`, and any tool that reads config from the repo (`eslint`, `prettier`, `jest`, etc.).

**Add an explicit `permissions:` block.** Set the minimum permissions needed. `contents: read` is enough for most CI jobs. Never leave permissions unset on a `pull_request_target` workflow.

**Consider splitting the workflow.** GitHub's [recommended pattern](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/) is to run untrusted code in a `pull_request` workflow (which has no secrets access), then use `workflow_run` to handle privileged operations after.

**Pin actions by commit SHA.** Tags are mutable. `@v4` can be force-pushed to point at anything. Pin to the full 40-character commit SHA.

You can run Fluxgate against your own repository to check:

```bash
# Local scan
fluxgate scan .

# Remote scan
fluxgate remote your-org/your-repo
```

## What's Next

Fluxgate v0.2.0 is built and running. It detects job-level conditionals, environment gates, and includes two new rules: FG-006 for fork PR code execution on `pull_request` triggers, and FG-007 for inconsistent GITHUB_TOKEN blanking. Sparse-checkout detection is on the v0.2.1 backlog.

The remaining confirmed-exploitable disclosures are being filed as I complete manual verification for each one.

The Trivy compromise was a wake-up call, but the underlying problem is structural. `pull_request_target` is a footgun that requires maintainers to understand a subtle trust boundary that GitHub's own documentation has historically under-explained. Even after corrections, the fact that over a dozen of the top 1,000 repositories on GitHub, projects with tens of thousands of stars, have exploitable variants of this vulnerability tells you the problem isn't developer negligence. It's a dangerous default in the platform.

Fluxgate is open source under Apache 2.0. Scan your repos. Fix your workflows. And if you get a security advisory from me, please don't ignore it.

---

*Christopher Lusk is a Principal Product Security Engineer at Red Hat. Fluxgate is an independent open-source project. Find it at [github.com/north-echo/fluxgate](https://github.com/north-echo/fluxgate).*

---

**Changelog:**
- **2026-03-22:** Added "What I Got Wrong" and "Why This Happened" sections. Corrected FG-001 critical count from 20 to ~12–14 after v0.2.0 rescan revealed pre-existing defensive controls (3 withdrawals, 3 downgrades, 1 triage correction). Updated disclosure campaign status, lessons learned, and what's next. Changed title.
- **2026-03-21:** Initial publication.
