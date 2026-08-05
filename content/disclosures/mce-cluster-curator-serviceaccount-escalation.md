---
title: "MultiCluster Engine — Namespace Admin Escalates to Cluster-Wide Authority via ClusterCurator ServiceAccount Token"
date: 2026-08-05
summary: "A tenant administrator holding only namespace-scoped privileges can create a namespaced ClusterCurator and, through it, mint a token for a ServiceAccount carrying cluster-wide administrative authority — a full escalation to cluster control."
vendor: "Red Hat"
product: "MultiCluster Engine for Kubernetes"
status: "unpatched"
disclosedDate: 2026-08-05
externalOnly: true
advisoryUrl: "https://access.redhat.com/security/cve/CVE-2026-10059"
cves:
  - id: "CVE-2026-10059"
    title: "Namespace admin can escalate to cluster-wide curator authority via ClusterCurator ServiceAccount token"
    cvss3: 9.1
    cwe: "CWE-266"
tags: ["kubernetes", "vulnerability-disclosure", "rbac", "privilege-escalation"]
build:
  render: never
  list: local
---
