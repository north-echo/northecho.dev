---
title: "Ignition — Low-Privilege Project Creation Leads to Gateway-Side Script Execution"
date: 2026-08-23
summary: "A roleless authenticated Vision client user in Ignition 8.1.53 could create a project seeded with a startup script and obtain code execution on the gateway when the project loaded. Root cause: project creation defaulted to unrestricted instead of requiring the Designer role. Fixed in 8.1.54."
vendor: "Inductive Automation"
product: "Ignition"
status: "fixed"
caseId: "SAG-76 / VRF#26-08-FVNGT"
reportedDate: 2026-04-04
disclosedDate: 2026-08-23
advisoryUrl: "https://security.inductiveautomation.com/?tcuUid=34477620-731d-4b70-b22b-9450f9a659a3"
cves:
  - id: "CVE-2026-77393"
    title: "Low-privilege project creation leads to gateway-side script execution"
    cvss3: 8.8
    cvss4: 8.7
    cwe: "CWE-276 (Incorrect Default Permissions)"
tags: ["ics", "ot", "scada", "vulnerability-disclosure", "inductive-automation", "ignition"]
---

North Echo reported an authenticated privilege-escalation issue in **Inductive Automation Ignition 8.1.53** to the Inductive Automation security team. A **roleless authenticated Vision client user** could create a new project containing attacker-controlled resources and obtain **code execution on the gateway** when that project loaded. Inductive Automation confirmed the lead finding, fixed it in **Ignition 8.1.54** (released May 20, 2026), and credited North Echo in its Trust Center advisory.

---

## Background: projects and gateway scripting

Ignition is a widely deployed **SCADA / HMI platform**. A gateway hosts *projects*, and projects can carry **gateway-scoped event scripts** — including startup scripts that run **server-side, in the gateway's own context**, when a project starts. Authoring projects and their scripts is meant to be a **Designer**-level activity: it is, by design, a code-execution surface, so it should sit behind a privileged role.

The Vision client, by contrast, is the runtime an ordinary operator logs into. A Vision client user is not supposed to be able to author gateway-executing code.

## The finding

That boundary did not hold. The project-management RPC path that creates projects **defaulted to unrestricted** rather than requiring the Designer role. As a result, a normal authenticated Vision client user with **no special roles** could:

1. Authenticate as a low-privilege user.
2. Call the project-creation RPC to create a new project.
3. Include an `ignition/event-scripts` **startup script** in that project.
4. Wait for the gateway to load the project (on restart or project start), at which point the seeded script **executed on the gateway**.

The seeded startup script was ordinary Jython running in the gateway context:

```python
f = open('/tmp/lp3_startup_hit', 'w')
f.write('hit')
f.close()
```

After the gateway loaded the malicious project, the marker file existed inside the container, and the gateway log recorded the project starting:

```text
-rw-r--r--. 1 ignition ignition 3 /tmp/lp3_startup_hit
Starting project: BleachExec
Project started. project-name=BleachExec
```

The low-privilege `Projects.create` call returned `errorNo=0`, and the malicious project — manifest plus the encoded event-script resource — was written to disk under `data/projects/`. Validated end-to-end against a fresh Ignition 8.1.53 gateway on April 4, 2026.

## Why it matters

This is an **authenticated remote-code-execution-class** outcome reached from an unprivileged account. A user provisioned only for Vision client access could pivot to running arbitrary server-side scripting in the gateway's context — a far more severe result than an information leak or an isolated authorization bypass. On an operational-technology asset, code execution on the gateway is about as high as impact goes.

The root cause is a classic **incorrect-default-permission** problem (CWE-276): a code-execution-capable operation whose default posture was permissive, so the privileged-role gate that *should* have protected it was never consulted.

## The fix

Inductive Automation confirmed the finding, framing the root cause as a project-creation permission that defaulted to unrestricted when it should require the Designer role. The corrected behavior shipped in:

- **Ignition 8.1.54** (released **May 20, 2026**) — project creation is now restricted to Designer sessions, and the permissive default is no longer consulted.
- The **8.3** series was **not affected**.

Operators on 8.1.x should update to **8.1.54 or later**.

## Disclosure timeline

| Date (2026) | Event |
|-------------|-------|
| Apr 04 | Initial report to Inductive Automation security team (Ignition 8.1.53) |
| Apr 29 | Vendor confirms the lead finding; agrees to CVE and credit; coordinates embargo |
| May 20 | Fix ships in **Ignition 8.1.54** |
| Aug 22 | Vendor closes the loop; Trust Center advisory live; CVE submitted to CISA |
| Sep 03 | **CVE-2026-77393** assigned; [CISA ICS advisory ICSA-26-246-06](https://www.cisa.gov/news-events/ics-advisories/icsa-26-246-06) published, crediting North Echo |

## A note on the second observation

The same campaign also surfaced a `ModuleInvoke` RPC path where **deserialization of client-supplied arguments occurred before method-level authorization** on a protected method. The reachability and the ordering were demonstrable, but no working gadget chain was produced through that path. The vendor assessed it as covered by the 2023 deserialization remediations (CVE-2023-50218 through CVE-2023-50221), with further hardening in 8.3, and it is treated as closed absent a concrete exploit. It is noted here for completeness rather than as a live issue.

## Coordination & credit

This was reported and fixed under coordinated disclosure with the **Inductive Automation security team**, who handled it professionally throughout. The fix is public in **Ignition 8.1.54**. The issue is tracked as **CVE-2026-77393** (CVSS v3.1 8.8 / v4.0 8.7), published in **[CISA ICS advisory ICSA-26-246-06](https://www.cisa.gov/news-events/ics-advisories/icsa-26-246-06)**, and the Inductive Automation **[Trust Center advisory](https://security.inductiveautomation.com/?tcuUid=34477620-731d-4b70-b22b-9450f9a659a3)** credits North Echo as the reporter.

*North Echo Security Research performs independent, logic-level vulnerability research on ICS/OT and edge platforms. — [northecho.dev](https://northecho.dev)*
