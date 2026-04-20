---
layout: page
title: Custom Software
permalink: /custom-software/
---

[🇨🇳 中文版]({{ '/custom-software/custom-software_cn' | relative_url }})

# Custom Software Components

## cockpit-gst-manager

A Cockpit plugin for managing GStreamer streaming/encoding pipelines on Amlogic A311D2.

- Repository: https://github.com/aml-streambox/cockpit-gst-manager
- Platform: Amlogic A311D2, GStreamer 1.20.7, Cockpit 220

### Features

- **Multi-instance** - Run multiple GStreamer pipelines simultaneously
- **AI Assistant** - Natural language to GStreamer CLI (BYO API key)
- **Manual Editor** - Direct CLI editing for advanced users
- **HDR 10-bit Support** - Automatic HDR detection and pipeline generation with GPU-accelerated Vulkan conversion
- **Event Triggers** - Auto-start on HDMI signal, USB plug, boot
- **Video Compositor** - OSD/overlay via ge2d hardware acceleration
- **Import/Export** - Share pipeline configurations
- **Localization** - English and Chinese UI

### Current v0.5.1 Status

- **Auto capture recovery** - HDMI auto capture pipelines now recover after signal loss, resolution changes, and transient errors via debounced restart and exponential backoff
- **No-signal screen** - HDMI TX shows bouncing "NO SIGNAL" / "STREAMBOX" boxes when HDMI RX is disconnected or unstable
- **Double-buffered rendering** - Flicker-free fb0 page flipping at 1080p, OSD hardware scales to 4K
- **Passthrough stability** - HDMI RX → TX passthrough no longer corrupted by fb0 manipulation during mode changes
- **Auto start/stop** - No-signal UI activates on RX invalid/unstable and disappears on RX stable
- **Path A independence** - `vfmcap` capture starts from HDMI RX stability only, no TX dependency
- **4-stage shutdown** - Hung encoder pipelines cleanly stopped via SIGUSR1 → SIGINT → SIGTERM → SIGKILL
- **UVC serial tracking** - UVC devices now tracked by USB serial ID for persistent identification across reconnections; auto-start when device is connected

### Previous v0.5 Status

- **Wave521 HEVC B-frame support** - Working and validated on target hardware
- **Extended GOP presets** - `gop-pattern=0..8` exposed and verified, including `RA_IB`
- **HEVC lossless mode** - Working in the current encoder stack
- **UVC transcoding path** - UVC H.264 -> hardware decode -> hardware HEVC encode -> SRT works end-to-end
- **SRT production path** - Validated with the fixed HEVC B-frame encoder path

### Development History

