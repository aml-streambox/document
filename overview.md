---
layout: page
title: Overview
permalink: /overview/
---

# Project Overview

## 1. Project Introduction

Streambox is a project that transforms Amlogic A311D2 SoC-based systems into dedicated stream computers with video capture capabilities. It enables high-performance video processing for streaming applications.

Key technical features:
- Passthrough up to 4K60fps HDR video from HDMI RX to HDMI TX with ultra low latency and VRR support
- Capture, encode, and stream 4K60 YUV420 SDR video via network
- Support for H.264/H.265 encoding
- Support for SRT/RTMP streaming protocols

## 2. Why This Project Exists

The project was created to stream arcade games at 1080p120fps via internet wtih SBC. The A311D2 chip was selected because it supports:
- HDMI RX and HDMI TX at 4K60 with VRR
- Hardware encoder up to 4K60fps
- Powerful CPU performance
- Wireless network including WiFi 6 (via Khadas VIM4 SBC)

These capabilities exceed the original requirements, making the A311D2 ideal for building a dedicated streaming PC.

## 3. What This Project Does

This project provides driver fixes and a minimal software stack for video streaming:
- Modified aml_tvserver for video pipeline control
- Modified GStreamer v4l2src plugin to enable video capture
- Cockpit plugin for GStreamer pipeline management
- Minimal web UI based on Cockpit

Note: System robustness is limited. Green screen, black screen, and system crashes may occur. Rebooting is expected when issues arise.

## 4. Supported Hardware

Any device with A311D2 chip featuring HDMI RX, HDMI TX, Ethernet, and WiFi should be compatible.

Tested devices:
- Khadas VIM4 (main development platform)
- TVPro 4G (A311D2 core board based system)

## 5. How to Use

### Build the Image

Follow the [Build Guide](build-guide.md) to compile your own image. Pre-built images are not distributed.

### Flash the Image

1. Locate the built image at `build/tmp/deploy/images/CONFIGURATION/<boardname>-yocto-<date>.img`
2. Use AmlBurn tool V3 to flash directly to device eMMC

### First Boot

After flashing:
- Connect via serial console (baud rate: 921600) or SSH
- Default login: `root` / `Streamb0x`
- Ethernet uses DHCP to obtain IP address
- WiFi AP is enabled by default

### Access Cockpit Web UI

Connect to device WiFi AP or via ethernet, then access:
```
http://192.168.127.1:9090 (via WIFI AP)
or
http://<obtained dhcp ip>:9090 (via eherent or WIFI client mode)
```

### HDMI Passthrough

HDMI passthrough is enabled by default. HDMI TX must be connected to a valid output (TV or monitor) for video capture to work.

### Video Capture and Streaming

By default, the GStreamer Manager plugin capture hdmitx's output (it is the reason why hdmitx must be plugged in) and provides an SRT stream at port 8888. You can modify streaming settings in the Cockpit GStreamer Manager tab.

## 6. Documentation Index

- [Overview](overview.md) - This page
- [Build Guide](build-guide.md) - How to build the image
- [Driver Modifications](driver-modifications.md) - Kernel and driver changes
- [Custom Software](custom-software.md) - Cockpit plugins
- [Source Code Reference](source-code-reference.md) - License and origins
- [FAQ and Roadmap](faq-and-roadmap.md) - Known issues, Q&A, and future plans

## 7. Development Method

This project was created entirely through **vibe coding** using various AI models including Claude Opus, Kimi 2.5, GLM-4.7, Gemini 3 Pro, and others.

**Important:** You can assume that the author has **no idea how the AI-generated code works** (but it does work, somehow).

### Use With Caution

- No formal code review has been conducted
- AI-generated code may contain unexpected behaviors or "funny" logic
- The project is functional but may have unusual implementations
- Contributions and bug reports are welcome

## 8. License

This project is licensed under the [MIT License](LICENSE).

### Original Code License

This project is based on the Khadas VIM4 Yocto release. Original components inherit their respective licenses from the upstream project (GPL/LGPL for kernel and many components). See [Source Code Reference](source-code-reference.md) for more details.

### Attribution (Appreciated but Not Required)

For commercial use, mentioning this project as a reference is appreciated but not required by the license.

### Ethical Use

The author strongly opposes using this project for cheating purposes, including academic dishonesty, gaming fraud, or examination fraud. Please use responsibly.

---

*This project was created through vibe coding with AI assistance. See [Development Method](#7-development-method) for details.*
