# StreamBox v0.4 — Dual-Path HDMI Capture Architecture

## 1. Problem Statement

The current video capture path routes through VPP writeback:

```
hdmirx -> vdin0 (VFM) -> deinterlace -> amvideo -> HDMI TX
                                            |
                                      VPP writeback
                                            |
                                      vdin1 (V4L2, /dev/video71)
                                            |
                                      GStreamer v4l2src -> amlvenc -> SRT
```

**Issues:**
- 1-3 frames of added latency from VPP writeback path
- Extra hardware copy (vdin1 re-captures VPP output)
- Two vdin hardware blocks used when one suffices
- ~280 MB CMA consumed by vdin1 that could be reclaimed
- No standard event mechanism -- consumers rely on tvserver Binder IPC polling
- vdin1 doesn't handle hdmirx signal change events (it's in the video output domain)
- When VPP resolution/framerate changes, vdin1 doesn't auto-restart

## 2. Target Architecture: Dual Capture Paths

The A311D2's TruLife Image Engine (VPP) already performs HDR10 PQ->SDR tone mapping,
3D LUT (17x17x17), gamut mapping, and full color management in the display pipeline.
The vdin1 loopback captures this VPP post-processed output with correct colors.

This means we need TWO capture paths, not one:

```
HDMI Source --> hdmirx0 --> vdin0 (VFM mode)
                              |
                     ISR fires, frame complete
                              |
                     VFM chain:
                     vdin0 --> vfm_cap --> deinterlace --> amvideo --> HDMI TX
                                 |                                      |
                           [Path A]                               VPP (TruLife)
                         Raw capture                             HDR->SDR, gamut,
                         Ultra low latency                       3D LUT, etc.
                         No color processing                          |
                                |                               vdin1 loopback
                       /dev/video_cap                            [Path B]
                                |                           Color-processed capture
                     libvfmcap SDK                          Good looking frames
                     AMLY -> P010/NV12                      Ready out of the box
                     (raw format packing)                         |
                                |                          /dev/video71
                                |                                |
                     +----------+-----------+          +---------+---------+
                     |          |           |          |                   |
                 GStreamer   PiKVM     Sunshine     GStreamer          Other
                (streambox-src)                   (streambox-src)    consumers
```

### Path A: vfm_cap (Raw / Low Latency)

- **Source:** vdin0 CMA buffers via VFM chain interception
- **Format:** Raw AMLY 10-bit YUV422, converted to P010 or NV12 via GPU
- **Color:** No color processing -- output preserves original BT.2020 PQ colorimetry
- **Latency:** Minimal (no VPP writeback, no vdin1 re-capture)
- **Use case:** Consumer is responsible for color tuning, or raw data leaves the
  A311D2 system entirely (e.g., sent to a cloud transcoder or external processor)
- **CPU:** Never touches pixel data -- ARM CPU does control plane only

### Path B: vdin1 (Color-Processed / Ready to Use)

- **Source:** VPP post-processed output via vdin1 loopback
- **Format:** NV21 8-bit (VPP output after HDR->SDR conversion)
- **Color:** Full hardware color pipeline applied -- HDR10 PQ->SDR tone mapping,
  BT.2020->BT.709 gamut mapping, 3D LUT, EOTF/OETF conversion
- **Latency:** Higher (VPP writeback + vdin1 re-capture adds 1-3 frames)
- **Use case:** Consumer wants good-looking frames out of the box, doesn't care
  about the extra latency

### Why Not GPU HDR Shaders?

We tried implementing HDR10 PQ->SDR tone mapping as a Vulkan compute shader on the
Mali-G52 MC1. It was ALU-bound at ~36.6fps for 4K60 (~80-100 FLOPs/pixel for PQ
EOTF + gamut matrix + Reinhard tonemap + gamma). Since the VPP hardware already does
this in the display path at full rate with zero CPU/GPU cost, and vdin1 captures that
output, there is no reason to duplicate this work on the GPU.

There is NO standalone M2M hardware HDR engine on A311D2 (VICP only exists on S5/T3X
SoCs), but the existing display-path VPP + vdin1 loopback achieves the same result.

## 3. Signal Event Model

### 3.1 Two-Tier Event Architecture

The kernel signal state machine (in `vdin_sm.c`) has two tiers of parameter changes.
The `vfm_cap` module must handle both correctly.

**Tier 1 -- Full pipeline restart (UNSTABLE -> STABLE cycle):**

These changes cause the hdmirx FSM to detect timing instability, forcing
`STABLE -> UNSTABLE -> PRESTABLE -> STABLE` with ALL parameters re-read.

| Change | Detection Layer |
|--------|----------------|
| Resolution (hactive/vactive) | hdmirx `rx_is_timing_stable()` |
| Htotal/Vtotal | hdmirx timing check |
| Frame rate (large change) | hdmirx `diff_frame_th` |
| Interlaced <-> progressive | hdmirx direct comparison |
| Color depth (8/10/12 bit) | hdmirx `COLOR_DEP_EN` |
| DVI <-> HDMI mode | hdmirx `DVI_EN` |
| TMDS link loss | hdmirx `is_tmds_valid()` |
| Colorspace (RGB <-> YUV) | hdmirx `rx_is_avi_stable()` (T7+) |
| HDR on/off, HDR10+ | Detected while STABLE, then `re_cfg` forces UNSTABLE |
| Dolby Vision on/off | Detected while STABLE, then `re_cfg` forces UNSTABLE |
| 5V HPD loss | hdmirx `rx_nosig_monitor()` |

**Tier 2 -- In-STABLE events (informational, no pipeline restart):**

Detected by `vdin_hdmirx_fmt_chg_detect()` while the SM stays STABLE.
They generate `TVIN_SIG_CHG_*` event flags but do NOT restart the pipeline:

| Change | Event Flag | Notes |
|--------|-----------|-------|
| FPS (small change) | `TVIN_SIG_CHG_VS_FRQ` | e.g., VRR frequency drift |
| Aspect ratio | `TVIN_SIG_CHG_AFD` | |
| ALLM mode | `TVIN_SIG_CHG_DV_ALLM` | Game mode / filmmaker mode |
| IT content type / CN type | `TVIN_SIG_CHG_DV_ALLM` | |
| VRR state | `TVIN_SIG_CHG_VRR` | FreeSync/G-Sync enable/disable |
| QMS state | `TVIN_SIG_CHG_QMS` | Quick Media Switching |
| Color format/range | (inline CSC reconfig) | No event flag generated |

### 3.2 V4L2 Event Mapping (vfm_cap)

The `vfm_cap` module translates kernel signal changes into standard V4L2 events:

| Kernel Signal | V4L2 Event | Consumer Action |
|---------------|-----------|-----------------|
| STABLE (new params) | `V4L2_EVENT_SOURCE_CHANGE` | Drain, reconfigure caps, restart |
| NOSIG | `V4L2_EVENT_SOURCE_CHANGE` (status=nosig) | Drain, pause, wait |
| NOTSUP | `V4L2_EVENT_SOURCE_CHANGE` (status=notsup) | Same as NOSIG |
| UNSTABLE | No event | Module internally drains; consumers see nothing |
| VRR change | `V4L2_EVENT_PRIVATE_START + 1` | Informational |
| ALLM change | `V4L2_EVENT_PRIVATE_START + 2` | Informational (e.g., switch encoding mode) |
| FPS change | `V4L2_EVENT_PRIVATE_START + 3` | Informational |
| HDR metadata | Embedded in vframe `signal_type` | Per-frame metadata, no separate event |

### 3.3 SOURCE_CHANGE Event Payload

```c
struct vfm_cap_signal_info {
    __u32 width;
    __u32 height;
    __u32 fps;            /* frames per second */
    __u32 color_format;   /* V4L2_PIX_FMT_* */
    __u32 signal_type;    /* HDR/DV/colorimetry packed bitfield */
    __u32 hdr_status;     /* 0=SDR, 1=HDR10, 2=HLG, 3=HDR10+, 4=DV */
    __u32 is_interlaced;
    __u32 status;         /* 0=STABLE, 1=NOSIG, 2=NOTSUP */
};
```

