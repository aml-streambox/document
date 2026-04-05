# StreamBox v0.5.1 — HDMI Passthrough Stability & No-Signal UI

## 1. Executive Summary

StreamBox v0.5.1 focuses on four areas:

1. **Auto capture error recovery** — `cockpit-gst-manager` now robustly recovers auto
   HDMI capture pipelines after signal loss, resolution changes, and encoder failures
2. **No-signal screen** — HDMI TX shows a visible bouncing-box UI when HDMI RX has no
   valid signal, instead of plain black
3. **HDMI frame rate detection fix** — Fixed incorrect frame rate reporting for VRR content
   where kernel reported 120Hz base framerate instead of actual 60Hz input
4. **UVC device serial tracking** — UVC device pipelines are now linked to USB serial IDs
   for persistent identification across reconnections and hot-plug events

## 2. Auto Capture Error Recovery

### 2.1 Problem

The previous auto capture flow had several stuck states:

- Resolution changes without signal drop caused `streamboxsrc` to exit with EOS, but no
  restart event was generated — the instance stayed `stopped`
- Short HDMI instability windows could be missed by the polling logic
- AUTO instances did not retry after non-signal pipeline failures
- Path A (`vfmcap`) was incorrectly gated on HDMI TX readiness
- Duplicate decision paths in `_check_initial_state()`, `_periodic_polling()`, and
  `HdmiMonitor` created edge-case races

### 2.2 Architecture

The core design decision: keep `streamboxsrc` unchanged, let the pipeline process exit
cleanly on EOS, and move all recovery responsibility into `cockpit-gst-manager`.

```
streamboxsrc detects signal change
    -> posts hdmi-signal-change message
    -> returns EOS
    -> gst-launch exits
    -> instances.py parses stderr for hdmi-signal-change details
    -> auto_instance.py schedules debounce / restart
    -> event manager provides current RX or TX readiness
    -> new pipeline is regenerated and started
```

### 2.3 Capture Path Rules

| Capture Source | Meaning | Start Condition | Runtime Parameter Source |
|----------------|---------|-----------------|--------------------------|
| `vfmcap` | Path A direct HDMI RX capture | HDMI RX stable only | HDMI RX status |
| `vdin1` | Path B VPP / screen-recording path | HDMI RX stable + HDMI TX ready | HDMI TX status |
| `v4l2_legacy` | Legacy HDMI capture | HDMI RX stable | HDMI RX status |

### 2.4 Implementation

#### Instance Exit Handling (`instances.py`)

- `ProcessExitInfo` dataclass carries exit code, signal-change details, and whether the
  stop was intentional
- `_parse_signal_change()` extracts `hdmi-signal-change` details from `gst-launch-1.0`
  stderr (reason, width, height, fps, color-space, hdr-eotf, etc.)
- `_handle_error()` classifies errors as transient or fatal:
  - **Transient** (network disconnect, timeout, buffer underrun): auto-retry up to
    `max_retries` with configurable delay
  - **Fatal** (device not found, invalid pipeline, encoder failure): stop and notify
  - **Signal-change exit** on AUTO instances: treat as clean stop, let `auto_instance.py`
    handle recovery
- 4-stage shutdown for hung pipelines: `SIGUSR1` (encoder EOS injection) → `SIGINT`
  (GStreamer EOS) → `SIGTERM` → `SIGKILL`
- Startup watchdog: instances stuck in `STARTING` for too long are moved to `ERROR`,
  allowing auto-recovery to retry

#### Auto Instance Manager (`auto_instance.py`)

- `on_instance_exit()` handles all auto-instance process exits:
  - **Signal-lost**: mark `WAITING_SIGNAL`, cancel restarts
  - **Signal-change** (resolution/timing shift): schedule debounced restart with new
    parameters
  - **Unexpected exit**: schedule retry with exponential backoff
- `_ensure_auto_instance_running()` reconciles pipeline state:
  - Rebuilds pipeline if current parameters don't match the live signal
  - Recreates instance if it's in `stopped` / `error` / `waiting_signal`
  - Path B gets a 2-second stabilization delay for TX readiness
- `_schedule_restart()` with generation-based cancellation prevents stale restarts from
  firing after newer events arrive
