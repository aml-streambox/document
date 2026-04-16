# VFM Capture SDK (libvfmcap) — Developer Guide

## 1. Overview

The **VFM Capture SDK** (`libvfmcap`) is a zero-copy HDMI capture library for the **Amlogic A311D2 (T7)** SoC. It wraps the kernel `vfm_cap` driver (`/dev/video_cap`) with a simple C API and optional **Vulkan-based GPU format conversion** on the Mali-G52 MP4 GPU.

### Key Design Principles

- **Zero-copy by default**: Frames are exported as DMA-buf file descriptors directly from vdin0's CMA buffers. The CPU never touches pixel data.
- **GPU conversion on demand**: Vulkan compute/graphics pipelines convert between raw capture formats (AMLY 10-bit, NV12 8-bit) and downstream-friendly formats (NV12, P010, AFBC) only when explicitly requested.
- **Integrated tone mapping**: HDR10 (PQ) and HLG content can be tone-mapped to BT.709 SDR in real time using a 33³ 3D LUT.
- **Dynamic reconfiguration**: Resolution changes, signal loss, and signal recovery are handled gracefully without crashing the pipeline.

### Supported Output Formats

| Format | Description | Use Case |
|--------|-------------|----------|
| `VFMCAP_FMT_RAW` | Raw passthrough (zero GPU) | Minimal latency, opaque pipelines |
| `VFMCAP_FMT_NV12` | 8-bit semi-planar YUV 4:2:0 | Standard encoder input |
| `VFMCAP_FMT_NV21` | 8-bit semi-planar YVU 4:2:0 | Android/media compatibility |
| `VFMCAP_FMT_P010` | 10-bit semi-planar YUV 4:2:0 | High-bitdepth encoder input |
| `VFMCAP_FMT_NV12_AFBC` | AFBC-compressed 8-bit NV12 | Display/compositor zero-copy |
| `VFMCAP_FMT_A2B10G10R10_AFBC` | AFBC-compressed 10-bit RGB | HDR display paths |
| `VFMCAP_FMT_VYUY_10BIT` | Amlogic private VYUY 10-bit passthrough | Internal debug/testing |

### Verified Performance (4K60)

| Mode | FPS | GPU Proc Time |
|------|-----|---------------|
| P010 Passthrough | 45.2 | 3.33 ms |
| P010 HDR10→SDR | 45.2 | 3.33 ms |
| NV12 AFBC HDR10→SDR | **61.5** | **2.44 ms** |
| A2B10G10R10 AFBC HDR10→SDR | **62.6** | **~2.4 ms** |

---

## 2. Quick Start

### Minimal Example — Raw Passthrough

```c
#include <vfmcap.h>
#include <stdio.h>

int main(void)
{
    vfmcap_ctx_t *ctx = vfmcap_open("/dev/video_cap", NULL);
    if (!ctx) {
        fprintf(stderr, "open failed: %s\n", vfmcap_last_error(NULL));
        return 1;
    }

    if (vfmcap_start(ctx, 6) != VFMCAP_OK) {
        fprintf(stderr, "start failed: %s\n", vfmcap_last_error(ctx));
        vfmcap_close(ctx);
        return 1;
    }

    vfmcap_frame_t frame;
    int rc = vfmcap_acquire_frame(ctx, &frame, 1000);
    if (rc == VFMCAP_OK) {
        printf("Frame %u: %ux%u fd=%d size=%u\n",
               frame.sequence, frame.width, frame.height,
               frame.dmabuf_fd, frame.size);
        // Pass frame.dmabuf_fd to consumer...
        vfmcap_release_frame(ctx, &frame);
    }

    vfmcap_stop(ctx);
    vfmcap_close(ctx);
    return 0;
}
```

### Minimal Example — GPU Conversion with Tone Mapping

```c
vfmcap_config_t cfg = {0};
cfg.output_format = VFMCAP_FMT_NV12;
cfg.target_width  = 1920;   /* 0 = match source */
cfg.target_height = 1080;   /* 0 = match source */
cfg.target_fps    = 30.0f;  /* 0 = match source */
cfg.color_mode    = VFMCAP_COLOR_HDR10_TO_SDR;

vfmcap_ctx_t *ctx = vfmcap_open("/dev/video_cap", &cfg);
```

