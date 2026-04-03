# StreamBox v0.5 — Encoder Robustness & UVC Device Support

## 1. Executive Summary

StreamBox v0.5 focuses on encoder stability and feature completeness. The Wave521
hardware encoder has significant untapped capabilities and critical multi-instance
reliability issues that must be resolved before expanding to additional video sources.

Update: HEVC B-frame output, extended GOP preset exposure, and deep-reorder RA_IB
runtime stability are now working in the current development branch. The remaining
work is cleanup, broader soak coverage, and folding the validated fixes into the
main operator and setup documentation.

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

### 4.1 Problem Statement

The Wave521 hardware supports B-frame encoding with multiple GOP presets (IBBBP,
IBPBP, IBBB, etc.), and the GStreamer plugin already exposes a `gop-pattern`
property (values 0-4). However, B-frame encoding is **broken at the GStreamer
plugin level** due to incorrect timestamp handling and frame management.

### 4.2 Root Cause

The Wave521 firmware handles B-frame reference management and reordering internally.
When B-frames are enabled:

1. The firmware buffers input frames and may output them out of order
2. `encOutputInfo.encSrcIdx == 0xfffffffe` indicates a "delay frame" (reorder delay)
   — the firmware needs more input before it can output
3. The `AML_MultiEncNAL()` layer handles this by returning `buf_nal_size = 0`

**Three bugs prevent correct B-frame operation:**

| Bug | Current Code | Correct Behavior |
|-----|-------------|------------------|
| `max_delayed_frames = 0` | GStreamer reports 0 latency | Must be >0 for B-frames (typically GOP-size dependent: IBBBP needs 3-4 delayed frames) |
| `frame->dts = frame->pts` | DTS equals PTS for every frame | B-frames require DTS < PTS offset; the decode order differs from presentation order |
| `gst_video_encoder_get_oldest_frame()` | Assumes output order = input order | With B-frame reordering, the firmware outputs frames out of order; need to match by `encSrcIdx` |

### 4.3 Fix Plan

#### Phase 2A: Latency Reporting

```c
// In gst_amlvenc_set_format() or gst_amlvenc_start():
if (self->gop_pattern == GOP_PATTERN_IBBBP) {
    self->max_delayed_frames = 4;  // 3 B-frames + 1 reorder delay
} else if (self->gop_pattern == GOP_PATTERN_IBPBP) {
    self->max_delayed_frames = 2;
} else if (self->gop_pattern == GOP_PATTERN_IBBB) {
    self->max_delayed_frames = 3;
} else {
    self->max_delayed_frames = 0;  // IPP, All-I
}
```

#### Phase 2B: DTS Calculation

With B-frames, DTS must account for reorder delay:

```c
// For IPP (no B-frames): DTS = PTS (current behavior, correct)
// For IBBBP: DTS = PTS - (max_delayed_frames * frame_duration)
//   - First max_delayed_frames output frames: DTS starts before first PTS
//   - Subsequent frames: DTS = PTS of the frame that was decoded at this position

// Approach: maintain a PTS queue and assign DTS from queue head
// The firmware tells us which source frame index was encoded via encSrcIdx
```

#### Phase 2C: Frame Matching

Replace `gst_video_encoder_get_oldest_frame()` with lookup by encoder source index.
The firmware reports `encOutputInfo.encSrcIdx` which maps back to the original
input frame number. Use this to find the correct `GstVideoCodecFrame`.

#### Phase 2D: B-Frame QP Configuration

Currently hardcoded to 0. Expose via GStreamer properties:

| Property | Type | Default | Range | Description |
|----------|------|---------|-------|-------------|
| `qp-b` | int | 0 (auto) | 0-51 | Base QP for B-frames (0 = use encoder default) |
| `min-qp-b` | int | 0 | 0-51 | Minimum QP for B-frames |
| `max-qp-b` | int | 51 | 0-51 | Maximum QP for B-frames |

### 4.4 Testing

