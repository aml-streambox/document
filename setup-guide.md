---
layout: page
title: Setup Guide
permalink: /setup-guide/
---

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
- cockpit webui via https://<device-ip>:9090 or https://192.168.172.1:9090 (WIFI AP default)

Default Linux username: root
Default Linux root password: Streamb0x

Default WIFI AP name: streambox-<your device MAC address>
Default WIFI AP password: streambox

We highly recommend changing the device password and WiFi AP password immediately!

# Start Streaming

By default, the streambox will automatically stream video via SRT at srt://<device-ip>:8888 when both HDMI RX and HDMI TX are working. You can change the streaming setting at https://<device-ip>:9090/gst-manager

However, since this project is still under development and the gstreamer manager plugin might be unstable, we highly encourage to run gstreamer pipeline via cli for now

Here are some gstreamer pipeline templates. Please change the pipeline options according to your actual needs:


For HDMI streaming at 4K(3840x2160) 60fps, h265 50mbps CBR, using HDMI RX audio and stream via SRT at port 8888
```
gst-launch-1.0 -e -v \
  v4l2src device=/dev/video71 io-mode=dmabuf do-timestamp=true ! videorate ! \
  video/x-raw,format=NV21,width=1920,height=1080 ,framerate=120/1 ! \
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
  v4l2src device=/dev/video71 io-mode=dmabuf do-timestamp=true ! \   # By default, HDMI TX loopback video is routed to /dev/video71 via v4l2
  video/x-raw,format=NV12,width=3840,height=2160,framerate=60/1  ! \ # tell the HDMI TX loopback hardware that we need 3840x2160 NV12 60fps video.
  videorate ! \                                                      # always try to feed the video encoder some frame to prevent some bug
  amlvenc gop=60 gop-pattern=0 bitrate=50000 framerate=60 rc-mode=1 ! video/x-h265 ! h265parse config-interval=-1 ! \ #encode using gop interval 60 frames. IPP gop structure with 50mbps bitrate at 60fps CBR (rc-mode=1) with h265
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \  
  mux. \
  alsasrc device=hw:0,6 buffer-time=100000 provide-clock=false slave-method=re-timestamp ! \  #hw:0,6 is HDMI RX audio loopback device.
  audio/x-raw,rate=48000,channels=2,format=S16LE ! \     
  audioconvert ! audioresample ! \
  avenc_aac bitrate=128000 ! aacparse ! \
  mux. \
  mpegtsmux name=mux alignment=7 latency=100000000 ! \   #change it according to your need
  srtsink uri="srt://:8888" wait-for-connection=false sync=false   # we suggest to make wait-for-connection=false, the video encode often malfunction when pipeline is stalled.
```

For HDMI streaming at 1080p(1920x1080) 120fps, h265 50mbps CBR, using line in audio and stream via SRT at port 8888
```
gst-launch-1.0 -e -v v4l2src device=/dev/video71 io-mode=dmabuf do-timestamp=true ! \
    video/x-raw,format=NV21,width=1920,height=1080,framerate=120/1 ! \
    queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
    amlvenc gop=120 framerate=120 bitrate=50000 rc-mode=1 ! \
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
gst-launch-1.0 -e -v v4l2src device=/dev/video71 io-mode=dmabuf do-timestamp=true ! \
    video/x-raw,format=NV21,width=1920,height=1080,framerate=120/1 ! \    #change resolution to 1920x1080 and framerate to 120fps
    queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
    amlvenc gop=120 gop-pattern=0 framerate=120 bitrate=50000 rc-mode=1 ! \    #change gop interval according to your need
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
    srtsink uri="srt://:8888" wait-for-connection=false latency=600 sync=false  #you can also add some latency for smoother streaming
```

You can also explore gstreamer and create your own pipeline according to your requirements.