## 4. Kernel Module: `vfm_cap`

### 4.1 Overview

A loadable kernel module that:
- Registers as a VFM receiver (receives frames from vdin0)
- Registers as a VFM provider (forwards frames to deinterlace)
- Exposes a V4L2 capture device (`/dev/video_cap`) with multi-open support
- Implements per-frame reference counting for safe multi-consumer buffer sharing
- Exports vdin0's CMA buffers as DMA-buf for zero-copy access
- Generates V4L2 events from kernel signal state changes

### 4.2 VFM Integration

**Modified VFM map:**
```
Old: "tvpath": "vdin0 deinterlace amvideo"
New: "tvpath": "vdin0 vfm_cap deinterlace amvideo"
```

**Frame flow:**

```
vdin0 ISR fires
    |
    v
vf_notify_receiver(VFRAME_EVENT_PROVIDER_VFRAME_READY)
    |
    v
vfm_cap receives notification
    |
    v
vfm_cap calls vf_get("vdin0") -> gets vframe_s pointer
    |
    v
Creates cap_frame wrapper, sets refcount = 1 + active_v4l2_consumers
    |
    +---> Adds to internal ready_list (for display path)
    |    deinterlace calls vfm_cap's vf_get() -> gets same vframe
    |    deinterlace calls vfm_cap's vf_put() -> decrements refcount
    |
    +---> For each active V4L2 consumer:
         Delivers buffer via vb2_buffer_done()
         Consumer DQBUFs, processes frame
         Consumer QBUFs -> decrements refcount
                |
                v
         When refcount == 0 -> vf_put("vdin0", vframe) -> recycled to vdin0
```

### 4.3 Per-Frame Reference Counting

```c
struct cap_frame {
    struct vframe_s     *vf;           /* original vframe from vdin0 */
    atomic_t            refcount;      /* 1 (display) + N (v4l2 consumers) */
    struct list_head    list;          /* internal list linkage */
    ktime_t             acquired_at;   /* for timeout/backpressure detection */
    bool                is_fallback;   /* true if this is a copy-fallback buffer */
};
```

### 4.4 Adaptive Zero-Copy with Fallback

The zero-copy design creates back-pressure: if a V4L2 consumer is slow to QBUF,
it holds vdin0 buffers, potentially starving vdin0 of write buffers.

A small pool of module-owned CMA buffers (default: 4) provides fallback:

1. Monitor held frame count vs vdin0 pool size
2. If threshold exceeded, DMA copy oldest held frame to fallback buffer
3. Release original back to vdin0
4. V4L2 consumer now holds the copy

**This fallback is the exception, not the norm.** Under typical operation (consumers
keeping up at 60fps), all frames are zero-copy.

### 4.5 Signal Change Handling Sequence

```
Time --------->

hdmirx:  STABLE --> timing_unstable --> ... --> STABLE (new params)
                         |                           |
vdin SM: STABLE --> UNSTABLE --> PRESTABLE --> STABLE
                         |                           |
vfm_cap:                 |                           |
  1. No new frames       |                           |
  2. Drains in-flight    |                           |
  3. Waits for QBUFs     |                           |
  4. (STABLE event) -----|                           |
     V4L2_EVENT_SOURCE_CHANGE to all consumers       |
  5. Consumers reconfigure                           |
  6. Resume frame delivery                           |
```

### 4.6 Module Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `provider_name` | `"vdin0"` | VFM provider to receive from |
| `receiver_name` | `"deinterlace"` | VFM receiver to forward to |
| `max_consumers` | 4 | Maximum simultaneous V4L2 consumers |
| `fallback_pool_size` | 4 | Number of copy-fallback buffers |
| `backpressure_threshold` | 2 | Min free vdin0 buffers before copy fallback |
| `video_nr` | -1 | V4L2 device number (-1 = auto-assign) |
| `debug` | 0 | Debug logging level (0=off, 1=info, 2=verbose) |

### 4.7 Sysfs Attributes

Exposed under the V4L2 device node (e.g., `/dev/video_cap`):

| Attribute | Mode | Description |
|-----------|------|-------------|
| `stats` | R | Frame counters: received, dropped, delivered |
| `status` | R | Module state: vfm_started, consumers, fmt_valid, sm_state, draining |
| `pool_state` | R | Per-slot pool dump: in_use, refcount, vframe index, physical address |
| `pool_drain` | W | Emergency: force-release all stuck frames back to vdin0 |

## 5. libvfmcap SDK (Path A Userspace)

### 5.1 Overview

A shared library (`libvfmcap.so`) that wraps `/dev/video_cap` V4L2 operations and
provides GPU-accelerated raw format conversion via Vulkan compute shaders on the
Mali-G52.

**This SDK performs raw format packing ONLY:**
- AMLY -> P010 (10-bit left-justified in 16-bit): `val << 6`
- AMLY -> NV12 (8-bit truncation): `val >> 2`

**No color space conversion, tone mapping, or gamut mapping is performed.**
Output preserves the original BT.2020 PQ colorimetry from vdin0.

### 5.2 Architecture

```
+-----------------------------------------------------------------+
|                    Consumer (GStreamer, app)                      |
|  vfmcap_open() -> vfmcap_start() -> vfmcap_acquire_frame()     |
|  -> vfmcap_convert_p010() or vfmcap_convert_nv12()             |
|  -> vfmcap_release_frame() -> vfmcap_stop() -> vfmcap_close()  |
+---------------+------------------------------------+-------------+
                | V4L2 + custom ioctl                | Vulkan compute
                v                                    v
+--------------------------+         +--------------------------------+
|   vfmcap.c               |         |  vfmcap_vulkan.c               |
|                          |         |                                |
| - open /dev/video_cap    |         | - Vulkan 1.2 instance          |
| - REQBUFS(MMAP, N)       |         | - Mali-G52 compute pipeline    |
| - STREAMON               |         | - DMA-buf import (cached)      |
| - DQBUF -> index         |         | - AMLY->P010 shader            |
| - IOC_GET_DMABUF(idx)    |--fd-->  | - AMLY->NV12 shader            |
|   -> DMA-buf fd          |         | - 2 pipelines (P010 + NV12)    |
| - QBUF (recycle)         |         | - Submit/wait (async)          |
| - STREAMOFF              |         |                                |
| - Event polling          |         | Output: DMA-buf fd for         |
|   (SOURCE_CHANGE)        |         |   downstream (encoder, etc.)   |
+--------------------------+         +--------------------------------+
```

### 5.3 Key Design Decisions

1. **CPU never touches frame data.** ARM CPU only does control plane (V4L2 ioctls,
   Vulkan command recording, fence waits). All pixel data flows:
   vdin0 CMA -> DMA-buf fd -> Vulkan GPU import -> GPU compute -> output DMA-buf

2. **Consumer provides output DMA-buf.** The SDK does NOT allocate output buffers.
   The caller allocates the output DMA-buf and passes its fd.

3. **Pipelined async API.** `vfmcap_convert_submit()` dispatches GPU work and
   returns immediately. The next `vfmcap_acquire_frame()` blocks in poll() while
   the GPU executes in parallel. GPU ~6ms completes during the ~10ms acquire wait,
   so `vfmcap_convert_wait()` typically returns instantly.

### 5.4 Source Tree

```
libvfmcap/
+-- include/
|   +-- vfmcap.h              # Public API header
+-- src/
|   +-- vfmcap.c              # V4L2 capture core + DMA-buf streaming
|   +-- vfmcap_vulkan.c       # Vulkan compute converter (2 pipelines)
|   +-- vfmcap_vulkan.h       # Internal Vulkan header
+-- shaders/
|   +-- amly_to_p010.comp     # GLSL: AMLY -> P010 (val << 6)
|   +-- amly_to_nv12.comp     # GLSL: AMLY -> NV12 (val >> 2)
|   +-- compile_shaders.sh    # Compiles .comp -> .spv -> _spv.h
|   +-- amly_to_p010_spv.h    # Pre-compiled SPIR-V (checked in)
|   +-- amly_to_nv12_spv.h    # Pre-compiled SPIR-V (checked in)
+-- demo/
|   +-- vfmcap-demo.c         # Validation + profiling demo
+-- Makefile                   # Builds libvfmcap.so + vfmcap-demo
```

