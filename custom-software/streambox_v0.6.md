# StreamBox v0.6 — Headless HDMI RX Capture Mode

## 1. Executive Summary

StreamBox v0.6 adds **headless mode**: when no HDMI TX display is connected, the device
auto-detects this condition at startup and operates as a pure HDMI RX frame capturer.
It also includes **enhanced EDID** advertising non-standard high-refresh-rate and ultrawide
display modes, **cockpit-gst-manager integration** for headless operation, and **HDMI RX
audio capture support**.

**Primary goals:**

1. Headless auto-detection — no config file, no manual toggle
2. Minimal capture-only VFM path — `vdin0 vfm_cap`, no display pipeline
3. vfm_cap standalone mode — frame management for V4L2 without downstream refcounting
4. EDID enhancement — non-standard modes (1440p120/60/144, 1080p144/240, 3440x1440@60)
5. PS5 1440p fix — base block DTD addition so PS5 detects and confirms 1440p output
6. cockpit-gst-manager Phase 2 — headless detection, HDMI RX Direct audio, TX HPD monitoring
7. HDMI RX audio routing — audio patch created in headless mode, DTS unified across platforms

---

## 2. Headless HDMI RX Capture Mode

### 2.1 Auto-Detection

At startup, `streambox-tv` checks HDMI TX HPD state. If HPD is 0 (no display connected),
headless mode is activated automatically — no config file or toggle needed.

```
streambox-tv main()
    → GetHdmiTxHpdState()
    → HPD == 0?
        YES → g_headless_mode = 1
              → skip HPD wait loop
              → SetHeadlessMode(pTvClientWrapper, 1)
              → skip DisplayInit() / no-signal UI / uevent thread
        NO  → normal passthrough mode (existing behavior)
```

### 2.2 IPC Chain

`SetHeadlessMode` propagates through 4 layers:

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

### 2.3 VFM Path

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

### 2.5 What is Skipped in Headless Mode

| Component | Normal | Headless |
|-----------|--------|----------|
| HPD wait loop | Blocks until TX | Bypassed |
| DisplayInit() (fb0) | Initializes framebuffer | Skipped |
| HDMI TX sync | Sets TX output mode per-frame | Skipped |
| Audio passthrough | Unmutes/mutes on signal changes | Skipped |
| VRR/ALLM/Game mode | Configures RX for VRR | Skipped |
| No-signal UI | Renders bouncing box on fb0 | Skipped |
| Uevent monitor | Watches HDMI hotplug | Skipped |
| Video layer control | Shows/hides via sysfs | Skipped |
| Audio patch | Routes HDMI audio to output | **Still created** |

### 2.6 Files

| File | Layer | Change |
|------|-------|--------|
| `aml_tvserver_streambox/test/streambox-tv.c` | App | Headless auto-detect, HPD bypass, callback guards |
| `aml_tvserver_streambox/client/include/tvcmd.h` | IPC | `TV_CONTROL_SET_HEADLESS_MODE = 13` |
| `aml_tvserver_streambox/client/TvClient.cpp` | IPC | SetHeadlessMode implementation |
| `aml_tvserver_streambox/client/TvClientWrapper.cpp` | IPC | C wrapper |
| `aml_tvserver_streambox/service/TvService.cpp` | IPC | Dispatch handler |
| `aml_tvserver_streambox/libtv/CTv.h` | Library | SetHeadlessMode API, mHeadlessMode member |
| `aml_tvserver_streambox/libtv/CTv.cpp` | Library | SetHeadlessMode impl, headless guards |
| `aml_tvserver_streambox/libtv/CTvin.h` | Library | `TV_PATH_VDIN_VFMCAP_ONLY` enum |
| `aml_tvserver_streambox/libtv/CTvin.cpp` | Library | Capture-only VFM path handler |
| `aml-comp/kernel/aml-5.15/common_drivers/drivers/media/vfm_cap/vfm_cap.h` | Kernel | `standalone` field |
| `aml-comp/kernel/aml-5.15/common_drivers/drivers/media/vfm_cap/vfm_cap.c` | Kernel | Standalone mode implementation |

---

## 3. EDID Enhancement

### 3.1 Problem

