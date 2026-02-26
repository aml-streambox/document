---
layout: page
title: FAQ and Roadmap
permalink: /faq-and-roadmap/
---

# FAQ and Roadmap

## Known Issues

### System Stability

The current system has limited robustness. The following issues may occur:

- **Green screen** - Video pipeline error, usually requires reboot
- **Black screen** - HDMI signal or pipeline issue, may recover or require reboot
- **System crash** - Kernel or driver crash, requires reboot

These issues are more likely to occur during:
- HDMI hot-plug events (cable connect/disconnect)
- Resolution or framerate changes
- Extended operation periods

**Workaround:** Reboot the device when issues arise. The system is designed to auto-start pipelines after boot.

### HDR Encoding Limitation

While the hardware encoder supports 10-bit HDR encoding, VDIN1 and v4l2src do not currently support 10-bit HDR capture due to firmware limitations. HDR passthrough works, but HDR streaming is not yet available.

### HEVC Lossless Encoding Bug

The hardware supports lossless HEVC encoding, but this feature currently has a bug and cannot be activated. CBR and VBR modes work correctly.

### 1080p240Hz Support

The hardware supports 1080p240Hz, but the VIC (Video Identification Code) definition does not support this timing yet. Only 1080p120Hz is achievable currently.

---

## Frequently Asked Questions

### Q: Why must HDMI TX be connected for capture to work?

**A:** By default, the GStreamer Manager plugin captures the HDMI TX output, not directly from HDMI RX. The pipeline flow is:

```
HDMI RX → tvserver → HDMI TX → GStreamer (v4l2src) → Encoder → Network
```

This design allows for ultra-low latency passthrough while simultaneously capturing the same video for streaming. If HDMI TX is not connected, the video pipeline does not produce output for capture.

### Q: Does the system support HDR passthrough?

**A:** Yes, HDR passthrough is supported up to 4K60fps. The HDR metadata is properly passed from HDMI RX to HDMI TX.

### Q: Can I stream 4K60fps?

**A:** Yes, 4K60fps encoding and streaming is supported. The encoder was overclocked from 500MHz to 666MHz to enable this capability.

### Q: What streaming protocols are supported?

**A:** The supported protocols depend on what GStreamer supports. The developer primarily uses and tests with **SRT**, which is recommended for better stability over unstable networks.

Other protocols such as **RTMP**, **NDI**, and more should be supported through GStreamer plugins, but they have not been extensively tested. You can configure custom protocols using the manual pipeline editor in the GStreamer Manager.

### Q: Is HDCP supported?

**A:** The A311D2 hardware supports HDCP, but HDCP support will not be implemented in this project due to legal and licensing restrictions.

### Q: Is Dolby Vision/Atmos supported?

**A:** The hardware is capable of Dolby Vision and Dolby Atmos processing, but support for these formats will not be implemented due to legal and licensing restrictions.

### Q: How do I access the web UI?

**A:** Connect to the device via WiFi AP (default: `http://192.168.172.1:9090`) or via Ethernet using the DHCP-assigned IP address.

### Q: Can I use custom GStreamer pipelines?

**A:** Yes, the Cockpit GStreamer Manager plugin supports manual pipeline editing and also provides an AI assistant to help generate pipelines from natural language descriptions.

### Q: Why does audio not work with my source?

**A:** Ensure your HDMI source is outputting PCM audio. Some compressed audio formats may not be properly handled. Also verify that the audio loopback device is properly configured.

---

## Future Plans


- **HDR Streaming** - Enable 10-bit HDR capture and streaming once firmware support is available
- **Stability Improvements** - Improve HDMI hot-plug handling and pipeline recovery
- **Bug Fixes** - Fix HEVC lossless encoding issue


- **1080p240Hz Support** - Add VIC definitions for 1080p240Hz timing
- **Audio Enhancements** - Better support for compressed audio formats

---

## Reporting Issues

If you encounter issues not listed above:

1. Check the [Known Issues](#known-issues) section above
2. Reboot the device and try again
3. Report issues at: https://github.com/aml-streambox/yocto/issues

When reporting, please include:
- Device model (VIM4, TVPro, etc.)
- Input source (PS5, Xbox, PC, etc.)
- Resolution and framerate
- Steps to reproduce
