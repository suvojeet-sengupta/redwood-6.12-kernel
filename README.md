# redwood on Linux 6.12: kernel source

The full source tree for the Qualcomm 6.12 kernel platform port to the
Redmi Note 12 Pro 5G / POCO X5 Pro 5G (redwood, SM7325), with the redwood
changes applied. Device tree work, bring-up tools and notes live in
[redwood-6.12](https://github.com/suvojeet-sengupta/redwood-6.12).

| Path | Upstream |
|---|---|
| `kernel_platform/common` | CodeLinaro `KERNEL.PLATFORM.5.0.r35-02100-kernel.0` (android16-6.12) |
| `kernel_platform/soc-repo` | same, plus the lahaina HWKM v1 config |
| `kernel_platform/qcom/opensource/devicetree` | `kernel.lnx.6.12.r45-rel` plus redwood |
| `vendor/display-drivers`, `vendor/mm-drivers` | `display-kernel.lnx.12.13.r1-rel` |
| `vendor/graphics-kernel` | `gfx-kernel.lnx.16.0.r14-rel` |
| `vendor/graphics-devicetree` | `gfx-devicetree-oss.lnx.16.0.r8-rel` |
| `vendor/securemsm-kernel`, `vendor/wlan/*`, `vendor/dataipa`, `vendor/datarmnet*` | `LA.VENDOR.16.2.8.r1-06100-LAHAINA.0` |
| `vendor/audio-kernel` | `audio-kernel-legacy.lnx.6.12.r1-rel` |
| `vendor/camera-kernel` | `camera-iot-kernel.qclinux.1.1.r5-rel` |
| `vendor/redwood-modules` | redwood's own modules (usbrole, touch, subsys_compat, Wi-Fi build scripts) |

This is a snapshot without history; the upstream history is on CodeLinaro.