### 5.5 Public API

```c
/* Lifecycle */
vfmcap_ctx_t *vfmcap_open(const char *device);
int  vfmcap_start(vfmcap_ctx_t *ctx, unsigned int num_buffers);
void vfmcap_stop(vfmcap_ctx_t *ctx);
void vfmcap_close(vfmcap_ctx_t *ctx);

/* Frame acquisition (zero-copy) */
int  vfmcap_acquire_frame(vfmcap_ctx_t *ctx, vfmcap_frame_t *frame, int timeout_ms);
void vfmcap_release_frame(vfmcap_ctx_t *ctx, vfmcap_frame_t *frame);

/* GPU format conversion -- raw packing only, no color conversion */
int  vfmcap_convert_p010(vfmcap_ctx_t *ctx, vfmcap_frame_t *frame, int out_dmabuf_fd);
int  vfmcap_convert_nv12(vfmcap_ctx_t *ctx, vfmcap_frame_t *frame, int out_dmabuf_fd);

/* Async GPU conversion (pipelined) */
int  vfmcap_convert_submit(vfmcap_ctx_t *ctx, vfmcap_frame_t *frame,
                           int out_dmabuf_fd, vfmcap_output_fmt_t fmt);
int  vfmcap_convert_wait(vfmcap_ctx_t *ctx);

/* Signal events */
int  vfmcap_poll_event(vfmcap_ctx_t *ctx, int timeout_ms);
int  vfmcap_get_signal_info(vfmcap_ctx_t *ctx, vfmcap_signal_info_t *info);

/* Utility */
const char *vfmcap_last_error(vfmcap_ctx_t *ctx);
uint32_t    vfmcap_output_size(uint32_t width, uint32_t height, vfmcap_output_fmt_t fmt);
```

### 5.6 Performance (Verified on Device)

4K60 HDR10 source (3840x2160 AMLY 10-bit YUV422):

| Format | FPS | GPU Submit | GPU Wait (pipelined) | Gaps |
|--------|-----|-----------|---------------------|------|
| P010   | 59.94 | 0.42ms avg | 0.90ms avg (hidden during acquire) | 0 |
| NV12   | 59.06 | 0.28ms avg | 0.97ms avg (hidden during acquire) | 0 |

Throughput: ~1185 MB/s input, ~1422 MB/s output (P010), ~701 MB/s output (NV12).

## 6. vdin1 Enhancements (Path B)

### 6.1 Current State

vdin1 loopback captures VPP post-processed frames with correct HDR->SDR colors.
Amlogic's GStreamer v4l2src plugin drives it today. However:

1. **vdin1 doesn't handle hdmirx signal change events.** vdin1 is in the video
   output domain -- it captures from VPP writeback, not from hdmirx. When the HDMI
   source changes resolution, tvserver detects it via hdmirx, reconfigures vdin0
   and VPP, but vdin1 doesn't know about the change until it gets garbled frames
   or times out.

2. **No auto-restart on resolution change.** When VPP parameters change (triggered
   by tvserver monitoring hdmirx), vdin1 doesn't automatically restart with new
   dimensions. The GStreamer pipeline stalls or crashes.

3. **Amlogic's patched v4l2src is a mess.** Their GStreamer plugin has extensive
   private patches that are hard to modify or extend.

### 6.2 Required Enhancements

**6.2.1 hdmirx Event Forwarding to vdin1 — SOLVED via vfm_cap Signal Monitor**

vdin1's V4L2 driver does NOT support `VIDIOC_SUBSCRIBE_EVENT` (returns ENOTTY).
However, the vfm_cap kernel module (`/dev/video_cap`) already monitors the vdin0
tvin state machine and translates signal changes into standard V4L2 events.

**Solution (implemented):** The streambox-src plugin opens `/dev/video_cap` as a
**signal monitor fd** in Path B mode — separate from any frame capture. It
subscribes to `V4L2_EVENT_SOURCE_CHANGE` and polls this fd alongside the vdin1
capture fd. When SOURCE_CHANGE fires, the plugin triggers STREAMOFF → reconfigure
→ STREAMON on vdin1.

**Key insight: vfm_cap serves a dual role:**
1. **Frame capture** (Path A): provides raw AMLY frames via V4L2 MMAP/DMA-buf
2. **Signal monitor** (Path B): provides V4L2 SOURCE_CHANGE events for vdin1
   consumers that cannot get events from their own V4L2 driver

This means the vfm_cap kernel module must be loaded even when using Path B only,
since it is the only userspace-accessible source of HDMI signal change events.

**6.2.2 vdin1 Auto-Restart Protocol**

When a signal change is detected, the streambox-src plugin must:

1. STREAMOFF on vdin1
2. Wait for VPP reconfiguration to complete (tvserver signals "ready")
3. Re-read format via VIDIOC_G_FMT
4. REQBUFS with new dimensions
5. STREAMON
6. Send new GStreamer CAPS downstream (triggers encoder renegotiation)

## 7. GStreamer Plugin: `streamboxsrc` [IMPLEMENTED]

### 7.1 Overview

A unified GStreamer source element (`streamboxsrc`) that can source from EITHER
vfm_cap (Path A) OR vdin1 (Path B). This replaces both:
- Amlogic's patched v4l2src for vdin1
- The planned vfmcapsrc for vfm_cap

**Current state:** ~3430 lines of C, fully working, committed and pushed.

### 7.2 Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `source` | enum | `vfmcap` | Capture source: `vfmcap` (Path A), `vdin1` (Path B) |
| `device` | string | auto | Device path (auto-detected from source) |
| `output-format` | enum | `p010` | Output: `p010`, `nv12` (Path A: both; Path B 8-bit: NV21 passthrough; Path B 10-bit: P010) |
| `num-buffers` | uint | 0 (unlimited) | Frame count limit (0 = unlimited) |

### 7.3 Path A Mode (vfm_cap)

Uses libvfmcap SDK internally. Rotating pool of 4 CMA output buffers avoids
fd recycling that breaks encoder's kernel DMA-buf import cache.

```
streamboxsrc source=vfmcap output-format=p010
    |
    +-- vfmcap_open("/dev/video_cap")
    +-- vfmcap_start(ctx, 6)
    +-- Allocate rotating CMA output pool (4 buffers from heap-codecmm)
    +-- loop:
    |     vfmcap_acquire_frame()
    |     vfmcap_convert_submit(P010, pool[i])  -- async GPU
    |     vfmcap_convert_wait()
    |     push GstBuffer (wraps output DMA-buf fd via dup())
    |     vfmcap_release_frame()
    |     i = (i + 1) % 4
    +-- vfmcap_stop()
    +-- vfmcap_close()
```

Output caps: `video/x-raw, format={P010_10LE,NV12}, width=W, height=H`

### 7.4 Path B Mode (vdin1)

Direct V4L2 capture from vdin1 using V4L2_MEMORY_DMABUF mode with CMA-backed
DMA-bufs. For 8-bit sources: NV21 zero-copy passthrough. For 10-bit sources:
AMLY format captured, then Vulkan GPU converts to Wave521 MSB P010 format using
double-buffered async dispatch.

