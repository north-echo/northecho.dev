---
title: "Linux kernel m68k: seccomp filters ignored on ColdFire and 68000"
date: 2026-09-13T13:17:53Z
summary: "The ColdFire and 68000 syscall entry paths skipped seccomp filtering unless the process was also being traced."
fixType: "security fix"
merged: "merged for Linux 7.4"
links:
  - name: "Fix commit"
    url: "https://git.kernel.org/linus/48859f8018d93bd93ef35f7430fdf5b97f2f9715"
  - name: "Patch discussion"
    url: "https://lore.kernel.org/all/20260913131753.605893-1-clusk@northecho.dev/"
tags: ["linux-kernel", "m68k", "seccomp", "upstream-patch"]
build:
  render: never
  list: local
---

m68k advertises seccomp filter support on every CPU type, but the ColdFire and 68000 system-call entry paths only checked for a tracer before taking the slow path. A process that installed a seccomp filter without being traced skipped that path, so its filters were never applied. Classic MMU m68k was not affected.

I wrote the fix with AI assistance. In review, Andreas Schwab suggested combining the two flag checks into a single test, which the final version adopted. Under QEMU on an emulated ColdFire board, filters denying getpid, openat, unlinkat and reboot each returned -EPERM after the change. The 68000 path gets the identical change and was compile-tested only. Backported to the 6.6, 6.12, 6.18 and 7.3 stable kernels. The bug dates to Linux 6.3.
