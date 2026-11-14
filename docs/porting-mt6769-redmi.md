# Bringing this up on the Redmi 10 family (selene / selenes / eos, MT6769)

selene (Redmi 10, Redmi 10 Prime), selenes (Redmi 10 2022, Redmi Note 11 4G, Redmi 10 Prime 2022)
and eos are one board family: Helio G88, which MediaTek's kernel builds as MT6769, in the same tree
as MT6768 -- Xiaomi's own `selene` kernel has `arch/arm64/boot/dts/mediatek/mt6768.dts`. So the SoC
blocks this repository drives (GIC, generic timer, MSDC0, MFG/Mali, PWRAP, SCP, CONNSYS/BTIF,
APMIXEDSYS) are the same registers. What is not shared is everything a board wires around them,
and the boot chain.

eos is not a product alias of selene: it is **selene's China variant, and it carries NFC**
(`mediatek,nfc` on i2c3 at 0x1100f000, address 0x08). The merged device tree of a global unit
carries that node too, with `status = okay`, so the tree does not tell the two apart -- the fitted
part does. eos therefore gets its own entry and its own per-board facts, not a line under selene.

This is the map; it is not a description of a working port. Nothing here has run on a Redmi.

## What is the phone in this code, and where

Every one of these is a fact about the X6826B written into a file. Each has to be re-derived per
board (from the phone's own `/proc/device-tree`, its `dtbo_a`, and Xiaomi's GPL kernel as a
reference for register values only).

| fact | where it is now | what a Redmi changes |
| --- | --- | --- |
| free DRAM window and its size | `mtkloader/mtkloader.c` `XNU_PHYS_BASE` 0xc0000000, `XNU_MEM_SIZE` 0xfe000000 | 4 GB vs 6 GB phones, and the window LK/TEE/ATF leave free, differ |
| panel size, LK framebuffer, OVL layer | `mtkloader/mtkdbg.h` `PANEL_W/H` 720x1640, `LK_FB_BASE` 0x787b0000, pitch 2944, `OVL0_BASE` 0x1400b000 | 1080x2400 on all three; framebuffer address and pitch come from LK and must be read, not assumed (the loader already prefers the live OVL registers, the constants are only the fallback) |
| display scale in XNU's device tree | `build_device_tree()`, `display-scale` 2, `display-rotation` 0 | 1080x2400 at 2x is a different surface budget |
| processors offered, core order | `core_order[8]`, A55 0-5 / A75 6-7, big cores 1176 MHz | same topology; G88's A75 runs 2000 MHz, so `big_cores_clock()`'s ARMPLL target and the OPP table are different |
| touch controller and its transport | `mtkloader/tcm.h` + `mtkloader.c` `touch_bring_up()`: OmniVision TD4160, SPI2, GPIO25-28, firmware sent by the loader | the Redmi's touch IC (Novatek / FocalTech / Goodix / ...) is a different protocol and firmware; if it is not a TCM part, `tcm.h` and `redwood_tcm.c` do not transfer at all |
| buttons | `kernel/bsd/dev/redwood_mtkkeys.c`: keypad at 0x10010000, power through an **MT6358** PMIC TOPSTATUS bit | the G88 boards use a different PMIC (MT6359 family); both the keypad map and the PMIC register/bit must be read from the board's DTS |
| eMMC | `msdc.h`, `redwood_emmc.c`: MSDC0 0x11230000, PIO, CMD17/18 | same controller; the partition named `xnusys` does not exist yet and has to be carved |
| GPU | `redwood_mali.c` MFG 0x13000000 / MALI 0x13040000 | same block, same addresses expected; still verify against the board DTS |
| SCP / sensors | `redwood_scp.c` SCP_CFG 0x105C0000, SRAM 0x10500000 | same blocks; SCP firmware and the sensor on its I2C bus differ |
| CONNSYS / Wi-Fi / Bluetooth | `redwood_connsys.c` PWRAP 0x1000d000, BTIF 0x1100c000, and the PMIC rails it switches on | same MT6631-class CONNSYS expected; the rails, the WMT config and the per-unit calibration are Xiaomi's |
| camera | `redwood_mtkcam.c`: S5K JN1 on I2C4, CAMSV_1, GPIO rails | the 50 MP main camera may be the same sensor; the pins, rails and CAMSV instance are per board |
| boot image layout | `tools/chainboot.py`: header v2 only, `CHAIN_OFFSET` 0x40000, boot partition fixed at 32 MB | Redmi 10 shipped Android 11; if its `boot.img` is header v3/v4 the chain has to be rebuilt around `vendor_boot`; the partition size is whatever `getvar partition-size:boot_a` says |
| system image partition | `tools/gpt-carve.py`: 4 GiB off the end of `userdata` | Xiaomi's dynamic partitions (`super`) and `userdata` layout are different; the carve must be recomputed from the phone's GPT |

