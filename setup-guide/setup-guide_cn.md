---
layout: page
title: 配置指南
---

[🇺🇸 English Version]({{ '/setup-guide/setup-guide_en' | relative_url }})

## 进入 USB 升级模式并刷写系统镜像

### Khadas VIM4

刷写方式与普通操作系统镜像相同。

请参考 VIM4 官方文档：
- [进入升级模式](https://docs.khadas.com/products/sbc/vim4/install-os/boot-into-upgrade-mode)
- [使用 USB 工具刷写系统](https://docs.khadas.com/products/sbc/vim4/install-os/install-os-into-emmc-via-usb-tool)

[下载 USB 升级工具](https://dl.khadas.com/products/vim4/tools/aml-burn-tool-v3.2.0.zip)

### TVPro 设备

串口模式与 VIM4 基本一致。

按键模式：按住复位键的同时接通电源，保持复位键不放。

**注意**：刷写方法可能因板卡版本和引导程序版本而异。请确保使用与设备匹配的镜像文件。


# 系统默认信息

可通过以下方式连接 Streambox 设备：
- UART 串口
- SSH
- Cockpit Web 界面（https://[设备 IP]:9090 或 https://192.168.172.1:9090，WiFi 热点默认地址）

默认 Linux 用户名：root
默认 root 密码：Streamb0x

默认 WiFi 热点名称：streambox-<设备 MAC 地址>
默认 WiFi 热点密码：streambox

**强烈建议**首次登录后立即修改设备密码和 WiFi 热点密码！

# 开始直播

默认配置下，当 HDMI RX 和 HDMI TX 均正常工作时，设备会自动通过 SRT 协议在 srt://[设备 IP]:8888 提供视频串流。可在 https://[设备 IP]:9090/gst-manager 调整直播参数。

由于项目仍在开发中，GStreamer Manager 插件可能不够稳定，建议目前通过命令行启动直播管线。

以下为 GStreamer 管线模板，请根据实际需求调整参数：

> **注意**：`streamboxsrc` 插件是推荐的采集方式。旧的 `v4l2src` 方式已弃用。详见[软件组件]({{ '/custom-software/custom-software_cn' | relative_url }})。

## SDR 8-bit 直播

### 4K 60fps H.265 50Mbps CBR 直播（使用 HDMI RX 音频）

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

参数说明：
```
gst-launch-1.0 -e -v \   # -e 参数至关重要，确保编码器正常停止
  streamboxsrc source=vdin1 ! \   # 使用 streamboxsrc Path B 进行色彩处理采集，替代已弃用的 v4l2src
  video/x-raw,format=NV21,width=3840,height=2160 ! \   # 请求 4K60 NV21（8-bit）输出，帧率自动协商
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  amlvenc gop=60 gop-pattern=0 bitrate=50000 framerate=60 rc-mode=1 ! video/x-h265 ! h265parse config-interval=-1 ! \   # GOP 间隔 60 帧，IPP 结构，50Mbps 码率，60fps CBR，H.265
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \  
  mux. \
  alsasrc device=hw:0,6 buffer-time=100000 provide-clock=false slave-method=re-timestamp ! \  # hw:0,6 为 HDMI RX 音频环回设备
  audio/x-raw,rate=48000,channels=2,format=S16LE ! \     
  audioconvert ! audioresample ! \
  avenc_aac bitrate=128000 ! aacparse ! \
  mux. \
  mpegtsmux name=mux alignment=7 latency=100000000 ! \   # 可根据需求调整
  srtsink uri="srt://:8888" wait-for-connection=false sync=false   # 建议设置 wait-for-connection=false，管线阻塞时编码容易异常
```

### 1080p 120fps H.265 50Mbps CBR 直播（使用线路输入音频）

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

参数说明：
```
gst-launch-1.0 -e -v \
  streamboxsrc source=vdin1 ! \
  video/x-raw,format=NV21,width=1920,height=1080 ! \    # 帧率从 HDMI 源自动协商
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  amlvenc gop=120 gop-pattern=0 framerate=120 bitrate=50000 rc-mode=1 ! \    # 根据需求调整 GOP 间隔
  video/x-h265 ! \
  h265parse config-interval=-1 ! \
  queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! \
  mux. alsasrc device=hw:0,0 buffer-time=50000 provide-clock=false slave-method=re-timestamp ! \   # 线路输入音频对应 hw:0,0
  audio/x-raw,rate=48000,channels=2,format=S16LE ! \
  queue max-size-buffers=0 max-size-time=500000000 max-size-bytes=0 ! \
  audioconvert ! \
  audioresample ! \
  avenc_aac bitrate=128000 ! \
  aacparse ! \
  queue max-size-buffers=0 max-size-time=500000000 max-size-bytes=0 ! \
  mux. mpegtsmux name=mux alignment=7 latency=100000000 ! \
  srtsink uri="srt://:8888" wait-for-connection=false latency=600 sync=false  # 可增加延迟以获得更流畅的直播体验
```


## HDR 10-bit 直播

当 HDMI 源输出 HDR10、HLG 或 HDR10+ 内容时，使用 Path A（vfmcap）保留原始 HDR 色彩信息：

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

与 SDR 管线的关键区别：
- `source=vfmcap output-format=p010` — 使用 Path A 原始 HDR 采集，GPU AMLY→P010 转换
- `format=P010_10LE` — Path A 输出的 10-bit 格式，保留原始 BT.2020 PQ 色彩信息
- `internal-bit-depth=10` — 启用 10-bit 编码；VUI（HDR10 BT.2020/PQ）由采集插件自动检测并写入
- 无需 `videorate` — streamboxsrc 原生处理帧率
- 建议使用更高的比特率（40000-50000 kbps）以保留 10-bit 渐变细节

可根据实际需求探索 GStreamer 功能，构建自定义管线。

### Path A 与 Path B 选择指南

| 特性 | Path A（`source=vfmcap`） | Path B（`source=vdin1`） |
|------|-------------------------|------------------------|
| 延迟 | 超低（无 VPP 处理） | 较高（VPP 处理 1-3 帧） |
| 色彩 | 原始，无处理 | HDR→SDR 转换，开箱即用 |
| 格式 | P010_10LE、NV12 | NV21（8-bit）、P010_10LE（10-bit） |
| HDR | 保留原始 HDR | VPP 色调映射为 SDR |
| 适用场景 | HDR 直播、原始采集 | SDR 直播、通用场景 |
