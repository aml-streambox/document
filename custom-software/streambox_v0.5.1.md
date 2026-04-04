# StreamBox v0.5.1 — No-Signal UI and HDMI Passthrough Stability

## 1. Executive Summary

StreamBox v0.5.1 focuses on the HDMI TX output experience when HDMI RX is
disconnected, unstable, or switching modes. The core deliverables are:

1. A visible no-signal screen rendered on HDMI TX when HDMI RX has no valid signal
2. Double-buffered flicker-free rendering at 1080p with OSD hardware scaling to 4K
3. HDMI passthrough video remains clean across TX mode changes (no fb0 disruption)
4. Automatic start/stop of the no-signal UI based on HDMI RX signal state

## 2. Architecture

### 2.1 Rendering Path

The no-signal UI runs inside `streambox-tv` as a dedicated worker thread:

```
HDMI RX signal event (TvEventCallback)
    -> UpdateNoSignalUiFromCurrentSource()
    -> StartNoSignalUi() / StopNoSignalUi()
    -> NoSignalUiThread()
       -> opens /dev/fb0
       -> resizes to 1920x1080, requests yres_virtual=2160 for double buffering
       -> renders bouncing "NO SIGNAL" and "STREAMBOX" boxes
       -> page-flips via FBIOPAN_DISPLAY
```

Key design decisions:

- Always render at `1920x1080` regardless of actual TX output mode
- fb0 `smem_len` is ~16 MB, enough for exactly 2 pages of 1080p@32bpp
- OSD hardware scaler upscales the 1080p content to match the TX output resolution
- Framebuffer path is the active renderer; dedicated OSD/GE2D path remains future work

### 2.2 Signal Flow

```
HDMI RX cable / source
    -> hdmirx driver
    -> vdin state machine
    -> aml_tvserver_streambox (TvEventCallback)
    -> streambox-tv no-signal UI thread
    -> /dev/fb0 (OSD plane)
    -> OSD hardware scaler
    -> HDMI TX output
```

When HDMI RX signal becomes valid:

```
TvEventCallback: TVIN_SIG_STATUS_STABLE
    -> UpdateNoSignalUiFromCurrentSource()
    -> StopNoSignalUi()  (joins thread, blanks fb0)
    -> SynchronizeHdmitxToHdmirx()  (sets TX mode, starts passthrough)
    -> StartAudioPassthrough()
```

### 2.3 Double Buffering

Single-buffered rendering caused visible flashing because `memset(black)` cleared the
currently displayed page before new content was drawn. The fix requests
`yres_virtual = height * 2` when `smem_len` allows it:

- `smem_len = 16,588,800` (16 MB)
- One page = `1920 * 4 * 1080 = 8,294,400` (8 MB)
- Two pages = `16,588,800` — exactly fits

The render loop draws on the back page, then flips with `FBIOPAN_DISPLAY`. Runtime
logs confirm `page_count=2` and `map_size=16588800`.

## 3. Fixes Completed

### 3.1 Passthrough Video Corruption After TX Mode Change

**Root cause**: `ResetFb0AfterTxModeChange()` directly manipulated fb0 (blank, resize,
FBIOPUT_VSCREENINFO, mmap+memset) during active video passthrough. This disrupted the
VPP compositor and corrupted the live HDMI RX → TX video path.

**Fix**: Removed the `ResetFb0AfterTxModeChange()` call from
`SynchronizeHdmitxToHdmirx()`.

**Result**: Live passthrough video remains clean after TX mode changes.

### 3.2 Broken No-Signal UI at 4K (Garbled Display)

**Root cause**: fb0 physical memory (`smem_len`) is ~16 MB. A 4K 32bpp framebuffer
requires ~33 MB. The code was mmapping 33 MB against 16 MB of physical memory.

**Fix**: Always render the no-signal UI at `1920x1080` regardless of TX mode. Added
`smem_len` safety guards in both `OpenNoSignalFramebuffer` and
`RemapNoSignalFramebuffer`.

**Result**: The UI renders correctly even when HDMI TX is at `3840x2160p60`.

### 3.3 No-Signal UI Flickering

**Root cause**: Single-buffered rendering (`yres_virtual = yres = 1080`, `page_count=1`).
Each frame cleared the visible buffer to black, causing a visible flash before the new
content appeared. At ~30 fps this produced continuous flashing.

**Fix**: Request `yres_virtual = height * 2` for double buffering when `smem_len`
permits. The render loop alternates pages with `current_page ^ 1`, draws on the back
buffer, and flips with `FBIOPAN_DISPLAY`.

**Result**: Flicker eliminated. Logs confirm `page_count=2`.

## 4. Deferred: OSD Scaling to Full 4K

When HDMI TX outputs at 4K but the DRM CRTC mode remains at 1080p (because
`SynchronizeHdmitxToHdmirx` sets TX mode through amhdmitx sysfs, bypassing DRM),
the OSD plane destination rectangle is only 1920x1080 and the no-signal UI appears in
the top-left quarter.

**Workaround confirmed**: Manually writing the mode works:
```sh
echo '2160p60hz' > /sys/class/drm/card0/crtc0/mode
```

**Code status**: `SyncDrmCrtcMode()` is implemented with retry logic and `O_WRONLY`,
but the programmatic write does not reliably take effect from within `streambox-tv`.
This is deferred for future investigation.

## 5. Files

| File | Role |
|------|------|
| `aml_tvserver_streambox/test/streambox-tv.c` | Main implementation: no-signal UI, fb0 renderer, TX sync |
| `aml_tvserver_streambox/client/CTvClientLog.cpp` | Logging level filter (default INFO, LOGD filtered) |
| `aml_tvserver_streambox/client/include/CTvClientLog.h` | LOGD/LOGE macros |

## 6. Build and Deploy

Build:
```sh
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15 && bitbake aml-tvserver
```

Deploy:
```sh
ssh root@192.168.12.156 "systemctl stop streambox-tv.service"
scp build/tmp/work/armv8a-poky-linux/aml-tvserver/0.4+git999-r0/image/usr/bin/streambox-tv \
    root@192.168.12.156:/usr/bin/streambox-tv
ssh root@192.168.12.156 "systemctl start streambox-tv.service"
```

## 7. Validation

Validated:
- No-signal UI starts on RX invalid / unstable state
- Live passthrough no longer corrupted after TX mode changes
- 4K output no longer produces garbled/broken framebuffer content
- Double buffering active (`page_count=2`, `map_size=16588800`)

Pending:
- Automatic DRM CRTC mode sync for full-screen 4K scaling
- End-to-end testing across all resolution transitions
- Future dedicated OSD/GE2D renderer path

## 8. Future Work

- **OSD scaling auto-sync**: Investigate why `crtc0/mode` programmatic write fails and
  find a reliable mechanism to sync DRM CRTC mode after TX mode changes
- **Dedicated OSD/GE2D path**: Move the no-signal renderer to a dedicated OSD plane
  via GE2D hardware acceleration for richer UI features
- **Richer UI**: Image assets, configurable messages, anti-burn-in color cycling
- **HDMI auto-capture refactor**: The planned `cockpit-gst-manager` auto-capture
  recovery improvements described in the original v0.5.1 plan