- Configurable via `AutoInstanceConfig`:
  - `signal_debounce_seconds` (default 2.0)
  - `max_restart_retries` (default 5)
  - `restart_backoff_base` (default 1.0s)
  - `restart_backoff_max` (default 30.0s)

#### Event Coordination (`events.py`)

- Path A instances receive `on_hdmi_signal_ready()` — RX stability only
- Path B instances receive `on_passthrough_ready()` — requires TX readiness
- Signal-lost callbacks: `on_hdmi_signal_lost()` and `on_passthrough_lost()`

#### Codec and Encoder Controls

- Auto instance now supports `output_codec` selection (H.265 or H.264)
- `gop_pattern` (Wave521 GOP preset) and `rc_mode` (VBR/CBR/FixQP) exposed in the
  auto-configurator UI
- Lossless mode validation: auto-disables for non-H.265 codecs

### 2.5 Error Classification

| Error Type | Examples | Recovery |
|------------|----------|----------|
| Transient | Network disconnect, timeout, buffer underrun | Auto-retry up to 3 times, 5s delay |
| Signal-lost | HDMI cable unplugged | Stop, set `WAITING_SIGNAL`, wait for event |
| Signal-change | Resolution/framerate/HDR change | Debounce, rebuild pipeline, auto-restart |
| Fatal | Device not found, encoder crash, invalid pipeline | Stop, notify user |

### 2.6 Files

| File | Role |
|------|------|
| `backend/instances.py` | Process lifecycle, exit parsing, error classification, 4-stage shutdown |
| `backend/auto_instance.py` | Auto instance manager, recovery scheduling, pipeline builder |
| `backend/events.py` | HDMI RX/TX event dispatch, Path A/B readiness callbacks |
| `backend/api.py` | REST API for auto instance configuration and codec controls |
| `frontend/auto-configurator.js` | Auto capture UI: source, codec, GOP, encoder settings |
| `doc/implementation_plan/error_recovery.md` | Error recovery design specification |

## 3. No-Signal UI

### 3.1 Rendering Path

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

### 3.2 Fixes Completed

#### Passthrough Video Corruption After TX Mode Change

**Root cause**: `ResetFb0AfterTxModeChange()` directly manipulated fb0 during active
video passthrough, disrupting the VPP compositor.

**Fix**: Removed the fb0 reset call from `SynchronizeHdmitxToHdmirx()`.

#### Broken No-Signal UI at 4K

**Root cause**: fb0 `smem_len` (~16 MB) too small for 4K 32bpp (~33 MB) mmap.

**Fix**: Always render at `1920x1080` with `smem_len` safety guards.

#### No-Signal UI Flickering

**Root cause**: Single-buffered rendering cleared the visible page every frame.

**Fix**: Request `yres_virtual = height * 2` for double buffering. The render loop
draws on the back page and flips with `FBIOPAN_DISPLAY`.

### 3.3 Deferred: OSD Scaling to Full 4K

When HDMI TX outputs at 4K but the DRM CRTC mode stays at 1080p (because TX mode is
set through amhdmitx sysfs, bypassing DRM), the OSD plane destination rectangle is
only `1920x1080` and the no-signal UI appears in the top-left quarter.

**Workaround**: `echo '2160p60hz' > /sys/class/drm/card0/crtc0/mode` works manually.
Programmatic write from `streambox-tv` does not reliably take effect — deferred.

### 3.4 Files

| File | Role |
|------|------|
| `aml_tvserver_streambox/test/streambox-tv.c` | No-signal UI, fb0 renderer, TX sync |
| `aml_tvserver_streambox/client/CTvClientLog.cpp` | Logging level filter |
| `aml_tvserver_streambox/client/include/CTvClientLog.h` | LOGD/LOGE macros |

## 4. HDMI Frame Rate Detection Fix

### 4.1 Problem Statement

When an HDMI source sends VRR (Variable Refresh Rate) content at 60Hz within a 120Hz
timing envelope, the kernel driver reported the VRR base framerate (120Hz from VIC 63)
instead of the actual input frame rate (60Hz). This caused the HDMI TX output mode to
be incorrectly set to 120Hz, resulting in:

- Incorrect display mode synchronization
- Failed auto HDMI capture due to framerate mismatch
- `streamboxsrc` unable to start properly

### 4.2 Root Cause

