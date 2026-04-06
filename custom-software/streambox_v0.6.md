# StreamBox v0.6 — Headless HDMI RX Capture Mode

## 1. Executive Summary

StreamBox v0.6 adds **headless mode**: when no HDMI TX display is connected, the device
auto-detects this condition at startup and operates as a pure HDMI RX frame capturer.

Key points:

1. **Auto-detection** — HPD state is checked at startup; no config file setting needed
2. **Minimal VFM path** — `vdin0 vfm_cap` only, no display pipeline
3. **All display code bypassed** — no fb0/DRM, no HDMI TX sync, no audio passthrough,
   no VRR/ALLM/game mode, no no-signal UI
4. **Decoder still runs** — vdin0 captures frames for V4L2 consumers via vfm_cap
5. **vfm_cap standalone mode** — auto-detects no downstream receiver, manages frames
   purely for V4L2 without display path refcounting

## 2. Architecture

### 2.1 Detection Flow

```
streambox-tv main()
    → GetHdmiTxHpdState()
    → HPD == 0?
        YES → g_headless_mode = 1
              → skip HPD wait loop
              → SetHeadlessMode(pTvClientWrapper, 1)
              → skip DisplayInit()
              → skip no-signal UI
              → skip uevent monitor thread
        NO  → normal passthrough mode (existing behavior)
```

### 2.2 IPC Chain

SetHeadlessMode propagates through 4 layers:

```
streambox-tv.c        → SetHeadlessMode(wrapper, 1)
TvClientWrapper.cpp   → sends "control.13.1"
TvClient.cpp          → SendMethodCall("control.13.1")
TvService.cpp         → dispatches TV_CONTROL_SET_HEADLESS_MODE
CTv.cpp               → SetHeadlessMode(true)
                         → removes old VFM path
                         → sets TV_PATH_VDIN_VFMCAP_ONLY
                         → writes "add tvpath vdin0 vfm_cap" to VFM map
```

### 2.3 VFM Path Comparison

| Mode | VFM tvpath |
|------|-----------|
| Normal | `vdin0 vfm_cap deinterlace amvideo` |
| Headless | `vdin0 vfm_cap` |

### 2.4 vfm_cap Standalone Mode

When vfm_cap is at the end of the VFM chain (no downstream receiver), it auto-detects
standalone mode via `vf_get_receiver()` returning NULL:

| Aspect | Tee Mode (normal) | Standalone Mode (headless) |
|--------|-------------------|---------------------------|
| Downstream | deinterlace → amvideo | none |
| Frame refcount base | 1 (display path) | 0 (V4L2 only) |
| ready_list | populated for downstream | not used |
| vf_notify_receiver | called per frame | skipped |
| No V4L2 consumers | frames wait for downstream | frames immediately recycled |
| Provider registration | vf_reg_provider on START | skipped |
| Event forwarding | RESET/FR_HINT passed through | skipped |

## 3. Hard Dependencies on HDMI TX — All Addressed

### 3.1 HPD Blocking (streambox-tv.c)

`main()` previously blocked forever polling for `GetHdmiTxHpdState() == 1`.

**Fix**: Headless mode bypasses the HPD wait loop entirely and enters the main
processing path immediately.

### 3.2 VFM Path Always Includes Display Pipeline (CTvin.cpp)

`Tvin_AddVideoPath()` always appended `deinterlace amvideo` suffix.

**Fix**: New `TV_PATH_VDIN_VFMCAP_ONLY` enum creates path `"add tvpath vdin0 vfm_cap"`
with early return before the suffix is appended.

### 3.3 SynchronizeHdmitxToHdmirx (streambox-tv.c)

Called on every `TVIN_SIG_STATUS_STABLE` event, writes to HDMI TX sysfs nodes.

**Fix**: `TvEventCallback` returns early in headless mode with a log message, skipping
all TX synchronization, audio passthrough, and UI code.

### 3.4 EDID Passthrough (CTv.cpp / CHDMIRxManager.cpp)

Passthrough mode reads TX EDID — fails without display. However, the passthrough
config key `edid.passthrough.tx.to.rx` defaults to `0` (disabled), so the file-based
EDID loading path runs instead. The file-based path:

- Loads `port1_20.bin` which already advertises maximal capabilities (4K60, HDR10, HLG,
  10-bit, YCbCr 4:2:0, all major audio formats)
- `PatchEdidFor120Hz()` adds high-refresh DTDs and VIC 63
- `PatchEdidMonitorName()` gracefully handles no TX EDID (returns 0, keeps original name)

No separate maximal EDID file is needed — the existing EDID loading works in headless.

## 4. Code Changes

### 4.1 App Layer (streambox-tv.c)