---

## 3. API Reference

### 3.1 Lifecycle

#### `vfmcap_open()`

```c
vfmcap_ctx_t *vfmcap_open(const char *device, const vfmcap_config_t *config);
```

Opens the capture device and optionally initializes the Vulkan conversion pipeline.

- `device`: Path to V4L2 device. Pass `NULL` for default `/dev/video_cap`.
- `config`: Output format, target resolution, target FPS, and color mode. Pass `NULL` for raw passthrough.
- **Returns**: Context pointer, or `NULL` on error.

#### `vfmcap_start()`

```c
int vfmcap_start(vfmcap_ctx_t *ctx, unsigned int num_buffers);
```

Starts V4L2 streaming with `num_buffers` MMAP buffers (recommended: **6**). Also initializes Vulkan resources if conversion is configured.

#### `vfmcap_stop()`

```c
void vfmcap_stop(vfmcap_ctx_t *ctx);
```

Stops streaming and releases V4L2 buffers. Vulkan resources are kept alive for fast restart.

#### `vfmcap_close()`

```c
void vfmcap_close(vfmcap_ctx_t *ctx);
```

Tears down everything: stops streaming if needed, destroys Vulkan resources, and closes the device.

> **Important**: Do **not** call `vfmcap_stop()` immediately before `vfmcap_close()` in the GPU conversion path. `vfmcap_close()` already performs the correct teardown order (Vulkan first, then V4L2) to avoid kernel use-after-free bugs.

---

### 3.2 Frame Acquisition

#### `vfmcap_acquire_frame()`

```c
int vfmcap_acquire_frame(vfmcap_ctx_t *ctx,
                         vfmcap_frame_t *frame,
                         int timeout_ms);
```

Dequeues one frame. When conversion is configured, the GPU conversion is performed automatically and the returned `frame` points to the **converted output buffer**.

**Return values:**

| Value | Meaning |
|-------|---------|
| `VFMCAP_OK` (0) | Frame acquired successfully |
| `VFMCAP_RECONFIGURED` (1) | Frame acquired after dynamic reconfiguration (resolution/format changed) |
| `VFMCAP_ERR_TIMEOUT` (-3) | Poll/DQBUF timed out |
| `VFMCAP_ERR_NOSIG` (-4) | No signal detected |
| `< 0` | Other error |

**Frame fields:**

| Field | Description |
|-------|-------------|
| `dmabuf_fd` | DMA-buf fd (Y plane for multi-planar formats) |
| `dmabuf_fd2` | UV plane fd for NV12/P010 linear, or `-1` |
| `width` / `height` | Frame dimensions (may differ from source if target resolution is set) |
| `pixelformat` | V4L2 fourcc of the returned data (e.g. `NV12`, `P010`) |
| `size` | Total buffer size in bytes |
| `drm_modifier` | DRM format modifier (`0` for linear, AFBC modifier for compressed) |
| `is_repeated` | `1` if this frame was repeated for framerate conversion |

#### `vfmcap_release_frame()`

```c
void vfmcap_release_frame(vfmcap_ctx_t *ctx, vfmcap_frame_t *frame);
```

Releases the frame back to the capture pool. **Must be called once for every successful `acquire_frame()`.**

---

### 3.3 Signal & Event Handling

#### `vfmcap_poll_event()`

```c
int vfmcap_poll_event(vfmcap_ctx_t *ctx, int timeout_ms);
```

Polls for V4L2 events without consuming a frame.

**Returns:**
- `VFMCAP_EVENT_SOURCE_CHANGE` — resolution or format changed
- `VFMCAP_EVENT_NOSIG` — signal lost
- `VFMCAP_EVENT_TIMEOUT` — no event within timeout
- `VFMCAP_EVENT_ERROR` — polling error

#### `vfmcap_get_signal_info()`

```c
int vfmcap_get_signal_info(vfmcap_ctx_t *ctx, vfmcap_signal_info_t *info);
```

Fills `info` with current HDMI RX parameters (width, height, FPS, HDR status, etc.).

---

### 3.4 Utility

#### `vfmcap_output_size()`