The kernel's `vdin_get_base_fr()` function in `vdin_ctl.c` determines the frame rate
for VRR content by:

1. Checking if `vtem_data.vrr_en` is set (VRR mode active)
2. If so, using `hdmirx_get_base_fps(hw_vic)` which maps VIC 63 to 120Hz
3. The actual frame rate from HDMI RX (`Frame Rate: 5993` in centi-Hz) was ignored

The tvserver's `control.8` (TV_CONTROL_GET_FRAME_RATE) command returned this incorrect
120Hz value through `CTv::GetFrontendInfo()` → `mpTvin->Tvin_GetFrontendInfo()`.

### 4.3 Solution

Modified `CTv::GetFrontendInfo()` in `aml_tvserver_streambox/libtv/CTv.cpp` to read the
actual frame rate from the HDMI RX sysfs interface and override the kernel-reported fps:

```cpp
int CTv::GetFrontendInfo(tvin_frontend_info_t *frontendInfo)
{
    int ret = -1;
    if (frontendInfo == NULL) {
        LOGD("%s: param is NULL.\n", __FUNCTION__);
    } else {
        ret = mpTvin->Tvin_GetFrontendInfo(frontendInfo);

        char buf[SYS_STR_LEN+1] = {0};
        int actual_fps = 0;
        if (tvReadSysfs("/sys/class/hdmirx/hdmirx0/info", buf) > 0) {
            char *fps_line = strstr(buf, "Frame Rate:");
            if (fps_line) {
                if (sscanf(fps_line, "Frame Rate: %d", &actual_fps) == 1 && actual_fps > 0) {
                    int rounded_fps = (actual_fps + 50) / 100;
                    if (rounded_fps > 0 && rounded_fps != frontendInfo->fps) {
                        LOGD("%s: overriding fps from kernel %d to actual %d (raw %d)\n",
                             __FUNCTION__, frontendInfo->fps, rounded_fps, actual_fps);
                        frontendInfo->fps = rounded_fps;
                    }
                }
            }
        }
    }
    // ...
}
```

The HDMI RX sysfs `Frame Rate` field contains the actual measured frame rate in
centi-Hz (e.g., 5993 for 59.93Hz), which is then rounded to the nearest integer Hz.

### 4.4 Verification

Before fix:
```
HDMI RX info:     Frame Rate: 5993 (actual 60Hz)
control.8 return: 120 (VRR base framerate)
Display output:   1920x1080p120hz
```

After fix:
```
HDMI RX info:     Frame Rate: 5993 (actual 60Hz)
control.8 return: 60 (corrected from sysfs)
Display output:   1080p60hz
```

### 4.5 Files

| File | Role |
|------|------|
| `aml_tvserver_streambox/libtv/CTv.cpp` | GetFrontendInfo() fix to read actual fps from sysfs |
| `meta-meson/recipes-multimedia/aml-tvserver/aml-tvserver_git.bb` | SRCREV update |

### 4.6 Related Kernel Code

The kernel VRR base framerate logic is in:
- `aml-comp/kernel/aml-5.15/common_drivers/drivers/media/vin/tvin/vdin/vdin_ctl.c:7507-7575`
- `vdin_get_base_fr()` function
- `hdmirx_get_base_fps()` in `hdmirx/hdmi_rx_drv.c:982-1027`

---

## 5. UVC Device Serial Tracking

### 5.1 Problem Statement

Previously, UVC device pipelines were identified by their `/dev/videoX` path. When multiple
UVC devices are connected or devices are reconnected, the video device path can change,
causing saved configurations to be applied to the wrong device.

### 5.2 Solution

UVC devices are now tracked by their USB serial number, which provides persistent
identification across reconnections and hot-plug events:

- **Serial ID extraction** — USB serial number is read from `/sys/class/video4linux/videoX/device/../serial`
- **Persistent device matching** — Pipelines are linked to serial IDs instead of device paths
- **Auto-start on connect** — UVC instances configured for auto-start will automatically start
  when their associated device is connected
- **Hot-plug monitoring** — Device connect/disconnect events trigger automatic instance start/stop

### 5.3 Architecture

