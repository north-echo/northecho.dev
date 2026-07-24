---
title: "OpenClaw — Path Traversal in ACP Dispatch Allows Arbitrary File Read"
date: 2026-04-03
summary: "Inbound channel attachment paths in ACP dispatch bypassed the attachment-cache and root-directory checks, allowing an authenticated remote attacker to read arbitrary files. Fixed in 2026.3.31."
vendor: "OpenClaw"
product: ""
status: "fixed"
disclosedDate: 2026-04-03
externalOnly: true
advisoryUrl: "https://github.com/openclaw/openclaw/security/advisories/GHSA-58q2-7r52-jq62"
cves:
  - id: "CVE-2026-41370"
ghsa: "GHSA-58q2-7r52-jq62"
tags: ["ai-agents", "vulnerability-disclosure", "openclaw", "path-traversal"]
build:
  render: never
  list: local
---