## selene, as measured on a phone (21061119AG, 14 November 2026)

Every row of the table above was a guess. These are the phone's own numbers, read from its merged
DTB (`/sys/firmware/fdt`), `/proc/partitions`, `/proc/iomem`, `boot_a` and `/proc/bus/input/devices`.
The files are in `fw-selene/` (git ignores `fw-*/`); none of them goes into the tree.

| fact | X6826B | selene |
| --- | --- | --- |
| SoC | MT6768 | **MT6769H**, root `compatible` still `mediatek,MT6768` |
| RAM | 4 GB given to the kernel, base 0xc0000000 | 6 GiB: `memory` = 0x40000000 + 0x180000000; free above 0xc0000000 to 0x1bfff5fff, with 0xfeff6000-0xffffffff reserved inside it |
| panel | 720x1640, fb 0x787b0000 | **1080x2400** (0x438 x 0x960), NT36672C `fhdp` VDO, no DSC in the fallback name; LK's framebuffer **0x7e605000**, vram 0x17e8000 (24 MB), 60 Hz |
| touch | OmniVision TD4160 on **SPI2** 0x11012000, GPIO25-28, firmware sent by the loader | **Novatek NT36672C on SPI0 0x1100a000** (`mediatek,mt6765-spi`), cs 0, 9.6 MHz, reset **GPIO 92**, irq **GPIO 1** (active low); SPI pins **25-28, function 2** -- the same pins, a different controller (`/proc/bus/input/devices`: `NVTCapacitiveTouchScreen`) |
| PMIC | MT6358 | **MT6358** at PWRAP 0x1000d000 -- the same part, so `redwood_mtkkeys.c`'s power-key bit is right |
| keypad | 0x10010000 | `10010000.kp`, `mtk-kpd` -- the same |
| boot image | header v2, partition 32 MB | **header v2**, page 2048, kernel 12.4 MB at 0x40080000, ramdisk 30 MB at 0x47c80000 (tags 0x4bc80000), partition **64 MiB**; no `vendor_boot` partition exists, so `tools/chainboot.py` needs no change of format |
| partitions | `xnusys` carved from `userdata` | A/B, `super` 8.5 GiB, `userdata` 104 GiB, `scp_a` 6 MiB, `lk_a` 2 MiB, `dtbo_a` 8 MiB; no `xnusys` |
| CONNSYS | MT6631 | MT6631: `WIFI_RAM_CODE_soc1_0_1a_1.bin`, `WMT_SOC.cfg`, `BT_FW.cfg`, `mt6631_fm_v1_*.bin` in `/vendor/firmware` |
| fingerprint | -- | Goodix (`uinput-goodix`) / FPC, on the same 0x10005000 pin controller |
| NFC | -- | `mediatek,nfc` on i2c3 0x1100f000, address 0x08, plus an `irq_nfc` eint node: present in the merged tree of a global unit with `status = okay`, so the tree does not tell eos from selene -- the fitted chip does. **eos' own device**, and its own driver, is this row |

What this changes in the plan: the touch driver is a new one either way (Novatek SPI, not TCM), but
the loader's SPI pin setup and `mtkspi.h` transfer -- only the controller base changes. The PMIC is
the one already driven, so buttons come almost for free. The boot chain is the one already written.

## What transfers unchanged

`start.S`, `loop.S`, the chain-and-return protocol (`mtkloader.c` markers, ramoops save, pstore
read-back), the payload/Mach-O loader and symbol patcher, the device-tree writer and identity
properties, `msdc.h`'s read path, `mtkspi.h`'s SPI core (pins excepted), `mtkpwrap.h`, the Mali
bring-up, the WMT/STP state machine, the XNU-side block device, WPA supplicant, and the whole
`tools/` run loop. The kernel drivers' structure (boot-arg gated, numbered steps, read-only
first) carries over as written in `AGENTS.md`.

## Order of work

1. **Host only, no phone**, on `mtk-selene`: a board table in the loader holding selene beside
   the X6826B (panel, framebuffer, touch pins and controller, memory window, taken from the
   measured table above) and the `xl`/`mkxnuimg` pair taking a board name, the way `xl_x6826b`
   and `mkxnuimg_x6826b.py` do today; `chainboot.py`'s boot partition size taken from the device
   rather than the 32 MB constant. The header needs no change: selene's `boot.img` is v2. eos is
   the same work plus its NFC part, on `mtk-eos`.
2. **Loader diagnostics only, no XNU**: chain in front of Android's kernel, print the live OVL0
   layer registers, the panel's DSI mode, the PMIC's ID and the keypad map, then hand back to
   Android. The table above says what these should be; the run proves the loader reads them.