| Change | Details |
|--------|---------|
| `g_headless_mode` global | `static int`, set by HPD auto-detection |
| `main()` | Auto-detect headless, bypass HPD wait, call SetHeadlessMode, skip DisplayInit / no-signal UI / uevent thread |
| `TvEventCallback()` | Early return in headless (log only, no TX sync/audio/UI) |
| `signal_handler()` | Skip fb0 unblank in headless |
| `StartTvProcessing()` | Skip game mode/ALLM/VRR setup in headless |
| Cleanup section | Guard uevent thread join and StopNoSignalUi with headless check |

### 4.2 IPC Layer

| File | Change |
|------|--------|
| `tvcmd.h` | `TV_CONTROL_SET_HEADLESS_MODE = 13` |
| `TvClient.h/.cpp` | `SetHeadlessMode(bool)` → sends `"control.13.<0|1>"` |
| `TvClientWrapper.h/.cpp` | C wrapper `SetHeadlessMode(wrapper, headless)` |
| `TvService.cpp` | Dispatch `TV_CONTROL_SET_HEADLESS_MODE` → `mpTv->SetHeadlessMode()` |

### 4.3 Library Layer (CTv / CTvin)

| File | Change |
|------|--------|
| `CTv.h` | `SetHeadlessMode()`, `GetHeadlessMode()`, `mHeadlessMode` member |
| `CTv.cpp` | SetHeadlessMode impl (VFM path switch with rollback), headless guards in: StartTv, StopTv, onSigToStable, onSigToUnstable, onSigToUnSupport, onSigToNoSig, isVideoFrameAvailable, muteVideoOnHDMISource |
| `CTvin.h` | `TV_PATH_VDIN_VFMCAP_ONLY` enum value |
| `CTvin.cpp` | Handler for capture-only VFM path (early return, no suffix) |

### 4.4 Kernel Module (vfm_cap)

| File | Change |
|------|--------|
| `vfm_cap.h` | `bool standalone` field in `struct vfm_cap_dev` |
| `vfm_cap.c` | Standalone auto-detection and frame management: |

Standalone mode changes in `vfm_cap.c`:

- **PROVIDER_START**: Check `vf_get_receiver()` — if NULL, set `standalone=true`, skip
  provider registration
- **PROVIDER_UNREG**: Skip `vf_unreg_provider()` in standalone
- **QUREY_STATE**: In standalone, skip downstream query; report based on pool state only
- **VFRAME_READY**: In standalone:
  - Skip late-start provider registration
  - If V4L2 consumers streaming: refcount=1 (held V4L2 ref only), one-frame delay delivery
  - If no V4L2 consumers: immediately recycle frame to vdin0 (no pool accumulation)
  - Skip `vf_notify_receiver()` (no downstream)
- **PROVIDER_RESET / FR_HINT / FR_END_HINT**: Skip downstream forwarding in standalone
- **Module exit**: Skip provider unreg in standalone
- **Sysfs status**: Shows `standalone` field
- **File header comment**: Documents both tee and standalone modes

### 4.5 What is Skipped in Headless Mode

| Component | Normal Mode | Headless Mode |
|-----------|------------|---------------|
| HPD wait loop | Blocks until TX connected | Bypassed |
| DisplayInit() (fb0) | Initializes framebuffer | Skipped |
| HDMI TX sync | Sets TX output mode per-frame | Skipped |
| Audio passthrough | Unmutes/mutes audio on signal changes | Skipped |
| VRR/ALLM/Game mode | Configures RX for VRR content | Skipped |
| No-signal UI | Renders bouncing box on fb0 | Skipped |
| Uevent monitor | Watches HDMI hotplug events | Skipped |
| Video layer control | Shows/hides video via sysfs | Skipped |
| Snow effect | Shows snow pattern on no-signal | Skipped |
| VIDEO_FREERUN_MODE | Sets freerun for VRR passthrough | Skipped |
| Video axis setup | Sets display crop/position | Skipped |
| Audio patch | Routes HDMI audio to output | Skipped |
| fb0 unblank (signal handler) | Unblanks fb on SIGTERM | Skipped |

### 4.6 What Still Runs in Headless Mode

| Component | Purpose |
|-----------|---------|
| EDID loading | File-based EDID (port1_20.bin + patches) |
| vdin0 decoder | Captures HDMI RX frames |
| Signal state machine | Detects stable/unstable/nosig |
| vfm_cap V4L2 device | Exposes /dev/video_cap for frame access |
| Source change events | V4L2_EVENT_SOURCE_CHANGE for consumer notification |
| Format detection | Tracks resolution, pixel format, HDR type |
| SM polling | Monitors signal state transitions |

## 5. Build and Deploy

### tvserver

```sh
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15 && bitbake aml-tvserver
```

### vfm_cap kernel module

```sh
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15 && bitbake vfm-cap
```

### Deploy

