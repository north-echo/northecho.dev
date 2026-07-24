---
title: "MultiCluster Engine — Pull-Secret Disclosure via InfraEnv Status Conditions"
date: 2026-05-29
summary: "assisted-service wrote raw referenced pull-secret contents into InfraEnv.status.conditions[].message on validation failure. A namespace principal holding only the stock view ClusterRole cannot read Secrets directly but can read InfraEnv objects, exposing the secret."
vendor: "Red Hat"
product: "MultiCluster Engine for Kubernetes"
status: "unpatched"
disclosedDate: 2026-05-29
externalOnly: true
advisoryUrl: "https://access.redhat.com/security/cve/CVE-2026-10101"
cves:
  - id: "CVE-2026-10101"
tags: ["kubernetes", "vulnerability-disclosure", "rbac", "secrets-exposure"]
build:
  render: never
  list: local
---