- **HDR 10-bit pipeline support** - Auto-detect HDR source and generate appropriate 10-bit or 8-bit pipeline
- **Auto pipeline implementation** - Automatic GStreamer pipeline configuration
- **TVService integration** - Use tvserver for HDMI RX signal change detection
- **AI prompt optimization** - Refined prompts with HDR pipeline knowledge
- **Boot order fix** - Service boots at last order to ensure dependencies ready
- **v0.4 dual-path capture** - New `streamboxsrc` plugin replacing `v4l2src`, supporting raw low-latency capture (Path A) and color-processed capture (Path B)
- **v0.4 new display modes** - Support for 2560x1440@60/120/144Hz, 3440x1440@60Hz, 1920x1080@144/240Hz, 2560x1080@60Hz
- **Capture tearing fix** - Kernel one-frame delay plus userspace output buffer pool and DMA_BUF_SYNC synchronization for Path A
- **vfm_cap stability fixes** - Fixed `prev_v4l2_frame` cleanup races and DMA-buf lifetime bug in zero-copy V4L2 re-queue path
- **v0.5 Wave521 B-frame fix** - Fixed delay-frame header prepend bug that created fake I-frames and broke B-frame output ordering
- **v0.5 full GOP preset exposure** - Added Wave5 preset support for `IPP`, `IBBBP`, `IBPBP`, `IBBB`, `ALL_I`, `IPPPP`, `IBBBB`, `RA_IB`, and `IPP_SINGLE`
- **v0.5 RA_IB startup fix** - Increased source-side and capture-side buffering for deep-reorder GOP startup
- **v0.5 UVC integration** - Added end-to-end UVC H.264 decode/transcode support for Cockpit-managed pipelines
- **v0.5.1 no-signal UI** - Framebuffer-based bouncing box renderer with double buffering, triggered by HDMI RX signal state
- **v0.5.1 passthrough fix** - Removed fb0 reset from TX mode sync to prevent VPP compositor corruption
- **v0.5.1 4K safety** - Always render at 1080p with smem_len guards, OSD hardware scaler handles upscaling
- **v0.5.1 auto capture recovery** - Instance-exit-driven pipeline rebuild and restart after HDMI signal changes, with debounce and backoff
- **v0.5.1 Path A independence** - vfmcap auto capture no longer gated on HDMI TX readiness
- **v0.5.1 4-stage shutdown** - SIGUSR1→SIGINT→SIGTERM→SIGKILL for hung encoder pipelines
- **v0.5.1 startup watchdog** - Instances stuck in STARTING are auto-aborted and retried
- **v0.5.1 UVC serial tracking** - UVC devices identified by USB serial ID for persistent config; auto-start on device connect

## streamboxsrc GStreamer Plugin

A unified GStreamer source element (`streamboxsrc`) that replaces the deprecated `v4l2src` capture method. See [Dual-Path HDMI Capture Architecture]({{ '/custom-software/streambox_v0.4' | relative_url }}) for full technical details.

- **Path A (vfmcap)**: Raw capture, ultra low latency, no color processing. Outputs P010 or NV12.
- **Path B (vdin1)**: Color-processed capture with VPP HDR-to-SDR conversion. Ready to use out of the box.
- **Zero-copy safety**: Path A now uses explicit DMA-buf lifetime management and output buffer acquire/release tracking to avoid tearing and use-after-free under sustained streaming.
- **Deep reorder safe**: Path A now has enough DMA-buf pool depth for deep Wave521 GOP presets such as `RA_IB`.

Basic usage:
```bash
# SDR capture (Path B) — color-processed, ready to stream
streamboxsrc source=vdin1 ! video/x-raw,format=NV21,width=3840,height=2160 ! amlvenc ...

# HDR 10-bit capture (Path A) — preserves original HDR colorimetry
streamboxsrc source=vfmcap output-format=p010 ! video/x-raw,format=P010_10LE,width=3840,height=2160 ! amlvenc internal-bit-depth=10 ...
```

> **Note:** The `v4l2src` capture method is **deprecated**. Use `streamboxsrc` for all video capture. See [Setup Guide]({{ '/setup-guide/setup-guide_en' | relative_url }}) for complete pipeline examples.

## Wave521 Encoder v0.5

The Wave521 hardware encoder is the core of the StreamBox v0.5 media path.

Current validated status:
- **HEVC B-frames** - working
- **HEVC lossless** - working
- **Extended GOP presets** - working
- **SRT output with B-frames** - working

Verified `gop-pattern` values:

| Value | Pattern | Status |
|------|---------|--------|
| 0 | IPP | Verified |
| 1 | IBBBP | Verified |
| 2 | IBPBP | Verified |
| 3 | IBBB | Verified |
| 4 | ALL_I | Verified |
| 5 | IPPPP | Verified |
| 6 | IBBBB | Verified |
| 7 | RA_IB | Verified |
| 8 | IPP_SINGLE | Verified |

Important v0.5 fixes:
- Fixed fake SPS/PPS-only outputs during B-frame startup delay
- Removed redundant plugin-side HEVC header prepend
- Increased `streamboxsrc` Path A pool depth for deep-reorder GOP startup
- Increased Wave521 userspace source slot count to match command queue depth

