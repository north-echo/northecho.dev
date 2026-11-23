---
title: "Linux kernel SMB client: request-buffer leak on SMB Direct reads"
date: 2026-07-29T22:00:17Z
summary: "smb2_new_read_req() leaked its request buffer when RDMA memory registration failed."
cves: ["CVE-2026-90125"]
kind: "memory leak"
merged: "merged in Linux 7.3"
links:
  - name: "CVE record"
    url: "https://www.cve.org/CVERecord?id=CVE-2026-90125"
  - name: "Fix commit"
    url: "https://git.kernel.org/linus/deb6468f4164640e4dc875f008aa449cf55987a5"
  - name: "Patch discussion"
    url: "https://lore.kernel.org/linux-cifs/20260729220017.944651-1-clusk@northecho.dev/"
tags: ["linux-kernel", "smb", "upstream-patch"]
build:
  render: never
  list: local
---

`smb2_new_read_req()` allocated a request buffer and then, if RDMA memory registration failed, returned early without freeing it. On hard mounts the request is retried without limit, so the leak repeats on every retry. Only affects SMB Direct (RDMA) mounts.

Sashiko, an AI code-review bot, flagged the failure path while reviewing an earlier patch of mine. I confirmed it against the source, narrowed its scope, wrote the fix, and tested it with kmemleak. Backported to the 6.1, 6.6, 6.12, 6.18 and 7.2 stable kernels. The bug dates to Linux 4.16.
