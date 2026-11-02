# redwood-6.12-kernel

Linux **6.12** for the **Redmi Note 12 Pro 5G / POCO X5 Pro 5G** (codename
`redwood`, Snapdragon 778G, SM7325).

This is Qualcomm's own 6.12 kernel platform (the one newer Snapdragon phones
ship with) moved onto redwood, which came with a 5.4 kernel. It runs the
normal 5.4-era vendor and firmware of the phone, so most of the work here is
making the new kernel speak the way the old vendor blobs expect.

The current release is **Suvkernel 3.0** (`6.12.97-Suvkernel-3.0`):
Qualcomm's `KERNEL.PLATFORM.5.0` with Google's `android16-6.12` (Linux
6.12.97) merged in.

## Branches and tags

| Ref | What |
|---|---|
| `main` | **Use this.** Qualcomm `KERNEL.PLATFORM.5.0.r48-00400`, its common kernel at `ks-kernel.lnx.5.0.r81` and AOSP `android16-6.12` (6.12.97), with every redwood fix |
| `v3.0` | Suvkernel 3.0 |
| `v2.0` | Suvkernel 2.0 |
| `r35` | The first port, on the older `r35-02100` release. Kept for history |
| `archive/pre-2.0` | `main` as it was before its history was tidied for 2.0. The prebuilt's older commits name kernel commits from it |

## What works

Display at 120 and 60 Hz, touch with double tap to wake (Goodix and
FocalTech panels), GPU, Wi-Fi, Bluetooth and its audio, SIMs, mobile data
including 5G, speakers, earpiece, microphones, video playback, camera,
fingerprint, IR blaster, vibration, deep sleep, PD fast charging with
Xiaomi's battery and adapter authentication, Widevine L1, USB file transfer,
and KernelSU (as a loadable module).

Known issues:

- Hardware video encoding fails for some encoder settings (HFI error
  0x1008); decoding is not affected.

## What's new in 3.0

- **Sleep.** The phone no longer goes dead after its first suspend
  following a boot. 6.12.97 brought a PM change made for newer kernels
  (`device_pm_sleep_init()` signalling its completion only once) that
  could leave a suspend waiting forever on 6.12; it is reverted, as the
  stable trees are doing.
- **Crashes.** A kernel oops no longer leaves a black screen that ignores
  the power key: a panic reboots the phone after five seconds. A NULL
  dereference in `finish_task_switch()`, a 6.12.95 regression in the block
  layer's plug timestamp, is fixed.