Recommended quick checks:
```bash
# Check frame types in the produced bitstream
ffprobe -show_frames -select_streams v -show_entries frame=pict_type -of csv /tmp/out.h265

# Check for flashback/reorder artifacts
python3 scripts/analyze_neighbor_frames.py /tmp/out.h265
```

For full design and investigation notes, see [StreamBox v0.5]({{ '/custom-software/streambox_v0.5' | relative_url }}).

## HDMI Auto Capture Recovery (v0.5.1)

When HDMI signal changes or the capture pipeline exits unexpectedly, `cockpit-gst-manager`
now automatically recovers:

- **Exit-driven recovery** — `streamboxsrc` posts `hdmi-signal-change` and exits with EOS;
  the manager parses the exit details and schedules a debounced restart with updated parameters
- **Path A independence** — `vfmcap` auto capture starts from HDMI RX stability only,
  no TX dependency
- **Path B TX gating** — `vdin1` auto capture waits for both RX stability and TX readiness
- **Transient error retry** — Network disconnect, timeout, and buffer underrun errors
  auto-retry up to 3 times
- **4-stage shutdown** — SIGUSR1→SIGINT→SIGTERM→SIGKILL for hung encoder pipelines
- **Startup watchdog** — Instances stuck in STARTING are auto-aborted and retried

Technical details: [StreamBox v0.5.1]({{ '/custom-software/streambox_v0.5.1' | relative_url }})

## HDMI TX No-Signal Screen (v0.5.1)

When HDMI RX has no valid signal, StreamBox now displays a visible no-signal screen
on HDMI TX instead of plain black output.

Key features:
- **Bouncing boxes** — "NO SIGNAL" (red border) and "STREAMBOX" (blue border) boxes
  bounce DVD-style across a black background
- **Double-buffered** — page flipping via `FBIOPAN_DISPLAY` eliminates visible flicker
- **4K safe** — renders at 1080p and relies on OSD hardware scaling to fill 4K output
- **Passthrough clean** — fb0 manipulation does not disrupt live HDMI RX → TX video path
- **Auto managed** — starts on RX invalid, stops on RX stable, audio passthrough paused

Technical details: [StreamBox v0.5.1]({{ '/custom-software/streambox_v0.5.1' | relative_url }})

## UVC Device Serial Tracking (v0.5.1)

UVC device pipelines are now tracked by USB serial ID instead of device path:

- **Persistent identification** — Devices identified by USB serial number, not `/dev/videoX` path
- **Auto-start on connect** — Configured instances auto-start when their device is plugged in
- **Hot-plug aware** — Device connect/disconnect events trigger appropriate start/stop actions
- **Config preservation** — Saved settings apply to correct device even when path changes

Technical details: [StreamBox v0.5.1]({{ '/custom-software/streambox_v0.5.1' | relative_url }})

## UVC Transcoding

The UVC pipeline in v0.5 is no longer limited to basic device discovery. The working production path is:

`v4l2src (UVC H.264) -> h264parse -> amlv4l2h264dec -> amlvenc -> h265parse -> mpegtsmux -> srtsink`

Current status:
- Hardware decode and re-encode path works
- DMA-buf-backed hardware encode path works
- SRT output works
- This path is intended to be managed by `cockpit-gst-manager`

## Dual-Path HDMI Capture Architecture

StreamBox v0.4 introduces a completely new dual-path HDMI capture architecture with a custom kernel module (`vfm_cap`), userspace SDK (`libvfmcap`), and unified GStreamer plugin (`streamboxsrc`).

Full technical document: [Dual-Path HDMI Capture Architecture]({{ '/custom-software/streambox_v0.4' | relative_url }})

Key components:
- **vfm_cap** — Kernel module that intercepts the VFM pipeline for zero-copy frame capture via `/dev/video_cap`
- **libvfmcap** — Userspace SDK with Vulkan GPU-accelerated AMLY to P010/NV12 format conversion
- **streamboxsrc** — Unified GStreamer source plugin supporting Path A (raw low-latency) and Path B (color-processed)
- **amlvenc** — Encoder plugin enhancements: VBR/CBR fix, colorimetry/VUI signaling, 0-240fps support

