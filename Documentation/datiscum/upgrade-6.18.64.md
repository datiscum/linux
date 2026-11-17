# Linux 6.18.64 Upgrade Record

## Source History

Branch: `linux-6.18.64-datiscum`.
Initial source tag: `v6.18.64-datiscum-source`.
Prepared kernel release: `6.18.64-datiscum1`.

The existing custom 6.18.60 history was merged with official stable
`v6.18.64`, retaining all 2,571 stable commits since `v6.18.60`, including
the intermediate 6.18.61, 6.18.62 and 6.18.63 releases.

| Official Tag | Commit |
| --- | --- |
| v6.18.61 | `7ab573237315f5cba0603cb039698adebeb29094` |
| v6.18.62 | `c3f63d70f236bb7685e6d11d1d9b5b351552e600` |
| v6.18.63 | `1cdda838e8475edf89b73ff0cc952005dd0b1fa7` |
| v6.18.64 | `fd7cfd39c88596fec816639ad83cbf3e9d316372` |

The kernel-source merge is commit
`2f24942962823f831e90014fdf41cbdc13811ff6`.

## Conflict Resolution

There was one content conflict, in `mm/vmscan.c:shrink_one()`.
Stable commit `2f0a531cf436` (upstream
[`15da1e4978dd`](https://github.com/torvalds/linux/commit/15da1e4978dd4ea8f5faa6e15ee967ba4ad9a30a))
adds protection calculation for non-kswapd reclaim. The local workingset
protection already inserts code at the same location.

The merged ordering is:

1. Calculate non-kswapd memcg protection using the new stable helper.
2. Run the unchanged local `prepare_workingset_protection()` block and guard.
3. Continue with the existing memcg minimum/low checks and reclaim path.

The new helper therefore runs before the local early return. Both the
stable fix and the local protection are retained, including the helper's
declaration, implementation and configuration stub from the stable history.

## Preservation Checks

These comparisons refer to the kernel-source merge commit above, before
the separate configuration snapshot and documentation updates.

Of 17 paths changed both locally and upstream, the other 16 retain identical
stable patch IDs for their local modifications. In `mm/vmscan.c`, every local
added and removed code line is unchanged; only the surrounding upstream
context and the placement after the new protection calculation differ.

The other 139 locally modified paths are byte-identical to the custom 6.18.60
version. The 2,121 paths changed only upstream match official `v6.18.64`.
Counts refer to paths with renames expanded to their old and new names.

Preserved changes include:

* Scheduler overflow fixes, 128-bit eligibility, zero-weight guard and diagnostics.
* The local x86-64-v3 scheduler fast path and preemptible IPI backport.
* io_queue queue/IRQ masks, CPU-offlining protection and housekeeping helpers.
* Xe changes, NTFS-Plus, GUD bounds checks and the existing Cachy-derived changes.

The verified difference between the two custom Git versions was applied
to the detached build copy using `git apply`, after a successful strict
`--check --whitespace=error-all` test.

## Detached Build And ZFS

`/ADMIN/Github/linux-6.18.60` was copied with `cp -a --reflink=always` to
`/ADMIN/Github/linux-6.18.64`. Only the copy was cleaned and upgraded.
The existing 6.18.60 build directory remains unchanged.

Before the additional fix below, the 1,171 ZFS files remaining after
`make LLVM=1 clean` were SHA256-identical to their originals.
The 700 removed ZFS files were generated build products
and dependency files. The ZFS Kbuild, generated feature header, and hooks in
`fs/Kconfig` and `fs/Makefile` remain unchanged. No ZFS reintegration or
reconfiguration was performed.

ZFS sources remain external to the kernel Git repository as previously
requested. A Git archive alone is not the ZFS-integrated build directory.

## Additional Kernel-Only ZFS Fix

After the initial source tag, OpenZFS commit
[`48f048c0b7e831c78e714ad62ac2db2ed5c58e65`](https://github.com/openzfs/zfs/commit/48f048c0b7e831c78e714ad62ac2db2ed5c58e65)
([PR #19411](https://github.com/openzfs/zfs/pull/19411)) was applied to
`fs/zfs/icp/algs/modes/gcm.c` in the prepared 6.18.64 build directory.

The single added line records the allocated table size immediately after
successful allocation in `gcm_init_avx()`:

```c
ctx->gcm_htab_len = htab_len;
```

This gives `gcm_clear_ctx()` the correct length when wiping and freeing
the AVX and AVX2/VAES GCM tables. It changes no userspace interface or pool
format. The separate ZFS package sources and installed userspace libraries
remain unchanged; this applies the fix only to the kernel-integrated copy.

The reproducible [backport patch](zfs-backports/0001-gcm-hash-table-allocation-size.patch)
is tracked here because the integrated ZFS sources themselves remain external.
Its paths are adapted to the kernel root so it can be applied with `patch -p1`
after ZFS integration. It is already applied in `/ADMIN/Github/linux-6.18.64`.
The initial source tag is retained unchanged as the pre-fix snapshot.

The forward check before editing and reverse check afterward passed.
After path normalization, the patch ID matches the original upstream patch.
SHA256 comparison confirms that this one-line edit is the only change among
the 1,171 retained ZFS files; `.config` and `.version` remain unchanged.
No compilation or runtime testing was performed for this fix.

## Configuration And Validation Limits

`make LLVM=1 olddefconfig` retained the existing settings. The only changes
to the copied `.config` are the 6.18.64 header and removal of
`CONFIG_SATA_INIC162X`, whose driver was removed by upstream.

The full updated configuration is stored as
`arch/x86/configs/datiscum_lto_defconfig`. O3, x86-64-v3, full Clang LTO,
built-in ZFS and `CONFIG_LOCALVERSION="-datiscum1"` remain enabled.
The detached build's `.version` was reset to `0` for its first build.

Source comparisons, patch checks and configuration preparation were performed.
Only configuration helper programs were built for olddefconfig; no kernel,
kernel modules, Debian packages or runtime tests were built or run.