```
streamboxsrc source=vdin1 output-format=p010
    |
    +-- open("/dev/video71")
    +-- S_INPUT(6) for VPP post-blend loopback
    +-- Detect 8-bit vs 10-bit from HDMI RX sysfs
    +-- Allocate CMA DMA-bufs for V4L2 DMABUF mode (6 capture + 4 output)
    +-- If 10-bit: init Vulkan P010 pipeline (double-buffered)
    +-- Open /dev/video_cap as signal monitor fd
    +-- Subscribe V4L2_EVENT_SOURCE_CHANGE on monitor fd
    +-- STREAMON
    +-- loop:
    |     poll(vdin1_fd + monitor_fd, POLLIN | POLLPRI)
    |     If POLLPRI on monitor_fd: signal change detected
    |         -> drain async GPU -> STREAMOFF vdin1
    |         -> post hdmi-signal-change message -> return EOS
    |     If POLLIN on vdin1_fd: DQBUF
    |         8-bit: wrap NV21 DMA-buf in GstBuffer, push downstream
    |         10-bit: submit Vulkan AMLY->P010 async, push previous result
    |     QBUF (recycle capture buffer)
    |     G_FMT polling every N frames as backup signal change detection
    +-- STREAMOFF
    +-- close()
```

Output caps (8-bit): `video/x-raw, format=NV21, width=W, height=H`
Output caps (10-bit): `video/x-raw, format=P010_10LE, width=W, height=H`

### 7.5 Signal Change Handling: Clean Exit Model

**Design decision:** Do NOT reconfigure the GStreamer pipeline in-place on signal
change. Instead, exit cleanly and let an external manager restart with new params.

```
Signal change detected (V4L2_EVENT_SOURCE_CHANGE or G_FMT mismatch)
    |
    v
Drain in-flight async GPU work (10-bit: wait for pending Vulkan fence)
    |
    v
Force STREAMOFF on vdin1 (prevents driver stall -- see Section 7.6)
    |
    v
Post GST_MESSAGE_ELEMENT named "hdmi-signal-change" with GstStructure:
    width, height, fps, color-depth, hdr-eotf, color-space,
    dolby-vision, interlace (from /sys/class/hdmirx/hdmirx0/info)
    |
    v
Return GST_FLOW_EOS
    |
    v
GStreamer propagates EOS downstream -> encoder flushes -> pipeline tears down
    |
    v
External GStreamer Manager reads message, relaunches pipeline with new caps
```

### 7.6 vdin1 Driver Stall Workaround

When HDMI RX stops supplying frames (signal change, unplug), VPP starves and
vdin1 enters an infinite internal wait for the next frame. If the plugin returns
EOS without stopping vdin1 first, the subsequent `stop()` call from GStreamer's
PLAYING->NULL state change will hang in VIDIOC_STREAMOFF because the kernel driver
is stuck in its wait loop.

**Fix:** Call `vdin1_streamoff()` in the `clean_exit:` path BEFORE returning
`GST_FLOW_EOS`. This breaks the driver's wait and allows the subsequent `stop()`
to complete immediately. STREAMOFF is idempotent, so calling it again in
`stop_path_b()` during teardown is harmless.

### 7.7 Verified Pipelines

```bash
# Path A P010 CQP (4K60)
gst-launch-1.0 streamboxsrc source=vfmcap output-format=p010 num-buffers=30 \
    ! "video/x-raw,format=P010_10LE,width=3840,height=2160" \
    ! amlvenc rc-mode=0 internal-bit-depth=10 framerate=60 \
    ! "video/x-h265,stream-format=byte-stream" ! filesink location=/tmp/test.h265

# Path A P010 VBR (4K60)
gst-launch-1.0 streamboxsrc source=vfmcap output-format=p010 num-buffers=30 \
    ! "video/x-raw,format=P010_10LE,width=3840,height=2160" \
    ! amlvenc rc-mode=2 encoder-buffer-size=4096 internal-bit-depth=10 framerate=60 \
    ! "video/x-h265,stream-format=byte-stream" ! filesink location=/tmp/test.h265

# Path A NV12 CQP (4K60)
gst-launch-1.0 streamboxsrc source=vfmcap output-format=nv12 num-buffers=10 \
    ! "video/x-raw,format=NV12,width=3840,height=2160" \
    ! amlvenc rc-mode=0 framerate=60 \
    ! "video/x-h265,stream-format=byte-stream" ! filesink location=/tmp/test.h265

# Path A full A/V SRT output (4K60 HDR10 + 48kHz stereo AAC -> MPEG-TS -> SRT)
gst-launch-1.0 -e streamboxsrc source=vfmcap output-format=p010 \
    ! "video/x-raw,format=P010_10LE,width=3840,height=2160" \
    ! amlvenc internal-bit-depth=10 gop=60 gop-pattern=0 bitrate=30000 framerate=60 \
    ! video/x-h265 ! h265parse config-interval=-1 \
    ! queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 ! mux. \
    alsasrc device=hw:0,6 buffer-time=500000 provide-clock=false slave-method=re-timestamp \
    ! "audio/x-raw,rate=48000,channels=2,format=S16LE" ! audioconvert ! audioresample \
    ! avenc_aac bitrate=128000 ! aacparse ! mux. \
    mpegtsmux name=mux alignment=7 latency=100000000 \
    ! srtsink uri="srt://:8888" wait-for-connection=false sync=false

# Path B NV21 VBR (4K60, 8-bit)
gst-launch-1.0 streamboxsrc source=vdin1 num-buffers=10 \
    ! "video/x-raw,format=NV21,width=3840,height=2160" \
    ! amlvenc rc-mode=2 encoder-buffer-size=4096 framerate=60 \
    ! "video/x-h265,stream-format=byte-stream" ! filesink location=/tmp/test.h265

# Path B P010 CQP (4K60, 10-bit)
gst-launch-1.0 streamboxsrc source=vdin1 output-format=p010 num-buffers=10 \
    ! "video/x-raw,format=P010_10LE,width=3840,height=2160" \
    ! amlvenc rc-mode=0 internal-bit-depth=10 framerate=60 \
    ! "video/x-h265,stream-format=byte-stream" ! filesink location=/tmp/test.h265

# Path B P010 VBR (1080p60, 10-bit) -- stability tested at 30,000 frames
gst-launch-1.0 streamboxsrc source=vdin1 output-format=p010 num-buffers=30000 \
    ! "video/x-raw,format=P010_10LE,width=1920,height=1080" \
    ! amlvenc rc-mode=2 encoder-buffer-size=4096 internal-bit-depth=10 framerate=60 \
    ! "video/x-h265,stream-format=byte-stream" ! fakesink
```

### 7.8 Colorimetry / VUI Signaling

HDMI sources carry color metadata (primaries, transfer function, matrix coefficients)
that must propagate through the capture pipeline into the H.264/H.265 encoded bitstream
as VUI (Video Usability Information) parameters. Without correct VUI, decoders assume
BT.709 SDR, producing washed-out or incorrect colors for HDR content.

**Source detection:** `streamboxsrc` reads HDMI RX signal info from
`/sys/class/hdmirx/hdmirx0/info` (fields: `Color Depth`, `HDR EOTF`, `Color Space`)
in the new `hdmirx_detect_colorimetry()` function, called during both `start_path_a()`
and `start_path_b()`.

**Colorimetry mapping by capture path:**

| Path | Condition | GStreamer colorimetry | VUI (cp/tc/mc) |
|------|-----------|----------------------|----------------|
| Path A (vfmcap) | HDR EOTF = PQ | `bt2100-pq` | 9/16/9 |
| Path A (vfmcap) | HDR EOTF = HLG | `bt2100-hlg` | 9/14/9 |
| Path A (vfmcap) | 10-bit SDR | `bt2020` | 9/1/9 |
| Path A (vfmcap) | 8-bit SDR | `bt709` | 1/1/1 |
| Path B 8-bit (NV21) | Any | `bt709` | 1/1/1 |
| Path B 10-bit (AMLY) | HDR EOTF = PQ | `bt2100-pq` | 9/16/9 |
| Path B 10-bit (AMLY) | HDR EOTF = HLG | `bt2100-hlg` | 9/14/9 |
| Path B 10-bit (AMLY) | 10-bit SDR | `bt2020` | 9/1/9 |
| Path B 10-bit (AMLY) | 8-bit SDR | `bt709` | 1/1/1 |