The stock EDID only advertises standard HDMI VICs. Non-standard modes like 1440p120,
1440p144, 1080p144, 1080p240, and ultrawide 3440x1440@60 are not advertised, so source
devices cannot select them. Additionally, the PS5 requires 1440p to be advertised in
the **base block DTDs** — it does not look at additional CEA extensions.

### 3.2 Solution

A **second CEA extension block** (bytes 256-383 of the 2.0 EDID) contains 6 DTDs for
non-standard modes. The kernel's `edid_with_port` sysfs interface supports per-port
EDID delivery up to 512 bytes, using `EDID_TYPE_256_PLUS_512` to send 384-byte 2.0
EDIDs (base block + 2 CEA extension blocks).

### 3.3 Modes Added

| Mode | Resolution | Refresh | Pixel Clock | Location |
|------|-----------|---------|-------------|----------|
| 1440p120 | 2560x1440 | 119.90 Hz | 483.0 MHz | Ext block 2 DTD |
| 1440p60 | 2560x1440 | 59.95 Hz | 241.5 MHz | **Base block slot 3** + ext block 2 |
| 1080p144 | 1920x1080 | 144.00 Hz | 356.4 MHz | Ext block 2 DTD |
| 1080p240 | 1920x1080 | 240.00 Hz | 594.0 MHz | Ext block 2 DTD |
| 3440x1440@60 | 3440x1440 | 59.65 Hz | 319.75 MHz | Ext block 2 DTD |
| 1440p144 | 2560x1440 | 143.99 Hz | 571.04 MHz | Ext block 2 DTD |
| 1080p120 | 1920x1080 | 120.00 Hz | — | VIC 63 injected into ext block 1 video data block |

### 3.4 EDID Structure

```
Byte 0-127:    Base block (extension count = 2)
                 Slot 1: 3840x2160@60Hz (preferred)
                 Slot 2: 1920x1080@60Hz
                 Slot 3: 2560x1440@60Hz (was Monitor Name)
                 Slot 4: Range Limits (48-165Hz, 600MHz max pclk)
Byte 128-255:  CEA extension 1 (original, with VIC 63 added)
Byte 256-383:  CEA extension 2 (6 DTDs for non-standard modes)
```

Total: 384 bytes delivered to kernel as 2.0 EDID via `edid_with_port`.

### 3.5 PS5 1440p Fix

The PS5 requires 1440p support in the **base block DTDs** (slot 3 at offset 90). The
Monitor Name descriptor was replaced with a 2560x1440@60Hz CVT-RBv2 DTD.

Result: PS5 now detects 1440p in display settings and confirms output after user
presses OK on the confirmation dialog.

### 3.6 Files

| File | Change |
|------|--------|
| `CHDMIRxManager.h` | `PATCHED_EDID_MAX_SIZE=384`, `EDID_TYPE_*` constants |
| `CHDMIRxManager.cpp` | `PatchEdidFor120Hz()` rewritten: injects VIC 63, adds base block 1440p DTD, builds second CEA extension with 6 DTDs |
| `CHDMIRxManager.cpp` | `UpdataEdidDataWithPort()` rewritten: per-port delivery via `edid_with_port` ioctl |
| `CTv.cpp` | `LoadEdidData()` rewritten: per-port delivery with `EDID_TYPE_256_PLUS_512` |
| `TvCommon.h` | `PATCHED_EDID_MAX_SIZE=384` |
| `aml-tvserver_git.bb` | SRCREV updated |

---

## 4. cockpit-gst-manager Phase 2

### 4.1 Headless Detection

At startup, `events.py` reads `/sys/class/amhdmitx/amhdmitx0/hpd_state`. If HPD is 0,
headless mode is activated automatically via `tvservice.set_headless_mode(True)`.

A polling loop monitors HPD for hot-plug transitions in both directions.

### 4.2 HDMI RX Direct Audio

Added `HDMI_RX_DIRECT` audio source enum mapped to `hw:0,2` (TDM-C, the HDMI RX input).
Device selection is now dict-based for cleaner mapping.

### 4.3 Files

