## Introduction ##
- A port of SUSFS v2.3.0 kernel patches for **Linux kernel 4.9** (non-GKI).
- SUSFS is an addon root hiding framework for KernelSU. The userspace tool `ksu_susfs` and any KSU module using SUSFS require a SUSFS-patched kernel to work.

# Warning #
- This is experimental code. It can harm your system or cause a performance hit. **YOU ARE !! W A R N E D !!** already.

## Compatibility ##
- Targets **Linux kernel 4.9** non-GKI devices.
- Kernel 4.9 differs significantly from GKI (5.10+). Key adaptations made in this port:
  - `fs/compat.c` hooks added (compat readdir path; GKI embeds this in `readdir.c`)
  - `fs/proc/cmdline.c` spoofed (kernel 4.9 has no bootconfig)
  - fsnotify API: `handle_event` (4.9, 8-param) vs `handle_inode_event` (5.1+, 6-param) — handled via version-gated compat wrapper
  - IDA API: `ida_get_new_above`/`ida_remove` (4.9) instead of `ida_alloc_min`/`ida_free`
  - `vfs_kern_mount` hooked instead of `vfs_create_mount` (doesn't exist in 4.9)
  - `show_smaps_rollup` not hooked (function doesn't exist in 4.9)
  - `mmap_sem` used instead of `mmap_lock` API
  - `CL_COPY_MNT_NS` flag defined in `fs/pnode.h`
  - SUS_KSTAT: kernel 4.9 has no statx, so the `STATX_SUS_KSTAT[_FUSE]` marker upstream keeps in
    `kstat->result_mask` is carried in a local inside `vfs_getattr_nosec()` instead
  - OPEN_REDIRECT: 4.9's `do_last()` resolves and opens the final component together, so the
    GKI pre-open hooks in `path_openat()`/`do_o_path()`/`do_tmpfile()` are replaced by the
    `do_filp_open()` post-open re-open (covers tmpfile/O_PATH/regular uniformly); `vfs_readlink()`
    does not exist, so its hook lives in `generic_readlink()`
  - `filename_lookup()` un-staticed and declared in `fs/internal.h` for the new `fs/open.c` and
    `fs/stat.c` su-compat hooks
  - `<linux/security.h>` added to `susfs.c` for `security_sb_statfs()` (GKI pulls it in transitively)
  - Dropped vs upstream: `include/linux/mount.h` (`susfs_mnt_id_backup` replaced by the
    `VFSMOUNT_MNT_FLAGS_KSU_UNSHARED_MNT` bit in `mnt_flags`), `fs/devpts/inode.c`
    (`ksu_handle_devpts` is not built under `CONFIG_KSU_SUSFS`), and
    `security/selinux/{hooks,selinuxfs}.c` (4.9 has no `struct selinux_state`; selinux_hide is 5.10+)

## Patch Instructions (Non-GKI, kernel 4.9) ##

**Prerequisites:**
1. Set up your kernel 4.9 source tree with a KernelSU that still builds on non-GKI.
   [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) is the recommended choice: it carries a
   `compat/` backport layer for old kernels (`ksu_access_ok`, `ksu_strncpy_from_user_nofault`,
   a `fallthrough` backport, `<5.1` selinux compat) and ships susfs integration in `main`.
   **SukiSU-Ultra's `builtin` branch no longer compiles on kernels older than 4.14.**
2. The SUSFS `Kconfig` options are expected to already be present in `$KERNEL_ROOT/KernelSU/kernel/Kconfig`. If not, add them manually (all `config KSU_SUSFS*` entries from the upstream SUSFS repo).
3. On ReSukiSU, `CONFIG_KSU_SUSFS` is itself the "KernelSU Hooking Method" choice (mutually
   exclusive with `KSU_TRACEPOINT_HOOK` / `KSU_MANUAL_HOOK`) — select it and the kernel-side
   su-compat hooks come from this patch, guarded by `CONFIG_KSU_SUSFS*`.

**Apply SUSFS patches:**
1. Copy the core files into the kernel source tree:
   ```sh
   cp kernel_patches/fs/susfs.c $KERNEL_ROOT/fs/
   cp kernel_patches/include/linux/susfs.h $KERNEL_ROOT/include/linux/
   cp kernel_patches/include/linux/susfs_def.h $KERNEL_ROOT/include/linux/
   ```
2. Apply the kernel patch:
   ```sh
   cd $KERNEL_ROOT
   patch -p1 < /path/to/susfs4ksu-kernel-4.9/kernel_patches/50_add_susfs_in_kernel-4.9.patch
   ```
   If any hunks fail, apply them manually by inspecting the patch.
3. Enable SUSFS options in your defconfig:
   ```
   CONFIG_KSU_SUSFS=y
   CONFIG_KSU_SUSFS_SUS_PATH=y
   CONFIG_KSU_SUSFS_SUS_MOUNT=y
   CONFIG_KSU_SUSFS_SUS_KSTAT=y
   CONFIG_KSU_SUSFS_SPOOF_UNAME=y
   CONFIG_KSU_SUSFS_ENABLE_LOG=y
   CONFIG_KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS=y
   CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG=y
   CONFIG_KSU_SUSFS_OPEN_REDIRECT=y
   CONFIG_KSU_SUSFS_SUS_MAP=y
   ```

## Build ksu_susfs userspace tool ##
> **⚠ Important:** The `kernel-4.9` branch of the [susfs4ksu](https://github.com/simonpunk/susfs4ksu) repository ships the **v1.5.5** userspace tool, which is **incompatible** with this v2.3.0 kernel patch. Using a mismatched userspace tool will result in broken or missing functionality.
>
> To obtain a v2.3.0-compatible `ksu_susfs` binary, use one of:
> - A **KSU manager that bundles the v2 tool**, such as [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) — the recommended option.
> - The **GKI kernel branches** of the susfs4ksu repository (e.g. `gki-android13-5.10`, `gki-android14-6.1`, etc.), which track v2 and include a compatible userspace tool. Build with `./build_ksu_susfs_tool.sh` from one of those branches.
> - Always verify the version reported by `ksu_susfs show version` matches `v2.3.0` before use.
> - Note: in v2.3.0 `add_open_redirect` takes a third `<UID_SCHEME>` argument.

- Push the compiled binary to `/data/adb/ksu/bin/ksu_susfs` so it can be called from module scripts or a root shell.

## Usage of ksu_susfs and supported features ##
- Run `ksu_susfs` in a root shell for detailed usage.
- See `$KERNEL_ROOT/KernelSU/kernel/Kconfig` for the full list of supported features after applying the patches.

**Important runtime note:** `susfs_hide_sus_mnts_for_non_su_procs` is **0 by default** in the kernel. To hide KSU bind-mounts from `/proc/[pid]/mounts` for non-root processes, call:
```sh
ksu_susfs hide_sus_mnts_for_non_su_procs 1
```
This should be called from a module `service.sh` or configured via the KSU manager.

## Building Tips ##
- To remove the `-dirty` suffix from the kernel release string, open `$KERNEL_ROOT/scripts/setlocalversion`, find all lines containing `printf '%s' -dirty`, and replace with `printf '%s' ''`.
- To hardcode the kernel release string, find the last `echo "$res"` line in `setlocalversion` and replace it, e.g.:
  ```sh
  echo "-android13-01-gb123456789012-ab12345678"
  ```
- To hardcode the kernel version string (visible in `/proc/version`), open `$KERNEL_ROOT/scripts/mkcompile_h`, find `UTS_VERSION=`, and append:
  ```sh
  LINUX_COMPILE_BY=build-user
  LINUX_COMPILE_HOST=build-host
  ```
- To spoof `/proc/config.gz` with the stock config:
  1. Pull the stock `/proc/config.gz` from the device and decompress it.
  2. Copy to `$KERNEL_ROOT/arch/arm64/configs/stock_defconfig`.
  3. In `$KERNEL_ROOT/kernel/Makefile`, change `$(obj)/config_data: $(KCONFIG_CONFIG) FORCE` to `$(obj)/config_data: arch/arm64/configs/stock_defconfig FORCE`.

## Known Issues ##
- Some file explorer apps cannot display files/directories properly when a sub-path of `/sdcard` or `/storage/emulated/0` is added to `sus_path`:
  1. Make sure the file explorer app has root access granted by the KSU manager, since `sus_path` only applies to processes without root access.
  2. It is strongly **not recommended** to add sub-paths of `/sdcard` or `/storage/emulated/0` to `sus_path`. File explorer apps typically use Android APIs that delegate file lookups to a system media provider (e.g. Google provider) running under a different UID, which SUSFS will treat as non-root and hide the paths.

## Credits ##
- KernelSU: https://github.com/tiann/KernelSU
- KernelSU fork: https://github.com/5ec1cff/KernelSU
- @Kartatz: for ideas and original commit from https://github.com/Dominium-Apum/kernel_xiaomi_chime/pull/1/commits/74f8d4ecacd343432bb8137b7e7fbe3fd9fef189
- SukiSU-Ultra: https://github.com/SukiSU-Ultra/SukiSU-Ultra

## Telegram ##
- @simonpunk

## Buy @simonpunk a coffee ##
- PayPal: kingjeffkimo@yahoo.com.tw
- BTC: bc1qgkwvsfln02463zpjf7z6tds8xnpeykggtgk4kw
