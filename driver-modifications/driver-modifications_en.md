---
layout: page
title: Driver Modifications
permalink: /driver-modifications/
---

[🇨🇳 中文版]({{ '/driver-modifications/driver-modifications_cn' | relative_url }})

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

The capture pipeline now uses the dual-path architecture with `streamboxsrc` (the old `v4l2src` method is deprecated):

```
Path A (raw HDR):   HDMI RX → vdin0 → vfm_cap (/dev/video_cap) → streamboxsrc → amlvenc → H.265
Path B (SDR):       HDMI RX → vdin0 → VPP → vdin1 → streamboxsrc → amlvenc → H.265
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

### common_drivers Submodule

**Key fixes:**

#### Frame Rate Limitation

Removed hardcoded 60fps maximum frame rate limitation. The system now supports 2K120fps and 1080p240Hz. However, the VIC definition does not support 1080p240Hz yet, so only 1080p120Hz is achievable currently.

Code change in `drivers/media/vout/hdmitx21/hw/hdmi_tx_hw.c`:

```c
case MESON_CPU_ID_T7:
    // Allow all resolutions above 4K60
    break;
```

#### HDMI RX Black Screen Fix

Ported from Khadas common_drivers fix. Resolves black screen on HDMI input resolution or framerate change.

Modified files:
- `drivers/media/video_sink/video_hw.c`
- `drivers/media/video_sink/video_priv.h`

The fix adds accurate tracking of VPP mute state, preventing RX mute from causing TX output to also be muted.

```c
// New mute state tracking enum
enum video_vpp_mute_type {
    VIDEO_BE_MUTED,
    VIDEO_BE_UNMUTED,
    VPP_BE_MUTED,
    VPP_BE_UNMUTED,
};
```

Related original Khadas fix: `hdmirx: adjust vpp mute cnt` (PD#SWPL-156133)

#### Capture Path Tearing Fix (Game Mode 2)

When encoding the vdin0 capture output via amlvenc (H.265), horizontal tearing appeared in the top ~50-145 rows of each frame. This occurred at 4K60 with game mode 2 + VRR active.

**Root cause:** In game mode 2, the vdin ISR delivers `next_wr_vfe` — a buffer that hasn't been written to yet (RDMA write is deferred to the next VSYNC). The V4L2 capture path reads stale data from this buffer, causing tearing.

**Fix (3 layers):**

1. **vdin_drv.c:** Changed game mode 2 ISR delivery from `next_wr_vfe` (stale) to `curr_wr_vfe` (buffer vdin is currently writing). VPP continues reading this same buffer with `line_dly` offset — zero additional display latency. Also disabled one-buffer mode (`dbg_force_one_buffer=2`) so each buffer retains its unique physical address.

2. **vfm_cap one-frame delay:** Added `prev_v4l2_frame` hold-back in vfm_cap's VFRAME_READY handler. When frame N arrives from vdin, frame N-1 (guaranteed fully written) is delivered to V4L2 consumers. The current frame is held until the next ISR.

3. **Userspace defense in depth:** streamboxsrc uses acquire/release output buffer pool; libvfmcap adds DMA_BUF_SYNC write brackets around Vulkan GPU output.

**Commits:**
- `common_drivers @ 9ee3fb8c7` — kernel fixes
- `multimedia @ 120b4f2` — userspace fixes

#### vfm_cap Crash Fix (prev_v4l2_frame Cleanup)

After deploying the tearing fix, random system crashes occurred due to broken `prev_v4l2_frame` cleanup in multiple paths.

**Root cause:** Multiple cleanup paths (PROVIDER_START/UNREG/RESET, late-start, drain, release, module exit) called `frame_release()` directly on `prev_v4l2_frame` without decrementing its refcount, or set the pointer to NULL without releasing the held reference. This caused the display path to call `vf_put()` on an already-released frame → double `vf_put` → VFM list corruption → kernel crash.

**Fix:** Introduced `frame_pool_recycle_all()` — a single function that atomically (under `ready_lock`) clears `prev_v4l2_frame`, returns all held vframes to vdin0 via `vf_put()`, resets all refcounts to 0, and reinits list heads. All 8 cleanup paths now use this centralized function.

Additional fixes in the same commit:
- `vfm_cap_drain_pending()`: moved refcount decrements inside `ready_lock` to prevent race with `frame_pool_recycle_all()` from ISR context
- `vfm_cap_exit()`: moved pool flush before `vf_unreg_receiver()` so `vf_put()` calls go through the still-registered receiver

**Commit:** `common_drivers @ ddf215410`

#### vfm_cap DMA-buf Use-After-Free Fix

KFENCE detected a use-after-free on dma_buf objects during V4L2 QBUF (buffer re-queue).

**Root cause:** `dma_buf_fd()` consumes the dma_buf reference — the fd becomes the sole owner. When userspace closes the fd (after importing into Vulkan/encoder), the dma_buf is freed. But `buf->dbuf` still pointed to the freed object. On QBUF, `vfm_cap_buf_queue()` called `dma_buf_put(buf->dbuf)` for cleanup → use-after-free → kernel crash.

**Fix:** Added `get_dma_buf(buf->dbuf)` after `dma_buf_fd()` succeeds, so both the fd and `buf->dbuf` each hold an independent reference. Userspace closing the fd drops one ref, our cleanup drops the other.

**Commit:** `common_drivers @ f6ae75fac`

#### AFBCE/Double-Write Disable (960x540 Downscaling)

Disabled AFBCE (Amlogic Frame Buffer Compression) and double-write on vdin0 to fix a downscaling bug where 960x540 resolution produced corrupted frames. AFBCE compresses frame buffers for bandwidth savings but is incompatible with certain capture configurations.

**Commit:** `common_drivers @ cf76bc55a`

### GStreamer streamboxsrc Plugin (replaces v4l2src)

The old `v4l2src` plugin has been replaced by the unified `streamboxsrc` plugin. The old plugin had HDMI RX control logic that conflicted with tvserver, and lacked proper signal change handling.

**`streamboxsrc`** is a unified GStreamer source element (~3430 lines of C) supporting both capture paths:
- **Path A (`source=vfmcap`)**: Raw low-latency capture via vfm_cap kernel module, with GPU AMLY→P010/NV12 conversion
- **Path B (`source=vdin1`)**: Color-processed capture via vdin1 VPP writeback, with signal change monitoring via vfm_cap events

Key features over the old `v4l2src`:
- Auto-detects framerate from HDMI RX sysfs (up to 240Hz)
- Event-driven signal change monitoring (no polling)
- Clean exit on signal change with HDMI info message for external pipeline manager
- Proper colorimetry/VUI signaling (BT.2020 PQ, BT.2100 HLG, BT.709)
- Zero-copy DMA-buf throughout, CPU never touches frame data

See [Dual-Path HDMI Capture Architecture]({{ '/custom-software/streambox_v0.4' | relative_url }}) for full details.

### GStreamer amlvenc multienc Plugin

Exposed additional encoder features that existed in hardware but were not available to users:

- **B-frame support (gop_pattern)**: Enable B-frame encoding for better compression. The `gop-pattern` property is accepted for API compatibility but B-frames are not actually produced (GopPreset=0x0 in encoder initialization).
- **RC mode selection**: Support for CBR (Constant Bitrate), VBR (Variable Bitrate), and constant QP modes
- **10-bit HDR encoding (internal-bit-depth)**: Enables 10-bit encoding with GPU-accelerated YUV422-to-P010 conversion via Vulkan compute shader or GLES backend
- **Conversion backend selection (v10conv-backend)**: Choose between Vulkan (0, default) and GLES (1) for 10-bit format conversion
- **HDR10 VUI signaling**: Dynamic colorimetry from capture plugin caps (BT.2020/PQ, HLG, HDR10+, SDR) — propagated to H.265 bitstream when `internal-bit-depth=10` is set
- **HEVC lossless encode option**: Hardware supports lossless HEVC encoding (currently has a bug -- see FAQ)
- **New display modes (v0.4)**: 2560x1440@60/120/144Hz, 3440x1440@60Hz, 1920x1080@144/240Hz via new HDMI RX/TX VIC enums and relaxed DTD matching
- **VRR support**: Force VRR for stable passthrough even when devices don't support VRR natively

### vfm_cap Kernel Module

A custom kernel module that intercepts the VFM (Video Frame Manager) pipeline to provide zero-copy frame capture from vdin0. Exposes `/dev/video_cap` as a V4L2 capture device with:
- Per-frame reference counting for safe multi-consumer buffer sharing
- DMA-buf export of vdin0 CMA buffers for zero-copy GPU access
- V4L2 SOURCE_CHANGE events for HDMI signal state changes
- Adaptive zero-copy with copy-fallback when consumers are slow

See [Dual-Path HDMI Capture Architecture]({{ '/custom-software/streambox_v0.4' | relative_url }}) for full details.

### libvfmcap SDK

A userspace library that wraps `/dev/video_cap` V4L2 operations and provides GPU-accelerated raw format conversion via Vulkan compute shaders:
- AMLY 10-bit YUV422 → P010 (10-bit) or NV12 (8-bit)
- Pure format packing — no color space conversion or tone mapping
- Pipelined async API for overlapping GPU work with frame acquisition
- Performance: 59.94fps at 4K60 for both P010 and NV12

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
