# Build fixes log — bug 57101 (fscache/stats memory leak)

Target: Linux v3.9-rc8 (pre-fix), built inside the `kernel-builder` container
(ubuntu:14.04, gcc 4.8.4), `make ARCH=x86_64 -j$(nproc) bzImage modules`.

## Result

The build completed with **zero errors** on the first attempt
(`DONE_RC=0` in `proof/build.log`, `arch/x86/boot/bzImage` produced).
gcc 4.8.4 is period-correct for a v3.9-era kernel, so no toolchain
compatibility patches were required.

No changes were made to any file in the source tree, and in particular
`fs/fscache/stats.c` was left untouched — the bug (`.release = seq_release`
instead of `single_release` in `fscache_stats_fops`, `fs/fscache/stats.c:290`)
remains present in the built kernel.

No entries below because no fixes were needed.
