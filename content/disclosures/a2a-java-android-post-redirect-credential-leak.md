---
title: "A2A Java SDK: Android Synchronous POST Overrides Redirect Fix and Forwards Credentials Across Authorities"
date: 2026-09-09
summary: "The Android HTTP client in the Agent2Agent (A2A) Java SDK disabled redirects at connection setup, but the synchronous POST builder re-enabled them by default. On a cross-authority redirect, Android's OkHttp strips Authorization but keeps nonstandard headers, so API keys placed in X-API-Key, API-Key, X-Auth-Token or X-Authentication by the shipped AuthInterceptor were forwarded to a different host. An incomplete fix for GHSA-9rhm-2h4x-jwmx. Fixed in 1.3.2.Final."
vendor: "Agent2Agent (A2A) Project"
product: "a2a-java"
status: "fixed"
reportedDate: 2026-09-07
disclosedDate: 2026-09-09
externalOnly: true
advisoryUrl: "https://github.com/a2aproject/a2a-java/security/advisories/GHSA-4fpj-g9jf-9rgg"
cves:
  - id: "CVE-2026-111765"
    title: "Android synchronous POST overrides the GHSA-9rhm redirect fix and forwards credentials across authorities"
    cvss3: 7.5
    cwe: "CWE-522"
ghsa: "GHSA-4fpj-g9jf-9rgg"
tags: ["ai-agents", "vulnerability-disclosure", "variant-analysis"]
build:
  render: never
  list: local
---
