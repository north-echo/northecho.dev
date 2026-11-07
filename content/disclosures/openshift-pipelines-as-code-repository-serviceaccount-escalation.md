---
title: "OpenShift Pipelines-as-Code: Repository Creators Run PipelineRuns as the Auto-Provisioned Pipeline ServiceAccount"
date: 2026-08-27
summary: "Pipelines-as-Code did not check whether the creator of a Repository custom resource was allowed to create PipelineRuns. A namespace user with only Repository and Secret permissions could have PAC execute attacker-controlled PipelineRuns as the auto-provisioned pipeline ServiceAccount, which carries ClusterRole/edit by default, yielding namespace-scoped privilege escalation including Secret read and CI/CD supply chain injection."
vendor: "Red Hat"
product: "OpenShift Pipelines"
status: "unpatched"
reportedDate: 2026-05-12
disclosedDate: 2026-08-27
externalOnly: true
advisoryUrl: "https://access.redhat.com/security/cve/CVE-2026-50240"
cves:
  - id: "CVE-2026-50240"
    title: "Missing authorization on Repository creation lets namespace users run PipelineRuns as the edit-bound pipeline ServiceAccount"
    cvss3: 8.8
    cwe: "CWE-862"
tags: ["kubernetes", "openshift", "vulnerability-disclosure", "ci-cd", "supply-chain"]
build:
  render: never
  list: local
---
