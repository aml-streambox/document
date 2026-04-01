---
layout: page
title: 驱动改动
---

[🇺🇸 English Version]({{ '/driver-modifications/driver-modifications_en' | relative_url }})

# 驱动与软件改动

本文档记录了对 Khadas VIM4 Yocto 原始源码的各项修改。

## U-Boot 改动

### TVPro 开发板支持

新增 TVPro（Streambox）设备支持，基于 A311D2 SoC。

## 内核改动

### 视频编码器超频

编码器默认频率为 500MHz，仅支持 4K50fps。超频至 666MHz 后可实现 4K60fps 编码（实际可略高于 60fps）。

### 视频环回缓冲区扩展

增大 VDIN1 CMA 缓冲区，支持最高 4K HDR 帧处理。原始缓冲区仅支持 1080p。

### HDR 编码支持

硬件编码器支持 10-bit HDR 编码。`v4l2src` 采集方式已弃用，请使用 `streamboxsrc` 插件进行 HDR 10-bit 采集。详见[双路径 HDMI 采集架构]({{ '/custom-software/streambox_v0.4' | relative_url }})。HDR 透传功能正常，HDR 直播通过 `streamboxsrc source=vfmcap output-format=p010` 实现。

### common_drivers 子模块

**关键修复：**

#### 帧率限制移除

移除硬编码的 60fps 帧率上限。系统现支持 2K120fps 和 1080p240Hz。然而 VIC 定义暂不支持 1080p240Hz，因此目前仅支持 1080p120Hz。

`drivers/media/vout/hdmitx21/hw/hdmi_tx_hw.c` 代码修改：

```c
case MESON_CPU_ID_T7:
    // T7 支持：4k60、2k120、1080p240
    ret = (soc_resolution_limited(timing, 4320) && soc_freshrate_limited(timing, 60)) ||
           (soc_resolution_limited(timing, 2160) && soc_freshrate_limited(timing, 120)) ||
           (soc_resolution_limited(timing, 1080) && soc_freshrate_limited(timing, 240));
    break;
```

#### HDMI RX 黑屏修复

从 Khadas common_drivers 修复移植。解决 HDMI 输入分辨率或帧率变化时的黑屏问题。

修改文件：
- `drivers/media/video_sink/video_hw.c`
- `drivers/media/video_sink/video_priv.h`

该修复添加了对 VPP 静音状态的准确追踪，避免 RX 静音导致 TX 输出也被静音。

```c
// 新增静音状态追踪枚举
enum video_vpp_mute_type {
    VIDEO_BE_MUTED,
    VIDEO_BE_UNMUTED,
    VPP_BE_MUTED,
    VPP_BE_UNMUTED,
};
```