Path B 8-bit always reports `bt709` because VPP has already performed HDR-to-SDR
tone mapping and BT.2020-to-BT.709 gamut mapping before vdin1 captures the frame.

**Caps propagation:** The colorimetry string is set on GStreamer caps at all negotiation
points (`push_current_caps`, `gst_streambox_src_get_caps`) using
`gst_video_colorimetry_from_string()` for validation, then `gst_caps_set_simple()`.

**Encoder VUI:** `amlvenc` (multienc-wave521) reads `GstVideoInfo.colorimetry` from
the negotiated input caps and converts to ITU-T H.273 integer codes using
`gst_video_color_primaries_to_iso()`, `gst_video_transfer_function_to_iso()`, and
`gst_video_color_matrix_to_iso()`. These are written into `vl_encode_info_t` VUI
fields and propagated by the encoder library into the H.264 SPS VUI or H.265 SPS VUI.
A colorimetry change in caps triggers encoder re-initialization via the
`gst_video_colorimetry_is_equal()` check in `gst_amlvenc_set_format()`.

**Note:** `chroma-site` (chroma sample location) is not currently signaled because
the encoder library API (`vl_encode_info_t`) does not expose `chroma_loc_info_present_flag`
fields, even though the underlying bitstream writer supports them.

## 8. Integration Updates

### 8.1 Pipeline Manager Changes

| File | Change |
|------|--------|
| Pipeline manager: device discovery | Add detection for `/dev/video_cap` + streambox-src plugin |
| Pipeline manager: auto-instance | Switch from `v4l2src device=/dev/video71` to `streambox-src` |
| Pipeline manager: events | Option to get events from V4L2 instead of tvserver polling |
| Pipeline manager: tvservice | Retain as-is (tvserver still controls display path) |

### 8.2 Third-Party Consumer Integration

**PiKVM:**
- Standard V4L2 capture from `/dev/video_cap`
- Uses MMAP or DMABUF mode
- Subscribe to `V4L2_EVENT_SOURCE_CHANGE` for resolution tracking

**Sunshine (remote gaming):**
- V4L2 capture with DMA-buf export for GPU-based encoding
- ALLM events for game mode detection
- VRR events for variable framerate handling

### 8.3 tvserver Compatibility

The tvserver display path is **completely unaffected**:
- tvserver still controls hdmirx (EDID, HDCP, signal detection)
- tvserver still manages vdin0 (open port, start/stop decoder)
- The VFM chain modification is transparent to deinterlace
- tvserver's Binder event system continues to work for existing clients

## 9. Phased Implementation Plan

### Phase 1: vfm_cap Kernel Module -- VFM Tee + V4L2 Device [COMPLETE]

**Deliverables:**
- [x] `vfm_cap.c` / `vfm_cap.h` kernel module (~2270 lines)
- [x] VFM receiver + provider registration
- [x] Per-frame reference counting (display + 1 consumer)
- [x] V4L2 capture device with MMAP support
- [x] Module parameters and sysfs
- [x] Yocto recipe, udev rule, VFM map modification

**Results:** 59.94fps, zero drops, display path unaffected.

### Phase 2: Signal Events + DMA-buf Export [COMPLETE]

**Deliverables:**
- [x] SM polling via `tvin_get_sm_status()` (no kernel patch needed)
- [x] `V4L2_EVENT_SOURCE_CHANGE` with full signal info payload
- [x] 10-bit AMLY format detection (bitdepth, FULL_PACK_422_MODE)
- [x] Zero-copy DMA-buf export from vdin0 CMA buffers
- [x] `VFM_CAP_IOC_GET_DMABUF` custom ioctl
- [x] Per-frame refcounting via DMA-buf fd lifecycle

**Results:** SM transitions detected, SOURCE_CHANGE events working,
DMA-buf zero-copy confirmed at 59.94fps.

### Phase 3: libvfmcap SDK [COMPLETE]

**Deliverables:**
- [x] `include/vfmcap.h` -- public API (raw format conversion only)
- [x] `src/vfmcap.c` -- V4L2 capture + DMA-buf lifecycle
- [x] `src/vfmcap_vulkan.c` -- Vulkan compute (2 pipelines: P010 + NV12)
- [x] `src/vfmcap_vulkan.h` -- internal Vulkan header
- [x] `shaders/amly_to_p010.comp` -- P010 shader
- [x] `shaders/amly_to_nv12.comp` -- NV12 shader (raw truncation, no CSC)
- [x] Pre-compiled SPIR-V headers
- [x] `demo/vfmcap-demo.c` -- validation + profiling demo
- [x] `Makefile` + Yocto recipe `libvfmcap_1.0.bb`

**Results:** P010 59.94fps, NV12 59.06fps, 0 gaps, 0 errors at 4K60.

**Commit history:**
- Initial libvfmcap add
- BT.2020->BT.709 CSC for NV12 (superseded)
- 3-pipeline split with HDR (superseded)
- Remove HDR/CSC code, simplify to raw-only

### Phase 4: streambox-src GStreamer Plugin [COMPLETE]

**Goal:** Unified GStreamer source element for both capture paths.

**Deliverables:**
- [x] `gst_streambox_src.c` / `gst_streambox_src.h` -- source element (~3430 lines)
- [x] Path A mode: libvfmcap integration with DMA-buf output (P010 + NV12)
- [x] Path B mode: direct V4L2 vdin1 capture with V4L2_MEMORY_DMABUF zero-copy
- [x] Path B 10-bit: Vulkan AMLY->P010 conversion (Wave521 MSB byte-swap format)
- [x] Double-buffered async GPU for Path B 10-bit (39fps -> 60fps)
- [x] Path A rotating CMA output buffer pool (4 buffers) to fix fd recycling bug
- [x] amlvenc VBR/CBR fix: propagate encoder-buffer-size to hardware bitstream DMA buffer
- [x] All output is DMA-buf backed from CMA `heap-codecmm` (required by Wave521 encoder)
- [x] CPU never touches frame data -- ARM CPU does control plane only
- [x] Pipeline integration testing with amlvenc (CQP, VBR, CBR all working)

**Results:** All capture paths verified at 1080p60 and 4K60:
- Path A P010 CQP/VBR: 59.94fps, zero errors
- Path A NV12 CQP: 59.94fps, zero errors
- Path B NV21 VBR: 59.94fps, zero errors
- Path B P010 CQP/VBR: 59.94fps, zero errors

**Commit history:**
- gst-plugin-vfmcap: add streamboxsrc dual-path GStreamer plugin
- streamboxsrc: fix allocation, DMA-buf output for Path B, fd leak
- streamboxsrc: Path B zero-copy via V4L2_MEMORY_DMABUF with CMA DMA-bufs
- streamboxsrc: add Path B 10-bit P010 output via Vulkan AMLY->P010 conversion
- streamboxsrc: double-buffered async GPU for Path B 10-bit (39fps -> 60fps)
- amlvenc: fix VBR/CBR INT_BSBUF_FULL by propagating bitstream buffer size to hardware
- streamboxsrc: fix vdin1 AMLY field mapping -- swap Y<->UV in decode_pair
- fix P010 encoding: use CMA DMA-bufs and Wave521 MSB byte-swap format
- streamboxsrc: fix Path A fd recycling with rotating CMA output buffer pool
- amlvenc (submodule): propagate encoder-buffer-size to hardware bitstream DMA buffer

### Phase 5: vdin1 Signal Enhancement + Clean Exit [COMPLETE]

**Goal:** Make vdin1 capture robust across HDMI signal changes.

**Approach:** Use vfm_cap (`/dev/video_cap`) as a signal monitor for Path B.
vdin1's V4L2 driver does not support `VIDIOC_SUBSCRIBE_EVENT`, but vfm_cap does.
The plugin opens `/dev/video_cap` read-only, subscribes to
`V4L2_EVENT_SOURCE_CHANGE`, and polls it alongside the vdin1 capture fd in
`create_path_b()`. This is event-driven with no polling overhead.

