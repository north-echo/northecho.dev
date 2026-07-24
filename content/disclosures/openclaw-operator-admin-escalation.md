---
title: "OpenClaw — operator.admin Escalation via Trusted-Proxy Auth Mode"
date: 2026-04-03
summary: "An incomplete scope-clearing fix let self-declared operator scopes survive on identity-bearing trusted-proxy auth paths for non-Control-UI clients, escalating to operator.admin. Fixed in 2026.3.31."
vendor: "OpenClaw"
product: ""
status: "fixed"
disclosedDate: 2026-04-03
externalOnly: true
advisoryUrl: "https://github.com/openclaw/openclaw/security/advisories/GHSA-g374-mggx-p6xc"
cves:
  - id: "CVE-2026-41404"
ghsa: "GHSA-g374-mggx-p6xc"
tags: ["ai-agents", "vulnerability-disclosure", "openclaw", "authorization"]
build:
  render: never
  list: local
---
