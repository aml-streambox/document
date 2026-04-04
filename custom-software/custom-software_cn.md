---
layout: page
title: 软件组件
---

[🇺🇸 English Version]({{ '/custom-software/custom-software_en' | relative_url }})

# 软件组件

## cockpit-gst-manager

适用于 Amlogic A311D2 的 GStreamer 直播/编码管线管理 Cockpit 插件。

- 仓库：https://github.com/aml-streambox/cockpit-gst-manager
- 平台：Amlogic A311D2、GStreamer 1.20.7、Cockpit 220

### 功能特性

- **多实例支持** - 同时运行多条 GStreamer 管线
- **AI 助手** - 自然语言转 GStreamer CLI（需自备 API 密钥）
- **手动编辑** - 面向高级用户的 CLI 直接编辑
- **HDR 10-bit 支持** - 自动检测 HDR 源，GPU 加速 Vulkan 转换生成对应管线
- **事件触发** - HDMI 信号、USB 插入、系统启动时自动启动
- **视频合成** - 通过 ge2d 硬件加速实现 OSD/叠加
- **导入/导出** - 分享管线配置
- **本地化** - 中英文界面

### 当前 v0.5.1 状态

- **自动采集恢复** — HDMI 自动采集管线在信号丢失、分辨率变化和瞬态错误后自动恢复，通过防抖重启和指数退避实现
- **无信号画面** — HDMI RX 断开或不稳定时，HDMI TX 显示弹跳的 "NO SIGNAL" / "STREAMBOX" 方框
- **双缓冲渲染** — 基于 fb0 的页面翻转，1080p 无闪烁，OSD 硬件缩放至 4K
- **直通稳定性** — HDMI RX → TX 视频直通不再因模式切换期间的 fb0 操作而损坏
- **自动启停** — RX 信号无效时自动启动无信号 UI，RX 信号恢复后自动关闭
- **Path A 独立性** — `vfmcap` 自动采集仅依赖 HDMI RX 稳定，不依赖 TX
- **四阶段关闭** — 对卡死编码管线执行 SIGUSR1→SIGINT→SIGTERM→SIGKILL
- **UVC 序列号追踪** — UVC 设备通过 USB 序列号进行持久识别，设备重连后自动恢复配置；设备连接时自动启动

### 之前的 v0.5 状态

- **Wave521 HEVC B 帧支持** - 已在目标硬件上完成验证，可稳定输出真实 B 帧
- **扩展 GOP 预设** - 已开放 `gop-pattern=0..8`，包含 `RA_IB` 在内的完整常用预设
- **HEVC 无损模式** - 已在当前编码栈中恢复可用
- **UVC 转码链路** - `UVC H.264 -> 硬件解码 -> 硬件 HEVC 编码 -> SRT` 已完成端到端验证
- **SRT 实际输出链路** - 已按生产使用方式验证通过，B 帧场景可正常工作

### 开发历程

- **HDR 10-bit 管线支持** - 自动检测 HDR 源并生成对应的 10-bit 或 8-bit 管线
- **自动管线实现** - 自动 GStreamer 管线配置
- **TVService 集成** - 使用 tvserver 检测 HDMI RX 信号变化
- **AI 提示词优化** - 优化提示词，移除录制功能
- **启动顺序修复** - 服务最后启动确保依赖就绪
- **v0.4 双路径采集架构** - 新增 `streamboxsrc` 插件替代 `v4l2src`，支持原始低延迟采集（Path A）和色彩处理采集（Path B）
- **v0.4 新显示模式** - 支持 2560x1440@60/120/144Hz、3440x1440@60Hz、1920x1080@144/240Hz、2560x1080@60Hz
- **采集撕裂修复** - Path A 加入内核一帧延迟，并在用户空间增加输出 buffer 池和 DMA_BUF_SYNC 同步
- **vfm_cap 稳定性修复** - 修复 `prev_v4l2_frame` 清理竞争问题，以及零拷贝 V4L2 回收路径中的 DMA-buf 生命周期错误
- **v0.5 Wave521 B 帧修复** - 修复启动延迟帧错误追加头信息所导致的假 I 帧输出与 B 帧时序错乱
- **v0.5 完整 GOP 预设开放** - 补齐 `IPP`、`IBBBP`、`IBPBP`、`IBBB`、`ALL_I`、`IPPPP`、`IBBBB`、`RA_IB`、`IPP_SINGLE`
- **v0.5 RA_IB 启动修复** - 补足深重排 GOP 启动所需的源缓冲与采集缓冲深度
- **v0.5 UVC 集成** - 新增面向 Cockpit 管理的 UVC H.264 硬件解码/转码链路
- **v0.5.1 无信号 UI** - 基于 framebuffer 的弹跳方框渲染器，双缓冲，由 HDMI RX 信号状态触发
- **v0.5.1 直通修复** - 从 TX 模式同步中移除 fb0 重置，防止 VPP 合成器损坏
- **v0.5.1 4K 安全** - 始终以 1080p 渲染并使用 smem_len 保护，OSD 硬件缩放负责放大
- **v0.5.1 自动采集恢复** - 基于实例退出的重建与重启机制，在 HDMI 信号变化后通过防抖和退避自动恢复
- **v0.5.1 Path A 独立性** - `vfmcap` 自动采集不再依赖 HDMI TX 就绪
- **v0.5.1 四阶段关闭** - 对卡死编码管线使用 SIGUSR1→SIGINT→SIGTERM→SIGKILL
- **v0.5.1 启动看门狗** - 长时间停留在 STARTING 的实例会被自动中止并重试
- **v0.5.1 UVC 序列号追踪** - UVC 设备通过 USB 序列号进行持久识别，配置在设备重连后自动恢复；设备连接时自动启动