**Key architecture decision: Clean exit on signal change, no in-place reconfigure.**
GStreamer has poor support for dynamically reconfiguring a pipeline's resolution,
HDR mode, or other parameters mid-stream. The encoder, downstream muxers, and
network sinks all have state tied to the original caps. The robust pattern is:

1. Detect signal change (POLLPRI on signal monitor fd, G_FMT polling, or poll timeout)
2. Drain any in-flight async GPU work (10-bit path)
3. Force STREAMOFF on vdin1 to prevent driver stall (see below)
4. Post `GST_MESSAGE_ELEMENT` with `GstStructure` named `"hdmi-signal-change"`
   containing all HDMI RX signal info (width, height, fps, color depth, HDR EOTF, etc.)
5. Return `GST_FLOW_EOS` -- GStreamer propagates EOS downstream, encoder flushes,
   pipeline tears down cleanly
6. External GStreamer Manager watches for this message, reads new parameters,
   relaunches pipeline with correct caps

**Critical fix: vdin1 stall workaround.**
When HDMI RX stops supplying frames (signal change, unplug), VPP has no new data
and vdin1 enters an infinite internal wait loop for the next frame. If the plugin
returns EOS without stopping vdin1 first, the subsequent `stop()` call from
GStreamer's state change will hang in STREAMOFF because the driver is stuck waiting.
**Fix:** call `vdin1_streamoff()` in the `clean_exit:` path before returning
`GST_FLOW_EOS`. STREAMOFF is idempotent so calling it again in `stop_path_b()` is
harmless.

**Deliverables:**
- [x] Open `/dev/video_cap` as signal monitor fd in `start_path_b()`
- [x] Subscribe `V4L2_EVENT_SOURCE_CHANGE` on monitor fd
- [x] Integrate monitor fd into `create_path_b()` poll loop (POLLPRI)
- [x] On SOURCE_CHANGE: drain async GPU → STREAMOFF vdin1 → post message → EOS
- [x] G_FMT polling as backup detection (catches cases missed by event)
- [x] Force STREAMOFF before EOS return to prevent vdin1 driver stall
- [x] Post `GST_MESSAGE_ELEMENT` named `"hdmi-signal-change"` with full signal info
- [x] `GstStructure` includes: width, height, fps, color-depth, hdr-eotf, color-space,
      dolby-vision, interlace fields from HDMI RX sysfs

**Commit history:**
- streamboxsrc: add event-driven signal monitor for Path B via vfm_cap
- streamboxsrc: clean exit on signal change -- post message + EOS instead of in-place reconfigure

### Phase 6: Integration + Hardening [PARTIALLY COMPLETE]

**Stability testing: PASSED.**

| Test | Frames | Duration | FPS | Errors | Result |
|------|--------|----------|-----|--------|--------|
| Path B P010 VBR 1080p60 10-bit (30k) | 30,000 | 8m 20.5s | 59.94 | 0 | PASS |
| Path B P010 VBR 1080p60 10-bit (3k) | 3,000 | 50.1s | 59.92 | 0 | PASS |

Both tests: zero ERRORs, zero CRITICALs, zero FATALs, zero segfaults,
clean EOS, clean shutdown (async GPU drained, Vulkan cleaned up, vdin1 STREAMOFF).
Encoder sustained ~3.4ms/frame throughout.

**Remaining deliverables:**
- [x] Colorimetry / VUI signaling: streamboxsrc detects HDMI color metadata and sets
      GStreamer colorimetry caps; amlvenc reads caps and writes H.265/H.264 VUI parameters
- [ ] Pipeline manager updates (discovery, auto-instance, events)
- [ ] External GStreamer Manager that watches for `hdmi-signal-change` messages
      and auto-restarts pipelines with correct parameters
- [ ] Network streaming integration (RTP/RTSP/SRT output instead of filesink)
- [ ] 4K testing of clean exit on actual HDMI signal changes
- [ ] PiKVM integration testing
- [ ] Sunshine integration testing
- [ ] Optional: disable vdin1 when not in use, reclaim CMA memory

### Phase 7: Path A Stability — Pipeline Crash and Buffer Exhaustion Fixes [COMPLETE]

Running the full Path A GStreamer pipeline (`streamboxsrc ! amlvenc ! filesink`
and full A/V SRT output) at 4K60 HDR10 exposed multiple bugs that only manifest
under sustained real-time encoding. This phase addresses all of them, including a
hard deadlock in the kernel module that required root cause analysis of the vdin0
ISR frame delivery mechanism.

#### 7.1 Bug 1 — Pipeline stall after ~4 frames [FIXED]

**Root cause:** In `gstamlvenc_multienc.c`, the Vulkan double-buffered pipeline
"primes" on frame 0 by doing GPU conversion then returning early. The code called
`gst_video_codec_frame_unref(frame)` which only drops a reference but does NOT
remove the frame from `GstVideoEncoder`'s internal pending queue. Frame 0's input
DMA-buf stayed pinned forever, exhausting the upstream 4-buffer pool after ~4 frames.

**Fix:** Replaced with `gst_video_encoder_finish_frame()` using
`GST_VIDEO_CODEC_FRAME_SET_DECODE_ONLY` flag to properly retire frame 0.

#### 7.2 Bug 1b — NULL dereference crash [FIXED]

`gst_video_codec_frame_unref(frame)` was called when `frame` was NULL.
Fixed by removing the spurious unref.

#### 7.3 Bug 1c — Race condition causing crash on exit [FIXED]

`encoder_lock` mutex was declared and initialized but never locked. The encoder's
`handle_frame` and `stop` could race, causing use-after-free in the Wave521 library.
Fixed by adding `g_mutex_lock/unlock` around all encode and destroy calls.

#### 7.4 Bug 1d — Vulkan resource leak on stop [FIXED]

`yuv422_vulkan_cleanup()` was never called in the encoder's `stop()` method.
Also `vulkan_initialized` was a `static` local variable (shared across instances).
Fixed by moving the flag to the instance struct and adding cleanup call in `stop()`.

#### 7.5 Bug 1e — Pool size headroom [FIXED]

Increased `PATHA_OUT_POOL_SIZE` from 4 to 6 in `gst_streambox_src.h` for more
headroom in the rotating output buffer pool.

#### 7.6 Bug 2 — Kernel crash from CMA buffer starvation [FIXED]

**Root cause:** The Vulkan DMA-buf input cache in `vfmcap_vulkan.c` held Mali GPU
`dma_buf` references (via `dup(fd)` → `vkAllocateMemory` → Mali internal
`dma_buf_get`) for up to 8 CMA buffers long after frame processing completed.
Each cache entry kept its `VkDeviceMemory` alive, preventing the kernel `cap_frame`
from being recycled (refcount never reached 0). With `VFM_CAP_POOL_SIZE=16` and the
display path (deinterlace→amvideo→tvserver) holding 2-3 buffers, pinning 8 more via
the cache left only 5-6 for vdin0. Under sustained 4K60fps pipelined GPU conversion,
vdin0 ran out of free CMA buffers, causing memory corruption and kernel oops.

**Fixes applied to `vfmcap_vulkan.c`:**
1. Reduced `DMABUF_CACHE_SIZE` from 8 to 2 (only need current + in-flight)
2. Added immediate cache eviction in `vfmcap_vk_convert_wait()` — after GPU fence
   signals, the completed frame's cache entry is destroyed, releasing Mali's
   `dma_buf` ref so the CMA buffer can be recycled when userspace does
   `close(fd)` + QBUF
3. Added inode-based validation to detect stale cache entries after fd number reuse
4. Fixed fd leak in `import_dmabuf()` on `vkAllocateMemory` failure

**Fixes applied to `vfmcap.c`:**
- Reordered `vfmcap_close()` to do Vulkan cleanup BEFORE `vfmcap_stop()` (which
  calls `VIDIOC_REQBUFS(count=0)` freeing kernel CMA allocations)

**Commit:** (Bugs 1-1e + Bug 2 combined)