3. **XNU, one core, display, buttons**: memory window, `PANEL_W/H`, framebuffer, GIC/timer
   patches, the keypad/PMIC driver.
4. **Storage**: carve `xnusys`, read the image off MSDC0, `emmc_root=1`.
5. **GPU**, then **touch** (new IC driver + firmware path), then **CONNSYS** (Wi-Fi, Bluetooth),
   then **SCP**, **camera**, **audio**, **battery**, **haptics** -- in that order, because each
   one's log is the next one's starting point.

## What has to be collected first

Collected for selene on 14 November 2026, into `fw-selene/`: the merged DTB and `/proc/device-tree`,
`dmesg` and the previous boot's console from pstore, `/proc/partitions`, `/proc/iomem`, the
`by-name` map, `boot_a` (the running image, which already carries the root that `boot-xnu.sh`
needs), `dtbo_a`, `lk_a`, `scp_a`, and the touch/CONNSYS firmware. Still to come, per board:

- the same set from a **selenes** unit (Redmi 10 2022 / Note 11 4G) -- its panel and touch may be a
  different combination, which is the one thing that is not shared by construction;
- a **GPT backup** of each phone, needed only when `xnusys` is carved (the sizes above come from
  `/proc/partitions`, which was enough to know there is room);
- the phones themselves, one at a time, unlocked and rooted: every step above is proved by a run,
  and a run needs the hardware.

Xiaomi's kernel source (MiCode's `selene-r-oss`, or the `selene-devs` 4.14 tree) is a reference
for register addresses and hardware order, as `AGENTS.md` allows; none of it, and none of the
files above, goes into git.

## Where the MTK code actually is (checked 14 November 2026)

The common tree has two branches that do not meet, and the port has to pick one:

| | `main` | `mtk` (a962798, 10 November) |
| --- | --- | --- |
| layout | `devices/<name>/` + `socs/<name>/` + `drivers/<subsystem>/`, `devices/manifest.conf`, `tools/hakios-device.py`, per-device `defconfig` layering | one tree, the phone baked in: `mtkloader/`, `tools/`, `xnu/files/bsd/dev/redwood_*.c`, `xl_x6826b`, `mkxnuimg_x6826b.py` |
| MTK support | none: no `mtk_emmc`, no MTK loader, `drivers/` is UEFI-era (UFS, Adreno, QMI) | everything this repository has, as the XNU patch and the loader |
| a second MTK phone | what the tree is built for | 52 files name `x6826b`, and the per-device scripts are top-level files |

The device repository (`hakios-selene`, a copy of `hakios-mt6768`) holds the drivers as
`kernel/bsd/dev/redwood_*.c`, i.e. the same files `mtk` keeps under `xnu/files/`. So: bring selene
up on `mtk`, with the board facts moved out of `mtkloader.c`/`mtkdbg.h` into one board table that
`xl`/`mkxnuimg` take a name for; porting the result forward onto `main`'s layering is mechanical
once it boots, and doing it the other way round means carrying the whole MTK bring-up (eMMC root,
Mali, PWRAP, CONNSYS, camera) into a tree that has none of it.

## The branches

One branch per board that is **actually a different board**, off `mtk` -- not one per product name:

| branch | for | what it adds over `mtk` |
| --- | --- | --- |
| `mtk` | the Infinix X6826B, and everything shared | -- |
| `mtk-selene` | selene **and** selenes: Redmi 10, 10 Prime, Redmi 10 2022, Note 11 4G, 10 Prime 2022 | the measured board facts above. One branch, because they are one board: same SoC, same panel family, same Novatek part, same partitions. A unit that turns out to differ gets its own branch **then**, with the measurement that shows it |
| `mtk-eos` | selene for China | selene's, plus **NFC** (`mediatek,nfc`, i2c3 0x1100f000 @0x08) and its driver -- the one thing that genuinely separates it |

A fix that belongs to every MTK phone goes on `mtk` and is merged down, not fixed three times. A
worktree for the branch being worked on: `git worktree add ../hakios-xnu-kernel-selene mtk-selene`.

### `main` does not distinguish them either

Checked on 14 November: `devices/manifest.conf` on `main` names only `redwood` (a Qualcomm SM7325
phone, kept in its own repository) and `qemu`; `socs/` holds nothing but its README. There is no
selene, no selenes, no eos, and no MTK SoC layer at all -- `main` cannot tell these phones apart
because it does not know about them. The distinction exists only where the MTK code is, on `mtk`
and its per-board branches.

Putting them into `main`'s layering (`devices/<name>/device.conf`, `soc/configs/mt6769.config`,
`tools/hakios-device.py new`) is worth doing, and it is what makes the difference visible to the
tree's own tools -- but on `main` those files would describe phones nothing in that tree can boot
yet. It follows the port, it does not start it.
