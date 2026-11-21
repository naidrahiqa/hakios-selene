# MTK loader audit — 20–21 November 2026

Scope: the bare-metal `mtkloader/` trees that LK starts before XNU:
`hakios-selene/mtkloader/` (device repo) and `hakios-xnu-kernel-selene/mtkloader/`
(common tree, worktree of branch `mtk-selene`). The common tree carries the same
loader as `origin/mtk-selene`; `origin/mtk` is the older X6826B variant.

## Fixed (commits pushed to origin)

| commit | repo / branch | what |
| --- | --- | --- |
| `480308f` | hakios-selene / main | device identity is the Redmi 10 (`selene`, MT6769H) family across `AGENTS.md`, `README*`, `kernel/`, `userland/`, `docs/` and the loader; loader: payload-walk bounds, segment-fit checks, `uart_putc`/`uart_putc_options` → `loader_log_putc`, dead `connsys_dt_report` and its helpers removed; rebuilt `mtkloader.bin` |
| `60d813b` | hakios-xnu-kernel-selene / mtk-selene | define `platform_renoi` — the RENOIR platform branches referenced it but nothing defined it, so `xnuloader/build.sh renoir` failed to link |
| `5545d80` | hakios-selene / main | `rescue_log`: `ring_at == 0` on an unwrapped log is not a missing symbol; `jump_to_kernel`: write back dirty data cache before XNU brings up its own MMU |
| `750bcc5` | hakios-xnu-kernel-selene / mtk-selene | same two loader fixes, ported |

Housekeeping: generated Python bytecode caches (`__pycache__/`) and build objects
were removed from the working trees; both `.gitignore` files already exclude them.
No generated binary, firmware, IPSW extract or run log was deleted.

## Verified offline

* Rebuild reproducibility: the committed `mtkloader.bin` matched a fresh
  `mtkloader/build.sh` build in both trees (sha256 confirmed at the time).
* Both trees currently build with **zero warnings**; `xnuloader/loader.c`
  compiles for `-DXNU_PHONE_RENOIR` (no undefined platform).
* `tools/chainboot.py` packs the current loader + Android `boot_a.img` +
  `build-payload.bin` into a `48,398,336`-byte image (`boot_a` is 64 MiB).
* `entry_va = ((uint64_t *)(lc + 16))[32]` in `run_xnu()` matches
  `xnuloader/loader.c` (Apple thread state: x0..x28, fp, lr, sp, **pc** at [32]).
* Fixed memory addresses checked against `fw-selene/iomem.txt` (phone dump):
  `PANIC_LOG_BASE 0x80100000`, `MARKER_PA 0x80000000`, `TOUCH_STAGE 0x50000000`,
  `CHAIN_STUB_PA 0x4a000000` all land in free System RAM, outside reserved ranges;
  ramoops `0x4d010000–0x4d0effff` covers the loader console `0x4d09f000` and
  pmsg `0x4d0e0000`; the XNU window `0xc0000000–0xfe000000` sits inside
  `c0000000-1bfff5fff : System RAM` and does not overlap `feff6000-feffefff`.

## Open items (need an iPhone X DT / a phone run / an owner decision)

* Design constants in `build_device_tree()` that are not in `identity.h`:
  `chip-id 0x8015`, `board-id 0x06`, `clock-frequency 1800000000`,
  `firmware-version iBoot-8422.100.650`, `serial=3`, `darwin-board`.
* `KERNEL_VIRT_BASE` (`0xfffffff004000000`) vs the TCR constant
  (`0x22a51a251a`, decoded by `xnu_va_to_pa` as 4K/38-bit): confirm against
  Apple's iPhone X values before trusting the rescue/backtrace readers.
* `watcher_l1` L1 block descriptor bits and `dcache_clean_all`'s CLIDR decode
  against the ARM ARM; GICR offsets used by `run_xnu`/`watcher_main` are
  self-consistent but unverified against the hardware.
* M3: the `mtk-selene` watcher now **holds the panel** after a snapshot instead
  of resetting into Android — intentional in code, but needs owner sign-off.