#### 7.7 Bug 3 — vdin0 display pipeline buffer exhaustion [FIXED]

**Symptom:** After running the GStreamer capture+encode pipeline for 30+ seconds,
vdin0's display path (deinterlace→amvideo) accumulates 9-10 of the 10 CMA capture
buffers, leaving 0-1 writable. The system barely survives. This does NOT happen
with the standalone `vfmcap-demo` even under sustained 60fps GPU profiling.

**Critical finding: GStreamer-specific, not demo-specific.** Both paths use the
same libvfmcap code, but:
- `vfmcap-demo -n 1800 -c p010 -p` (GPU pipelined, 60fps, 30s): NO stall
- GStreamer `streamboxsrc ! amlvenc ! filesink` (GPU synchronous, 60fps, 35s): STALL

**Root cause:** Mali GPU's `vkFreeMemory` performs a **deferred `dma_buf_put`** —
the kernel dma_buf reference is not released immediately but on an internal worker
thread. In the demo's pipelined mode, there's a natural ~16ms delay between
`vkFreeMemory` (in `convert_wait`) and `release_frame` (called 1 frame later).
This gives Mali time to complete deferred cleanup. In GStreamer's synchronous mode,
`vkFreeMemory` → `release_frame` happens back-to-back with <1ms gap. Mali hasn't
released its dma_buf ref yet when `close(fd)` + `QBUF` fire, so the cap_frame's
`vfm_cap_dmabuf_release` callback (which recycles the CMA buffer to vdin0) is
delayed. At 60fps over 30+ seconds, this backlog accumulates and eventually
exhausts vdin0's buffer pool.

**DMA-buf reference lifecycle (for understanding the fix):**

```
Frame arrives from vdin0:
  cap_frame refcount = 2 (1 display-path, 1 V4L2-delivery) when consumer streaming
  cap_frame refcount = 1 (display-path only) otherwise

vfm_cap_export_frame_dmabuf():  atomic_inc(&refcount) → +1 for DMA-buf
deliver_work end:               frame_put() → -1 (drops pending-list ref)
Display path vf_put:            frame_put() → -1

get_dmabuf ioctl:  get_dma_buf() → dma_buf refcount=2 (1 kernel buf->dbuf, 1 userspace fd)
Userspace close(fd):            → dma_buf refcount=1
QBUF → buf_queue → dma_buf_put(buf->dbuf): → dma_buf refcount=0
  → vfm_cap_dmabuf_release → frame_put → frame_release → vf_put (CMA recycled)

BUT if Mali cache holds import via dup(fd):
  Mali's dma_buf ref keeps dma_buf alive even after close+QBUF
  → cap_frame NOT recycled until vkFreeMemory → Mali worker → dma_buf_put
```

**Fix:** Added `vkDeviceWaitIdle()` after `cache_entry_destroy()` in
`vfmcap_vk_convert_wait()`. This forces Mali to flush all pending internal work,
including the deferred `dma_buf_put`, before returning to the caller. The caller
then does `release_frame()` (close+QBUF) and the CMA buffer can be recycled
immediately.

**Verified: 5-minute stress test at 4K60 HDR10 (12,205 frames):**

| Time | Buffer Get Count | Writable | Notes |
|------|-----------------|----------|-------|
| Baseline | 4 | 6 | No pipeline |
| 15s  | 6 | 4 | Stable |
| 45s  | 6 | 3 | Stable |
| 75s  | 7 | 3 | Stable |
| 105s | 7 | 3 | Stable |
| 135s | 6 | 3 | Stable |
| 165s | 7 | 3 | Stable |
| 195s | 6 | 4 | Stable |
| 225s | 7 | 2 | Stable |
| 255s | 6 | 4 | Stable |

Buffer count oscillates 6-7 with NO upward drift over 5 minutes. System maintains
2-4 writable buffers at all times. Previous behavior without the fix: count reached
9-10 within 30 seconds.

**Note:** Buffer get count jumps to 9-10 during pipeline shutdown (SIGTERM → EOS
processing) as the display path temporarily holds frames while the V4L2 consumer
disconnects. This is the root cause of Bug 4 (see section 7.9). The auto-drain
fix in `vfm_cap_release()` recycles these stuck frames when the last consumer
closes, preventing the deadlock on subsequent pipeline starts.

*(Bug 3 + poll fix in one commit)*

#### 7.8 Poll loop fix in vfmcap_acquire_frame [FIXED]

**Problem:** The original `vfmcap_acquire_frame()` used a single-shot `poll()`.
If a stale `V4L2_EVENT_SOURCE_CHANGE` event (from a prior HDMI signal transition)
was queued, `poll()` would return immediately with `POLLPRI` but no `POLLIN`.
The function would handle the event, see no data, and return `VFMCAP_ERR_TIMEOUT`
— even though vdin0 was actively delivering frames. This wasted the entire timeout
budget on one obsolete event.

**Fix:** Converted to a deadline-based loop:
1. Calculate `deadline = now_ms_mono() + timeout_ms`
2. Loop: `poll()` with `remaining = deadline - now`
3. If `POLLPRI`: drain ALL pending events (inner `while` loop), then re-poll
4. If `POLLIN`: break out to DQBUF
5. If `EINTR`: retry (instead of returning timeout)
6. Added `now_ms_mono()` helper using `CLOCK_MONOTONIC`
7. Added stale event drain in `vfmcap_start()` before entering capture loop

**Commit:** (same commit as Bug 3 fix)

#### 7.9 Bug 4 — vdin0 frame delivery freeze after first pipeline run [FIXED]

**Symptom:** After the first full GStreamer+SRT pipeline run completes and tears
down, the vfm_cap kernel module stops receiving frames from vdin0. The
`frames_received` counter freezes permanently. No amount of waiting recovers it.
A device reboot is required to restore capture. This is a hard deadlock, not a
transient stall.

**Root cause: vdin0 write buffer pool exhaustion.**

1. vdin0 has **12 frame buffers** total (`VDIN_CANVAS_MAX_CNT = 12`).
2. During normal operation, the display path (deinterlace → amvideo) holds ~3
   frames in the vfm_cap pool via reference counting. When a V4L2 consumer
   (GStreamer) is streaming, each frame gets `refcount=2` (one for display path,
   one for V4L2 delivery).
3. After a GStreamer pipeline runs and stops, the display path holds **9-10 frames**
   instead of the normal ~3. These extra frames were in transit through the
   V4L2/DMA-buf/GPU pipeline during teardown. The V4L2 side properly releases its
   references (refcount drops from 2 to 1), but the display-path reference
   (refcount=1) persists because the frames are buffered in deinterlace/amvideo.
4. With 10 of 12 vdin0 buffers held downstream, vdin0 has only 2 spare write
   buffers. The vdin0 ISR needs at least 2-3 free buffers for its RDMA write
   path — `provider_vf_peek()` checks for the next available write buffer.
5. When `provider_vf_peek()` returns NULL in `vdin_isr()`, the ISR bails out
   and **never calls `vf_notify_receiver(VFRAME_EVENT_PROVIDER_VFRAME_READY)`**.
6. This is a permanent deadlock: vdin0 won't send frames because its write pool
   is exhausted, and the display path won't return frames because it isn't getting
   new ones to replace them.

**Why extra frames accumulate during pipeline shutdown:**

When a V4L2 consumer is actively streaming, the V4L2 delivery workqueue, DMA-buf
export, and GPU processing add processing delays that cause the display path to
hold more frames than in the idle case. The deinterlace and amvideo subsystems
buffer frames in their internal pipelines. After teardown, these extra frames
remain stuck with refcount=1 (display-path only) and are never recycled because
no new frames arrive to push them through.

**Debug tool: `pool_drain` sysfs (write-only)**

Added a `pool_drain` sysfs attribute as an emergency recovery tool. Writing any
value force-releases ALL in_use pool frames back to vdin0 regardless of refcount.
This confirmed the root cause: after drain, `in_use_total` dropped from 10/16 to
3/16, and `frames_received` immediately resumed at 60fps.