- **Heat and stutter.** `mi_thermald` capped the big cores down to
  940 MHz as soon as its skin estimate passed 35 C, often just from
  charging, and the UI stuttered. Its caps now stop at 1516.8 MHz on gold
  and 1862.4 MHz on prime (`mi_thermal`'s `gold_limit_floor` and
  `prime_limit_floor`, 0 for Xiaomi's own); the SoC's thermal limits and
  the charging limits are unchanged.
- **Upstream fixes.** AOSP `android16-6.12` proxy execution and
  blk-crypto fixes, Qualcomm display and kgsl (a6xx) fixes, and a bound on
  the battery driver's `verify_digest` copy.

## What's new in 2.0

- **Charging.** The ADSP's Type-C stack only reports the port once the host
  enables pin assignment notifications; that was waiting for a DisplayPort
  client redwood doesn't have. Chargers are now seen as PD PPS and fast
  charge.
- **Display.** 60/120 Hz switching is back, without the FIFO errors: the
  DSC link is sized for RGB888 and the rate's gamma bank is sent in HS mode
  inside the blanking, so a switch no longer blanks a frame.
- **Mobile data.** The downlink reaches the stack in pages instead of being
  copied (5G SA was held around 30 Mbps), TX completes in NAPI, and IPA and
  RmNet carry a set of Qualcomm fixes.
- **Battery and heat.** RT tasks no longer start at the top frequency, so
  the big cores stay low with a static screen; Xiaomi's thermal zones are
  all there for `mi_thermald`.
- **Sleep and wake.** Interrupts that fire while suspended are replayed
  instead of lost, plus pinctrl, RPMh, gdsc, UFS and tsens fixes.
- **Audio, Wi-Fi, video, Bluetooth.** Fixes from genie1997's redwood 6.12
  tree, including a use-after-free in the LSM (hotword) driver.

## Works with any redwood ROM

Every redwood ROM, AOSP or AOSPA, ships a device tree made for the 5.4
kernel. This kernel meets those trees where they are, so a ROM only has to
swap the kernel in (see "Using it in a ROM" below):

- **Old sysfs paths.** SPMI devices keep their 5.4 names (`spmi0-02`), so
  the torch and the PMIC ADC keep their paths, permissions and labels.
- **Scheduling.** The kernel schedules with WALT, Qualcomm's own load
  tracking, which the vendor perf HAL's boosts are made for. A 5.4
  `post_boot` finds its WALT tunables in `/proc/sys/kernel` and its
  `cpu_boost` nodes, `schedutil` is handed WALT's governor, core control
  never pauses the big cores, and the boot boost ends on its own.
  `sched_walt.disable=1` on the command line switches to PELT, EAS and
  `schedutil` instead.
- **Wi-Fi.** Runtime power management stays off whatever the ROM's
  `WCNSS_qcom_cfg.ini` says; with it on, a WCN6750 wake-up could reset the
  phone.
- **Video.** The video driver labels its `sku_version` node itself, so the
  media script picks redwood's media profiles before SELinux has caught up.
- **TrustZone.** `smcinvoke` also speaks the 5.4 ioctl layout the old
  vendor libraries use, and the real `qseecom` driver serves QTEE's listener
  requests, so Widevine runs at L1 and the feature enabler loads.
- **Memory buses.** `memlat` samples in the kernel instead of waiting for
  SCMI, which redwood's CPUCP firmware lacks, and the DDR, LLCC and L3
  boot boost ends a minute after boot even though the 5.4 `post_boot`
  doesn't know those nodes.
- **USB.** A USB-A to USB-C cable counts as connected even though the
  ADSP reports no Type-C partner for it, so USB preferences and file
  transfer work.
- **No stuck CPU pause.** The PMIC battery-current trips clear again once
  the voltage recovers; a single sag under load used to leave the big cores
  paused until reboot.
- **No command line changes.** Unused clocks and power domains stay on by
  default, which keeps the boot splash alive until the display takes over.

## Charger

redwood's charger firmware is Xiaomi's, so `qti_battery_charger` is Xiaomi's
driver from the LineageOS sm8350 kernel, ported onto Qualcomm's 6.12 one. It
speaks Xiaomi's property layout, has all of Xiaomi's `qcom-battery` nodes
(battery and adapter authentication, fast charge mode, fuel gauge details),
tells the firmware when the screen is off, sends the thermal level and the
battery authentication again after the charger firmware restarts, and gives
the `battery` thermal zone the fuel gauge's temperature.

## Where things are

| Path | What | Upstream |
|---|---|---|
| `kernel_platform/common` | The kernel itself (Android `android16-6.12`) | `ks-kernel.lnx.5.0.r81-rel` + AOSP `android16-6.12` (6.12.97) |
| `kernel_platform/soc-repo` | Qualcomm SoC drivers built as modules | `kernel.lnx.6.12.r69-rel` |
| `kernel_platform/qcom/opensource/devicetree` | Device trees, redwood's in `qcom/redwood-*` | `kernel.lnx.6.12.r69-rel` |
| `vendor/display-drivers`, `vendor/mm-drivers` | Display | `display-kernel.lnx.12.13.r1-rel` |
| `vendor/graphics-kernel` | GPU (kgsl) | `gfx-kernel.lnx.16.0.r14-rel` |
| `vendor/graphics-devicetree` | GPU device tree | `gfx-devicetree-oss.lnx.16.0.r8-rel` |
| `vendor/audio-kernel` | Audio | `audio-kernel-legacy.lnx.6.12.r1-rel` |
| `vendor/camera-kernel` | Camera | `camera-iot-kernel.qclinux.1.1.r5-rel` |
| `vendor/wlan`, `vendor/dataipa`, `vendor/datarmnet*`, `vendor/securemsm-kernel` | Wi-Fi, mobile data, TrustZone | `LA.VENDOR.16.2.8.r1-06100-LAHAINA.0` |
| `vendor/redwood-modules` | redwood's own small modules | — |

`soc-repo` and the device trees stay on `r69`: Qualcomm's `r70` is another
release line that lacks much of `r69`. Qualcomm releases come in as plain
snapshot commits that the redwood branch then merges, and `android16-6.12`
is merged into the common kernel the same way.

redwood's hardware description is `qcom/redwood-overlay.dtso` and the
`qcom/redwood-*.dtsi` files it includes. Every change made for redwood has
a commit starting with `redwood:` or naming the driver it touches, and each
one says why in its message: `git log` is the best documentation here.

## Building

This repository has the sources only. The build system and compilers come
from Qualcomm's manifest for the same release:

```
git clone https://github.com/suvojeet-sengupta/redwood-6.12-kernel
mkdir kp && cd kp
repo init -u https://git.codelinaro.org/clo/la/kernelplatform/manifest.git \
	-b release -m KERNEL.PLATFORM.5.0.r48-00400-kernel.0.xml --depth=1
repo sync -c -j8 --no-tags --no-clone-bundle
cd kernel_platform
for x in bootable build build_with_bazel.py common-modules device.bazelrc \
		external MODULE.bazel prebuilts system tools WORKSPACE \
		WORKSPACE.bzlmod .source_date_epoch_dir; do
	mv $x ../../redwood-6.12-kernel/kernel_platform/
	echo kernel_platform/$x >> ../../redwood-6.12-kernel/.git/info/exclude
done
```

Keep `common`, `qcom` and `soc-repo` from this repository. The scripts that
build everything (kernel, vendor modules, `dtbo.img`) and pack it for the
ROM or as a flashable zip are in
[redwood-6.12](https://github.com/suvojeet-sengupta/redwood-6.12).

## Installing

- **Flashable zip.** Each release has an AnyKernel3 zip
  (`Suvkernel-<version>-redwood-*.zip`) that flashes from recovery. It replaces
  the kernel, the modules in `vendor_boot` and `dtbo`, and keeps the boot
  ramdisk, so KernelSU or Magisk stay installed.
- **In a ROM.** Don't build the kernel inside the ROM. The ready-made
  kernel, modules and `dtbo.img` are in
  [device_xiaomi_redwood-kernel-6.12](https://github.com/suvojeet-sengupta/device_xiaomi_redwood-kernel-6.12),
  whose README lists the few lines a device tree needs. The redwood device
  tree's `clo-6.12` branch already uses it.

ROMs that build with Dolby set `debug.c2.use_dmabufheaps=1`; on this kernel
that leaves the 32-bit video codecs without their ION buffers, so set it
back to `0` in the device tree.

## Credits

- Qualcomm and CodeLinaro for the kernel platform, and Google for the
  Android common kernel.
- genie1997 for the redwood 6.12 tree whose fixes many commits here come
  from, kept under their name.
- The upstream authors whose fixes were taken from Linux stable and
  Qualcomm's releases, also kept under their names.
- The LineageOS sm8350 kernel for the Xiaomi charger driver and for driver
  and device tree references.

Ported and maintained by Suvojeet Sengupta.

## License

The kernel is GPL-2.0, see `kernel_platform/common/COPYING`. Other parts
carry their own license headers.