## Safety

Only `boot_a` and `xnusys` may be written; always pass `-s 65cb03c00702`.
Never run `adb`/`fastboot`/USB commands against the OPPO hotspot device.

## Flash image source and backup (21 November 2026)

The boot image to flash was packed from the owner's own **ColorOS 16 boot backup**,
`ColorOS16_Selene_20261119.boot.img` (dated 2026-11-19, kept outside this repo in the
ColorOS project tree: `$HOME/Projects/ColorOS/COS16_Selene/`). It is a 64 MiB Android
boot image, header v2, page 2048, kernel 11,624,783 B, ramdisk 34,568,566 B — the
`chainboot.py` v2 path applies unchanged.

Packed output (workspace root, not tracked):
`$HOME/Projects/IOS/xnu-selene-boot_a-ColorOS16-20261119.img` — **52,207,616 bytes**
(program 35,936, Android Image 30,052,191, XNU payload 19,870,480, gz kernel
17,494,934), inside the 64 MiB `boot_a` limit with ~11.9 MiB headroom.

Flash (only `boot_a`, explicit serial, no `fastboot boot` support on this phone):
```sh
fastboot -s 65cb03c00702 flash boot_a \
  $HOME/Projects/IOS/xnu-selene-boot_a-ColorOS16-20261119.img
fastboot -s 65cb03c00702 reboot
```

Restore the owner's ColorOS 16 boot image (reversible):
```sh
fastboot -s 65cb03c00702 flash boot_a \
  $HOME/Projects/ColorOS/COS16_Selene/ColorOS16_Selene_20261119.boot.img
fastboot -s 65cb03c00702 reboot
```

Do **not** run `fastboot flashing unlock` (wipes `userdata`), `-w`, `erase`, or flash
anything except `boot_a`. Check `fastboot -s 65cb03c00702 getvar current-slot` first:
if the active slot is B, `boot_a` is not used until the slot changes.


## Kernel rebuild and flash-root progress (21 November 2026, evening)

- Installed the full XNU toolchain (TheOS cctools ld64-609, iPhoneOS16.5 SDK via theos/sdks)
  and ran `xnu-deps/setup.sh --stage1` and `--stage2` successfully.
- `xnu/apply.sh` fetched xnu-8796.121.2 and applied `redwood.patch` (77 changed, 126 new files).
- `./xnu-build.sh -j8 -k` built a NEW kernel (`mach.development.bcm2837`, 15,082,000 B).
  Unlike the old `prebuilt/mach.development.bcm2837`, it contains the `emmc_root` hook:
  strings `emmc: xnusys`, `xnusys partition could not`, `redwood_emmc_root` are present.
  The old prebuilt had none of them (backed up to `prebuilt/mach.development.bcm2837.orig`).
- Boot image `xnu-selene-boot_a-NEWKERNEL2.img` (51,992,576 B) with the new kernel, LITTLE1
  loader (cpus=1, pmsg log, warm reset) was flashed and run. Log
  `hakios-xnu-kernel-selene/logs/newkernel2-20261121-1818.log` (65 KB pmsg) confirms:
  `emmc: xnusys at block 235765760, 2361 MB of 4096 MB in its APFS containers`,
  `emmc: 4 MB in 256 ms`, `BSD root: emmc0, major 2, minor 0` — the phone's xnusys
  partition became the root for the first time.
- `FLASHBOOT=1 bash userland/build.sh` built `flashboot.apfs` (975 MB, 3 APFS containers,
  dyld cache present) — too large for the chainboot payload; the documented phone path is
  `emmc_root=1` with the small root.apfs as md1 (docs/x6826b-emmc.md).
- Remaining gap: with `emmc_root=1` alone, `init` does not call `flash_root()`, so the OS
  cryptex is not mounted (`/System/Cryptexes/OS` missing) and dyld cannot load
  `/usr/lib/libSystem.B.dylib`. Adding `root_apfs=` would take the flash-root path, but
  that path expects a `flashboot` payload and a device node not verified on this phone.
  Do not combine the two paths without a phone run to confirm.
