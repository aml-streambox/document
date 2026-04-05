---
name: streambox-usage
description: Control and operate a Streambox device via SSH
license: MIT
compatibility: opencode
metadata:
  audience: ai-agent
  device: amlogic-a311d2
  platform: yocto-linux
---

## What I do

- Authenticate to Streambox device via SSH (public key preferred)
- Perform first-time device setup (password change, WiFi, streaming config)
- Control GStreamer pipelines via `gst-manager-cli`
- Manage system services and network configuration
- Debug hardware issues using sysfs interfaces and logs

## When to use me

Use this when you need to operate a Streambox device for streaming, configuration, or debugging.

Always check `/home/root/.streambox_complete_setup.md` existence before setup. Prompt user to change default passwords if setup is incomplete.

---

## Default Preferences

When configuring streaming, use these defaults unless user specifies otherwise:

- **Codec:** H.265
- **Capture Source:** `vfmcap` (Path A - raw capture, preserves HDR)
- **Bitrate:** 20Mbps (20000 kbps)
- **GOP Preset:** IBBBP (gop-pattern=1)
- **RC Mode:** CBR

---

## Authentication

**Default credentials:** `root` / `Streamb0x`

Public key auth is preferred. If not set up, ask user to run:
```bash
ssh-copy-id root@<device-ip>
```

## First-Time Setup

Check setup status:
```bash
ssh root@<device-ip> "ls /home/root/.streambox_complete_setup.md"
```

If missing, perform setup:

### 1. Change Passwords (Required)

```bash
# Root password
ssh root@<device-ip> "passwd root"

# WiFi AP password (if hostapd enabled)
ssh root@<device-ip> "sed -i 's/wpa_passphrase=.*/wpa_passphrase=NEW_PASSWORD/' /etc/hostapd/hostapd.conf && systemctl restart hostapd"
```

### 2. Configure Network

Connect to WiFi:
```bash
ssh root@<device-ip> "wpa_passphrase '<ssid>' '<password>' > /etc/wpa_supplicant.conf && wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant.conf"
```

Enable/disable WiFi AP:
```bash
# Enable: systemctl start hostapd && systemctl enable hostapd
# Disable: systemctl stop hostapd && systemctl disable hostapd
```

### 3. Configure HDMI Streaming

```bash
# CLI wrapper
alias gst-manager-cli='python3 /usr/lib/gst-manager/cli_client.py'

# Get current config
ssh root@<device-ip> "python3 /usr/lib/gst-manager/cli_client.py auto get"

# Set config with defaults (H.265, vfmcap, 20Mbps, IBBBP, CBR)
ssh root@<device-ip> "cat > /tmp/auto-config.json << 'EOF'
{
  \"enabled\": true,
  \"source\": \"vfmcap\",
  \"output_codec\": \"h265\",
  \"bitrate_kbps\": 20000,
  \"gop_pattern\": 1,
  \"rc_mode\": \"cbr\",
  \"audio_device\": \"hw:0,6\",
  \"port\": 8888
}
EOF
python3 /usr/lib/gst-manager/cli_client.py auto set --config-file /tmp/auto-config.json"
```

### 4. Complete Setup and Reboot

```bash
ssh root@<device-ip> "echo 'Setup complete: $(date)' > /home/root/.streambox_complete_setup.md && reboot"
```

---

## Pipeline Management

Use `python3 /usr/lib/gst-manager/cli_client.py` - only run `gst-launch-1.0` directly if CLI is broken or user explicitly requests.

```bash
# List instances
python3 /usr/lib/gst-manager/cli_client.py instances list

# UVC devices
python3 /usr/lib/gst-manager/cli_client.py uvc list
python3 /usr/lib/gst-manager/cli_client.py uvc create <name> /dev/video0 --format h264 --encoder h265 --bitrate-kbps 20000 --output-type srt --port 8889 --start

# Auto HDMI config
python3 /usr/lib/gst-manager/cli_client.py auto get
python3 /usr/lib/gst-manager/cli_client.py auto set --config-file <path>

# Board context
python3 /usr/lib/gst-manager/cli_client.py board-context

# Direct D-Bus call (escape hatch)
python3 /usr/lib/gst-manager/cli_client.py call GetBoardContext
python3 /usr/lib/gst-manager/cli_client.py call StartInstance <instance-id>
```

---

## Debug Commands

```bash
# Display info
cat /sys/class/display/vinfo

# HDMI TX info
cat /sys/class/amhdmitx/amhdmitx0/*

# VRR status
cat /sys/class/aml_vrr/vrr2/status

# HDMI RX debug (to dmesg)
echo state > /sys/class/hdmirx/hdmirx0/debug
dmesg | tail -50

# Video input info
echo state > /sys/class/vdin/vdin0/attr

# HDMI RX signal
cat /sys/class/hdmirx/hdmirx0/signal
cat /sys/class/hdmirx/hdmirx0/info

# Service logs
journalctl -u gst-manager -n 100
journalctl -u tvservice -n 100
```

---

## Capture Paths

| Path | Source | Use Case |
|------|--------|----------|
| Path A | `vfmcap` | Raw HDR capture, preserves original colorimetry (preferred) |
| Path B | `vdin1` | Color-processed, HDR-to-SDR, general streaming |

## Audio Devices

| Device | Path | Description |
|--------|------|-------------|
| HDMI Audio | `hw:0,6` | HDMI RX audio loopback |
| Line-in | `hw:0,0` | 3.5mm line-in (TVPro only) |

## GOP Patterns (Wave521 encoder)

`gop-pattern`: 0=IPP, 1=IBBBP, 2=IBPBP, 3=IBBB, 4=ALL_I, 5=IPPPP, 6=IBBBB, 7=RA_IB, 8=IPP_SINGLE

## RC Modes

`rc-mode`: `cbr` (Constant Bitrate), `vbr` (Variable Bitrate), `fixqp` (Fixed QP)

---

## Notes

- Always run `gst-launch-1.0 -e` for proper encoder shutdown
- Prefer Path A (`vfmcap`) for best quality with HDR preservation
- Default: H.265, 20Mbps, IBBBP GOP, CBR mode
- Reboot after initial setup
- Check `journalctl -u gst-manager` for pipeline errors
- HDMI passthrough corruption after TX mode change: `systemctl restart tvservice`
- reboot can often solve some unexpected hardware issue.