Recent capture-path fixes:
- **Tearing fix** — Path A uses kernel-side one-frame delay so V4L2 consumers only see fully-written frames
- **Output buffer pool fix** — streamboxsrc tracks output buffers with acquire/release instead of round-robin reuse
- **GPU sync fix** — libvfmcap wraps Vulkan writes with DMA_BUF_SYNC and waits for GPU completion before releasing buffers
- **DMA-buf lifetime fix** — vfm_cap keeps an extra dma_buf reference after `dma_buf_fd()` so QBUF cleanup cannot hit freed objects

## v0.4 New Display Modes

StreamBox v0.4 adds support for the following HDMI display modes:

| Mode | Resolution | Refresh Rates |
|------|-----------|---------------|
| 1440p | 2560x1440 | 60Hz, 120Hz, 144Hz |
| Ultrawide | 3440x1440 | 60Hz |
| 1080p high refresh | 1920x1080 | 144Hz, 240Hz |
| Ultrawide FHD | 2560x1080 | 60Hz |

Changes span kernel HDMI RX/TX drivers, tvin/vdin subsystem, tvserver EDID management, GStreamer plugins, and Cockpit UI. See the [architecture document]({{ '/custom-software/streambox_v0.4' | relative_url }}) for details.

## OTA Firmware Update (v0.6.2)

Over-the-air firmware update support via both Cockpit WebUI and CLI.

Technical details: [StreamBox v0.6.2]({{ '/custom-software/streambox_v0.6.2' | relative_url }})

### WebUI Method (Recommended)

1. Open `https://<device-ip>:9090` in a browser
2. Navigate to **"Firmware Update"** tab
3. Drag & drop `software.swu` file
4. Backend verifies SHA-256, board name match, and package signature
5. Click **"Start Update"** to trigger OTA flash
6. Device reboots into recovery, flashes all partitions, auto-reboots (~70s)

### CLI Method (SSH)

```bash
# Build the OTA package
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15
bitbake amlogic-yocto

# Upload and trigger
scp build/tmp/deploy/images/mesont7-tvpro-5.15/software.swu \
    root@<device-ip>:/data/software.swu
ssh root@<device-ip> '/usr/bin/update_swfirmware.sh'
```

### OTA Bugs Fixed in v0.6.2

Three critical bugs were fixed in the Amlogic SWUpdate chain:

| Bug | Symptom | Root Cause |
|-----|---------|------------|
| sw-description indentation | libconfig syntax error | Pack scripts used 3-tab vs 4-tab indentation |
| Dots in board name | libconfig syntax error / board mismatch | `mesont7_tvpro_5.15` contains dots; libconfig rejects dots in identifiers |
| `swupdate -c` env corruption | Device enters USB burning mode | Check-only mode wrote `write_boot=1` to u-boot env without dry_run guard |

> **WARNING**: Never run `swupdate -c` on devices without the dry_run patch applied. It will brick the device.

## cockpit-streambox-settings

A Cockpit plugin for managing system settings on Amlogic A311D2 Streambox.

- Repository: https://github.com/aml-streambox/cockpit-streambox-settings
- Platform: Amlogic A311D2, Cockpit 220

### Features

- **Basic Settings** - Device name, timezone, system locale
- **Network Settings** - Wired, WiFi client, WiFi AP configuration
- **TVServer Settings** - Video, audio, HDCP, debug configuration
- **Storage Settings** - SDCard management, formatting, mount points
- **Firmware Update** - OTA firmware upload, verify, and flash (v0.6.2)

### Requirements

- Amlogic A311D2/T7 Streambox hardware
- Yocto-built image with Cockpit
- tvservice (aml_tvserver_streambox) running
