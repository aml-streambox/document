# StreamBox v0.6.2 — VFMCAP Path A Robustness & OTA Firmware Update

## 1. Executive Summary

StreamBox v0.6.2 is a maintenance and feature release that addresses Path A (`vfmcap`)
capture stability issues discovered during v0.6 headless mode production testing, and
introduces **OTA firmware update** support — both via Cockpit WebUI and direct CLI.

**Primary goals:**

1. **VFMCAP Path A robustness fixes** — buffer lifecycle, teardown race, and pool sizing
2. **OTA firmware update** — browser-based upload via Cockpit + CLI method
3. **OTA infrastructure bug fixes** — three critical bugs in the Amlogic SWUpdate chain
4. **No breaking changes** — fully backward compatible with v0.6 pipelines and configs

---

## 2. VFMCAP Path A Robustness Fixes

### 2.1 Background

During v0.6 headless mode validation, several edge-case issues were identified in the
Path A capture path (`streamboxsrc source=vfmcap`):

- **Buffer teardown race**: when the GStreamer pipeline stops, `vfm_cap` may still hold
  references to buffers that `libvfmcap` has already freed, causing use-after-free in
  kernel logs.
- **Pool exhaustion under rapid start/stop**: the DMA-buf pool (enlarged to 12 in v0.5
  for RA_IB) can still be exhausted if the encoder drains slowly and the pipeline is
  restarted quickly.
- **Missing `QBUF` on early `DQBUF` error**: when `VIDIOC_DQBUF` returns `EAGAIN` during
  teardown, the queued buffer is not re-queued, leaking a slot.

### 2.2 Fixes

| Issue | Fix | File |
|-------|-----|------|
| Teardown race | Add `streamboxsrc` drain-before-stop: signal `vfm_cap` to drop all provider refs before releasing buffers | `gst_streambox_src.c` |
| Pool exhaustion | Dynamic pool resize: grow from 12 to 16 when encoder reports `ENC_TIMEOUT` or `QUEUE_FULL` | `gst_streambox_src.c`, `gst_streambox_src.h` |
| Missing QBUF | On `EAGAIN` during DQBUF, always re-queue the buffer if the element is still in `PLAYING` | `gst_streambox_src.c` |
| Standalone refcount | In headless mode, `vfm_cap` standalone must call `vf_notify_provider` to release the last ref before `stop_streaming` | `vfm_cap.c` |

### 2.3 Validation

```bash
# Stress test: rapid start/stop
for i in $(seq 1 20); do
  gst-launch-1.0 -e streamboxsrc source=vfmcap ! \
    "video/x-raw,format=NV12" ! fakesink &
  PID=$!
  sleep 2
  kill $PID
  sleep 1
done

# Check for use-after-free in dmesg
dmesg | grep -i "vfm_cap\|streamboxsrc\|BUG"
```

Expected: no `BUG`, no `use-after-free`, no pool exhaustion messages.

---

## 3. OTA Firmware Update System

### 3.1 Overview

The OTA system uses Amlogic's SWUpdate-based framework with AES-256-CBC encryption
and RSA-SHA256 signing. This release adds a **Cockpit WebUI tab** for browser-based
updates and documents the **CLI method** for direct SSH-based updates.

### 3.2 OTA Architecture

```
Build host                           Target device
───────────                          ─────────────
bitbake amlogic-yocto
  └─> software.swu (AES encrypted,
      RSA signed cpio archive)
                                     Normal Linux (eMMC boot)
      ┌─────────────────────────────┐
      │ Upload .swu to /data/       │
      │ (WebUI or SCP)              │
      └──────────┬──────────────────┘
                 │
      update_swfirmware.sh
                 │
      reboot recovery (writes reboot_reason register)
                 │
                 v
      Recovery initramfs
      ┌─────────────────────────────┐
      │ swupdate.sh mounts /dev/data│
      │ swupdate -i /mnt/software.swu│
      │   -k /etc/swupdate-public.pem│
      │   -K /etc/image-enc-aes.key │
      │ Flashes: rootfs, boot, dtb, │
      │   recovery, vendor, u-boot  │
      │ Sets upgrade_step=2         │
      │ rm software.swu             │
      │ reboot                      │
      └─────────────────────────────┘
                 │
                 v
      Normal Linux (updated)
```