```
UVC device connected → UVCDiscovery discovers device + serial
                        ↓
UVCInstanceManager.on_devices_changed()
                        ↓
                    Match serial to saved config
                        ↓
        ┌───────────────┴───────────────┐
        ↓                               ↓
Device matched                      Device not found
    → Update device_path             → Instance in WAITING_SIGNAL
    → Auto-start if configured       → Wait for device connect
```

### 5.4 Key Components

| Component | Role |
|-----------|------|
| `uvc_utils.py` | USB serial ID extraction from sysfs, device discovery |
| `uvc_instance.py` | UVC instance manager with serial-based tracking, hot-plug handling |
| `events.py` | UVC device monitor loop (2s polling), triggers `on_devices_changed` |
| `main.py` | Creates UVCInstanceManager, calls `start_all_autostart()` on boot |
| `frontend/uvc-manager.js` | UI shows serial number, auto-start checkbox |

### 5.5 Instance Configuration

UVC instances now store:
- `device_serial`: USB serial number for persistent identification
- `device_path`: Current device path (resolved on connect)
- `device_name`: Human-readable device name
- `autostart`: Whether to start when device is connected
- `trigger_event`: Set to `"uvc_device_ready"` when autostart enabled

### 5.6 API Changes

| Method | Change |
|--------|--------|
| `CreateUVCInstance` | New `autostart` parameter, returns `device_serial` |
| `UpdateUVCInstance` | New `autostart` parameter |
| `GetUVCDevices` | Returns devices with `serial` field |
| `ListInstances` | UVC instances include `device_serial` in `uvc_config` |

### 5.7 Files

| File | Role |
|------|------|
| `backend/uvc_utils.py` | Added `_get_usb_serial()`, `find_device_by_serial()`, serial field in UVCDevice |
| `backend/uvc_instance.py` | New file — UVCInstanceManager with serial tracking, auto-start, hot-plug |
| `backend/events.py` | Added `_uvc_monitor_loop()` for device hot-plug detection |
| `backend/main.py` | Added UVCInstanceManager initialization and autostart on boot |
| `backend/api.py` | Updated CreateUVCInstance/UpdateUVCInstance with autostart parameter |
| `frontend/uvc-manager.js` | UI shows serial, adds autostart checkbox |
| `frontend/gst-manager.js` | Instance details show device serial for UVC instances |

## 6. Build and Deploy

**streambox-tv** (no-signal UI):
```sh
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15 && bitbake aml-tvserver
```

**cockpit-gst-manager** (auto capture recovery):
```sh
# Deployed via Yocto package or direct install
```

## 7. Validation

### Auto Capture Recovery

Validated:
- HDMI resolution change triggers debounced pipeline rebuild and restart
- Signal-lost moves instance to `WAITING_SIGNAL`, restarts on signal return
- Path A starts without HDMI TX dependency
- Path B waits for TX readiness before starting
- 4-stage shutdown cleanly stops hung encoder pipelines
- Startup watchdog catches stuck instances

### No-Signal UI

Validated:
- No-signal UI starts on RX invalid / unstable state
- Live passthrough no longer corrupted after TX mode changes
- 4K output no longer produces garbled framebuffer content
- Double buffering active (`page_count=2`, `map_size=16588800`)

### HDMI Frame Rate Detection

Validated:
- VRR content at 60Hz within 120Hz timing correctly reports 60Hz
- HDMI TX output mode correctly matches actual input frame rate
- Auto HDMI capture works with VRR sources

### UVC Serial Tracking

Validated:
- USB serial number correctly extracted from sysfs
- Device path resolved from serial on boot and reconnection
- Auto-start triggers when configured device is connected
- Instance stops when device is disconnected
- Save/load of UVC config preserves serial ID

### Pending

- Automatic DRM CRTC mode sync for full-screen 4K no-signal scaling
- End-to-end testing across all resolution transitions (720p, 1080p24/30/60/120, 4K)
- Repeated rapid mode switching stress testing
- Future dedicated OSD/GE2D renderer path for richer UI

## 8. Future Work

- **OSD scaling auto-sync**: Investigate why `crtc0/mode` programmatic write fails
- **Dedicated OSD/GE2D path**: Move the no-signal renderer to a dedicated OSD plane
  via GE2D hardware acceleration for richer UI features
- **Richer no-signal UI**: Image assets, configurable messages, anti-burn-in cycling
- **Long-duration soak**: Extended mixed-workload stability testing
