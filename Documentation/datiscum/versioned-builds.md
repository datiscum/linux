# Versioned Kernel Sources

The two versions are retained independently. Updating the 6.18.60 branch
does not change the 6.18.55 branch or either detached build directory.
The earlier datiscum branches and their original patch history are retained.

| Kernel | Branch | Tag | Status |
| --- | --- | --- | --- |
| 6.18.55-datiscum1 | `linux-6.18.55-datiscum` | `v6.18.55-datiscum1` | Preserved sources of the completed Debian build |
| 6.18.60 | `linux-6.18.60-datiscum` | `v6.18.60-datiscum-source` | Prepared sources; not built during this synchronization |

## Source Verification

The branches were compared byte-for-byte with the Linux sources in
`/ADMIN/Github/linux-6.18.55` and `/ADMIN/Github/linux-6.18.60`, respectively.
Build products, packaging output, and generated kernel headers are excluded.
The existing, newer Git documentation was retained instead of restoring
older README copies from the build directories.

The scheduler guards and diagnostics, scheduler fast path, preemptible IPI
backport, NVMe/io_queue isolation, Xe changes, and the local GUD bounds fix
are retained. Official v6.18.55 and v6.18.60 were merged with their stable
commit history, not imported as anonymous source snapshots.

As previously requested, integrated OpenZFS sources are not imported into
this kernel repository. Both detached build directories retain their
OpenZFS 2.4.4 integration unchanged: `fs/zfs`, `include/zfs`, and its two
hooks in `fs/Kconfig` and `fs/Makefile`. Those integration hooks are the only
intentional differences in the tracked Linux code versus the build folders.
A Git archive alone is therefore not the complete ZFS-integrated build tree.

## Exact LTO Configuration

`arch/x86/configs/datiscum_lto_defconfig` contains the exact `.config` used
for the completed 6.18.55 build. It also matches the unchanged configuration
in the prepared 6.18.60 directory and the configuration embedded in the
6.18.55 Debian image package.

SHA256:

```text
ebcfe4ce71881f259626816367aa4532d8b625c8b8d4988e927040072ac68590
```

Settings include O3, x86-64-v3, full Clang LTO, and built-in ZFS. The stored
configuration is not rewritten by oldconfig during this synchronization;
it still requires the external ZFS integration when used for that build.
The conventional `datiscum_defconfig` is left unchanged.

## Completed 6.18.55 Build

Compiler and linker: Debian Clang 19.1.7 and LLD 19.1.7.
Debian package version: `6.18.55-1`.
Kernel release: `6.18.55-datiscum1`.

Image package and SHA256:

```text
linux-image-6.18.55-datiscum1_6.18.55-1_amd64.deb
d150795b701218b145279bad52cbe6dceb01efd9cb4258769af727cc0b1d7c79
```

The GUD module's DWARF source checksum also matches the preserved
`drivers/gpu/drm/gud/gud_connector.c`, including the local Fortify bounds fix.
These checks identify the preserved build; they do not claim binary
reproducibility or a completed 6.18.60 build.

Neither detached build directory was modified or compiled during this
Git reconstruction and publication.