## streamboxsrc GStreamer 插件

统一的 GStreamer 视频源插件（`streamboxsrc`），替代已弃用的 `v4l2src` 采集方式。完整技术细节见[双路径 HDMI 采集架构]({{ '/custom-software/streambox_v0.4' | relative_url }})。

- **Path A（vfmcap）**：原始采集，超低延迟，无色彩处理，支持 P010/NV12 输出
- **Path B（vdin1）**：色彩处理采集，VPP HDR→SDR 转换，开箱即用
- **零拷贝安全性**：Path A 已加入显式 DMA-buf 生命周期管理，以及输出缓冲的 acquire/release 跟踪机制，可避免长时间推流时出现撕裂和 use-after-free。
- **深重排安全性**：Path A 的 DMA-buf 缓冲池深度已补足，可支撑 `RA_IB` 等深重排 GOP 预设的启动过程。

基本用法：
```bash
# SDR 采集（Path B）— 色彩处理，可直接直播
streamboxsrc source=vdin1 ! video/x-raw,format=NV21,width=3840,height=2160 ! amlvenc ...

# HDR 10-bit 采集（Path A）— 保留原始 HDR 色彩信息
streamboxsrc source=vfmcap output-format=p010 ! video/x-raw,format=P010_10LE,width=3840,height=2160 ! amlvenc internal-bit-depth=10 ...
```

> **注意**：`v4l2src` 采集方式已**弃用**，请使用 `streamboxsrc` 进行视频采集。完整管线示例见[配置指南]({{ '/setup-guide/setup-guide_cn' | relative_url }})。

## Wave521 编码器 v0.5

Wave521 硬件编码器是 StreamBox v0.5 媒体链路的核心组件。

当前已验证状态：
- **HEVC B 帧** - 正常工作
- **HEVC 无损** - 正常工作
- **扩展 GOP 预设** - 正常工作
- **带 B 帧的 SRT 输出** - 正常工作

已验证的 `gop-pattern` 值：

| 值 | 结构 | 状态 |
|------|---------|--------|
| 0 | IPP | 已验证 |
| 1 | IBBBP | 已验证 |
| 2 | IBPBP | 已验证 |
| 3 | IBBB | 已验证 |
| 4 | ALL_I | 已验证 |
| 5 | IPPPP | 已验证 |
| 6 | IBBBB | 已验证 |
| 7 | RA_IB | 已验证 |
| 8 | IPP_SINGLE | 已验证 |

v0.5 关键修复：
- 修复 B 帧启动延迟阶段产生假的 `SPS/PPS-only` 输出的问题
- 删除插件侧重复追加 HEVC 头信息的逻辑
- 增加 `streamboxsrc` Path A 的缓冲池深度，以支持深重排 GOP 的启动阶段
- 增加 Wave521 用户态 source slot 数量，使其与命令队列深度保持一致

推荐快速检查命令：
```bash
# 检查码流中的帧类型
ffprobe -show_frames -select_streams v -show_entries frame=pict_type -of csv /tmp/out.h265

# 检查是否存在明显的倒回闪帧或重排错误
python3 scripts/analyze_neighbor_frames.py /tmp/out.h265
```

完整设计和排查记录见 [StreamBox v0.5]({{ '/custom-software/streambox_v0.5' | relative_url }})。

## HDMI 自动采集恢复（v0.5.1）

当 HDMI 信号变化或采集管线异常退出时，`cockpit-gst-manager` 现在可以自动恢复：

- **基于退出的恢复** — `streamboxsrc` 发出 `hdmi-signal-change` 后以 EOS 退出；
  管理器解析退出细节，并使用更新后的参数执行防抖重启
- **Path A 独立性** — `vfmcap` 自动采集仅依赖 HDMI RX 稳定，不依赖 TX
- **Path B TX 门控** — `vdin1` 自动采集需要同时满足 RX 稳定和 TX 就绪
- **瞬态错误重试** — 网络断开、超时、buffer underrun 等错误会自动重试
- **四阶段关闭** — 对卡死编码管线执行 SIGUSR1→SIGINT→SIGTERM→SIGKILL
- **启动看门狗** — 长时间停留在 STARTING 的实例会被自动中止并重试

技术细节：[StreamBox v0.5.1]({{ '/custom-software/streambox_v0.5.1' | relative_url }})