```bash
# B-frame encode with IBBBP pattern
gst-launch-1.0 -e streamboxsrc source=vfmcap output-format=p010 \
    ! amlvenc gop-pattern=1 gop=60 bitrate=30000 framerate=60 internal-bit-depth=10 \
    ! video/x-h265 ! h265parse ! filesink location=/tmp/test_bframe.h265

# Verify B-frames with ffprobe
ffprobe -select_streams v -show_frames -of csv /tmp/test_bframe.h265 \
    | grep "pict_type" | sort | uniq -c
# Expected: I, P, and B frames present

# Verify DTS < PTS for B-frames
ffprobe -select_streams v -show_frames -of csv /tmp/test_bframe.h265 \
    | head -20
# Expected: dts values differ from pts values for B-frames

# Dual-instance B-frame test (requires Feature 1 fixes)
# Instance 1: 4K60 IBBBP
# Instance 2: 1080p30 IBBBP
```

### 4.5 Files to Modify

| File | Changes |
|------|---------|
| `gstamlvenc_multienc.c` | DTS calculation, frame matching by encSrcIdx, latency reporting, B-frame QP properties |
| `gstamlvenc_multienc.h` | PTS queue, delayed frame tracking, B-frame QP fields |
| `libvpmulti_codec.c` | Propagate B-frame QP params |
| `AML_MultiEncoder.c` | Verify B-frame QP params are written to hardware registers |

---

## 5. Feature 3: HEVC Lossless Coding Fix

### 5.1 Problem Statement

When `lossless-enable=true` is set on the `amlvenc` GStreamer element, the Wave521
encoder reports an error and fails to initialize. The hardware supports lossless
coding (documented in enc_define.h), and the code path exists end-to-end:

```
GStreamer `lossless-enable` boolean
  → encode_info.lossless_enable
  → mEncParams.lossless_enable
  → SetupEncoderOpenParam() → param->losslessEnable
```

When enabled, the library correctly disables incompatible features:
- Rate control (rcEnable = 0)
- Noise reduction (nrYEnable/nrCbEnable/nrCrEnable = 0)
- ROI (roiEnable = 0)
- Background detection (bgDetectEnable = 0)
- Enables transform skip (skipIntraTrans = 1)
- Validated as HEVC-only (error returned for H.264)

### 5.2 Investigation Plan

The error likely comes from one of these sources:

1. **Firmware parameter validation**: The firmware may reject certain parameter
   combinations when lossless is enabled (e.g., specific profile requirements,
   bit depth settings, or GOP structure incompatibilities)

2. **Missing profile setting**: HEVC lossless requires Main RExt profile
   (`HEVC_PROFILE_MAIN_REXT`), but the driver may be selecting Main or Main10
   which don't support lossless per the HEVC specification

3. **Rate control conflict**: Even though `rcEnable` is set to 0, other RC-related
   parameters may still be configured in a conflicting way

4. **QP setting**: Lossless encoding requires QP=0. The initial QP may be set to
   a non-zero value

5. **Bitstream buffer size**: Lossless output can be significantly larger than
   lossy (3-5x). The bitstream buffer may be too small, causing overflow

### 5.3 Fix Plan

#### Phase 3A: Reproduce and Capture Error

```bash
# Reproduce the lossless encoding error
gst-launch-1.0 -e videotestsrc pattern=snow num-buffers=10 \
    ! video/x-raw,format=NV12,width=1920,height=1080 \
    ! amlvenc lossless-enable=true \
    ! video/x-h265 ! h265parse ! filesink location=/tmp/lossless.h265

# Capture full error log
GST_DEBUG=amlvenc:5 gst-launch-1.0 -e ...
```

#### Phase 3B: Diagnose Root Cause

1. Check firmware return code from `VPU_EncOpen()` or `VPU_EncStartOneFrame()`
2. Check if profile is correctly set to Main RExt
3. Verify QP=0 is being set when lossless enabled
4. Check bitstream buffer sizing
5. Check for any parameter validation that rejects lossless configuration

#### Phase 3C: Apply Fix

Based on likely root cause:
- Set profile to `HEVC_PROFILE_MAIN_REXT` when lossless enabled
- Force initial QP to 0
- Increase bitstream buffer size for lossless
- Ensure all conflicting parameters are properly disabled

#### Phase 3D: Verification

