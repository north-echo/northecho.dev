---
title: "FUXA — Guest and Invalid-Token Access to Protected Read APIs in Secure Mode"
date: 2026-05-28
summary: "With secureEnabled set, FUXA 1.3.0-2773 still served the project, alarms, and scheduler read APIs to guest and invalid-token requests. Fixed in v1.3.1."
vendor: "frangoteam"
product: "FUXA"
status: "fixed"
disclosedDate: 2026-05-28
externalOnly: true
advisoryUrl: "https://github.com/frangoteam/FUXA/security/advisories/GHSA-r9g5-7q8j-958c"
cves:
  - id: "CVE-2026-47718"
ghsa: "GHSA-r9g5-7q8j-958c"
tags: ["scada", "hmi", "vulnerability-disclosure", "fuxa", "authentication"]
build:
  render: never
  list: local
---