```c
uint32_t vfmcap_output_size(uint32_t width, uint32_t height,
                            vfmcap_output_fmt_t fmt);
```

Returns the minimum DMA-buf size required for `fmt` at the given resolution.

#### `vfmcap_last_error()`

```c
const char *vfmcap_last_error(vfmcap_ctx_t *ctx);
```

Returns the last human-readable error string. Pointer is valid until the next API call.

---

## 4. Configuration Deep Dive

### 4.1 Output Formats

#### `VFMCAP_FMT_RAW` (default)
No GPU involvement. `dmabuf_fd` points directly to the vdin0 CMA buffer. The pixel format depends on the HDMI source (typically AMLY 10-bit or NV12 8-bit).

#### `VFMCAP_FMT_NV12` / `VFMCAP_FMT_NV21`
8-bit 4:2:0 semi-planar output. For 10-bit AMLY input, the GPU decodes each pixel pair, averages chroma vertically for 4:2:0 subsampling, and writes Y and UV planes.

#### `VFMCAP_FMT_P010`
10-bit 4:2:0 semi-planar output. Data is stored as **left-justified 10-bit in 16-bit words** (bits 15:6 contain the 10-bit value, bits 5:0 are zero). This matches the standard P010 layout expected by most video codecs.

#### `VFMCAP_FMT_NV12_AFBC`
AFBC (Arm Frame Buffer Compression) compressed NV12. A single multi-planar DMA-buf fd is returned. The DRM modifier is set to:
```
ARM_AFBC(BLOCK_SIZE_16x16 | SPARSE | SPLIT)
```

#### `VFMCAP_FMT_A2B10G10R10_AFBC`
AFBC compressed 10-bit RGB output. The GPU performs full AMLY decode → YCbCr→RGB → 3D LUT tone mapping → A2B10G10R10 in a single TBDR pass. Returns a single-plane DMA-buf fd with the AFBC modifier.

### 4.2 Target Resolution

Setting `target_width` and/or `target_height` enables **GPU scaling**:

```c
cfg.target_width  = 1920;
cfg.target_height = 1080;
```

Scaling is performed by the Vulkan TBDR pipeline using bilinear filtering via the GPU's texture sampler. 4K→1080p scaling has been verified at ~9.2 ms total processing time.

> **Note**: Both dimensions must be even for YUV 4:2:0 output. Odd dimensions will be rejected during `vfmcap_start()`.

### 4.3 Target Framerate

Setting `target_fps` enables **framerate conversion** by frame dropping:

```c
cfg.target_fps = 30.0f;  /* Drop every other frame from 60 fps source */
```

When the source FPS is higher than the target, `vfmcap_acquire_frame()` may return frames with `is_repeated = 1` to maintain the target rate. No temporal interpolation is performed — this is simple frame repeat/drop.

### 4.4 Color Modes

| Mode | Description |
|------|-------------|
| `VFMCAP_COLOR_PASSTHROUGH` | No color conversion. YCbCr values pass through directly. |
| `VFMCAP_COLOR_HDR10_TO_SDR` | BT.2020 PQ → BT.709 SDR using Reinhard tone map + 3D LUT |
| `VFMCAP_COLOR_HLG_TO_SDR` | BT.2020 HLG → BT.709 SDR using Reinhard tone map + 3D LUT |

The 3D LUT is generated at initialization as a **33×33×33 RGBA16 texture** and uploaded to GPU device-local memory. Each LUT texel stores the pre-computed SDR YCbCr result for one input YCbCr coordinate. The fragment shaders sample this LUT with trilinear filtering.

---

## 5. Architecture & Data Flow

### 5.1 Raw Passthrough Path

```
HDMI RX → vdin0 → vfm_cap (/dev/video_cap) → DQBUF → DMA-buf fd
                                                        |
                                                        v
                                                   caller (zero-copy)
```

### 5.2 GPU Conversion Path (TBDR)

