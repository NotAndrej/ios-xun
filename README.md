# ios-xun

**XNU/iOS 16 bring-up on a Xiaomi Redmi Pad SE (Snapdragon 680 "khaje"), 8 GB RAM / 256 GB eMMC.**

This repository is the `xun` adaptation of [MakrSas/ios-topaz](https://github.com/MakrSas/ios-topaz): Apple's open-source XNU on `SM6225` hardware, with the unmodified iOS 16.7.16 userland running on a tablet. It started from the topaz Redmi Note 12 4G work and is being adapted to the Redmi Pad SE's eMMC storage, 1200x1920 panel and Cortex-A73 "Gold" cluster.

> **Important:** This port is still early bring-up. **No iOS home screen has been reached on the Pad SE yet.** The kernel boots and paints the console to the splash framebuffer; the current blocker is storage, because the Pad SE stores its HFS+ root on eMMC/SDHCI, not the UFS device assumed by the topaz port. The machine's Android installation and slot A must be preserved while testing.

> **Safety first.** Use `fastboot boot` for experimentation unless replacing root storage is intentional. Keep boot slot A untouched; do not flash `boot_a`, `vbmeta_a`, `recovery_a`, `super`, or `userdata` from this project while bringing up the driver. Slot B is the natural experimentation slot.

**Thanks to MakrSas/topaz-woa**, whose SM6225 hardware documentation covers the GIC, panel, touch, power and storage details reused here.

## Current bring-up state

| Milestone | State |
| --- | --- |
| Shim (Silver -> Gold hand-off) | Verified on a connected Redmi Pad SE |
| XNU boot through loader | Verified |
| Kernel console / panic display | Live on the tablet splash framebuffer |
| Storage root discovery | **Blocked on eMMC**: `Sweet UFS root ufsroot0 unavailable` when an internal root is attempted |
| USB / touchscreen / GPU / Wi-Fi | Not adapted for the tablet yet |

Verified facts for this machine (from `adb getprop` and a 2026-10-29 device tree dump):

- SoC: `SM6225` (Snapdragon 680 "khaje")
- CPU: four Kryo 260 Silver + four Kryo 260 Gold; only Gold has the 16K granule
- Memory: 8 GB (`0x0000000040000000+0x3bb00000`, `0x0000000080000000+0xc0000000`, `0x0000000140000000+4 GiB`)
- Panel: 1200x1920 splash framebuffer at `0x5c000000`; framebuffer stride `4800`
- Storage: eMMC (`mmcblk0`, 256 GB), standard SDHCI host at `0x04744000`
- Network/Board: Wi-Fi-only tablet variant; does not contain the topaz modem/Wi-Fi path

## How it boots

```text
ABL (stock Xiaomi bootloader, unlocked)
  -> fastboot boot shim-boot.img (boot image v4, RAM only)
     -> shim on a Silver core: prints MMFR0 / MIDR, wakes Gold CPU 0x100, parks
        -> shim on Gold: GIC setup, Gold cpufreq, payload located after _end
           -> XNU Mach-O + Apple Device Tree (ADT) + boot-args
              -> XNU -> eMMC driver (in progress) -> HFS+ root from userdata
                 -> dyld + shared cache -> launchd -> backboardd + SpringBoard
```

There is no serial console. The kernel draws into ABL's framebuffer. Flashing root storage is not currently part of the happy path.

## How to build the current branch

The kernel is built on GitHub Actions macOS runners using [blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build):

```sh
gh workflow run xnu.yml -R YOU/ios-xun --ref topaz
gh run download RUN_ID -R YOU/ios-xun -n xnu-kernel -D xnu/out
```

Build the shim and boot from RAM:

```sh
MKBOOTIMG="python3 /usr/share/android-tools/mkbootimg/mkbootimg.py" bash shim/build.sh /tmp/payload-xun.bin
fastboot boot shim/out/shim-boot.img
```

To test without touching slot A or userdata, prefer `fastboot boot` only. Partition writes, if any, should target slot B tools only after the eMMC driver is working and the storage plan is understood.

## Hardware notes (xun)

- Core quartet: 4 Gold cores (0x100-0x103) are used; Silver remains parked
- GICv3 at `0xf200000` with redistributors at `0xf300000`
- Splash region: `0x5c000000`, size `0x00f00000`, 1200x1920
- eMMC host: `0x04744000`, reg size `0x00001000`
- DDR banks and console/memory note inherited from the same SoC family; 8 GB is the observed configuration here
- Battery/touch/eMMC/IP setup must still be ported from Android sources or the topaz reference

## Roadmap for xun

1. Replace polled UFS access with a polled SDHCI eMMC `emmcroot*` driver.
2. Validate read-only eMMC access on a small spare partition.
3. Add HFS+ root mounting with a safe storage plan; the 16 GiB iOS root needs substantial room, usually `userdata`.
4. Correct panel/stride presentation so XNU console text is coherent (currently corrupted).
5. Re-add device features relevant to the tablet: keys/touch, brightness, charging, and any Wi-Fi supported by the hardware.
6. Add GNSS/Bluetooth/audio only as hardware support allows; the Pad SE is Wi-Fi-first.
7. Preserve slot A and Android data until the storage decision is explicit.

## Repository layout

| Path | What |
| --- | --- |
| `shim/` | Bare-metal boot shim for the Silver -> Gold hand-off and payload loader |
| `xnu/patches/` | XNU patch series; includes topaz patches plus xun eMMC work |
| `.github/workflows/xnu.yml` | Kernel CI build |
| `tools/mkadt.py` | Apple Device Tree builder |
| `tools/mkpayload.py` | Packs kernel + ADT + boot-args |
| `tools/mkiosroot.sh`, `tools/sweet-root-tweaks.py` | Build and adjust the iOS root image |
| `NOTES.md` | Project handoff notes |
| `README.md` | This document |

## Credits

Based on [dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet) and its successors, including [MakrSas/ios-topaz](https://github.com/MakrSas/ios-topaz) and the topaz-woa hardware documentation. Apple owns iOS and its components. The XNU portions are Apple open-source under their respective licenses. Do not redistribute Apple firmware or root images.
