# StreamBox HDMI TX No-Signal Screen - v0.5.1 Status

## Goal

When HDMI RX is disconnected or unstable, HDMI TX should show a visible
StreamBox no-signal screen instead of a plain black passthrough output.

## What v0.5.1 Implements

The current implementation lives in `aml_tvserver_streambox/test/streambox-tv.c`
and uses the framebuffer-backed OSD path as the active renderer.

- Trigger on HDMI RX unstable / no-signal conditions
- Stop the UI automatically when HDMI RX becomes valid again
- Stop audio passthrough while the no-signal UI is active
- Render a black background with two bouncing boxes:
  - `NO SIGNAL`
  - `STREAMBOX`
- Keep the current framebuffer path available as the fallback renderer

## Current Rendering Path

### Active path

- `fb0` framebuffer renderer inside `streambox-tv`
- Lightweight built-in bitmap text renderer
- Dedicated UI thread for animation
- Double-buffered page flipping using `FBIOPAN_DISPLAY`

### Future path

- A dedicated OSD / GE2D path is still the long-term direction
- The current framebuffer path remains useful as a fallback and bring-up path

## Key Fixes Completed

### 1. Passthrough corruption after TX mode change

Root cause:

- Resetting `fb0` directly during HDMI TX mode changes disturbed the VPP
  compositor state and corrupted live HDMI RX -> TX passthrough.

Fix:

- Removed the `ResetFb0AfterTxModeChange()` call from the HDMI TX sync path.

Result:

- Live passthrough remains clean after TX mode changes.

### 2. Broken no-signal UI at 4K

Root cause:

- `fb0` physical memory (`smem_len`) is about 16 MB.
- A full 4K 32bpp framebuffer needs about 33 MB.
- Mapping a full 4K framebuffer against 16 MB caused invalid rendering.

Fix:

- Always render the no-signal UI at `1920x1080`.
- Add `smem_len` guards around framebuffer mapping and remapping.

Result:

- The UI no longer breaks or corrupts when HDMI TX is `3840x2160p60`.

### 3. Flickering / flashing no-signal UI

Root cause:

- Single-buffered rendering cleared the currently visible page every frame.
- The render loop briefly showed a black frame before the next draw completed.

Fix:

- Request `yres_virtual = height * 2` when framebuffer memory allows it.
- Map the full virtual framebuffer.
- Draw on the hidden page and flip with `FBIOPAN_DISPLAY`.

Result:

- Double buffering is active at `1920x1080@32bpp` because the 16 MB fb memory
  exactly fits two pages.
- Runtime logs confirm:
  - `yres_virtual=2160`
  - `map_size=16588800`
  - `page_count=2`

## Scaling Investigation Status

There is still a separate scaling issue when HDMI TX is 4K.

### Root cause

- HDMI TX mode is changed through amhdmitx sysfs.
- DRM CRTC mode does not automatically follow that change.
- fbdev pan/display code uses the DRM CRTC mode to size the OSD plane
  destination rectangle.
- If DRM still thinks the mode is `1080p60hz` while HDMI TX is actually
  `2160p60hz`, the no-signal UI appears only in the top-left quarter.

### Confirmed workaround

Writing the DRM CRTC mode manually works:

```sh
echo '2160p60hz' > /sys/class/drm/card0/crtc0/mode
```

That forces the DRM atomic modeset and makes the OSD plane fill the full 4K
output.

### Current code status

- `streambox-tv` includes a `SyncDrmCrtcMode()` helper that tries to update
  `/sys/class/drm/card0/crtc0/mode`
- It uses `O_WRONLY`, retry logic, and both short-name and long-name mode forms
- In testing, the programmatic write still does not reliably take effect
- Manual root-shell writes do work

### Decision

- Leave automatic DRM CRTC mode sync deferred for now
- Keep the stable framebuffer renderer and double-buffering fixes in place

## Files Involved

- Code: `aml_tvserver_streambox/test/streambox-tv.c`
- Logging support: `aml_tvserver_streambox/client/CTvClientLog.cpp`
- Logging macros: `aml_tvserver_streambox/client/include/CTvClientLog.h`

## Build and Deploy Flow

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

## Validation Status

Validated so far:

- no-signal UI starts on RX invalid / unstable state
- live passthrough is no longer corrupted by the old fb reset path
- 4K output no longer produces the broken / garbled framebuffer result
- double buffering is active and deployed on target

Still to validate / revisit:

- visual confirmation that flicker is fully gone in all no-signal scenarios
- automatic DRM CRTC mode sync from inside `streambox-tv`
- full scaling behavior across all HDMI TX mode transitions
- future dedicated OSD / GE2D renderer path
