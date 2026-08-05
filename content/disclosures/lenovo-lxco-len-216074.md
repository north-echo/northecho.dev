---
title: "Lenovo XClarity Orchestrator — TLS Validation Bypass and OS Profile Command Injection"
date: 2026-08-04
summary: "Two defects in LXCO 2.2.0: multiple microservices skipped TLS certificate validation, exposing HTTPS connections to an adjacent-network machine-in-the-middle; and an OS profile password field passed shell metacharacters through unneutralized, giving an authenticated attacker command execution as a privileged user. Fixed in GAFix 2.2.0."
vendor: "Lenovo"
product: "XClarity Orchestrator"
caseId: "LEN-216074"
status: "fixed"
disclosedDate: 2026-08-04
externalOnly: true
advisoryUrl: "https://support.lenovo.com/us/en/product_security/LEN-216074"
cves:
  - id: "CVE-2026-16792"
    title: "Global TLS certificate validation bypass in Lenovo XClarity Orchestrator"
    cvss3: 6.1
    cvss4: 7.0
    cwe: "CWE-295"
  - id: "CVE-2026-16793"
    title: "Remote command injection via OS profile password in Lenovo XClarity Orchestrator"
    cvss3: 8.8
    cvss4: 8.7
    cwe: "CWE-78"
tags: ["vulnerability-disclosure", "lenovo", "tls", "command-injection"]
build:
  render: never
  list: local
---