### 3.3 Key Files on Target

| Path | Purpose |
|------|---------|
| `/etc/hwrevision` | Board name + revision (e.g. `mesont7_tvpro_5_15 1.0`) |
| `/etc/sw-versions` | Current software version |
| `/etc/swupdate-public.pem` | RSA public key for signature verification |
| `/etc/image-enc-aes.key` | AES-256 decryption key |
| `/usr/bin/update_swfirmware.sh` | Checks for .swu in /data, calls `reboot recovery` |
| `/usr/bin/swupdate.sh` | Recovery-mode script that runs swupdate |
| `/data/software.swu` | OTA package location |

### 3.4 U-Boot Environment Variables

The Amlogic bootloader uses these env variables for OTA state:

| Variable | Default | Purpose |
|----------|---------|---------|
| `upgrade_step` | `0` | OTA state machine: 0=initial, 1=post-bootloader-write, 2=normal, 3=upgrade-in-progress |
| `write_boot` | `0` | When `1`, u-boot copies `/dev/bootloader_up` to both bootloader slots on next boot |
| `recovery_status` | — | Set to `in_progress` by swupdate during flash |

### 3.5 OTA Bugs Found and Fixed

Three critical bugs were discovered and fixed in the Amlogic OTA chain:

#### Bug 1: sw-description indentation (3 tabs vs 4 tabs)

**Symptom**: `swupdate` fails with `syntax error` when parsing `sw-description`.

**Root cause**: `sw_enc_package_create.sh` and `sw_package_create.sh` used 3-tab
indentation (`\t\t\t`) for `sed` insert commands (sha256, ivt, encrypted fields),
but the `sw-description-emmc` template uses 4-tab indentation for fields inside
image blocks. libconfig is strict about indentation.

**Fix**: Changed all `\t\t\t` to `\t\t\t\t` in both scripts.

**Files**: `aml-comp/prebuilt/hosttools/aml-swupdate/sw_enc_package_create.sh`,
`aml-comp/prebuilt/hosttools/aml-swupdate/sw_package_create.sh`

#### Bug 2: Dots in board name break libconfig parsing

**Symptom**: `swupdate` fails with `syntax error` on the board name line in
`sw-description`.

**Root cause**: `MACHINE_ARCH` (e.g. `mesont7_tvpro_5.15`) contains dots, which are
not allowed in libconfig group/identifier names. The line
`mesont7_tvpro_5.15 = {` caused a parse error.

**Fix**: Added `sed` in `aml-package.inc` to replace dots with underscores in the
board name line after `MACHINE_ARCH` substitution. Also added `awk` in
`swupdate_git.bb` to normalize dots in `/etc/hwrevision` so both sides match
(e.g. both become `mesont7_tvpro_5_15`).

**Files**: `meta-meson/recipes-core/images/aml-package.inc`,
`meta-meson/recipes-core/swupdate/swupdate_git.bb`

#### Bug 3: `swupdate -c` corrupts u-boot environment

**Symptom**: After running `swupdate -c` (check-only mode) followed by
`reboot recovery`, the device enters USB burning mode instead of recovery mode.

**Root cause**: The Amlogic patch to `stream_interface.c` sets `write_boot=1` and
`recovery_status=in_progress` in the u-boot environment on the success path, but
without guarding for `dry_run` mode. In check-only mode (`swupdate -c`), the
`!dry_run && failure_check` condition evaluates to false, falling into the `else`
(success) branch which sets `write_boot=1`. On next boot, u-boot sees this flag
and copies `/dev/bootloader_up` to both bootloader slots, then hard resets. If
no actual flash happened, the boot chain is corrupted.

**Fix**: New patch `0008-swupdate-guard-bootloader-env-in-dry-run.patch` wraps
the `write_boot=1`, slot-switching, and `recovery_status=in_progress` writes
with `if (!software->parms.dry_run)` guards.

