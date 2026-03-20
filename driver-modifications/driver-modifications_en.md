---
layout: page
title: Driver Modifications
permalink: /driver-modifications/
---

[🇨🇳 中文版](driver-modifications_cn.md)

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

### HDR 10-bit Capture and Encoding Support

HDR 10-bit capture and encoding is fully supported. The implementation spans kernel driver fixes, a GPU-accelerated format conversion pipeline, and encoder-level HDR metadata signaling.

#### Data Flow

The capture pipeline uses the VPP writeback path:

```
HDMI RX (HDR10 PQ/BT.2020) → vdin0 → VPP display pipeline → VD1 writeback → vdin1 → v4l2src → amlvenc → H.265 bitstream
```

VDIN1 captures from the VPP output via writeback (VIU1_WB0_VD1). It has its own HDR processing module (`VDIN1_HDR`).

#### VDIN1 HDR Bypass (Kernel Driver)

By default, VDIN1's `vdin_set_hdr()` function applies HDR-to-SDR tone mapping when the input signal is HDR10. This converts the HDR pixel data to SDR before capture, destroying HDR information.

A bypass was added that checks for 10-bit V4L2 pixel formats (YUV422_10BIT_PACKED, P010). When using these formats, the VDIN1 HDR processing module is set to bypass mode (`HDR_BYPASS | HLG_BYPASS`), preserving the original HDR pixel data. Standard 8-bit formats (NV12, NV21) continue to receive tone mapping as before.

#### VDIN1 CMA Memory Routing (Kernel Driver)

VDIN1 buffer allocation was routed to a dedicated CMA region to prevent memory contention with other subsystems when allocating large 10-bit frame buffers. CMA pool sizes were also enlarged across all T7/T7C board DTS files.

#### GPU-Accelerated YUV422-to-P010 Conversion (GStreamer Plugin)

The Wave521 hardware encoder does not accept the raw YUV422 10-bit packed format from VDIN1 directly. A Vulkan compute shader performs the conversion:

- **Input**: YUV422 10-bit packed (40 bits per 2 pixels) from VDIN1 via dmabuf
- **Output**: P010-like format with custom byte-swap for Wave521
- **Backend**: Vulkan 1.2 compute shader running on Mali-G52 GPU
- **Performance**: Double-buffered pipeline achieves ~16.3ms/frame (~61fps) at 4K
- **Alternative**: GLES backend available (selectable via `v10conv-backend` property)

The conversion uses cached dmabuf imports with LRU eviction for input buffers and a persistent output buffer pool, minimizing per-frame allocation overhead.

To customize the Vulkan compute shader (`.comp` file), `glslangValidator` is needed to compile GLSL to SPIR-V. This is not required for the normal build process since a pre-compiled SPIR-V header is included.

#### HDR10 VUI Signaling (GStreamer Plugin)

The encoder automatically adds HDR10 VUI (Video Usability Information) fields to the H.265 bitstream when `internal-bit-depth=10`:

- `colour_primaries = 9` (BT.2020)
- `transfer_characteristics = 16` (SMPTE ST 2084 / PQ)
- `matrix_coefficients = 9` (BT.2020 non-constant luminance)

This ensures downstream players and displays correctly interpret the HDR content.

#### Color Matrix Preservation

The `vdin_set_color_matrix()` function for YUV-to-YUV422 at 4K only performs range adjustment (full-range to limited-range conversion), not BT.2020-to-BT.709 gamut conversion. This is safe for HDR passthrough — the original BT.2020 color gamut is preserved.

#### GStreamer Usage

HDR 10-bit streaming:

```
gst-launch-1.0 -e -v \
  v4l2src device=/dev/video71 io-mode=dmabuf do-timestamp=true ! \
  video/x-raw,format=ENCODED,width=3840,height=2160,framerate=60/1 ! \
  videorate ! \
  amlvenc internal-bit-depth=10 v10conv-backend=0 gop=60 gop-pattern=0 bitrate=50000 framerate=60 ! \
  video/x-h265 ! h265parse config-interval=-1 ! \
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  mux. \
  alsasrc device=hw:0,6 buffer-time=500000 provide-clock=false slave-method=re-timestamp ! \
  audio/x-raw,rate=48000,channels=2,format=S16LE ! \
  audioconvert ! audioresample ! avenc_aac bitrate=128000 ! aacparse ! \
  mux. \
  mpegtsmux name=mux alignment=7 latency=100000000 ! \
  srtsink uri="srt://:8888" wait-for-connection=false sync=false
```

Key differences from SDR pipeline:
- `format=ENCODED` instead of `NV21` — requests 10-bit YUV422 packed from VDIN1
- `videorate` element between caps filter and encoder for frame pacing
- `internal-bit-depth=10` — enables 10-bit encoding and GPU format conversion
- `v10conv-backend=0` — selects Vulkan compute shader (0=Vulkan, 1=GLES)
- Higher bitrate recommended (40000-50000 kbps) to preserve 10-bit gradients

Standard SDR 8-bit streaming (unchanged):

```
gst-launch-1.0 -e -v \
  v4l2src device=/dev/video71 io-mode=dmabuf do-timestamp=true ! \
  video/x-raw,format=NV21,width=3840,height=2160,framerate=60/1 ! \
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  amlvenc gop=60 gop-pattern=0 bitrate=20000 framerate=60 rc-mode=1 ! \
  video/x-h265 ! h265parse config-interval=-1 ! \
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  mux. \
  alsasrc device=hw:0,6 buffer-time=500000 provide-clock=false slave-method=re-timestamp ! \
  audio/x-raw,rate=48000,channels=2,format=S16LE ! \
  queue max-size-buffers=0 max-size-time=500000000 max-size-bytes=0 ! \
  audioconvert ! audioresample ! avenc_aac bitrate=128000 ! aacparse ! \
  queue max-size-buffers=0 max-size-time=500000000 max-size-bytes=0 ! \
  mux. \
  mpegtsmux name=mux alignment=7 latency=100000000 ! \
  srtsink uri="srt://:8888" wait-for-connection=false latency=600 sync=false
```

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

- **B-frame support (gop_pattern)**: Enable B-frame encoding for better compression. The `gop-pattern` property is accepted for API compatibility but B-frames are not actually produced (GopPreset=0x0 in encoder initialization).
- **RC mode selection**: Support for CBR (Constant Bitrate), VBR (Variable Bitrate), and constant QP modes
- **10-bit HDR encoding (internal-bit-depth)**: Enables 10-bit encoding with GPU-accelerated YUV422-to-P010 conversion via Vulkan compute shader or GLES backend
- **Conversion backend selection (v10conv-backend)**: Choose between Vulkan (0, default) and GLES (1) for 10-bit format conversion
- **HDR10 VUI signaling**: Automatic BT.2020/PQ metadata insertion in H.265 bitstream when encoding in 10-bit mode
- **HEVC lossless encode option**: Hardware supports lossless HEVC encoding (currently has a bug — see FAQ)

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
