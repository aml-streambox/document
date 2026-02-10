---
layout: page
title: Driver Modifications
permalink: /driver-modifications/
---

# Driver and Software Modifications

This document lists modifications made to the original Khadas VIM4 Yocto source code.

## U-Boot Modifications

### TVPro Board Support

Added new board support for TVPro (Streambox) device based on A311D2 SoC.

## Kernel Modifications

### Video Encoder Overclock

The video encoder runs at 500MHz by default, which only supports 4K50fps. It has been overclocked to 666MHz to provide 4K60fps encoding capability (can actually run slightly higher than 60fps).

### Video Loopback Buffer

Increased VDIN1 CMA buffer size to support up to 4K HDR frames. The original buffer only supported 1080p.

### HDR Encoding Support

The hardware encoder supports 10-bit HDR encoding. However, VDIN1 and v4l2src do not currently support 10-bit HDR due to firmware limitations. Future versions will address this to enable HDR streaming.

### common_drivers Submodule

**Key fixes:**

#### Frame Rate Limitation

Removed hardcoded 60fps maximum frame rate limitation. The system now supports 2K120fps and 1080p240Hz. However, the VIC definition does not support 1080p240Hz yet, so only 1080p120Hz is achievable currently.

Code change in `drivers/media/vout/hdmitx21/hw/hdmi_tx_hw.c`:

```c
case MESON_CPU_ID_T7:
    //For T7, max is 4k60, 2k120 and 1080p240
    ret = (soc_resolution_limited(timing, 4320) && soc_freshrate_limited(timing, 60)) ||
           (soc_resolution_limited(timing, 2160) && soc_freshrate_limited(timing, 120)) ||
           (soc_resolution_limited(timing, 1080) && soc_freshrate_limited(timing, 240));
    break;
```

#### HDMI RX Black Screen Fix

Ported from Khadas common_drivers fix. Fixes black screen issue when HDMI input resolution or framerate changes.

Files modified:
- `drivers/media/video_sink/video_hw.c`
- `drivers/media/video_sink/video_priv.h`

The fix adds proper tracking of VPP mute state to avoid RX mute causing TX output to be muted.

```c
// Added enum for tracking mute state
enum video_vpp_mute_type {
    VIDEO_BE_MUTED,
    VIDEO_BE_UNMUTED,
    VPP_BE_MUTED,
    VPP_BE_UNMUTED,
};
```

Related original Khadas fix: `hdmirx: adjust vpp mute cnt` (PD#SWPL-156133)

#### VRR Support Fixes

The system has a built-in game mode that writes one frame while reading the previous frame simultaneously. However, since HDMI RX and HDMI TX connect to different physical devices (e.g., a PS5 and a TV), their crystals produce slightly different frequencies, causing timing drift. This eventually leads to buffer read/write collisions, resulting in image glitches or stuttering.

To solve this, VRR (Variable Refresh Rate) is used even when one or both of RX/TX do not natively support VRR. VRR takes both input and output clocks and adjusts the output VBLANK slightly to keep RX and TX phases constant. This allows the system to write and read the same frame with line-level delay, enabling ultra-low latency HDMI video passthrough with latency reduced to a few milliseconds.

**Fixes implemented:**

##### Force VRR Low Latency Feature

This feature forces VRR operation even when one or both devices (RX/TX) do not natively support VRR. When both RX and TX support VRR, VRR operates naturally without forcing.

##### HDMI TX VRR EMP Packet Transmission Fix

In the original driver implementation, the HDMI TX VRR EMP (Extended Metadata Packet) packet mechanism could not be triggered by HDMI RX. This has been fixed to allow proper VRR signaling through EMP packets.

##### Audio Loopback Device Update

Modified device tree (DTS) to configure one loopback device to take HDMI RX audio input. This allows:
- tvserver to exclusively use HDMI RX audio input for passthrough
- GStreamer video streaming pipeline to still get audio from HDMI RX via the loopback device

This separation enables simultaneous passthrough and capture/encode operations on the same audio source.

### GStreamer v4l2src Plugin

Modified the v4l2src plugin in the built-in GStreamer. The original design made the plugin always try to control HDMI RX, which conflicted with the existing tvserver. The HDMI RX control logic has been removed from the plugin.

### GStreamer amlvenc multienc Plugin

Exposed additional encoder features that existed in hardware but were not available to users:

- **B-frame support (gop_pattern)**: Enable B-frame encoding for better compression
- **RC mode selection**: Support for CBR (Constant Bitrate), VBR (Variable Bitrate), and constant QP modes
- **HEVC lossless encode option**: Hardware supports lossless HEVC encoding, though this feature currently has a bug and cannot be activated

## TVServer Modifications

### aml_tvserver_streambox

Forked and modified from the original aml-tvserver to create aml_tvserver_streambox.

- Repository: `git@github.com:aml-streambox/aml_tvserver_streambox.git`
- Branch: streambox_v0.2
- Original source imported from yocto-kirkstone-202406-smarthome release

**Key Modifications:**

#### Low Latency and Game Mode
- Enabled low latency mode by default for minimal passthrough delay
- Set game mode to 2 by default for frame read/write synchronization
- Improved frac_rate_policy handling

#### Resolution and Framerate Support
- Added resolution/framerate follow for HDMI RX and TX
- Added 1080p120Hz and 1440p120Hz support at RX
- Default to HDMI 2.0 mode

#### VRR Support
- Added VRR support with custom force VRR feature
- Refined VRR handling for different modes
- Fixed VRR on EMP packet transmission at TX

#### Audio Passthrough
- Added audio passthrough support with increased buffer size
- Uses legacy audio path to solve hang issues
- Adjusted audio restart timing for better robustness

#### Video Pipeline
- Added extra video pipeline reset for HDMI RX resolution/framerate/HDR status changes
- Fixed color bitdepth passthrough issue
- Updated HDR handling

#### Configuration and Robustness
- Added config-driven design for flexible settings
- Added HDMI EDID passthrough feature
- Improved HDMI TX disconnect event handling
- Added extra reset logic for cable plug/unplug scenarios

**Note:** Despite improvements, green screen, black screen, or system crash may still occur during HDMI hot-plug events.
