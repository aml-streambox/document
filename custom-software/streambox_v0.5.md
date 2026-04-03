# StreamBox v0.5 — Encoder Robustness & UVC Device Support

## 1. Executive Summary

StreamBox v0.5 focuses on encoder stability and feature completeness. The Wave521
hardware encoder has significant untapped capabilities and critical multi-instance
reliability issues that must be resolved before expanding to additional video sources.

Update: HEVC B-frame output, extended GOP preset exposure, and deep-reorder RA_IB
runtime stability are now working in the current development branch. The remaining
work is cleanup, broader soak coverage, and folding the validated fixes into the
main operator and setup documentation.

Recent review-driven cleanup also corrected several implementation details in the
released v0.5 stack, including encoder error recovery, public metadata layout
compatibility, redundant timestamp rewriting, backend initialization behavior,
and Path A buffer-pool configurability.

**Primary goals:**
1. Fix Wave521 encoder multi-instance stability (crash/hang elimination)
2. Fix B-frame encoding support (broken DTS/latency handling)
3. Fix HEVC lossless coding (encoder error on enable)
4. Add UVC device support to Cockpit GStreamer Manager

**Dependency chain:**
```
[1] Multi-instance stability fix
        |
        +--> [2] B-frame fix (needs stable multi-instance for testing)
        |
        +--> [3] Lossless fix (independent, but same driver stack)
        |
        +--> [4] UVC device support (requires stable multi-instance
                 for simultaneous HDMI + UVC encoding)
```

---

## 2. Current Architecture Overview

### 2.1 Encoder Software Stack

```
GStreamer plugin (gstamlvenc_multienc.c)
    |  calls vl_multi_encoder_init/encode/destroy()
    v
libvpcodec.so (libvpmulti_codec.c)
    |  calls AML_MultiEncInitialize/SetInput/NAL/Release()
    v
libamvenc_api.so (AML_MultiEncoder.c)
    |  calls VPU_Init/EncOpen/EncStartOneFrame/EncGetOutputInfo/EncClose()
    v
VPU API layer (vpuapi.c / enc_driver.c)
    |  calls vdi_init/vdi_write_register/vdi_read_register/vdi_allocate_dma_memory()
    v
VDI layer (vdi.c)
    |  ioctl() calls to /dev/amvenc_multi
    v
Kernel Driver (amvenc_multi)
    |
    v
Wave521 VPU Hardware (firmware-driven command mailbox)
```

### 2.2 Key Source Files

| Layer | File | Lines |
|-------|------|-------|
| GStreamer | `multimedia/gst-plugin-venc/multienc-wave521/gstamlvenc_multienc.c` | ~2314 |
| GStreamer | `multimedia/gst-plugin-venc/multienc-wave521/gstamlvenc_multienc.h` | ~100 |
| GStreamer | `multimedia/gst-plugin-venc/multienc-wave521/yuv422_converter_vulkan.c` | ~960 |
| GStreamer | `multimedia/gst-plugin-venc/multienc-wave521/yuv422_converter_gpu_gles.c` | ~704 |
| libvpcodec | `hardware/aml-5.4/amlogic/libencoder/multiEnc/amvenc_lib/libvpmulti_codec.c` | ~1481 |
| libamvenc_api | `hardware/aml-5.4/amlogic/libencoder/multiEnc/amvenc_lib/AML_MultiEncoder.c` | ~2712 |
| API header | `hardware/aml-5.4/amlogic/libencoder/multiEnc/amvenc_lib/include/vp_multi_codec_1_0.h` | ~498 |
| Definitions | `hardware/aml-5.4/amlogic/libencoder/multiEnc/amvenc_lib/include/enc_define.h` | ~428 |
| VDI | `hardware/aml-5.4/amlogic/libencoder/multiEnc/vpuapi/vdi.c` | - |
| VPU API | `hardware/aml-5.4/amlogic/libencoder/multiEnc/vpuapi/vpuapi.c` | - |
| Encoder driver | `hardware/aml-5.4/amlogic/libencoder/multiEnc/vpuapi/enc_driver.c` | - |
| Cockpit plugin | `cockpit-gst-manager/` (separate repo) | - |

---

## 3. Feature 1: Multi-Instance Stability Fix

### 3.1 Problem Statement

When running multiple encoder instances simultaneously (e.g., HDMI capture + UVC
capture, or multiple resolution outputs), a failure in one instance often causes the
entire Wave521 encoder to crash or hang, requiring a device reboot to recover.

### 3.2 Root Cause Analysis — Known Issues