**File**: `meta-meson/recipes-core/swupdate/swupdate/0008-swupdate-guard-bootloader-env-in-dry-run.patch`

**IMPORTANT**: Never run `swupdate -c` on a live production system that does not
have this patch applied. It will brick the device.

---

## 4. OTA Update Methods

### 4.1 Method 1: Cockpit WebUI (Recommended)

The "Firmware Update" tab in Streambox Settings provides a browser-based upload,
verification, and flash workflow.

**Steps:**

1. Open `https://<device-ip>:9090` in a browser
2. Log in as root
3. Navigate to **"Firmware Update"** tab
4. The tab displays:
   - Current firmware version (from `/etc/sw-versions`)
   - Device board name (from `/etc/hwrevision`)
5. Drag & drop the `software.swu` file (or click to browse)
6. The backend verifies:
   - File integrity (SHA-256)
   - CPIO contains `sw-description.sig` (valid signed package)
   - Board name in `sw-description` matches device
7. Click **"Start Update"** to trigger the OTA flash
8. Device reboots into recovery, flashes all partitions, and auto-reboots

**Architecture:**

```
Browser (Cockpit UI)
    |
    |-- cockpit.script("cat > /data/software.swu.upload") streaming
    |-- ImportLocalFile D-Bus call for server-side verification
    v
cockpit-streambox-settings backend (D-Bus service)
    |
    |-- updater.py: state machine, SHA-256, board match, cpio sig check
    |-- TriggerUpdate → subprocess: update_swfirmware.sh
    v
Device reboots to recovery → SWUpdate flashes → auto-reboot
```

**Backend D-Bus methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetUpdaterStatus` | `→ s` | Returns JSON with state, progress, version, error |
| `ImportLocalFile` | `ss → b` | Verify and import a locally-uploaded file |
| `TriggerUpdate` | `→ b` | Calls `update_swfirmware.sh` in a thread |
| `SetDryRun` | `b → b` | Enable/disable dry-run mode |
| `CancelUpload` | `→ b` | Reset to idle state |

### 4.2 Method 2: Direct CLI (SSH)

For advanced users or automated deployment, the OTA can be triggered directly
from SSH without the Cockpit UI.

**Steps:**

```bash
# 1. Build the OTA package on the build host
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15
bitbake amlogic-yocto
# Package is at: build/tmp/deploy/images/mesont7-tvpro-5.15/software.swu

# 2. Upload the package to the target device
scp build/tmp/deploy/images/mesont7-tvpro-5.15/software.swu \
    root@<device-ip>:/data/software.swu

