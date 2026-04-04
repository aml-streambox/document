# StreamBox v0.5.1 — HDMI Passthrough Stability & No-Signal UI

## 1. Executive Summary

StreamBox v0.5.1 focuses on two areas of the HDMI passthrough experience:

1. **Auto capture error recovery** — `cockpit-gst-manager` now robustly recovers auto
   HDMI capture pipelines after signal loss, resolution changes, and encoder failures
2. **No-signal screen** — HDMI TX shows a visible bouncing-box UI when HDMI RX has no
   valid signal, instead of plain black

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

## 4. Build and Deploy

**streambox-tv** (no-signal UI):
```sh
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15 && bitbake aml-tvserver
```

**cockpit-gst-manager** (auto capture recovery):
```sh
# Deployed via Yocto package or direct install
```

## 5. Validation

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

### Pending

- Automatic DRM CRTC mode sync for full-screen 4K no-signal scaling
- End-to-end testing across all resolution transitions (720p, 1080p24/30/60/120, 4K)
- Repeated rapid mode switching stress testing
- Future dedicated OSD/GE2D renderer path for richer UI

## 6. Future Work

- **OSD scaling auto-sync**: Investigate why `crtc0/mode` programmatic write fails
- **Dedicated OSD/GE2D path**: Move the no-signal renderer to a dedicated OSD plane
  via GE2D hardware acceleration for richer UI features
- **Richer no-signal UI**: Image assets, configurable messages, anti-burn-in cycling
- **Long-duration soak**: Extended mixed-workload stability testing
