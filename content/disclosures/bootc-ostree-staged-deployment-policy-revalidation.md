---
title: "bootc/ostree: Staged Deployments Not Revalidated Against Container Signature Policy Before Finalization"
date: 2026-10-18
summary: "bootc and ostree finalize an already-staged deployment without rechecking its container image against the current signature policy. Unsigned OS content staged while the policy was permissive stays bootable after the policy is hardened to reject unsigned images, leading to full host compromise on reboot. Red Hat lists Hardened Images as affected."
vendor: "Red Hat"
product: "bootc / ostree"
status: "unpatched"
reportedDate: 2026-07-04
disclosedDate: 2026-10-18
externalOnly: true
advisoryUrl: "https://access.redhat.com/security/cve/CVE-2026-16460"
cves:
  - id: "CVE-2026-16460"
    title: "Staged deployments not revalidated against container signature policy before finalization"
    cvss3: 7.4
    cwe: "CWE-345"
tags: ["linux", "vulnerability-disclosure", "supply-chain", "containers"]
build:
  render: never
  list: local
---
