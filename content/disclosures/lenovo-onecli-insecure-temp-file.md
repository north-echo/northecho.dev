---
title: "Lenovo XClarity Essentials OneCLI — Predictable Temporary File Enables Symlink File Overwrite"
date: 2026-08-04
summary: "The Linux build of OneCLI 5.5.0 and below created temporary files at predictable paths. A local low-privileged attacker could pre-plant a symlink and, when OneCLI ran with elevated privileges, overwrite or truncate arbitrary files with program-generated data. Fixed in 5.6.0."
vendor: "Lenovo"
product: "XClarity Essentials OneCLI"
caseId: "LEN-216072"
status: "fixed"
disclosedDate: 2026-08-04
externalOnly: true
advisoryUrl: "https://support.lenovo.com/us/en/product_security/LEN-216072"
cves:
  - id: "CVE-2026-16791"
    title: "Predictable temporary file symlink vulnerability in Lenovo XClarity Essentials OneCLI"
    cvss3: 3.9
    cvss4: 1.0
    cwe: "CWE-377"
tags: ["vulnerability-disclosure", "lenovo", "local-privilege-escalation", "symlink"]
build:
  render: never
  list: local
---
