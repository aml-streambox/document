---
layout: page
title: One-KVM Usage
permalink: /custom-software/one-kvm-usage/
---

# One-KVM Usage

One-KVM is available in StreamBox images as an alternative active application to
the default GStreamer Manager workflow. It provides browser-based KVM access and
USB gadget functions for keyboard/mouse input and mass storage.

> **Status:** One-KVM support is experimental in StreamBox v0.7. Basic video,
> HID, and MSD functions are available, but stability is not yet recommended for
> production use.

## Default Behavior

By default, StreamBox starts the GStreamer Manager application. This means the
device boots into the normal streaming workflow managed by `gst-manager`.

To use One-KVM, switch the active application from GStreamer Manager to One-KVM.

One-KVM listens on port `8080` by default:

```text
http://<device-ip>:8080
```

Cockpit remains available on port `9090`:

```text
http://<device-ip>:9090
```

## Switch Using Cockpit

Recommended method:

1. Open Cockpit in a browser: `http://<device-ip>:9090`
2. Log in as `root`
3. Open the StreamBox Settings page
4. Find the active application setting
5. Change the active application from GStreamer Manager to One-KVM
6. Apply the change and restart the selected application if prompted
7. Open One-KVM at `http://<device-ip>:8080`

Use the same page to switch back from One-KVM to GStreamer Manager.

## Switch Using systemd

Advanced users can switch applications directly with `systemctl`.

Disable and stop GStreamer Manager:

```bash
systemctl disable --now gst-manager.service
```

Enable and start One-KVM:

```bash
systemctl enable --now one-kvm.service
```

Check service status:

```bash
systemctl status one-kvm.service
```

Switch back to GStreamer Manager:

```bash
systemctl disable --now one-kvm.service
systemctl enable --now gst-manager.service
```

## HID and MSD

One-KVM includes support for USB gadget functions:

- HID: keyboard and mouse input from the browser session
- MSD: USB mass-storage device support for virtual media workflows

The host computer should see StreamBox as a USB HID device when the gadget path
is active and connected. MSD availability depends on the configured One-KVM
storage/media settings.

## Notes and Limitations

- One-KVM is experimental and may crash or require manual service restart.
- Do not run GStreamer Manager and One-KVM as active video applications at the same time.
- If video capture fails after switching applications, restart the selected service or reboot the device.
- For production streaming, GStreamer Manager remains the recommended active application.
