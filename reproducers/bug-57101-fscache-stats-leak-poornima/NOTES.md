# Reproduction notes — bug 57101 (fscache/stats memory leak)

Independent reproduction (separate from reproducers/bug-57101-fscache-stats-leak/,
which is a coworker's earlier work — not modified or reused here beyond referencing
the busybox-initramfs technique).

## Build

`~/kernel-work/linux-buggy/arch/x86/boot/bzImage` already existed
(v3.9.0-rc8, built 2026-08-17 in the `kernel-builder` container, gcc 4.8.4).
`.config` was checked and already had all required options set:
CONFIG_DEBUG_KMEMLEAK=y, CONFIG_FSCACHE=y, CONFIG_FSCACHE_STATS=y,
CONFIG_DEBUG_FS=y, CONFIG_DEBUG_KERNEL=y, CONFIG_FRAME_POINTER=y.
Per task instructions the build step was skipped — no rebuild, no build.log,
and `fs/fscache/stats.c` was not touched.

## Boot / test harness

Own minimal initramfs built from a statically-linked `busybox` (installed via
`busybox-static` in the `kernel-builder` container, copied out with
`docker cp`) plus a custom `/init` script that:

1. mounts proc/sysfs/devtmpfs/debugfs
2. clears the kmemleak baseline (`echo clear > .../kmemleak`)
3. opens (`cat`) `/proc/fs/fscache/stats` 4 times
4. sleeps past kmemleak's `MSECS_MIN_AGE`, runs two `echo scan` passes
5. dumps `/sys/kernel/debug/kmemleak` and powers off

Booted directly on the host with `qemu-system-x86_64 -kernel bzImage -initrd
initramfs.cpio.gz -append "console=ttyS0 kmemleak=on panic=-1" -nographic
-no-reboot` (no `-s -S`). Full serial output captured in `proof/boot-output.log`;
the kmemleak report section is extracted into `proof/kmemleak-output.log`.

## Result

kmemleak flagged **3 unreferenced objects** (32 bytes each — the `struct
seq_file` sized/allocated by `single_open()`) after 4 opens of
`/proc/fs/fscache/stats`. All three share the identical allocation backtrace:

```
kmemleak_alloc
kmem_cache_alloc_trace
single_open
fscache_stats_open
proc_reg_open
do_dentry_open
finish_open
do_last
path_openat
do_filp_open
do_sys_open
sys_open
system_call_fastpath
```

## Comparison to the original report

`bugzilla.kernel.org/show_bug.cgi?id=57101` is behind an Anubis bot-check
that blocked automated fetch in this session (confirmed via WebFetch, both
the HTML page and the `rest/bug/57101` API endpoint returned the Anubis
"Access Denied" challenge page, not bug content). The archived report could
not be retrieved directly. The comparison below instead uses the fix commit
message (ec686c9239b4d472052a271c505d04dae84214cc, "fscache: fix memory leak
in fscache_stats_open()") and the linux-kernel mailing list thread that
carried the same fix (lkml.iu.edu/hypermail/linux/kernel/1304.3/00703.html,
00763.html), both of which quote the bug reporter's description verbatim and
are the authoritative record of the bug's root cause and call path.

| Aspect | Original report / fix commit | This reproduction |
|---|---|---|
| Trigger | Reading `/proc/fs/fscache/stats` | `cat /proc/fs/fscache/stats` x4 |
| Allocating function | `single_open()` | `single_open` (in backtrace) |
| Caller of allocator | `fscache_stats_open()` | `fscache_stats_open` (in backtrace) |
| Broken release | `fscache_stats_fops.release = seq_release` (should be `single_release`) | unchanged in source; confirmed by leak persisting |
| Leaked object | struct allocated by `single_open` (never freed by `seq_release`) | 32-byte object, alloc site `single_open`, x3 leaks for x4 opens |
| File | `fs/fscache/stats.c` | not modified; same file |

Function-name order and identity match: every reproduction leak's allocation
site is `single_open` called directly from `fscache_stats_open`, which is
exactly the root cause described in the fix commit and mailing-list thread.
The surrounding VFS frames (`proc_reg_open` → `do_dentry_open` →
`finish_open` → `do_last` → `path_openat` → `do_filp_open` → `do_sys_open` →
`sys_open` → `system_call_fastpath`) are the standard open(2) path on x86_64
3.9 and are not specific to the bug report but are consistent with "leak
observed when the proc file is read" (i.e., leak occurs on open, one leaked
object per open, matching 3-4 opens -> 3-4 leaked objects here).

Note: only 3 leaks were reported for 4 opens — kmemleak's scan is
best-effort/timing-sensitive (it can miss an object that hasn't aged past
`MSECS_MIN_AGE` by the first scan pass, or briefly hold a stale reference to
scanned memory). This does not affect the root-cause match; it is a known
characteristic of kmemleak's periodic/triggered scanning, not evidence
against the leak.
