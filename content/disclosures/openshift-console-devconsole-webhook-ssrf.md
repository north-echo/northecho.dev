---
title: "OpenShift Console — Authenticated SSRF via Dev Console Webhook Helpers"
date: 2026-08-11
summary: "The Dev Console webhook helpers fetched user-supplied target URLs server-side without validation. Path neutralization allowed arbitrary endpoint targeting, and the full response was reflected back to the caller from the console pod's privileged network position."
vendor: "Red Hat"
product: "OpenShift Container Platform 4"
status: "unpatched"
disclosedDate: 2026-08-11
externalOnly: true
advisoryUrl: "https://access.redhat.com/security/cve/CVE-2026-50236"
cves:
  - id: "CVE-2026-50236"
    title: "Authenticated SSRF with full response reflection and path neutralization via Dev Console webhook helpers"
    cvss3: 7.4
    cwe: "CWE-918"
tags: ["kubernetes", "openshift", "vulnerability-disclosure", "ssrf"]
build:
  render: never
  list: local
---