相关的原始 Khadas 修复：`hdmirx: adjust vpp mute cnt` (PD#SWPL-156133)

#### 采集路径画面撕裂修复（Game Mode 2）

在使用 amlvenc（H.265）对 vdin0 采集输出进行编码时，画面顶部约 50-145 行会出现水平撕裂。该问题出现在 4K60、game mode 2 和 VRR 同时开启的场景。

**根因：** 在 game mode 2 下，vdin ISR 交付的是 `next_wr_vfe`，而这个 buffer 在交付时尚未真正写入（RDMA 写入要等到下一个 VSYNC）。V4L2 采集路径读取到了过期内容，因此出现撕裂。

**修复分为 3 层：**

1. **vdin_drv.c：** 将 game mode 2 ISR 的交付对象从 `next_wr_vfe`（过期 buffer）改为 `curr_wr_vfe`（当前正在被 vdin 写入的 buffer）。VPP 仍然通过 `line_dly` 偏移读取同一个 buffer，因此显示侧不增加额外帧延迟。同时关闭 one-buffer 模式（`dbg_force_one_buffer=2`），保证每个 buffer 保留独立物理地址。

2. **vfm_cap 一帧延迟：** 在 vfm_cap 的 VFRAME_READY 处理中加入 `prev_v4l2_frame` 保留逻辑。当第 N 帧到达时，交付给 V4L2 消费者的是第 N-1 帧（已经完整写完），当前帧保留到下一次 ISR 再交付。

3. **用户空间加固：** streamboxsrc 改为 acquire/release 输出 buffer 池；libvfmcap 在 Vulkan GPU 输出前后增加 DMA_BUF_SYNC 写访问同步。

**相关提交：**
- `common_drivers @ 9ee3fb8c7` — 内核侧修复
- `multimedia @ 120b4f2` — 用户空间修复

#### vfm_cap 崩溃修复（prev_v4l2_frame 清理路径）

在部署撕裂修复后，系统出现随机崩溃，原因是多个 `prev_v4l2_frame` 清理路径实现错误。

**根因：** 多个清理路径（PROVIDER_START/UNREG/RESET、late-start、drain、release、module exit）直接对 `prev_v4l2_frame` 调用 `frame_release()`，但没有先递减 refcount，或者只是把指针设为 NULL 而没有释放 held ref。这样会导致显示路径后续再次对已经释放的 frame 调用 `vf_put()`，形成 double `vf_put`，最终破坏 VFM 链表并触发内核崩溃。

**修复：** 新增 `frame_pool_recycle_all()`，在 `ready_lock` 保护下统一完成以下操作：清空 `prev_v4l2_frame`、对所有 in-use vframe 调用 `vf_put()` 归还给 vdin0、把 refcount 清零、并重新初始化链表头。8 条清理路径全部改为使用这个统一函数。

同一提交中的附加修复：
- `vfm_cap_drain_pending()`：把 refcount 递减移到 `ready_lock` 内部，避免与 ISR 上下文中的 `frame_pool_recycle_all()` 竞争
- `vfm_cap_exit()`：把 pool flush 移到 `vf_unreg_receiver()` 之前，确保 `vf_put()` 仍然走已注册的 receiver 路径

**提交：** `common_drivers @ ddf215410`

#### vfm_cap DMA-buf Use-After-Free 修复

KFENCE 检测到 V4L2 QBUF（buffer 重新入队）阶段存在 dma_buf use-after-free。

**根因：** `dma_buf_fd()` 会消费掉 dma_buf 的引用，fd 成为唯一持有者。当用户空间在导入 Vulkan/编码器后关闭 fd 时，dma_buf 被释放，但 `buf->dbuf` 仍然指向已释放对象。后续在 QBUF 中，`vfm_cap_buf_queue()` 调用 `dma_buf_put(buf->dbuf)` 做清理，就会触发 use-after-free 并导致内核崩溃。

**修复：** 在 `dma_buf_fd()` 成功后调用 `get_dma_buf(buf->dbuf)`，使 fd 和 `buf->dbuf` 分别各自持有一份独立引用。用户空间关闭 fd 只会释放其中一份，我们在 cleanup 路径中释放另一份。

**提交：** `common_drivers @ f6ae75fac`

#### 禁用 AFBCE / Double-Write（960x540 缩放问题）

在 vdin0 上禁用 AFBCE（Amlogic Frame Buffer Compression）和 double-write，修复 960x540 分辨率下出现损坏帧的问题。AFBCE 虽然能降低带宽，但与某些采集配置不兼容。

**提交：** `common_drivers @ cf76bc55a`

#### VRR 支持修复

系统内置游戏模式支持同时写入当前帧并读取前一帧。但由于 HDMI RX 和 HDMI TX 连接不同物理设备（如 PS5 和电视），晶振频率存在微小差异，导致时序漂移，最终引发缓冲区读写冲突，造成画面撕裂或卡顿。

通过 VRR（可变刷新率）解决，即使 RX/TX 之一或两者均不支持原生 VRR。VRR 同时获取输入输出时钟，微调输出 VBLANK 以保持 RX 和 TX 相位同步。这使得系统能以行级延迟同时写入和读取同一帧，实现超低延迟 HDMI 视频透传，延迟降低至数毫秒级。

**实施的修复：**

##### 强制 VRR 低延迟功能

强制启用 VRR，即使 RX/TX 之一或两者不支持原生 VRR。当 RX 和 TX 均支持 VRR 时，无需强制即可自然启用。

##### HDMI TX VRR EMP 包传输修复

原始驱动实现中，HDMI TX VRR EMP（扩展元数据包）无法由 HDMI RX 触发。已修复此问题，支持通过 EMP 包进行正确的 VRR 信号传输。

##### 音频环回设备更新

修改设备树（DTS），配置独立的环回设备接收 HDMI RX 音频输入。这使得：
- tvserver 可专门使用 HDMI RX 音频进行透传
- GStreamer 视频直播管线仍可通过环回设备获取 HDMI RX 音频

这种分离实现同一音频源的同时透传和采集/编码。

### GStreamer v4l2src Plugin (Deprecated)

> **Note:** `v4l2src device=/dev/video71` is **deprecated**. It has been replaced by the `streamboxsrc` plugin which provides dual-path capture (Path A: raw HDR and Path B: color-processed SDR via `/dev/video_cap`. See [Custom Software]({{ '/custom-software/custom-software_en' | relative_url }}) for details.

 The original `v4l2src` had HDMI RX control logic removed to avoid conflicts with tvserver.

 `streamboxsrc` auto-detects signal source and handles HDMI signal changes cleanly.



### GStreamer amlvenc multienc 插件

公开硬件支持但此前未向用户开放的高级编码功能：

- **B 帧支持（gop_pattern）**：启用 B 帧编码以获得更高压缩率。`gop-pattern` 属性为 API 兼容性保留，但实际不生成 B 帧（编码器初始化中 GopPresetis=0x0）。
- **码率控制模式选择**：支持 CBR（恒定码率）、VBR（可变码率）及固定 QP 模式
- **HEVC 无损编码选项**：硬件支持无损 HEVC 编码

## TVServer 改动

### aml_tvserver_streambox

从原始 aml-tvserver 分叉并修改，创建 aml_tvserver_streambox。

- 仓库：`git@github.com:aml-streambox/aml_tvserver_streambox.git`
- 分支：streambox_v0.2
- 从 yocto-kirkstone-202406-smarthome 版本导入原始源码

**关键改动：**

#### 低延迟与游戏模式
- 默认启用低延迟模式，实现最小透传延迟
- 默认设置游戏模式为 2，实现帧读写同步
- 改进 frac_rate_policy 处理

#### 分辨率与帧率支持
- 新增 HDMI RX/TX 分辨率/帧率跟随功能
- 新增 1080p120Hz 和 1440p120Hz 接收支持
- 默认 HDMI 2.0 模式

#### VRR 支持
- 使用自定义强制 VRR 功能新增 VRR 支持
- 针对不同模式优化 VRR 处理
- 修复 TX 端 VRR EMP 包传输

#### 音频透传
- 新增音频透传支持并增大缓冲区
- 使用传统音频路径解决卡死问题
- 调整音频重启时序提升稳定性

#### 视频管线
- 新增 HDMI RX 分辨率/帧率/HDR 状态变化时的视频管线重置
- 修复色彩位深透传问题
- 更新 HDR 处理

#### 配置与鲁棒性
- 为灵活设置新增配置驱动设计
- 新增 HDMI EDID 透传功能
- 改进 HDMI TX 断开事件处理
- 新增线缆插拔场景的重置逻辑

**注意**：尽管已进行多项改进，HDMI 热插拔时仍可能出现绿屏、黑屏或系统崩溃。
