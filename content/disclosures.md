---
title: "Disclosures"
description: "Coordinated vulnerability disclosure policy for North Echo projects"
---

If you have found a security vulnerability in software I personally maintain or in this website, I want to hear about it. This page describes how to report it, what to expect in return, and the rules of the road for good-faith research.

## Reporting a vulnerability

**Email:** [clusk@northecho.dev](mailto:clusk@northecho.dev)

Please include, where possible:

- A description of the vulnerability and the affected component (repository, commit, version, or URL).
- Steps to reproduce, or a minimal proof-of-concept.
- The impact you believe it has.
- Any suggested remediation.

If you require encrypted communication, request a PGP key in your initial message and one will be provided.

The well-known endpoint at [`/.well-known/security.txt`](/.well-known/security.txt) restates the contact and policy URL in [RFC 9116](https://www.rfc-editor.org/rfc/rfc9116) form.

## Scope

**In scope**

- Software I personally author and publish under [github.com/north-echo](https://github.com/north-echo) (e.g., Fluxgate, WAINGRO, REAPER, 21csim).
- This website (`northecho.dev`) and its build/deploy pipeline.

**Out of scope**

- Third-party services and infrastructure I depend on but do not control (Cloudflare, GitHub, package registries, upstream dependencies). Report those directly to the owning vendor.
- Social engineering, phishing, or physical attacks against me or anyone else.
- Volumetric denial-of-service, brute-force, or rate-limit testing.
- Findings that require already-compromised credentials or privileged access you were not authorized to hold.
- Reports generated solely by automated scanners with no demonstrated impact.

**Red Hat products and services** are explicitly out of scope here. Please report those to the Red Hat Product Security team at [secalert@redhat.com](mailto:secalert@redhat.com) — see Red Hat's [security policy](https://access.redhat.com/security/team/contact) for details.

## What to expect

I run this as an individual, not a team, so timelines are conservative:

| Stage | Target |
|---|---|
| Acknowledgement of receipt | Within 3 business days |
| Initial triage and severity assessment | Within 10 business days |
| Status updates while a report is open | At least every 14 days |
| Coordinated public disclosure window | 90 days from initial report, by default |

The 90-day window is the default ceiling, not a floor — fixes are released as soon as they are ready. The window may be extended by mutual agreement when a fix is in active development, or shortened if the issue is being actively exploited.

## Safe harbor

I will not pursue or support legal action against researchers who:

- Make a good-faith effort to comply with this policy.
- Avoid privacy violations, destruction of data, and degradation of service for other users.
- Stop testing and report immediately if they encounter user data, credentials, or systems outside the documented scope.
- Give a reasonable opportunity to remediate before any public disclosure.

This is a personal commitment; it does not bind any third party (including any employer, hosting provider, or upstream project).

## Acknowledgement

If you would like to be credited in the resulting advisory, release notes, or commit message, say so in your report and tell me how you would like to be named and linked. Anonymous and pseudonymous reports are equally welcome.

## Reports I have sent

If I have contacted *you* about a vulnerability in software you maintain, the protocol I follow is documented in the [Fluxgate disclosure protocol](https://github.com/north-echo/fluxgate/blob/main/DISCLOSURE.md), which mirrors the timelines above.
