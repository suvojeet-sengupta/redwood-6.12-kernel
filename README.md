# redwood-6.12-kernel

Linux **6.12** for the **Redmi Note 12 Pro 5G / POCO X5 Pro 5G** (codename
`redwood`, Snapdragon 778G, SM7325).

This is Qualcomm's own 6.12 kernel platform (the one newer Snapdragon phones
ship with) moved onto redwood, which came with a 5.4 kernel. It runs the
normal 5.4-era vendor and firmware of the phone, so most of the work here is
making the new kernel speak the way the old vendor blobs expect.

The kernel calls itself **6.12.81-Suvkernel**.

## Branches

| Branch | What |
|---|---|
| `r48` | **Use this.** Qualcomm `KERNEL.PLATFORM.5.0.r48-00400` with every redwood fix |
| `main` | The first port, on the older `r35-02100` release. Kept for history |

## What works

Display (120 Hz), touch, GPU, Wi-Fi, Bluetooth, SIMs and mobile data,
speakers, microphones, video playback, charging and battery readings,
vibration and KernelSU (as a loadable module).

Ported but not fully tested yet: camera, fingerprint, IR blaster.

Known gaps:

- Widevine stays at L3 and `feature_enabler_client` keeps restarting. The
  6.12 `smcinvoke` driver changed its ioctl layout and the old vendor
  libraries don't understand it yet.
- `memlat` (memory latency governor) doesn't load: the CPUCP firmware on
  redwood lacks the SCMI protocols it needs.
- Wi-Fi runtime power management is turned off in the ROM. With it on, a
  WCN6750 wake-up could reset the phone.

## Where things are

| Path | What | Upstream |
|---|---|---|
| `kernel_platform/common` | The kernel itself (Android `android16-6.12`) | `ks-kernel.lnx.5.0.r80-rel` |
| `kernel_platform/soc-repo` | Qualcomm SoC drivers built as modules | `kernel.lnx.6.12.r69-rel` |
| `kernel_platform/qcom/opensource/devicetree` | Device trees, redwood's in `qcom/redwood-*` | `kernel.lnx.6.12.r69-rel` |
| `vendor/display-drivers`, `vendor/mm-drivers` | Display | `display-kernel.lnx.12.13.r1-rel` |
| `vendor/graphics-kernel` | GPU (kgsl) | `gfx-kernel.lnx.16.0.r14-rel` |
| `vendor/graphics-devicetree` | GPU device tree | `gfx-devicetree-oss.lnx.16.0.r8-rel` |
| `vendor/audio-kernel` | Audio | `audio-kernel-legacy.lnx.6.12.r1-rel` |
| `vendor/camera-kernel` | Camera | `camera-iot-kernel.qclinux.1.1.r5-rel` |
| `vendor/wlan`, `vendor/dataipa`, `vendor/datarmnet*`, `vendor/securemsm-kernel` | Wi-Fi, mobile data, TrustZone | `LA.VENDOR.16.2.8.r1-06100-LAHAINA.0` |
| `vendor/redwood-modules` | redwood's own small modules | — |

redwood's hardware description is `qcom/redwood-overlay.dtso` and the
`qcom/redwood-*.dtsi` files it includes. Every change made for redwood has
a commit starting with `redwood:` or naming the driver it touches, and each
one says why in its message: `git log` is the best documentation here.

## Building

This repository has the sources only. The build system and compilers come
from Qualcomm's manifest for the same release:

```
git clone -b r48 https://github.com/suvojeet-sengupta/redwood-6.12-kernel
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
and the device tree's `clo-6.12` branch uses them.

## Credits

Qualcomm and CodeLinaro for the kernel platform, the LineageOS redwood/sm8350
kernel for driver and device tree references.

Ported and maintained by Suvojeet Sengupta.