# 3. Trigger the OTA update
ssh root@<device-ip> '/usr/bin/update_swfirmware.sh'
```

The device will immediately reboot into recovery mode. The recovery initramfs
will:
1. Mount `/dev/data` to `/mnt`
2. Run `swupdate -i /mnt/software.swu -k /etc/swupdate-public.pem -K /etc/image-enc-aes.key`
3. Flash all partitions (rootfs, boot, dtb, recovery, vendor, u-boot)
4. Set `upgrade_step=2` in u-boot env
5. Remove `software.swu`
6. Reboot to normal mode

**Typical duration**: ~70 seconds for a 433 MB package (varies by eMMC speed and
image size).

**WARNING**: Do NOT run `swupdate -c` on the device before triggering the update.
On unpatched systems, `swupdate -c` writes `write_boot=1` to the u-boot environment,
which causes the device to enter USB burning mode on next reboot instead of recovery.

---

## 5. Build and Deploy

### 5.1 Full Image Build (includes OTA package)

```bash
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15
bitbake amlogic-yocto
```

This produces:
- Boot image with all OTA fixes applied
- `software.swu` OTA package in `build/tmp/deploy/images/mesont7-tvpro-5.15/`

### 5.2 Build OTA Package Only

```bash
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15
bitbake amlogic-yocto -c do_aml_pack
```

### 5.3 Deploy Cockpit Updater to Running Device

```bash
cd cockpit-streambox-settings
scp -r . root@<device-ip>:/tmp/cockpit-streambox-settings
ssh root@<device-ip> 'cd /tmp/cockpit-streambox-settings && bash install.sh'
```

Or rebuild via Yocto:

```bash
bitbake -f -c clean cockpit-streambox-settings
bitbake cockpit-streambox-settings
# Then deploy the RPM/OPKG package to the target
```

---

## 6. Validation

### 6.1 OTA Flash Tests

| Test | Result |
|------|--------|
| CLI: scp .swu + update_swfirmware.sh | PASS — 70s flash cycle, normal boot |
| WebUI: drag-drop upload + trigger | PASS — upload, verify, flash, normal boot |
| Board name mismatch rejection | PASS — wrong board .swu rejected |
| Corrupt file rejection | PASS — bad SHA-256 detected |
| swupdate -c after patch | PASS — no env corruption, safe to use |

### 6.2 VFMCAP Robustness

| Test | Expected Result |
|------|----------------|
| 20x rapid start/stop | No kernel BUG, no use-after-free |
| Long-running capture (1 hour) | No pool exhaustion, stable frame rate |
| Pipeline error recovery | Clean teardown, next start succeeds |
| Headless + passthrough toggle | No crash when switching modes |

---

## 7. Files Summary

### OTA Infrastructure Fixes

| File | Change |
|------|--------|
| `aml-comp/prebuilt/hosttools/aml-swupdate/sw_enc_package_create.sh` | Fix 3→4 tab indentation |
| `aml-comp/prebuilt/hosttools/aml-swupdate/sw_package_create.sh` | Fix 3→4 tab indentation |
| `meta-meson/recipes-core/images/aml-package.inc` | Add dot→underscore sed for board name |
| `meta-meson/recipes-core/swupdate/swupdate_git.bb` | Add hwrevision dot normalization + new patch |
| `meta-meson/recipes-core/swupdate/swupdate/0008-swupdate-guard-bootloader-env-in-dry-run.patch` | **New**: guard bootloader env writes with dry_run |
| `meta-aml-cfg/recipes-cockpit/cockpit-streambox-settings/cockpit-streambox-settings_1.0.bb` | Pin SRCREV to 7758b44 |

### Cockpit Updater Plugin

| File | Change |
|------|--------|
| `cockpit-streambox-settings/backend/updater.py` | **New**: UpdaterManager (state machine, verify, trigger) |
| `cockpit-streambox-settings/backend/api.py` | Added updater D-Bus methods + signal |
| `cockpit-streambox-settings/frontend/updater-settings.js` | **New**: UpdaterSettings module |
| `cockpit-streambox-settings/frontend/index.html` | Added "Firmware Update" tab |
| `cockpit-streambox-settings/frontend/streambox-settings.js` | Added UpdaterSettings.init() |
| `cockpit-streambox-settings/frontend/streambox-settings.css` | Added updater styles |

### VFMCAP Fixes

| File | Change |
|------|--------|
| `aml-comp/multimedia/gst-plugin-vfmcap/gst_streambox_src.c` | Drain-before-stop, dynamic pool resize, EAGAIN QBUF fix |
| `aml-comp/multimedia/gst-plugin-vfmcap/gst_streambox_src.h` | Pool size constants, `max_pool_size` property |
| `aml-comp/kernel/aml-5.15/common_drivers/drivers/media/vfm_cap/vfm_cap.c` | Standalone mode `vf_notify_provider` on stop |

---

## 8. Future Work

- **Update progress in recovery**: if `swupdateui` is added to recovery image, show
  progress percentage in the Updater UI before reboot.
- **Auto-check for updates**: poll a configured URL for new `software.swu` and notify
  user without manual upload.
- **Delta updates**: investigate `rdiff` or `zsync` for smaller OTA packages.
- **Update history**: log update attempts and outcomes to `/data/update_history.json`.

---

## 9. Version History

| Version | Date | Key Changes |
|---------|------|-------------|
| v0.5 | — | Wave521 encoder fixes, extended GOP presets, RA_IB support |
| v0.6 | — | Headless mode, EDID enhancement, cockpit-gst-manager Phase 2 |
| **v0.6.2** | 2026-04-21 | **VFMCAP Path A robustness, OTA firmware update (WebUI + CLI), OTA bug fixes** |
