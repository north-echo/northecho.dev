---
title: "OpenShift Console — Namespace Tenant SSRF and Catalog Poisoning via ProjectHelmChartRepository"
date: 2026-08-11
summary: "A namespace tenant could plant a ProjectHelmChartRepository pointing at an arbitrary URL that the console pod then fetched server-side, bypassing tenant egress restrictions. Combined with catalog metadata poisoning and admin-mediated chart installation, this yielded a supply chain path to privilege escalation."
vendor: "Red Hat"
product: "OpenShift Container Platform 4"
status: "unpatched"
disclosedDate: 2026-08-11
externalOnly: true
advisoryUrl: "https://access.redhat.com/security/cve/CVE-2026-50237"
cves:
  - id: "CVE-2026-50237"
    title: "Namespace tenant SSRF with egress bypass, catalog poisoning, and admin-mediated supply chain escalation via ProjectHelmChartRepository"
    cvss3: 7.4
    cwe: "CWE-918"
tags: ["kubernetes", "openshift", "vulnerability-disclosure", "ssrf", "supply-chain"]
build:
  render: never
  list: local
---
