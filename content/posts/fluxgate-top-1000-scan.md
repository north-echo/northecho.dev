---
title: "I Scanned 1,000 of GitHub's Most-Starred Repos for the Vulnerability That Took Down Trivy"
date: 2026-03-21
summary: "Fluxgate found 20 critical pwn request vulnerabilities across 16 of the top 1,000 most-starred repositories on GitHub — the same vulnerability class that enabled the Trivy supply chain compromise."
tags: ["fluxgate", "github-actions", "supply-chain", "disclosure"]
---

On March 19, 2026, an autonomous AI agent exploited a single misconfigured GitHub Actions workflow in [Trivy](https://github.com/aquasecurity/trivy) — the most popular open-source vulnerability scanner — to steal credentials, publish a malicious release, and poison 75 GitHub Actions version tags. The tool millions of developers trust to *find* malware was *delivering* malware.

The vulnerability class? A `pull_request_target` workflow that checks out attacker-controlled code and executes it with write access to the repository. It's been documented for years. It's still everywhere.

I built [Fluxgate](https://github.com/north-echo/fluxgate) to find out how widespread the problem actually is. Then I pointed it at the top 1,000 most-starred repositories on GitHub.

The results were worse than I expected.

## The Pattern

The dangerous pattern has three ingredients:

1. A workflow triggered by `pull_request_target` (runs with the *base* repository's secrets and permissions, not the fork's)
2. A checkout of the pull request's head code (`ref: ${{ github.event.pull_request.head.sha }}`)
3. Execution of that code — `npm install`, `pip install`, `make`, or anything that runs build scripts from the checked-out tree

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

For this research, I ran Fluxgate in batch mode against the top 1,000 repositories on GitHub by star count — projects like React, TensorFlow, Kubernetes, and VS Code. Fluxgate stores results in SQLite with WAL mode for resumability, which matters when you're making thousands of API calls and GitHub's rate limiter has opinions.

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
| FG-001 | Pwn Request (pull\_request\_target + fork checkout) | 44 | Critical / High |
| FG-002 | Script Injection (expression interpolation in `run:`) | 44 | High |
| FG-003 | Tag-Based Action Pinning (mutable references) | 23,850 | Medium / Info |
| FG-004 | Overly Broad Permissions | 455 | Medium |
| FG-005 | Secrets Exposed in Logs | 30 | Low |

FG-003 dominates the count because nearly every repository pins actions by tag (`@v4`) instead of by commit SHA. That's a known supply chain risk — it's how the second Trivy incident on March 19th worked, with the attacker force-pushing 75 `trivy-action` tags to point at malicious code. But the FG-001 findings are the ones that keep me up at night.

## 44 Pwn Request Findings: Confirmed, Likely, and Pattern-Only

Not all FG-001 findings are created equal. A `pull_request_target` workflow that checks out PR head code but only runs `diff` or `grep` against it is categorically different from one that runs `npm install`.

Fluxgate v0.2.0 (in development) formalizes this into confidence tiers, but the scan already applied this logic:

**Confirmed (post-checkout code execution detected):** The workflow runs a build tool — `npm install`, `pip install`, `make`, `cargo build` — on the checked-out code. An attacker can inject arbitrary commands through `package.json`, `setup.py`, `Makefile`, or equivalent.

**Likely (setup action + run step after checkout):** The workflow installs a language runtime (`actions/setup-node`, `actions/setup-python`) and then runs a command that isn't in the known read-only list. Tools like `eslint` or `prettier` load configuration files from the repo, which can execute arbitrary code.

**Pattern-only (checkout without detected execution):** The workflow checks out PR head code but post-checkout steps appear to be read-only — `diff`, `grep`, `test -f`. Still worth flagging for defense in depth, but not immediately exploitable.

The distribution:

- **20 critical findings** (confirmed or likely execution) across **16 unique repositories**
- **24 high findings** (pattern-only) across **12 unique repositories**

## What I Found in the Top 1,000

I can't name specific repositories until patches are in place or disclosure windows expire. But I can share the aggregate picture.

Among the 20 critical findings:

- **4 repositories** had `write-all` permissions (no `permissions:` block, which defaults to full write access) combined with confirmed code execution. This is the worst case — an attacker gets write access to the repository, its packages, its releases, and its secrets.
- **8 findings** involved `npm install` or `pnpm install` as the execution vector — the most common pattern by far.
- **5 findings** involved `pip install`.
- **3 findings** involved `make`.
- **4 findings** were "likely" — a setup action followed by a non-read-only run step.

Star counts for affected repositories ranged from 31,000 to 94,000. These are not obscure projects. They are foundational tools and frameworks used by millions of developers.

## The Disclosure Campaign

I've filed 8 responsible disclosures so far, through the appropriate channel for each project:

- **2** via GitHub Security Advisories
- **2** via HackerOne (bounty-eligible programs)
- **1** via Google's OSS Vulnerability Reward Program (bounty-eligible)
- **1** via Kubernetes' HackerOne program (bounty-eligible)
- **1** via Microsoft's MSRC portal
- **1** via direct email to the project's security contact

Each disclosure follows Fluxgate's [published disclosure protocol](https://github.com/north-echo/fluxgate/blob/main/DISCLOSURE.md): 30-day coordinated window, aggregate stats only until patches ship, no naming of unpatched repos.

8 more confirmed-critical repositories remain to be filed. Pattern-only findings will follow as a lower-priority batch.

Every disclosure platform has a different format. MSRC wants field-by-field structured input. HackerOne wants Summary/Steps/Impact/References with program-specific CVSS versions. Google VRP is free-form with dropdowns. GitHub Security Advisories are markdown. If you're doing bulk disclosure, plan for the formatting overhead.

## Lessons Learned

**Severity is contextual.** Early in this research, I filed an FG-001 finding against a project whose `pull_request_target` workflow checked out PR code but only ran `grep` and `git diff` on it. The maintainer pushed back — correctly. Post-checkout steps were read-only. No code execution, no exploitability. I sent an honest reassessment and revised the severity. That exchange directly motivated Fluxgate's confidence tiering system.

**Broken security contacts are common.** One project's `SECURITY.md` listed an email address that hard-bounced. I had to fall back to filing a GitHub Security Advisory and sending a backup email to an address I found in the project's original research paper. If you maintain an open-source project, verify your `SECURITY.md` contact actually works.

**Default permissions are the silent killer.** When a workflow doesn't include a `permissions:` block, GitHub defaults vary by configuration — and in many cases, the default is `write-all`. Several of the most dangerous findings I identified had no explicit permissions block at all. The maintainers likely didn't realize their CI workflows had write access to their entire repository.

## What Maintainers Should Do

If you maintain a project that uses `pull_request_target`:

1. **Audit your checkout refs.** If any step uses `ref: ${{ github.event.pull_request.head.sha }}` or `ref: ${{ github.event.pull_request.head.ref }}`, you are checking out fork code.

2. **Check what runs after checkout.** If anything builds, installs, tests, or lints the checked-out code, an attacker controls what executes. This includes `npm install`, `pip install -e .`, `make`, and any tool that reads config from the repo (`eslint`, `prettier`, `jest`, etc.).

3. **Add an explicit `permissions:` block.** Set the minimum permissions needed. `contents: read` is enough for most CI jobs. Never leave permissions unset on a `pull_request_target` workflow.

4. **Consider splitting the workflow.** GitHub's recommended pattern is to run untrusted code in a `pull_request` workflow (which has no secrets access), then use `workflow_run` to handle privileged operations after.

5. **Pin actions by commit SHA.** Tags are mutable. `@v4` can be force-pushed to point at anything. Pin to the full 40-character commit SHA.

You can run Fluxgate against your own repository to check:

```bash
# Local scan
fluxgate scan .

# Remote scan
fluxgate remote your-org/your-repo
```

## What's Next

Fluxgate v0.2.0 will formalize the confidence tier system into the scanner itself, so findings are automatically classified as confirmed, likely, or pattern-only. The spec is written; implementation is next.

The remaining 8 critical disclosures will be filed in the coming days. Pattern-only findings will follow after.

The Trivy compromise was a wake-up call, but the underlying problem is structural. `pull_request_target` is a footgun that requires maintainers to understand a subtle trust boundary that GitHub's own documentation has historically under-explained. The fact that 20 of the top 1,000 repositories on GitHub — projects with 30,000 to 94,000 stars — have this exact vulnerability class tells you the problem isn't developer negligence. It's a dangerous default in the platform.

Fluxgate is open source under Apache 2.0. Scan your repos. Fix your workflows. And if you get a security advisory from me, please don't ignore it.

---

*Christopher Lusk is a Principal Product Security Engineer at Red Hat. Fluxgate is an independent open-source project. Find it at [github.com/north-echo/fluxgate](https://github.com/north-echo/fluxgate).*
