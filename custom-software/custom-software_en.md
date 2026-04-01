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

## streamboxsrc GStreamer Plugin

A unified GStreamer source element (`streamboxsrc`) that replaces the deprecated `v4l2src` capture method. See [Dual-Path HDMI Capture Architecture]({{ '/custom-software/streambox_v0.4' | relative_url }}) for full technical details.

- **Path A (vfmcap)**: Raw capture, ultra low latency, no color processing. Outputs P010 or NV12.
- **Path B (vdin1)**: Color-processed capture with VPP HDR-to-SDR conversion. Ready to use out of the box.
- **Zero-copy safety**: Path A now uses explicit DMA-buf lifetime management and output buffer acquire/release tracking to avoid tearing and use-after-free under sustained streaming.

Basic usage:
```bash
# SDR capture (Path B) — color-processed, ready to stream
streamboxsrc source=vdin1 ! video/x-raw,format=NV21,width=3840,height=2160 ! amlvenc ...

# HDR 10-bit capture (Path A) — preserves original HDR colorimetry
streamboxsrc source=vfmcap output-format=p010 ! video/x-raw,format=P010_10LE,width=3840,height=2160 ! amlvenc internal-bit-depth=10 ...
```

> **Note:** The `v4l2src` capture method is **deprecated**. Use `streamboxsrc` for all video capture. See [Setup Guide]({{ '/setup-guide/setup-guide_en' | relative_url }}) for complete pipeline examples.

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

## cockpit-streambox-settings

A Cockpit plugin for managing system settings on Amlogic A311D2 Streambox.

- Repository: https://github.com/aml-streambox/cockpit-streambox-settings
- Platform: Amlogic A311D2, Cockpit 220

### Features

- **Basic Settings** - Device name, timezone, system locale
- **Network Settings** - Wired, WiFi client, WiFi AP configuration
- **TVServer Settings** - Video, audio, HDCP, debug configuration
- **Storage Settings** - SDCard management, formatting, mount points

### Requirements

- Amlogic A311D2/T7 Streambox hardware
- Yocto-built image with Cockpit
- tvservice (aml_tvserver_streambox) running