Investigation of the current codebase reveals **16 concrete bugs and design flaws**
that contribute to multi-instance instability:

#### Category A: Global/Static State Corruption (GStreamer Plugin Layer)

| # | Issue | File | Severity |
|---|-------|------|----------|
| A1 | **Static `dbg_frame_num`** shared across all instances | `gstamlvenc_multienc.c` | Medium |
| A2 | **Static `logged_p010_stats`** — logs once globally, not per-instance | `gstamlvenc_multienc.c` | Low |
| A3 | **Global `VulkanCtx ctx`** — Vulkan converter state is a single static global, NOT per-instance safe | `yuv422_converter_vulkan.c` | Critical |
| A4 | **Global `GpuCtx g`** — GLES converter state is a single static global, NOT per-instance safe | `yuv422_converter_gpu_gles.c` | Critical |
| A5 | **GLES shaders hardcoded to 3840x2160** — breaks for other resolutions | `yuv422_converter_gpu_gles.c` | High |
| A6 | **ROI buffer allocation bug**: `width * width` instead of `width * height` | `gstamlvenc_multienc.c` ~L1367 | High |
| A7 | **`ENCODE_TIME_STATISTICS` globals** (`total_encode_time`, `total_encode_frames`) shared across instances when profiling enabled | `gstamlvenc_multienc.c` | Low |

#### Category B: B-Frame / Reordering Bugs (GStreamer Plugin Layer)

| # | Issue | File | Severity |
|---|-------|------|----------|
| B1 | **`max_delayed_frames = 0`** — incorrect when B-frames enabled (causes wrong latency reporting) | `gstamlvenc_multienc.c` | High |
| B2 | **`frame->dts = frame->pts`** always — breaks B-frame decode ordering (DTS must be less than PTS for B frames) | `gstamlvenc_multienc.c` | Critical |
| B3 | **`gst_video_encoder_get_oldest_frame()`** — can mismatch with B-frame reordering since firmware reorders internally | `gstamlvenc_multienc.c` | High |

#### Category C: Library/VDI Layer Bugs

| # | Issue | File | Severity |
|---|-------|------|----------|
| C1 | **`VPU_EncStartOneFrame` QUEUEING_FAILURE** — infinite retry loop via `goto retry_point` with no escape | `AML_MultiEncoder.c` | Critical |
| C2 | **`ge2d_colorFormat()` uses static globals** (`SRC1_PIXFORMAT`, `SRC2_PIXFORMAT`, `DST_PIXFORMAT`) — not thread-safe for concurrent instances doing format conversion | `AML_MultiEncoder.c` | High |
| C3 | **`vdi_init()` memset bug** — `memset(&s_vdi_info[core_idx], 0x00, sizeof(s_vdi_info))` zeros the ENTIRE `s_vdi_info` array (all cores), not just `[core_idx]` entry | `vdi.c` | Critical |
| C4 | **`vdi_init_flag` race** — protected by `vid_mutex` but the full-array zeroing in C3 can corrupt other cores' initialized state | `vdi.c` | Critical |
| C5 | **PTS counter wraps at UINT32_MAX** — potential fragility for long recordings | `gstamlvenc_multienc.c` | Low |

#### Category D: Error Recovery Gaps

| # | Issue | File | Severity |
|---|-------|------|----------|
| D1 | **`HandlingInterruptFlag()` timeout** → `VPU_SWReset()` resets the ENTIRE VPU, killing ALL instances, not just the failing one | `AML_MultiEncoder.c` | Critical |
| D2 | **`RETCODE_VLC_BUF_FULL`** → logged but encoding continues (potential bitstream corruption) | `AML_MultiEncoder.c` | High |
| D3 | **Header encoding retries up to 100 times**, then `VPU_SWReset()` (same global reset problem as D1) | `AML_MultiEncoder.c` | High |

### 3.3 Reference: Mainline Linux Wave5 Driver Architecture

The mainline Linux kernel (`drivers/media/platform/chips-media/wave5/`) provides a
clean reference implementation that handles multi-instance correctly:

**Key architectural patterns to adopt:**

1. **Single `hw_lock` mutex** for firmware command serialization — the VPU firmware
   command mailbox is inherently serial; one mutex protects all register access

2. **Per-instance state machine** with validated transitions:
   ```
   NONE -> OPEN -> INIT_SEQ -> PIC_RUN -> STOP
   ```
   Invalid transitions trigger `WARN()` — catches bugs during development