**Debug tool: `pool_state` sysfs (read-only)**

Added a `pool_state` sysfs attribute that dumps all 16 cap_frame pool slots with
their in_use flag, refcount, vframe index, and physical address. Used to diagnose
the accumulation pattern.

**Production fix: auto-drain on last consumer close.**

In `vfm_cap_release()`, when `num_consumers` drops to 0, all in_use pool frames
are automatically recycled back to vdin0. The logic:
1. Cancel pending delivery work (no consumers to deliver to)
2. Iterate all 16 pool slots
3. For each in_use frame: `vf_put()` + `vf_notify_provider()` to return it to vdin0
4. Clear `in_use`, reset refcount, remove from list nodes

This is safe because with zero consumers, there are no V4L2/DMA-buf references
outstanding — every remaining in_use frame is held only by the display path.
Releasing them allows the display path to naturally re-acquire fresh frames from
vdin0 as they arrive.

**Verification: 5 consecutive pipeline runs (3 filesink + 2 full SRT):**

| Run | Pipeline Type | Auto-drained | frames_received after |
|-----|-------------|--------------|----------------------|
| 1 | filesink H.265 | 9 frames | Incrementing at 60fps |
| 2 | filesink H.265 | 9 frames | Incrementing at 60fps |
| 3 | filesink H.265 | 10 frames | Incrementing at 60fps |
| 4 | Full A/V SRT   | 9 frames | Incrementing at 60fps |
| 5 | Full A/V SRT   | 9 frames | Incrementing at 60fps |

Pool consistently returns to healthy 3-4/16 in_use after each drain. Before this
fix, the second pipeline run would freeze permanently.

**Additional kernel module improvements in this fix:**
- Forward declaration of `vfm_cap_vf_provider_ops` for late-start logic
- Upgraded `frame_release()` debug logging from level 2 to level 1 with slot index
- Fixed `vfm_cap_vf_states()` `buf_free_num` to count actual in_use slots
- Enhanced `QUREY_STATE` handler with pool_free check and diagnostic logging
- Enhanced pool exhaustion error logging in `VFRAME_READY` handler
- Added late-start logic in `VFRAME_READY` for auto-registering VFM provider
  when module is loaded into a running pipeline (hot-reload case)

#### 7.10 Commit History

| Description |
|-------------|
| Fix kernel crash: release Vulkan DMA-buf imports promptly (Bug 2 + Bugs 1-1e) |
| Fix poll loop and vdin0 buffer exhaustion during GStreamer pipeline (Bug 3 + poll fix) |
| Fix vdin0 frame delivery freeze: auto-drain pool on last consumer close (Bug 4) |

## 10. Files Created/Modified

### Phases 1-3 (COMPLETE)

| File | Status | Description |
|------|--------|-------------|
| `vfm_cap/vfm_cap.c` | Done | Kernel module (~2270 lines) |
| `vfm_cap/vfm_cap.h` | Done | Module header |
| `vfm_cap/Makefile` | Done | Kernel build |
| `vfm_cap/vfm-cap-setup.sh` | Done | Setup script |
| `vfm_cap/test_dmabuf.c` | Done | DMA-buf test |
| (Yocto recipe: `vfm-cap_1.0.bb`) | Done | Yocto recipe |
| (Yocto recipe: `99-vfm-cap.rules`) | Done | udev rule |
| (Yocto BSP layer machine config) | Done | EXTERNALSRC |
| (tvserver VFM chain configuration) | Done | VFM chain insert |
| `libvfmcap/include/vfmcap.h` | Done | Public API |
| `libvfmcap/src/vfmcap.c` | Done | V4L2 capture core |
| `libvfmcap/src/vfmcap_vulkan.c` | Done | Vulkan converter |
| `libvfmcap/src/vfmcap_vulkan.h` | Done | Internal header |
| `libvfmcap/shaders/amly_to_p010.comp` | Done | P010 shader |
| `libvfmcap/shaders/amly_to_nv12.comp` | Done | NV12 shader |
| `libvfmcap/demo/vfmcap-demo.c` | Done | Demo program |
| `libvfmcap/Makefile` | Done | Build system |
| (Yocto recipe: `libvfmcap_1.0.bb`) | Done | Yocto recipe |

### Phase 4: streambox-src Plugin (COMPLETE)

| File | Status | Description |
|------|--------|-------------|
| `gst-plugin-vfmcap/gst_streambox_src.c` | Done | Source element (~3430 lines) |
| `gst-plugin-vfmcap/gst_streambox_src.h` | Done | Plugin header (~200 lines) |
| `gst-plugin-vfmcap/Makefile` | Done | Plugin build |
| `gst-plugin-vfmcap/vdin1_amly_to_p010.comp` | Done | Vulkan compute shader: vdin1 AMLY -> Wave521 MSB P010 |
| `gst-plugin-vfmcap/vdin1_amly_to_p010.spv` | Done | Compiled SPIR-V binary |
| `gst-plugin-vfmcap/vdin1_amly_to_p010_spv.h` | Done | C header with embedded SPIR-V |
| `gst-plugin-venc/multienc-wave521/gstamlvenc_multienc.c` | Done | amlvenc: VBR/CBR bitstream buffer fix, colorimetry/VUI signaling |

### Phase 5: Signal Enhancement (COMPLETE)

Signal monitor and clean exit logic are integrated into `gst_streambox_src.c`
(no separate files). The vfm_cap kernel module must be loaded for Path B signal
monitoring.

### Phase 6: Integration (PARTIALLY COMPLETE)

| File | Status | Description |
|------|--------|-------------|
| `gst-plugin-vfmcap/gst_streambox_src.c` | Done | Colorimetry detection from HDMI RX sysfs, caps propagation |
| `gst-plugin-vfmcap/gst_streambox_src.h` | Done | Added `colorimetry[64]` field to instance struct |
| `gst-plugin-venc/multienc-wave521/gstamlvenc_multienc.c` | Done | Dynamic VUI from caps (replaces hardcoded HDR10), colorimetry change detection |
| Pipeline manager: device discovery | TODO | Detect new devices |
| Pipeline manager: auto-instance | TODO | Use streamboxsrc |
| Pipeline manager: events | TODO | V4L2 event source |

## 11. Risk Assessment

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| Holding vdin0 buffers starves display | High | Low | Auto-drain on consumer close; adaptive fallback copy; pool_drain sysfs for emergency |
| VFM chain insertion breaks display | High | Low | Module is removable; tested extensively |
| Per-frame refcount races | High | Medium | Atomic ops; spinlock; stress testing |
| Signal change during capture | Medium | High | Drain-and-reconfigure; force-release on timeout |
| vdin1 signal handling unreliable | Medium | High | Resolved: use vfm_cap as signal monitor fd |
| Amlogic v4l2src too patched to extend | Medium | High | Replace entirely with streambox-src |

## 12. Open Questions

1. **vdin1 event forwarding mechanism: RESOLVED.** Use vfm_cap `/dev/video_cap`
   as signal monitor fd. vdin1 does not support `VIDIOC_SUBSCRIBE_EVENT` (ENOTTY),
   but vfm_cap does. Plugin opens vfm_cap read-only and subscribes to
   `V4L2_EVENT_SOURCE_CHANGE` — purely event-driven, no polling, no kernel changes.

2. **Audio sync:** New lower-latency Path A may need `mpegtsmux latency` tuning
   to match audio from `alsasrc device=hw:0,6`.

3. **AFBCE:** vdin0 can output AFBCE-compressed frames. The module exposes only
   uncompressed MIF output. If vdin0 is in double-write mode, MIF path is used.

4. **VRR and variable frame rate:** With VRR, frame timing is not constant.
   streambox-src should handle this via `do-timestamp=true` or vframe timestamp
   passthrough.

5. **Path selection in production:** Should the cockpit UI expose a choice between
   Path A and Path B, or should it auto-select based on whether the source is HDR?