---
title: "Leapp RHEL 9→10 Upgrade — scan_mysql Runs mysqld as Root and Can Load mysql-Writable Plugins"
date: 2026-09-15
summary: "During a RHEL 9 to RHEL 10 upgrade, the scan_mysql actor in leapp-upgrade-el9toel10 runs `mysqld --validate-config` directly as root, bypassing the packaged unit that would drop to User=mysql. A process already compromised as the mysql identity can plant an attacker-writable plugin that mysqld then loads with root authority — escalating a mysql-level foothold to full root during the upgrade."
vendor: "Red Hat"
product: "Red Hat Enterprise Linux 9"
status: "unpatched"
disclosedDate: 2026-09-15
externalOnly: true
advisoryUrl: "https://access.redhat.com/security/cve/CVE-2026-75092"
cves:
  - id: "CVE-2026-75092"
    title: "scan_mysql runs mysqld --validate-config as root and can load mysql-writable plugins"
    cvss3: 7.3
    cwe: "CWE-250"
tags: ["linux", "vulnerability-disclosure", "privilege-escalation", "supply-chain"]
build:
  render: never
  list: local
---