```
HDMI RX → vdin0 → vfm_cap → DMA-buf fd → Vulkan import as SSBO
                                                |
                                                v
                    +------------------ TBDR Fragment Shader ------------------+
                    |  1. Read AMLY pair from SSBO (inline bswap64 correction) |
                    |  2. Decode Y0/Y1/Cb/Cr 10-bit values                     |
                    |  3. If HDR mode: normalize YCbCr → sample 3D LUT         |
                    |  4. Scale if target resolution differs from source         |
                    |  5. Write to output color attachment(s)                  |
                    +----------------------------------------------------------+
                                                |
                                                v
                                    Output DMA-buf pool (NV12/P010/AFBC)
                                                |
                                                v
                                            caller
```

The TBDR path uses **dynamic rendering** (no VkFramebuffer/VkRenderPass) and writes directly to DMA-buf backed images imported into Vulkan.

### 5.3 AFBC Output Path

For AFBC outputs, the output pool images are created with:
- `VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT`
- Modifier: `DRM_FORMAT_MOD_ARM_AFBC(16x16 | SPARSE | SPLIT)`
- Single `dma_buf_sync` ioctl before/after GPU work

The Mali-G52 on T7C supports AFBC for:
- `R8G8B8A8_UNORM` (COLOR_ATTACHMENT + SAMPLED)
- `G8_B8R8_2PLANE_420_UNORM` (NV12, COLOR_ATTACHMENT + SAMPLED)
- `A2B10G10R10_UNORM_PACK32` (COLOR_ATTACHMENT + SAMPLED)

> **Note**: `STORAGE_IMAGE` usage on AFBC formats is **not supported** on this GPU, so compute-shader direct-write paths were abandoned in favor of TBDR.

---

## 6. Dynamic Reconfiguration

When the HDMI source changes resolution, refresh rate, or color format, the `vfm_cap` driver emits a `V4L2_EVENT_SOURCE_CHANGE`. `libvfmcap` handles this internally:

1. **Drain pending GPU work** and wait for idle.
2. **Stop V4L2 streaming** (`STREAMOFF`).
3. **Re-query format** from the driver.
4. **Re-initialize Vulkan output pools** at the new resolution (or target resolution).
5. **Restart streaming** (`STREAMON`).
6. Return `VFMCAP_RECONFIGURED` from the next `vfmcap_acquire_frame()` call.

The caller should detect `VFMCAP_RECONFIGURED` and adjust downstream caps/encoder settings accordingly.

### Example Handling

```c
int rc = vfmcap_acquire_frame(ctx, &frame, 1000);
if (rc == VFMCAP_RECONFIGURED) {
    vfmcap_signal_info_t info;
    vfmcap_get_signal_info(ctx, &info);
    printf("Signal changed to %ux%u @ %u.%03u fps\n",
           info.width, info.height,
           info.fps / 1000, info.fps % 1000);
    /* Update encoder / GStreamer caps here */
}
```

---

## 7. Error Handling Best Practices

### Common Errors

| Error | Typical Cause | Resolution |
|-------|---------------|------------|
| `VFMCAP_ERR_NOSIG` | HDMI cable unplugged or source asleep | Wait and retry, or notify user |
| `VFMCAP_ERR_TIMEOUT` | Driver not producing frames | Check signal status, reboot if VPU stuck |
| `VFMCAP_ERR_VULKAN` | GPU init failed (missing `/dev/mali0`, old loader) | Verify Vulkan driver installation |
| `VFMCAP_RECONFIGURED` | Source changed resolution | Reconfigure downstream pipeline |

### Recommended Loop Structure

```c
while (running) {
    vfmcap_frame_t frame;
    int rc = vfmcap_acquire_frame(ctx, &frame, 1000);

    if (rc == VFMCAP_RECONFIGURED) {
        /* Handle reconfiguration */
        continue;
    }
    if (rc == VFMCAP_ERR_NOSIG) {
        g_usleep(100000);  /* 100 ms */
        continue;
    }
    if (rc != VFMCAP_OK) {
        fprintf(stderr, "acquire failed: %s\n", vfmcap_last_error(ctx));
        break;
    }

    /* Process frame */
    consumer_push_frame(frame.dmabuf_fd, frame.dmabuf_fd2,
                        frame.width, frame.height);

    vfmcap_release_frame(ctx, &frame);
}
```

---

## 8. Integration with GStreamer (streamboxsrc)