## HDMI TX 无信号画面（v0.5.1）

当 HDMI RX 没有有效信号时，StreamBox 现在会在 HDMI TX 上显示可见的无信号画面，
而不是纯黑输出。

核心特性：
- **弹跳方框** — "NO SIGNAL"（红色边框）和 "STREAMBOX"（蓝色边框）在黑色背景上以 DVD 风格弹跳
- **双缓冲** — 通过 `FBIOPAN_DISPLAY` 页面翻转消除可见闪烁
- **4K 安全** — 以 1080p 渲染，依赖 OSD 硬件缩放填满 4K 输出
- **直通无干扰** — fb0 操作不会干扰活跃的 HDMI RX → TX 视频直通路径
- **自动管理** — RX 信号无效时启动，RX 信号恢复时关闭，暂停音频直通

技术细节：[StreamBox v0.5.1]({{ '/custom-software/streambox_v0.5.1' | relative_url }})

## UVC 设备序列号追踪（v0.5.1）

UVC 设备管线现在通过 USB 序列号而非设备路径进行追踪：

- **持久识别** — 设备通过 USB 序列号识别，而非 `/dev/videoX` 路径
- **连接自动启动** — 已配置的实例在设备插入时自动启动
- **热插拔感知** — 设备连接/断开事件触发相应的启动/停止操作
- **配置保留** — 即使路径变化，保存的设置仍能应用到正确的设备

技术细节：[StreamBox v0.5.1]({{ '/custom-software/streambox_v0.5.1' | relative_url }})

## UVC 转码

在 v0.5 中，UVC 管线已不再停留在基础设备发现阶段，当前可用于实际转码输出的链路为：

`v4l2src (UVC H.264) -> h264parse -> amlv4l2h264dec -> amlvenc -> h265parse -> mpegtsmux -> srtsink`

当前状态：
- 硬件解码与重新编码链路工作正常
- 基于 DMA-buf 的硬件编码路径工作正常
- SRT 输出已验证可用
- 该链路的目标用法是由 `cockpit-gst-manager` 统一编排和管理

## 双路径 HDMI 采集架构

StreamBox v0.4 引入了全新的双路径 HDMI 采集架构，包含自定义内核模块（`vfm_cap`）、用户空间 SDK（`libvfmcap`）和统一 GStreamer 插件（`streamboxsrc`）。

完整技术文档：[双路径 HDMI 采集架构]({{ '/custom-software/streambox_v0.4' | relative_url }})

关键组件：
- **vfm_cap** — 内核模块，拦截 VFM 管线实现零拷贝帧采集（`/dev/video_cap`）
- **libvfmcap** — 用户空间 SDK，Vulkan GPU 加速 AMLY→P010/NV12 格式转换
- **streamboxsrc** — 统一 GStreamer 源插件，支持 Path A（原始低延迟）和 Path B（色彩处理）
- **amlvenc** — 编码器插件增强：VBR/CBR 修复、色彩度/VUI 信号、0-240fps 支持

近期采集路径修复：
- **撕裂修复** — Path A 在内核侧加入一帧延迟，确保 V4L2 消费者只读取已完整写入的帧
- **输出 buffer 池修复** — streamboxsrc 改为 acquire/release 跟踪，而不是简单 round-robin 复用
- **GPU 同步修复** — libvfmcap 在 Vulkan 写入前后增加 DMA_BUF_SYNC，并在释放 buffer 前等待 GPU 完成
- **DMA-buf 生命周期修复** — vfm_cap 在 `dma_buf_fd()` 后额外保留一份 dma_buf 引用，避免 QBUF 清理命中已释放对象

## v0.4 新显示模式

StreamBox v0.4 新增以下 HDMI 显示模式支持：

| 模式 | 分辨率 | 刷新率 |
|------|--------|--------|
| 1440p | 2560x1440 | 60Hz、120Hz、144Hz |
| 超宽 | 3440x1440 | 60Hz |
| 1080p 高刷 | 1920x1080 | 144Hz、240Hz |
| 超宽 FHD | 2560x1080 | 60Hz |

变更涉及内核 HDMI RX/TX 驱动、tvin/vdin 子系统、tvserver EDID 管理、GStreamer 插件和 Cockpit UI。详见[架构文档]({{ '/custom-software/streambox_v0.4' | relative_url }})。

## cockpit-streambox-settings

适用于 Amlogic A311D2 Streambox 的系统设置管理 Cockpit 插件。

- 仓库：https://github.com/aml-streambox/cockpit-streambox-settings
- 平台：Amlogic A311D2、Cockpit 220

### 功能特性

- **基础设置** - 设备名称、时区、系统区域设置
- **网络设置** - 有线、WiFi 客户端、WiFi 热点配置
- **TVServer 设置** - 视频、音频、HDCP、调试配置
- **存储设置** - SD 卡管理、格式化、挂载点

### 系统要求

- Amlogic A311D2/T7 Streambox 硬件
- 集成 Cockpit 的 Yocto 镜像
- 运行中的 tvservice (aml_tvserver_streambox)
