---
title: "Advanced Cluster Management — Namespace Edit User Reaches cluster-admin via Application Subscription"
date: 2026-08-05
summary: "The app-subscription controller applied attacker-supplied Helm chart contents with its own elevated authority, without checking for the subscription-admin role or confining applied resources to the subscription namespace. A namespace-scoped edit user could ship a ClusterRoleBinding granting themselves cluster-admin."
vendor: "Red Hat"
product: "Advanced Cluster Management for Kubernetes"
status: "unpatched"
disclosedDate: 2026-08-05
externalOnly: true
advisoryUrl: "https://access.redhat.com/security/cve/CVE-2026-10090"
cves:
  - id: "CVE-2026-10090"
    title: "Namespace edit user can deploy cluster-scoped ClusterRoleBinding and become cluster-admin via Application Subscription"
    cvss3: 9.9
    cwe: "CWE-267"
tags: ["kubernetes", "vulnerability-disclosure", "rbac", "privilege-escalation"]
build:
  render: never
  list: local
---
