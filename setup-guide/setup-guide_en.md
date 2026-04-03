---
layout: page
title: Setup Guide
permalink: /setup-guide/
---

[🇨🇳 中文版]({{ '/setup-guide/setup-guide_cn' | relative_url }})

## Put your device into USB upgrade mode and flash the system image

### Khadas VIM4

Just flash the image like a normal OS image.

Please refer to VIM4's official guide [boot-into-upgrade-mode](https://docs.khadas.com/products/sbc/vim4/install-os/boot-into-upgrade-mode) to boot into upgrade mode then using [flash os using usb tool](https://docs.khadas.com/products/sbc/vim4/install-os/install-os-into-emmc-via-usb-tool)

[download USB upgrade tool](https://dl.khadas.com/products/vim4/tools/aml-burn-tool-v3.2.0.zip)

### TVPro Device
Pretty much same as VIM4 for Serial Mode.

For Key Mode, press the reset key and plug in power supply without releasing the reset key.


**Note**: Flashing methods may vary based on the specific board revision and bootloader version. Always ensure you have the correct image for your device variant.


# Default OS information

You can connect to the Streambox device by using 
- uart serial
- ssh
- cockpit webui via https://[device-ip]:9090 or https://192.168.172.1:9090 (WIFI AP default)

Default Linux username: root
Default Linux root password: Streamb0x

Default WIFI AP name: streambox-<your device MAC address>
Default WIFI AP password: streambox

We highly recommend changing the device password and WiFi AP password immediately!

# Start Streaming

By default, the streambox will automatically stream video via SRT at srt://[device-ip]:8888 when both HDMI RX and HDMI TX are working. You can change the streaming setting at https://[device-ip]:9090/gst-manager

However, since this project is still under development and the gstreamer manager plugin might be unstable, we highly encourage to run gstreamer pipeline via cli for now

Here are some gstreamer pipeline templates. Please change the pipeline options according to your actual needs:

> **Note:** The `streamboxsrc` plugin is the recommended way to capture video. The old `v4l2src` method is deprecated. See [Custom Software]({{ '/custom-software/custom-software_en' | relative_url }}) for details.

## StreamBox v0.5 Notes

Latest validated v0.5 status:
- Wave521 HEVC B-frame encoding is working
- Wave521 HEVC lossless mode is working
- Extended `gop-pattern=0..8` presets are working
- `RA_IB` deep-reorder startup is working after buffer-depth fixes
- UVC H.264 -> HW decode -> HW HEVC encode -> SRT is working

Recommended capture rules for the current encoder stack:
- For HDMI Path A / raw capture validation, prefer `streamboxsrc ! "video/x-raw,format=NV12"` and let size/framerate negotiate unless you need fixed caps
- `videotestsrc` is not a valid replacement for this hardware path because the encoder expects DMA-buf-backed input
- If HDMI RX test-frame acquisition fails after repeated encoder tests, reboot the target to reset VDIN/VPU state

Useful `gop-pattern` values:

| Value | Pattern |
|------|---------|
| 0 | IPP |
| 1 | IBBBP |
| 2 | IBPBP |
| 3 | IBBB |
| 4 | ALL_I |
| 5 | IPPPP |
| 6 | IBBBB |
| 7 | RA_IB |
| 8 | IPP_SINGLE |

## SDR 8-bit Streaming

For HDMI streaming at 4K(3840x2160) 60fps, h265 50mbps CBR, using HDMI RX audio and stream via SRT at port 8888:
```
gst-launch-1.0 -e -v \
  streamboxsrc source=vdin1 ! \
  video/x-raw,format=NV21,width=3840,height=2160 ! \
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  amlvenc gop=60 gop-pattern=0 bitrate=50000 framerate=60 rc-mode=1 ! \
  video/x-h265 ! h265parse config-interval=-1 ! \
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  mux. \
  alsasrc device=hw:0,6 buffer-time=100000 ! \
  audio/x-raw,rate=48000,channels=2,format=S16LE ! \
  queue max-size-buffers=0 max-size-time=500000000 max-size-bytes=0 ! \
  audioconvert ! audioresample ! \
  avenc_aac bitrate=128000 ! aacparse ! \
  queue max-size-buffers=0 max-size-time=500000000 max-size-bytes=0 ! \
  mux. \
  mpegtsmux name=mux alignment=7 latency=100000000 ! \
  srtsink uri="srt://:8888" wait-for-connection=false latency=600 sync=false
```
Explain:
```
gst-launch-1.0 -e -v \   # -e is critical. It allows gstreamer to stop the encoder properly without any issue
  streamboxsrc source=vdin1 ! \   # Use streamboxsrc Path B for color-processed capture. Replaces deprecated v4l2src.
  video/x-raw,format=NV21,width=3840,height=2160  ! \   # Request 4K60 NV21 (8-bit) output. Framerate is auto-negotiated.
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  amlvenc gop=60 gop-pattern=0 bitrate=50000 framerate=60 rc-mode=1 ! video/x-h265 ! h265parse config-interval=-1 ! \   # Encode: GOP 60 frames, IPP structure, 50Mbps CBR at 60fps
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \  
  mux. \
  alsasrc device=hw:0,6 buffer-time=100000 provide-clock=false slave-method=re-timestamp ! \  # hw:0,6 is HDMI RX audio loopback device.
  audio/x-raw,rate=48000,channels=2,format=S16LE ! \     
  audioconvert ! audioresample ! \
  avenc_aac bitrate=128000 ! aacparse ! \
  mux. \
  mpegtsmux name=mux alignment=7 latency=100000000 ! \   # Change latency according to your need
  srtsink uri="srt://:8888" wait-for-connection=false sync=false   # We suggest wait-for-connection=false, the video encode often malfunctions when pipeline is stalled.
```

For HDMI streaming at 1080p(1920x1080) 120fps, h265 50mbps CBR, using line in audio and stream via SRT at port 8888:
```
gst-launch-1.0 -e -v \
  streamboxsrc source=vdin1 ! \
  video/x-raw,format=NV21,width=1920,height=1080 ! \
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  amlvenc gop=120 gop-pattern=0 framerate=120 bitrate=50000 rc-mode=1 ! \
  video/x-h265 ! \
  h265parse config-interval=-1 ! \
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  mux. alsasrc device=hw:0,0 buffer-time=50000 provide-clock=false slave-method=re-timestamp ! \
  audio/x-raw,rate=48000,channels=2,format=S16LE ! \
  queue max-size-buffers=0 max-size-time=500000000 max-size-bytes=0 ! \
  audioconvert ! \
  audioresample ! \
  avenc_aac bitrate=128000 ! \
  aacparse ! \
  queue max-size-buffers=0 max-size-time=500000000 max-size-bytes=0 ! \
  mux. mpegtsmux name=mux alignment=7 latency=100000000 ! \
  srtsink uri="srt://:8888" wait-for-connection=false latency=600 sync=false
```


Explain: 
```
gst-launch-1.0 -e -v \
  streamboxsrc source=vdin1 ! \
  video/x-raw,format=NV21,width=1920,height=1080 ! \    # Auto-negotiated framerate from HDMI source
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  amlvenc gop=120 gop-pattern=0 framerate=120 bitrate=50000 rc-mode=1 ! \    # Change gop interval according to your need
  video/x-h265 ! \
  h265parse config-interval=-1 ! \
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  mux. alsasrc device=hw:0,0 buffer-time=50000 provide-clock=false slave-method=re-timestamp ! \   # line in audio is at hw:0,0
  audio/x-raw,rate=48000,channels=2,format=S16LE ! \
  queue max-size-buffers=0 max-size-time=500000000 max-size-bytes=0 ! \
  audioconvert ! \
  audioresample ! \
  avenc_aac bitrate=128000 ! \
  aacparse ! \
  queue max-size-buffers=0 max-size-time=500000000 max-size-bytes=0 ! \
  mux. mpegtsmux name=mux alignment=7 latency=100000000 ! \
  srtsink uri="srt://:8888" wait-for-connection=false latency=600 sync=false  # You can add latency for smoother streaming
```


## HDR 10-bit Streaming

For HDR 10-bit streaming, when the HDMI source outputs HDR10, HLG, or HDR10+ content. This uses Path A (vfmcap) which preserves the original HDR colorimetry:

```
gst-launch-1.0 -e -v \
  streamboxsrc source=vfmcap output-format=p010 ! \
  video/x-raw,format=P010_10LE,width=3840,height=2160 ! \
  amlvenc internal-bit-depth=10 gop=60 gop-pattern=0 bitrate=50000 framerate=60 ! \
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
- `source=vfmcap output-format=p010` — Uses Path A for raw HDR capture with GPU AMLY-to-P010 conversion
- `format=P010_10LE` — 10-bit format from Path A, preserves original BT.2020 PQ colorimetry
- `internal-bit-depth=10` — Enables 10-bit encoding; VUI (HDR10 BT.2020/PQ) is automatically signaled from the capture plugin's colorimetry detection
- No `videorate` needed — streamboxsrc handles framerate natively
- Higher bitrate recommended (40000-50000 kbps) to preserve 10-bit gradients

## B-frame and GOP Examples

Example: HEVC IBBBP output with real B-frames:

```bash
gst-launch-1.0 -e -v \
  streamboxsrc ! "video/x-raw,format=NV12" ! \
  amlvenc bitrate=8000 gop=60 gop-pattern=1 ! \
  "video/x-h265" ! h265parse ! filesink location=/tmp/ibbbp.h265
```

Example: HEVC RA_IB output:

```bash
gst-launch-1.0 -e -v \
  streamboxsrc ! "video/x-raw,format=NV12" ! \
  amlvenc bitrate=8000 gop=60 gop-pattern=7 ! \
  "video/x-h265" ! h265parse ! filesink location=/tmp/ra_ib.h265
```

Verify frame types:

```bash
ffprobe -show_frames -select_streams v -show_entries frame=pict_type -of csv /tmp/ibbbp.h265
```

Verify there is no obvious flashback artifact:

```bash
python3 scripts/analyze_neighbor_frames.py /tmp/ibbbp.h265
```

## UVC Transcoding Example

The current working UVC production path is hardware decode plus hardware HEVC re-encode:

```bash
gst-launch-1.0 -e -v \
  v4l2src device=/dev/video0 io-mode=dmabuf ! \
  h264parse ! amlv4l2h264dec capture-io-mode=dmabuf ! \
  amlvenc bitrate=4000 framerate=30 gop=5 gop-pattern=0 ! \
  video/x-h265 ! h265parse config-interval=-1 ! \
  mpegtsmux ! srtsink uri="srt://:8888" wait-for-connection=false sync=false
```

You can also explore gstreamer and create your own pipeline according to your requirements.

### Path A vs Path B Guide

| Feature | Path A (`source=vfmcap`) | Path B (`source=vdin1`) |
|---------|-------------------------|------------------------|
| Latency | Ultra low (no VPP) | Higher (1-3 frames VPP) |
| Color | Raw, no processing | HDR→SDR converted, ready to use |
| Formats | P010_10LE, NV12 | NV21 (8-bit), P010_10LE (10-bit) |
| HDR | Preserves original HDR | VPP tone-mapped to SDR |
| Use case | HDR streaming, raw capture | SDR streaming, general use |