| File | Change |
|------|--------|
| `backend/tvservice.py` | Added `set_headless_mode()` method with ctypes binding |
| `backend/events.py` | TX HPD monitoring: startup headless detection, polling loop, transitions |
| `backend/auto_instance.py` | `HDMI_RX_DIRECT` audio source enum (hw:0,2), dict-based device selection |
| `backend/discovery.py` | Added `hw:0,2` and `hw:0,6` to AUDIO_DEVICES with labels |
| `backend/ai/agent.py` | Updated audio documentation for all three devices |
| `frontend/index.html` | Added "HDMI RX Direct (hw:0,2, headless)" dropdown option |

---

## 5. Audio Routing

### 5.1 Changes

| Component | Change |
|-----------|--------|
| CTv.cpp | `create_audio_patch()` / `release_audio_patch()` moved outside `!mHeadlessMode` guard — audio patch is created in headless mode |
| DTS (KVim4) | TDM-A disabled, dai-links renumbered so `hw:0,2` = TDM-C (HDMI RX) |
| audioserver | Permanently disabled (`systemctl mask audioserver`) |
| local.conf | `DISTRO_FEATURES:append = " disable-audioserver-init"` |

### 5.2 Audio Device Map

| Device | Function | Platform |
|--------|----------|----------|
| `hw:0,0` | TDM-B (HDMI TX output) | Both |
| `hw:0,2` | TDM-C (HDMI RX input) | Both (after DTS fix) |
| `hw:0,6` | FRHDMIRX (raw HDMI RX) | Both |

### 5.3 Required Mixer Controls

```sh
amixer -c 0 sset 'TDMIN_C source select' 'hdmirx'
amixer -c 0 sset 'Audio In Source' 'FRHDMIRX'
```

---

## 6. Build and Deploy

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
scp <build-output>/usr/lib/libtv.so <build-output>/usr/lib/libtvclient.so \
    <build-output>/usr/bin/tvservice <build-output>/usr/bin/streambox-tv \
    root@<device-ip>:/usr/lib/ /usr/bin/

# vfm_cap (if kernel module changed)
scp <build-output>/vfm-cap.ko root@<device-ip>:/lib/modules/$(uname -r)/

# Restart
ssh root@<device-ip> "systemctl restart tvserver && sleep 3 && systemctl restart streambox-tv"
```

---

## 7. Validation

### Test Environment

- Target: StreamBox device at `<device-ip>`, kernel 5.15.137-amlogic, T7 (A311D2)
- HDMI TX: disconnected (headless)
- HDMI RX: connected to PS5 sending 3840x2160p60 HDR

### Headless Mode

| Check | Result |
|-------|--------|
| Headless auto-detection | PASS — `HEADLESS MODE: no HDMI TX display detected` in logs |
| VFM path | PASS — `tvpath { vdin0(1) vfm_cap }` |
| Standalone detection | PASS — `STANDALONE mode: no downstream receiver` in dmesg |
| Signal detection | PASS — SM state transitions, format detection |
| vfmcap-demo capture | PASS — frames captured, no pool exhaustion |
| No display code executed | PASS — no fb0/DRM errors, no TX sync attempts |

### EDID Enhancement

| Check | Result |
|-------|--------|
| 641-byte EDID delivery | PASS — kernel accepts, all checksums valid |
| Base block 1440p DTD | PASS — 2560x1440@60Hz at slot 3, checksum valid |
| Extension block 2 | PASS — 6 DTDs with correct image sizes (800x450mm), checksums valid |
| PS5 1440p detection | PASS — PS5 detects 1440p in display settings |
| PS5 1440p confirmation | PASS — user presses OK, 1440p streaming confirmed |

### PS5 1440p Test Sequence

1. Connect PS5 to HDMI RX (port 1)
2. PS5 reads EDID — sees 2560x1440@60Hz in base block DTD
3. In PS5 settings, select 1440p resolution
4. PS5 sends 1440p test signal — device locks and captures successfully
5. PS5 shows confirmation dialog — user presses OK
6. PS5 outputs 1440p — SRT streaming works

---

## 8. Future Work

- **End-to-end audio capture test** — verify audio capture with mixer controls set
- **Direct mixer setup** — set mixer controls via `amixer` in streambox-tv startup
- **Runtime headless toggle** — support hot-plug of HDMI TX to transition between modes
- **Multi-source headless** — support multiple HDMI RX ports in headless mode
- **Config override** — allow forcing headless mode via config for testing