```bash
# Lossless encode test
gst-launch-1.0 -e streamboxsrc source=vfmcap output-format=p010 num-buffers=60 \
    ! amlvenc lossless-enable=true internal-bit-depth=10 framerate=60 \
    ! video/x-h265 ! h265parse ! filesink location=/tmp/lossless.h265

# Verify lossless with ffprobe/mediainfo
mediainfo /tmp/lossless.h265
# Expected: Format profile should show lossless/RExt

# Bitwise verification (decode and compare):
# Since true lossless, decoded output should be bit-identical to input
```

### 5.4 Files to Modify

| File | Changes |
|------|---------|
| `AML_MultiEncoder.c` | Profile selection, QP forcing, parameter validation for lossless |
| `gstamlvenc_multienc.c` | Bitstream buffer sizing, error reporting improvements |
| `vp_multi_codec_1_0.h` | If profile param needs to be added to API |

---

## 6. Feature 4: UVC Device Support in Cockpit GStreamer Manager

### 6.1 Problem Statement

The Cockpit GStreamer Manager currently only manages pipelines for the HDMI capture
path. Users who connect USB Video Class (UVC) devices (webcams, USB capture cards)
have no way to configure and manage GStreamer pipelines for them through the UI.

### 6.2 Dependency

**This feature REQUIRES Feature 1 (multi-instance stability) to be complete.**

UVC device pipelines will run simultaneously with HDMI capture pipelines, meaning
the Wave521 encoder must handle multiple concurrent instances reliably. If one
encoder instance hangs or crashes, it must not bring down the other.

### 6.2.1 Current Bring-up Status (2026-04-02)

The UVC H.264 transcoding path is now working end-to-end on the T7/A311D2 test
device with hardware decode, hardware encode, and DMA-BUF buffer sharing between
decoder and encoder.

Validated path:

```bash
v4l2src (UVC H.264) -> h264parse -> amlv4l2h264dec -> amlvenc -> h265parse \
    -> mpegtsmux -> srtsink
```

Validated findings:

1. `amlvenc bitrate` is in kbps, so `bitrate=4000` means 4 Mbps.
2. The decoder allocation path needed fixes so DMABUF-backed capture buffers stay
   on the V4L2 pool and are not deep-copied into generic memory.
3. Decode -> encode with DMA-BUF zero-copy is stable for sustained runs.
4. Live SRT streaming works reliably when the encoder produces frequent IDRs;
   `gop=5` is acceptable for normal use, while `gop=1` gives the cleanest
   late-join startup behavior.

Current working UVC command:

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

Fallback low-risk streaming variant: use `gop=1` if the receiver must attach
cleanly at arbitrary times with minimal startup decode errors.

### 6.3 Design

#### 6.3.1 New "UVC Devices" Tab

Add a new tab in the Cockpit GStreamer Manager alongside the existing tabs. This
tab provides:

1. **Automatic UVC device detection** — scan `/dev/video*` devices, filter to UVC
   class devices using V4L2 capabilities and USB bus info
2. **Device capability parsing** — enumerate supported formats, resolutions, and
   frame rates via `VIDIOC_ENUM_FMT` / `VIDIOC_ENUM_FRAMESIZES` /
   `VIDIOC_ENUM_FRAMEINTERVALS`
3. **User-configurable pipeline generation** — let user select format, resolution,
   framerate, encoder settings, output destination
4. **Pipeline save/load** — save generated pipelines as named instances

#### 6.3.2 UVC Device Detection

Distinguish UVC devices from HDMI capture devices (`/dev/video71`, `/dev/video_cap`):

```python
# Backend detection logic (Python):
# 1. Enumerate /dev/video* devices
# 2. Open each, query VIDIOC_QUERYCAP
# 3. Filter by V4L2_CAP_VIDEO_CAPTURE (single-planar) or V4L2_CAP_VIDEO_CAPTURE_MPLANE
# 4. Check bus_info field: UVC devices have "usb-*" prefix
# 5. Exclude known HDMI devices by path (/dev/video71, /dev/video_cap)
# 6. Read /sys/class/video4linux/videoN/device → follow USB symlinks
#    for vendor/product/serial info
```

#### 6.3.3 Capability Parsing

For each detected UVC device, build a capability tree:

```
USB Camera (Logitech C920, /dev/video0)
├── MJPEG
│   ├── 1920x1080 @ 30fps, 24fps
│   ├── 1280x720  @ 60fps, 30fps
│   └── 640x480   @ 120fps, 60fps, 30fps
├── YUYV (YUV 4:2:2)
│   ├── 1920x1080 @ 5fps
│   ├── 1280x720  @ 10fps
│   └── 640x480   @ 30fps
└── H.264 (if supported)
    ├── 1920x1080 @ 30fps
    └── 1280x720  @ 60fps
```

#### 6.3.4 Pipeline Generation

Based on user selections, generate a GStreamer pipeline:

```bash
# UVC MJPEG → decode → HW encode → SRT
gst-launch-1.0 -e \
    v4l2src device=/dev/video0 \
    ! image/jpeg,width=1920,height=1080,framerate=30/1 \
    ! jpegdec \
    ! videoconvert \
    ! video/x-raw,format=NV12 \
    ! amlvenc bitrate=5000 gop=60 framerate=30 \
    ! video/x-h265 ! h265parse config-interval=-1 \
    ! mpegtsmux alignment=7 \
    ! srtsink uri="srt://:8889" wait-for-connection=false

# UVC YUYV → HW encode → SRT
gst-launch-1.0 -e \
    v4l2src device=/dev/video0 \
    ! video/x-raw,format=YUY2,width=1280,height=720,framerate=30/1 \
    ! videoconvert \
    ! video/x-raw,format=NV12 \
    ! amlvenc bitrate=3000 gop=60 framerate=30 \
    ! video/x-h265 ! h265parse config-interval=-1 \
    ! mpegtsmux alignment=7 \
    ! srtsink uri="srt://:8889" wait-for-connection=false

# UVC H.264 passthrough → remux → SRT (no re-encoding)
gst-launch-1.0 -e \
    v4l2src device=/dev/video0 \
    ! video/x-h264,width=1920,height=1080,framerate=30/1 \
    ! h264parse config-interval=-1 \
    ! mpegtsmux alignment=7 \
    ! srtsink uri="srt://:8889" wait-for-connection=false

# UVC H.264 -> HW decode -> HW HEVC encode -> SRT
gst-launch-1.0 -e \
    v4l2src device=/dev/video0 \
    ! video/x-h264,width=1920,height=1080,framerate=30/1 \
    ! h264parse \
    ! amlv4l2h264dec \
    ! queue max-size-buffers=5 max-size-time=0 max-size-bytes=0 \
    ! video/x-raw,format=NV12 \
    ! amlvenc bitrate=4000 framerate=30 gop=5 gop-pattern=0 \
    ! video/x-h265 \
    ! h265parse config-interval=-1 \
    ! queue max-size-buffers=30 max-size-time=0 max-size-bytes=0 \
    ! mpegtsmux alignment=7 latency=100000000 \
    ! srtsink uri="srt://:8889" wait-for-connection=false sync=false
```

#### 6.3.5 UI Mockup