3. **Per-instance interrupt dispatch** via hardware bitmask register
   (`W5_RET_QUEUE_CMD_DONE_INST`) — each bit is an instance ID. IRQ handler reads
   bitmask, dispatches completions to per-instance `kfifo`

4. **Per-instance `kfifo`** for IRQ status — clean queuing of multiple completions
   per instance without loss

5. **Per-instance error isolation**: on encode failure, return `VB2_BUF_STATE_ERROR`
   on that instance's buffers only, transition instance to `STOP` state. Other
   instances unaffected

6. **Retry with `MAX_FIRMWARE_CALL_RETRY = 10`** (not infinite) — bounded retry with
   `hw_lock` released between attempts to let other instances proceed

7. **Per-instance memory allocation**: each instance gets own work buffer, task
   buffer, FBC tables, MV buffers — one instance's corruption doesn't overwrite
   another's data structures

8. **V4L2 M2M framework**: standard `v4l2_m2m_ctx` per instance with proper buffer
   lifecycle management

**Known mainline limitation**: No per-instance hardware reset. If firmware truly
hangs, all instances block. The Wave521 firmware's command queue depth is only 2.

### 3.4 Fix Plan

#### Phase 1A: GStreamer Plugin Global State Elimination

**Goal:** Make every `amlvenc` element instance fully independent.

| Fix | Description | Risk |
|-----|-------------|------|
| Move Vulkan `VulkanCtx` to per-instance | Allocate `VulkanCtx` in element instance struct, init in `start()`, cleanup in `stop()` | Medium — need to verify Mali-G52 supports multiple Vulkan instances |
| Move GLES `GpuCtx` to per-instance | Same pattern as Vulkan | Medium |
| Move `dbg_frame_num` to instance struct | Trivial rename | Low |
| Move `logged_p010_stats` to instance struct | Trivial rename | Low |
| Fix ROI allocation: `width * height` | One-line fix | Low |
| Move encode timing stats to instance struct | Conditional on `ENCODE_TIME_STATISTICS` | Low |
| Fix GLES shader resolution hardcoding | Pass width/height as uniforms | Medium |

#### Phase 1B: Library Layer Thread Safety Fixes

| Fix | Description | Risk |
|-----|-------------|------|
| Fix `vdi_init()` memset | Change to `memset(&s_vdi_info[core_idx], 0x00, sizeof(s_vdi_info[0]))` — only zero the specific core entry | Low — critical correctness fix |
| Fix `ge2d_colorFormat()` globals | Move to per-call local variables or per-instance context | Low |
| Bound `VPU_EncStartOneFrame` retry | Add `MAX_RETRY_COUNT` (e.g., 10) to the queueing failure retry loop, return error after exhaustion | Medium |
| Bound header encoding retry | Same pattern — bounded retry with error return | Medium |

#### Phase 1C: Error Isolation

| Fix | Description | Risk |
|-----|-------------|------|
| Per-instance error handling | On timeout in `HandlingInterruptFlag()`, close and destroy only the failing instance, NOT `VPU_SWReset()` the entire VPU | High — need firmware cooperation; may require fallback to global reset if per-instance close also hangs |
| `VLC_BUF_FULL` handling | Request IDR on next frame and skip current (like `AMVENC_TIMEOUT` handling), instead of continuing with potentially corrupt bitstream | Medium |
| Instance state machine | Add per-instance state tracking (INIT → RUNNING → ERROR → STOPPED) with `WARN()` on invalid transitions | Low |

#### Phase 1D: Integration Testing

| Test | Duration | Instances | Description |
|------|----------|-----------|-------------|
| Dual 1080p60 encode | 30 min | 2 | Two simultaneous 1080p60 encodes to file |
| 4K60 + 1080p30 | 30 min | 2 | Primary 4K capture + secondary downscaled |
| Instance crash recovery | - | 2 | Kill one instance mid-encode, verify other continues |
| Instance start/stop churn | 10 min | 2 | Rapidly start/stop one instance while other runs |
| Long duration dual encode | 8 hr | 2 | Overnight stability test |

### 3.5 Files to Modify

| File | Changes |
|------|---------|
| `gstamlvenc_multienc.c` | Per-instance state, fix globals, bounded frame counting |
| `gstamlvenc_multienc.h` | Add VulkanCtx/GpuCtx pointers, state machine enum, per-instance counters |
| `yuv422_converter_vulkan.c` | Refactor from global `ctx` to per-instance context pointer |
| `yuv422_converter_vulkan.h` | Update API signatures to take context pointer |
| `yuv422_converter_gpu_gles.c` | Refactor from global `g` to per-instance context pointer |
| `yuv422_converter_gpu_gles.h` | Update API signatures |
| `AML_MultiEncoder.c` | Bounded retries, per-instance error handling, fix ge2d globals |
| `vdi.c` | Fix memset bug, verify mutex robustness |
| `libvpmulti_codec.c` | Error propagation improvements |