The `gst-plugin-vfmcap` plugin wraps `libvfmcap` as a GStreamer `GstPushSrc` element. Key properties:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `source` | enum | `vfmcap` | Capture path (`vfmcap` or `vdin1`) |
| `output-format` | enum | `nv12` | Output pixel format (`nv12`, `p010`) |
| `target-width` | uint | 0 | Target output width (0 = source) |
| `target-height` | uint | 0 | Target output height (0 = source) |
| `target-fps` | float | 0.0 | Target output framerate (0 = source) |
| `color-mode` | uint | 0 | `0`=passthrough, `1`=HDR10→SDR, `2`=HLG→SDR |
| `capture-buffers` | uint | 6 | V4L2 buffer count |

### Example Pipelines

**Basic NV12 capture:**
```bash
gst-launch-1.0 streamboxsrc source=vfmcap output-format=nv12 \
    ! "video/x-raw,format=NV12" ! fakesink
```

**4K→1080p HDR10 tone mapping:**
```bash
gst-launch-1.0 streamboxsrc source=vfmcap output-format=nv12 \
    color-mode=1 target-width=1920 target-height=1080 \
    ! "video/x-raw,format=NV12" ! amlvenc ! fakesink
```

**10-bit P010 passthrough:**
```bash
gst-launch-1.0 streamboxsrc source=vfmcap output-format=p010 \
    ! "video/x-raw,format=P010_10LE" ! fakesink
```

---

## 9. Build & Deployment

### Cross-Compilation (Yocto)

```bash
cd /home/anshi/yocto/aml-comp/multimedia/libvfmcap
make clean
bash /home/anshi/yocto/build/tmp/work/armv8a-poky-linux/libvfmcap/1.0-r0/temp/run.do_compile
```

### Deploy to Target

```bash
scp libvfmcap.so root@192.168.12.176:/usr/lib/
scp vfmcap-demo root@192.168.12.176:/usr/bin/
```

### Clear GStreamer Registry (if plugin changed)

```bash
ssh root@192.168.12.176 'rm -f ~/.cache/gstreamer-1.0/registry.*'
```

---

## 10. Legacy API (Deprecated)

The following functions exist for backward compatibility but should **not be used in new code**:

- `vfmcap_convert_p010()`
- `vfmcap_convert_nv12()`
- `vfmcap_convert_submit()`
- `vfmcap_convert_wait()`

These require the caller to allocate output DMA-bufs manually and manage a separate conversion step. The integrated `vfmcap_config_t` approach handles buffer pooling, synchronization, and release automatically.

---

## 11. Troubleshooting

### "No signal or cannot acquire test frame"
- Verify HDMI cable is connected and source is powered on.
- Check `/sys/class/hdmirx/hdmirx0/info` for signal parameters.
- If the VPU/VDIN state is corrupted, **reboot the target**.

### "Vulkan 10-bit AFBC NV12 render failed: pool exhausted"
- This indicates `vfmcap_release_frame()` is not being called, or is being called on the wrong pool format.
- Ensure every `acquire_frame()` has a matching `release_frame()`.

### Dark/washed out P010 or HDR output
- Verify `color_mode` is actually being applied. Check kernel logs for `[vfmcap-vk] 3D LUT created: 33x33x33 RGBA16 (HDR10->SDR)`.
- If the log says `1x1x1 RGBA16 (passthrough dummy)`, the `color_mode` was not propagated to Vulkan context (this was a bug fixed in `v0.6.2_dev`).

### Performance below 4K60
- AFBC output is faster than linear P010 on this platform. If the downstream path supports AFBC, use `VFMCAP_FMT_NV12_AFBC`.
- Ensure the Mali GPU frequency governor is set to performance mode.

---

## 12. Version History

| Version | Changes |
|---------|---------|
| v0.6.2 | A2B10G10R10 AFBC output, streamboxsrc new properties (`target-width/height/fps`, `color-mode`), fixed `color_mode` propagation to Vulkan, fixed UV plane size in GStreamer plugin |
| v0.6.1 | AFBC NV12 output, dynamic reconfiguration, 3D LUT tone mapping |
| v0.6.0 | Initial architecture redesign: TBDR pipelines, integrated conversion, removed legacy compute-only path |

---

*Copyright (C) 2026 StreamBox — SPDX-License-Identifier: MIT*