```
┌─────────────────────────────────────────────────────────┐
│ GStreamer Manager  │ Auto │ Custom │ UVC Devices │       │
├────────────────────┴──────┴────────┴─────────────┴──────┤
│                                                         │
│  Detected UVC Devices:                                  │
│  ┌─────────────────────────────────────────────────┐    │
│  │ ● Logitech C920 (/dev/video0)                   │    │
│  │   USB 2.0, VID:046d PID:082d                    │    │
│  │   Formats: MJPEG, YUYV, H.264                   │    │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │ ○ USB Capture Card (/dev/video2)                │    │
│  │   USB 3.0, VID:1234 PID:5678                    │    │
│  │   Formats: YUYV, NV12                           │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  [Refresh Devices]                                      │
│                                                         │
│  ─── Configure Pipeline ────────────────────────────    │
│                                                         │
│  Input Format:    [MJPEG ▼]                             │
│  Resolution:      [1920x1080 ▼]                         │
│  Framerate:       [30 fps ▼]                            │
│                                                         │
│  Encoder:         [H.265 (HW) ▼]                       │
│  Bitrate (kbps):  [5000      ]                          │
│  GOP Size:        [60        ]                          │
│                                                         │
│  Output:          [SRT ▼]                               │
│  SRT URI:         [srt://:8889                  ]       │
│                                                         │
│  Pipeline Name:   [uvc-logitech-c920            ]       │
│                                                         │
│  [Generate Pipeline]  [Save & Start]                    │
│                                                         │
│  ─── Generated Pipeline ────────────────────────────    │
│  gst-launch-1.0 -e v4l2src device=/dev/video0 ...      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 6.3.6 Hot-plug Detection

Use `udev` monitoring via a backend service to detect USB device connect/disconnect:

- On connect: auto-detect capabilities, notify UI
- On disconnect: if pipeline is running, trigger clean shutdown (same pattern as
  HDMI signal change — EOS, flush, stop)
- Manual "Refresh Devices" button as fallback

### 6.4 Implementation Plan

#### Phase 4A: Backend UVC Discovery API

Add a new backend endpoint/tool that:
1. Scans `/dev/video*` for UVC devices
2. Queries capabilities via V4L2 ioctls
3. Returns JSON device tree

#### Phase 4B: Pipeline Builder Logic

Add pipeline template generation for UVC sources:
1. MJPEG → jpegdec → videoconvert → NV12 → amlvenc → output
2. YUYV → videoconvert → NV12 → amlvenc → output
3. H.264 passthrough → remux → output (no re-encoding)
4. Handle format-specific quirks (e.g., MJPEG quality varies by device)

#### Phase 4C: Cockpit UI Tab

1. New "UVC Devices" tab component
2. Device list with auto-refresh
3. Configuration form (format, resolution, fps, encoder, output)
4. Pipeline preview and save/start controls

#### Phase 4D: Instance Management

1. New instance type for UVC pipelines
2. Lifecycle management (start, stop, restart)
3. Status monitoring (encoding stats, errors)
4. Persist saved UVC pipeline configurations

### 6.4.1 Next Validation Sequence

Before Cockpit integration work starts, the following encoder multi-instance work
must be completed on the real target:

1. Run the HDMI RX pipeline and the UVC transcode pipeline together to validate
   dual encoder instance operation.
2. Verify that restarting one pipeline does not interrupt or corrupt the other
   running pipeline.
3. Create a stress script that randomly starts, stops, and restarts either the
   HDMI or UVC instance through full lifecycle transitions.
4. Run long-duration churn testing and fix any remaining cross-instance faults.
5. Only after this is stable, integrate UVC auto-pipeline generation into the
   Cockpit GStreamer Manager.

### 6.5 Files to Create/Modify

| File | Action | Description |
|------|--------|-------------|
| `cockpit-gst-manager/tools.py` | Modify | Add UVC device discovery tool, pipeline generation |
| `cockpit-gst-manager/agent.py` | Modify | Update agent prompt with UVC knowledge |
| `cockpit-gst-manager/events.py` | Modify | Handle UVC device connect/disconnect events |
| `cockpit-gst-manager/src/pipeline-editor.js` | Modify | Add UVC tab, device list, configuration form |
| `cockpit-gst-manager/src/uvc-discovery.js` | Create | Frontend UVC device enumeration |
| `cockpit-gst-manager/uvc_utils.py` | Create | Backend UVC V4L2 capability parsing |

---

## 7. Implementation Schedule

### Phase 1: Multi-Instance Stability (Weeks 1-3)

| Week | Tasks |
|------|-------|
| Week 1 | Phase 1A: GStreamer global state elimination (Vulkan/GLES per-instance, fix static globals, fix ROI bug) |
| Week 2 | Phase 1B + 1C: VDI memset fix, bounded retries, per-instance error handling, ge2d thread safety |
| Week 3 | Phase 1D: Multi-instance integration testing (dual encode, crash recovery, long-duration) |

### Phase 2: B-Frame Support (Week 4)

| Week | Tasks |
|------|-------|
| Week 4 | Phase 2A-2D: DTS calculation, frame matching by encSrcIdx, latency reporting, B-frame QP properties, testing |

### Phase 3: Lossless Fix (Week 4-5)

| Week | Tasks |
|------|-------|
| Week 4-5 | Phase 3A-3D: Reproduce, diagnose, fix, verify (can overlap with B-frame testing) |

### Phase 4: UVC Device Support (Weeks 5-7)

| Week | Tasks |
|------|-------|
| Week 5 | Phase 4A-4B: Backend UVC discovery and pipeline builder |
| Week 6 | Phase 4C: Cockpit UI tab implementation |
| Week 7 | Phase 4D: Instance management, testing, polish |

### Final: Integration Testing (Week 8)

| Test | Description |
|------|-------------|
| HDMI + UVC simultaneous | 4K60 HDMI capture + 1080p30 UVC webcam, both encoding to SRT |
| Signal change during dual encode | HDMI source changes while UVC pipeline running |
| UVC hot-plug during HDMI encode | Plug/unplug USB device while HDMI pipeline active |
| B-frame dual instance | Both HDMI and UVC using IBBBP GOP pattern |
| Long-duration combined | 8-hour test with HDMI + UVC simultaneous encoding |

### Current Immediate Next Step

The next active debugging task is now dual-instance validation rather than
single-pipeline UVC bring-up:

1. Keep one HDMI RX encoder pipeline running continuously.
2. Start and stop the UVC H.264 -> HEVC transcode pipeline repeatedly.
3. Confirm existing encoder instances survive unrelated instance lifecycle churn.
4. Convert the validated behavior into an automated randomized stress test.

---

## 8. Branch Strategy

All code changes for v0.5 will be made on `v0.5_dev` branches in each affected repository:

| Repository | Branch |
|-----------|--------|
| `aml-comp/multimedia/gst-plugin-venc` | `v0.5_dev` |
| `aml-comp/hardware/aml-5.4/amlogic/libencoder` | `v0.5_dev` |
| `cockpit-gst-manager` | `v0.5_dev` |
| `aml-comp/kernel/aml-5.15` | `v0.5_dev` (if kernel driver changes needed) |
| `meta-meson` | `v0.5_dev` (recipe updates) |
| `meta-aml-cfg` | `v0.5_dev` (recipe updates) |

---

## 9. Risk Assessment

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| VPU firmware doesn't support per-instance error recovery | High | Medium | Fall back to global reset with bounded retry + instance restart |
| Mali-G52 doesn't support multiple Vulkan instances | High | Low | If not, serialize Vulkan init/cleanup with a process mutex |
| B-frame DTS calculation edge cases | Medium | Medium | Reference mainline wave5 driver's handling; extensive testing with ffprobe |
| Lossless requires firmware update | High | Low | Test with current firmware first; document minimum firmware version if needed |
| UVC devices with non-standard V4L2 behavior | Medium | Medium | Test with popular devices (Logitech C920/C930, Elgato Cam Link); add quirk table |
| Multi-instance testing coverage insufficient | High | Medium | Automated test scripts, long-duration soak tests, KFENCE enabled for UAF detection |

---

## 10. Success Criteria

| Criterion | Measurement |
|-----------|-------------|
| Multi-instance stability | 2 encoder instances running 8 hours without crash/hang |
| Instance crash isolation | Killing one instance does not affect the other |
| B-frame correctness | ffprobe shows correct I/P/B frame types and DTS < PTS |
| B-frame compression | 15-20% bitrate reduction vs IPP at same quality |
| Lossless encoding | Successful encode with profile=RExt, bitwise-identical decode |
| UVC detection | Auto-detect connected UVC devices within 2 seconds |
| UVC pipeline generation | Generate working pipeline for MJPEG/YUYV/H.264 sources |
| HDMI + UVC simultaneous | Both pipelines encoding stable for 1 hour |

---

## 11. References

### Internal Documents
- `enc_doc/wave521_encoder_improvement_plan.md` — Hardware feature analysis
- `document/custom-software/streambox_v0.4.md` — v0.4 capture architecture
- `new_modes_plan.md` — v0.4 implementation plan

### External References
- Mainline Linux wave5 driver: `drivers/media/platform/chips-media/wave5/`
  - Source: https://github.com/torvalds/linux/tree/master/drivers/media/platform/chips-media/wave5
- ITU-T H.265 Specification (HEVC lossless, B-frame, RExt profile)
- USB Video Class 1.5 Specification (UVC device capabilities)
- V4L2 API documentation: https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/v4l2.html