### 3.6 Current Status Update

The encoder stack has moved beyond the original investigation state in this document.
The following items are now implemented and verified on the T7/A311D2 target:

- HEVC B-frame flashback fix: delay-frame SPS/PPS prepend bug fixed in
  `hardware/aml-5.4/amlogic/libencoder/multiEnc/amvenc_lib/libvpmulti_codec.c`
- Plugin header handling cleanup: redundant plugin-level HEVC header prepend removed
  from `multimedia/gst-plugin-venc/multienc-wave521/gstamlvenc_multienc.c`
- Extended GOP exposure: `gop-pattern` now exposes the full Wave5 preset set
  (`IPP`, `IBBBP`, `IBPBP`, `IBBB`, `ALL_I`, `IPPPP`, `IBBBB`, `RA_IB`,
  `IPP_SINGLE`)
- Deep reorder runtime fix for `RA_IB`: source/output buffering increased so the
  pipeline no longer stalls before first output

#### RA_IB Runtime Root Cause and Fix

`RA_IB` did not fail because of the preset mapping itself. Two queue-depth issues
caused the runtime failure:

1. `streamboxsrc` Path-A DMA-buf output pool was fixed at 6 buffers, but `RA_IB`
   needs substantially deeper in-flight buffering before first encoded output.
   This caused `Path A output pool exhausted after 300ms (6 buffers in use)`.
2. `AML_MultiEncoder.c` provisioned source slots as `minSrcFrameCount + 2`, which
   is too small for deep-reorder GOPs. The Wave sample path uses
   `minSrcFrameCount + COMMAND_QUEUE_DEPTH`. With `RA_IB`, the smaller count caused
   premature `srcIdx` reuse and `source frame was already in encoding index ...`.

The validated fixes are:

- `multimedia/gst-plugin-vfmcap/gst_streambox_src.h`: `PATHA_OUT_POOL_SIZE` from
  `6` to `12`
- `hardware/aml-5.4/amlogic/libencoder/multiEnc/amvenc_lib/AML_MultiEncoder.c`:
  `ctx->src_num = initialInfo->minSrcFrameCount + COMMAND_QUEUE_DEPTH`, clamped to
  `ENC_SRC_BUF_NUM`

#### Verified GOP Results

Verified on target with `gop=30` and `gop=60`:

- `gop-pattern=0` `IPP`: pass
- `gop-pattern=1` `IBBBP`: pass, real B frames present, 0 flashback suspects
- `gop-pattern=2` `IBPBP`: pass, real B frames present, 0 flashback suspects
- `gop-pattern=3` `IBBB`: pass, real B frames present, 0 flashback suspects
- `gop-pattern=4` `ALL_I`: pass
- `gop-pattern=5` `IPPPP`: pass
- `gop-pattern=6` `IBBBB`: pass, real B frames present, 0 flashback suspects
- `gop-pattern=7` `RA_IB`: pass after pool-depth and source-slot fixes, 0 flashback suspects
- `gop-pattern=8` `IPP_SINGLE`: pass

---

## 4. Feature 2: B-Frame Encoding Support

### 4.1 Current Status

Wave521 HEVC B-frame encoding is now working in the v0.5 stack.

Validated results:
- real B-frames are present in the output bitstream
- frame matching is stable under reorder
- visual flashback artifacts are eliminated
- file output and SRT output both work with B-frame HEVC streams

### 4.2 Implemented Fix Summary

The main correctness issue was caused by startup delay frames during deep reorder.
Those frames were incorrectly treated as real encoded output, which shifted frame
tracking and broke timestamp assignment for subsequent B-frames.

The implemented fixes include:
- suppressing fake output on delay frames with zero payload length
- removing redundant plugin-side HEVC header prepend logic
- updating reorder-related latency and timestamp handling for B-frame GOPs
- extending GOP handling so the plugin and userspace encoder agree on preset behavior

### 4.3 Public Verification

Example B-frame encode:

```bash
gst-launch-1.0 -e \
    streamboxsrc ! "video/x-raw,format=NV12" \
    ! amlvenc gop-pattern=1 gop=60 bitrate=30000 framerate=60 internal-bit-depth=10 \
    ! video/x-h265 ! h265parse ! filesink location=output_bframe.h265
```

