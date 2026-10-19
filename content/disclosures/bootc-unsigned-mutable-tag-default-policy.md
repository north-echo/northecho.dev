---
title: "bootc: Automatic Updates Can Apply Unsigned Mutable-Tag Content Under the Default Containers Policy"
date: 2026-10-10
summary: "Under the default containers signature policy, bootc upgrades of a mutable image tag can apply unsigned content from any registry that falls through to insecureAcceptAnything. An attacker who can update that tag gets arbitrary OS content installed as the host root filesystem on the next reboot. Affects bootc and containers-common on RHEL 8, 9 and 10 and Red Hat Hardened Images."
vendor: "Red Hat"
product: "bootc"
status: "unpatched"
reportedDate: 2026-07-04
disclosedDate: 2026-10-10
externalOnly: true
advisoryUrl: "https://access.redhat.com/security/cve/CVE-2026-16487"
cves:
  - id: "CVE-2026-16487"
    title: "bootc automatic updates can apply unsigned mutable-tag content under default containers policy"
    cvss3: 8.1
    cwe: "CWE-345"
tags: ["linux", "vulnerability-disclosure", "supply-chain", "containers"]
build:
  render: never
  list: local
---
