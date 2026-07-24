---
title: "OpenShift Pipelines — tekton-scheduler RBAC Grants Authenticated-Group Write"
date: 2026-06-04
summary: "The tekton-scheduler-rolebinding ClusterRoleBinding granted the system:authenticated group write access to Kueue and cert-manager custom resources via the tekton-scheduler-role ClusterRole. Fixed via RHSA-2026:36648 / RHSA-2026:41036."
vendor: "Red Hat"
product: "OpenShift Pipelines"
status: "fixed"
disclosedDate: 2026-06-04
externalOnly: true
advisoryUrl: "https://access.redhat.com/security/cve/CVE-2026-10840"
cves:
  - id: "CVE-2026-10840"
tags: ["kubernetes", "openshift", "vulnerability-disclosure", "rbac"]
build:
  render: never
  list: local
---
