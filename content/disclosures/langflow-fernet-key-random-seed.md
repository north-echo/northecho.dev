---
title: "Langflow — Weak Fernet Key Derivation via random.seed()"
date: 2026-08-12
summary: "When SECRET_KEY was shorter than 32 characters, Langflow derived the Fernet key protecting every stored credential (API keys, variables, MCP OAuth settings) by seeding Python's non-cryptographic Mersenne Twister with it, so anyone who knows or brute-forces the secret can reproduce the key and decrypt the database. Reported independently and consolidated into the canonical advisory. Fixed in 1.10.1."
vendor: "IBM"
product: "Langflow OSS"
status: "fixed"
reportedDate: 2026-03-29
disclosedDate: 2026-08-12
externalOnly: true
advisoryUrl: "https://github.com/advisories/GHSA-jxw3-mjmx-3pqm"
cves:
  - id: "CVE-2026-9205"
    title: "Weak Fernet key derivation via random.seed()"
    cvss3: 9.1
    cwe: "CWE-338"
ghsa: "GHSA-jxw3-mjmx-3pqm"
tags: ["vulnerability-disclosure", "langflow", "credentials", "ai", "cryptography"]
build:
  render: never
  list: local
---
