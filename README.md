# redwood-6.12-kernel

Linux **6.12** for the **Redmi Note 12 Pro 5G / POCO X5 Pro 5G** (codename
`redwood`, Snapdragon 778G, SM7325).

This is Qualcomm's own 6.12 kernel platform (the one newer Snapdragon phones
ship with) moved onto redwood, which came with a 5.4 kernel. It runs the
normal 5.4-era vendor and firmware of the phone, so most of the work here is
making the new kernel speak the way the old vendor blobs expect.

The kernel calls itself **6.12.96-Suvkernel**: Qualcomm's
`KERNEL.PLATFORM.5.0` with Google's `android16-6.12` (Linux 6.12.96) merged in.

## Branches

| Branch | What |
|---|---|
| `main` | **Use this.** Qualcomm `KERNEL.PLATFORM.5.0.r48-00400`, its common kernel at `ks-kernel.lnx.5.0.r81` and AOSP `android16-6.12` (6.12.96), with every redwood fix |
| `r35` | The first port, on the older `r35-02100` release. Kept for history |

## What works

Display (120 Hz), touch with double tap to wake, GPU, Wi-Fi, Bluetooth, SIMs
and mobile data, speakers, earpiece, microphones, video playback, camera,
fingerprint, IR blaster, vibration, deep sleep, charging with Xiaomi's
battery authentication, Widevine L1, USB file transfer, and KernelSU (as a
loadable module).

Known gaps:

- Fast PD charging (67 W) doesn't start yet: the charger firmware reports
  no PD contract.
- The panel stays at 120 Hz. Switching it to 90 or 60 Hz, as the 5.4 kernel
  could, floods the DSI host with FIFO errors until reboot, so dynamic fps is
  off.

## Works with any redwood ROM

Every redwood ROM, AOSP or AOSPA, ships a device tree made for the 5.4
kernel. This kernel meets those trees where they are, so a ROM only has to
swap the kernel in (see "Using it in a ROM" below):

- **Old sysfs paths.** SPMI devices keep their 5.4 names (`spmi0-02`), so
  the torch and the PMIC ADC keep their paths, permissions and labels.
- **Old tuning scripts.** A 5.4 `post_boot` writes `/proc/sys/kernel/sched_*`,
  `/sys/devices/system/cpu/cpu_boost` and picks the `schedutil` governor.
  Those paths exist again and lead to WALT, and `schedutil` is handed WALT's
  governor. redwood's tuning is also WALT's default, so it holds even if the
  script never runs.
- **No stuck boost.** WALT's boot-time boost ends on its own a minute after
  boot, and core control never pauses the big cores.
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
tells the firmware when the screen is off, and gives the `battery` thermal
zone the fuel gauge's temperature.

## Where things are

| Path | What | Upstream |
|---|---|---|
| `kernel_platform/common` | The kernel itself (Android `android16-6.12`) | `ks-kernel.lnx.5.0.r81-rel` + AOSP `android16-6.12` (6.12.96) |
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
ROM are in [redwood-6.12](https://github.com/suvojeet-sengupta/redwood-6.12).

## Using it in a ROM

Don't build the kernel inside the ROM. The ready-made kernel, modules and
`dtbo.img` are in
[device_xiaomi_redwood-kernel-6.12](https://github.com/suvojeet-sengupta/device_xiaomi_redwood-kernel-6.12),
whose README lists the few lines a device tree needs. The redwood device
tree's `clo-6.12` branch already uses it.

ROMs that build with Dolby set `debug.c2.use_dmabufheaps=1`; on this kernel
that leaves the 32-bit video codecs without their ION buffers, so set it
back to `0` in the device tree.

## Credits

Qualcomm and CodeLinaro for the kernel platform, the LineageOS sm8350 kernel
for the Xiaomi charger driver and for driver and device tree references.

Ported and maintained by Suvojeet Sengupta.