Check that B-frames are present:

```bash
ffprobe -select_streams v -show_frames -of csv output_bframe.h265
```

### 4.4 Verified GOP Presets

The following `gop-pattern` values are verified in the current v0.5 encoder stack:

| Value | Pattern | Status |
|------|---------|--------|
| 0 | IPP | Verified |
| 1 | IBBBP | Verified |
| 2 | IBPBP | Verified |
| 3 | IBBB | Verified |
| 4 | ALL_I | Verified |
| 5 | IPPPP | Verified |
| 6 | IBBBB | Verified |
| 7 | RA_IB | Verified |
| 8 | IPP_SINGLE | Verified |

---

## 5. Feature 3: HEVC Lossless Coding

### 5.1 Current Status

HEVC lossless mode is now working in the current v0.5 encoder stack.

The implementation enables the correct lossless path for HEVC and avoids the
configuration conflicts that previously caused encoder initialization failure.

### 5.2 Practical Notes

- lossless mode is intended for HEVC only
- higher bitstream usage is expected compared with lossy presets
- 10-bit input and output paths are supported by the current Wave521 pipeline

### 5.3 Public Verification

Example lossless encode:

```bash
gst-launch-1.0 -e \
    streamboxsrc source=vfmcap output-format=p010 \
    ! amlvenc lossless-enable=true internal-bit-depth=10 framerate=60 \
    ! video/x-h265 ! h265parse ! filesink location=output_lossless.h265
```

---

## 6. Feature 4: UVC Device Support in Cockpit GStreamer Manager

### 6.1 Current Status

The UVC media path in v0.5 has moved beyond basic device detection. The working
pipeline is now:

```bash
v4l2src (UVC H.264) -> h264parse -> amlv4l2h264dec -> amlvenc -> h265parse \
    -> mpegtsmux -> srtsink
```

Validated results:
- hardware decode and re-encode are working
- DMA-BUF based decode-to-encode handoff is working
- live SRT output is working
- the path is suitable for Cockpit-managed pipeline generation and control

### 6.2 Current Working Example

```bash
gst-launch-1.0 -e -v \
    v4l2src device=/dev/video0 \
    ! "video/x-h264,width=1920,height=1080,framerate=30/1" \
    ! h264parse \
    ! amlv4l2h264dec \
    ! queue max-size-buffers=5 max-size-time=0 max-size-bytes=0 \
    ! "video/x-raw,format=NV12" \
    ! amlvenc bitrate=4000 framerate=30 gop=5 gop-pattern=0 \
    ! video/x-h265 \
    ! h265parse config-interval=-1 \
    ! queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 \
    ! mpegtsmux alignment=7 latency=100000000 \
    ! srtsink uri="srt://:8889" wait-for-connection=false sync=false
```

Practical note:
- `bitrate=4000` means 4 Mbps
- `gop=5` is a good normal operating point for live streaming
- `gop=1` is a lower-risk choice when very fast decoder join behavior is required

### 6.3 Remaining Work

The remaining UVC work is mainly product integration rather than codec bring-up:
- improved UI workflow in Cockpit GStreamer Manager
- automatic UVC capability discovery and pipeline generation
- broader multi-instance soak testing with HDMI and UVC running together

---

## 7. Current Validation Focus

The main remaining validation theme for v0.5 is long-duration robustness under
real mixed workloads.

Priority validation areas:
- HDMI and UVC pipelines running simultaneously
- repeated start/stop and restart of one pipeline while another remains active
- long-duration SRT streaming stability
- extended soak testing across the verified GOP presets

---

## 8. Public Success Criteria

| Criterion | Measurement |
|-----------|-------------|
| Multi-instance stability | Multiple encoder pipelines run for long duration without crash or hang |
| B-frame correctness | Output bitstream contains correct I/P/B structure without flashback artifacts |
| Lossless encoding | HEVC lossless encode completes successfully |
| Extended GOP support | Verified operation across the exposed GOP presets |
| UVC transcoding | UVC H.264 to HEVC to SRT pipeline runs reliably |
| Mixed workload stability | HDMI and UVC pipelines can run together without corrupting each other |

---

## 9. References

- [StreamBox v0.4]({{ '/custom-software/streambox_v0.4' | relative_url }}) — dual-path capture architecture
- Mainline Linux wave5 driver: `drivers/media/platform/chips-media/wave5/`
- ITU-T H.265 specification
- USB Video Class specification
- V4L2 userspace API documentation