```sh
# tvserver
scp <build-output>/aml-tvserver*.ipk root@<device-ip>:/tmp/
ssh root@<device-ip> 'opkg install --force-reinstall /tmp/aml-tvserver*.ipk && reboot'

# vfm_cap
scp <build-output>/vfm-cap*.ipk root@<device-ip>:/tmp/
ssh root@<device-ip> 'opkg install --force-reinstall /tmp/vfm-cap*.ipk && reboot'
```

## 6. Validation

### Test Environment

- Target: StreamBox device at `<device-ip>`
- HDMI TX: disconnected (headless)
- HDMI RX: connected to source device sending 1080p60 or 4K60

### Expected Behavior

1. `streambox-tv` starts without blocking on HPD
2. Logs show `HEADLESS MODE: no HDMI TX display detected`
3. VFM path is `vdin0 vfm_cap` (check via `cat /sys/class/vfm/map`)
4. `/dev/video_cap` exists and accepts V4L2 operations
5. `vfmcap-demo` captures frames with non-zero pixel data
6. `cat /sys/class/video4linux/videoN/vfm_cap/status` shows `standalone: 1`
7. Pool does not exhaust when no V4L2 consumer is active

### Test Commands

```sh
# Verify headless mode in logs
journalctl -u streambox-tv | grep -i headless

# Check VFM path
cat /sys/class/vfm/map

# Verify vfm_cap standalone mode
cat /sys/class/video4linux/video*/status 2>/dev/null

# Run capture demo
vfmcap-demo --frames 10 --output /tmp/test_frame.raw

# Check pool state (should not exhaust)
cat /sys/class/video4linux/video*/pool_state 2>/dev/null
```

## 7. Files Modified

| File | Layer | Changes |
|------|-------|---------|
| `aml_tvserver_streambox/test/streambox-tv.c` | App | Headless auto-detect, HPD bypass, callback guards |
| `aml_tvserver_streambox/client/include/tvcmd.h` | IPC | TV_CONTROL_SET_HEADLESS_MODE = 13 |
| `aml_tvserver_streambox/client/include/TvClient.h` | IPC | SetHeadlessMode declaration |
| `aml_tvserver_streambox/client/TvClient.cpp` | IPC | SetHeadlessMode implementation |
| `aml_tvserver_streambox/client/include/TvClientWrapper.h` | IPC | C wrapper declaration |
| `aml_tvserver_streambox/client/TvClientWrapper.cpp` | IPC | C wrapper implementation |
| `aml_tvserver_streambox/service/TvService.cpp` | IPC | Dispatch handler |
| `aml_tvserver_streambox/libtv/CTv.h` | Library | SetHeadlessMode API, mHeadlessMode member |
| `aml_tvserver_streambox/libtv/CTv.cpp` | Library | SetHeadlessMode impl, headless guards |
| `aml_tvserver_streambox/libtv/CTvin.h` | Library | TV_PATH_VDIN_VFMCAP_ONLY enum |
| `aml_tvserver_streambox/libtv/CTvin.cpp` | Library | Capture-only VFM path handler |
| `aml-comp/kernel/aml-5.15/common_drivers/drivers/media/vfm_cap/vfm_cap.h` | Kernel | standalone field |
| `aml-comp/kernel/aml-5.15/common_drivers/drivers/media/vfm_cap/vfm_cap.c` | Kernel | Standalone mode implementation |

### Test Results (2026-04-06)

- **Device**: StreamBox at `<device-ip>`, kernel 5.15.137-amlogic, T7 (A311D2)
- **Source**: 1080p60 RGB 8-bit HDMI input on port 1
- **HDMI TX**: disconnected (headless auto-detected)

| Check | Result |
|-------|--------|
| Headless auto-detection | PASS — `HEADLESS MODE: no HDMI TX display detected` in logs |
| VFM path | PASS — `tvpath { vdin0(1) vfm_cap }` |
| Standalone detection | PASS — `STANDALONE mode: no downstream receiver` in dmesg |
| vf_notify_receiver errors | PASS — 0 occurrences (previously continuous errors) |
| vfm_cap provider list | PASS — vfm_cap in receiver list only, not in provider list |
| Signal detection | PASS — SM state 0→2→5, format 1920x1080 10-bit |
| vfmcap-demo capture | PASS — **100 frames, 127.94 fps, 0 dropped** |
| Frame latency | PASS — first frame in 0.0 ms |
| No display code executed | PASS — no fb0/DRM errors, no TX sync attempts |

## 8. Future Work

- **End-to-end headless testing** — full test with `vfmcap-demo` and GStreamer pipeline
- **Runtime headless toggle** — support hot-plug of HDMI TX to transition between modes
- **Headless audio capture** — route HDMI RX audio to ALSA capture device
- **Multi-source headless** — support multiple HDMI RX ports in headless mode
- **Config override** — allow forcing headless mode via config for testing
