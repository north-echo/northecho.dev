---
title: "Linux kernel SMB client: read retries skipped their back-off"
date: 2026-07-29T19:20:02Z
summary: "The read send-error retry path never set the replay flag, so retries skipped their back-off."
kind: "reliability fix"
merged: "merged in Linux 7.3"
links:
  - name: "Fix commit"
    url: "https://git.kernel.org/linus/45f84cf25a08"
  - name: "Patch discussion"
    url: "https://lore.kernel.org/linux-cifs/20260729192002.876156-1-clusk@northecho.dev/"
tags: ["linux-kernel", "smb", "upstream-patch"]
build:
  render: never
  list: local
---

When sending a read failed, the SMB client retried it but never set the replay flag that the write path sets. As a result the read retry ignored the exponential back-off it had just calculated and reissued at once, and the reissued request was not marked as a replay to the server.

I found it during my own AI-assisted audit of recent SMB client code, and wrote and submitted the fix. Backported to the 7.2 stable kernel; older kernels never had the bug.
