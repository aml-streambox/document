---
name: streambox-dev
description: Develop and build Streambox firmware and components
license: MIT
compatibility: opencode
metadata:
  audience: ai-agent
  device: amlogic-a311d2
  platform: yocto-linux
  kernel: "5.15"
---

## What I do

- Set up Yocto build environment for Streambox firmware development
- Build kernel, drivers, userspace components, and full system images
- Deploy and test changes on target devices
- Debug hardware issues via sysfs interfaces
- Manage development branches and documentation

## When to use me

Use this when developing firmware, kernel drivers, userspace applications, or GStreamer plugins for Streambox.

**Critical rules:**
- Always create a dev branch before any code modification
- Document all modifications in relevant docs
- Build, deploy, and test on target before claiming completion
- Update recipe revision hashes when repo commits change

---

## Build Environment Setup

```bash
# Install dependencies (Ubuntu 20.04/22.04)
sudo apt update && sudo apt install -y git ssh make gcc libssl-dev \
  liblz4-tool expect expect-dev g++ patchelf chrpath gawk texinfo \
  diffstat binfmt-support qemu-user-static live-build bison flex \
  fakeroot cmake gcc-multilib g++-multilib unzip device-tree-compiler \
  ncurses-dev libgucharmap-2-90-dev bzip2 expat gpgv2 cpp-aarch64-linux-gnu \
  libgmp-dev libmpc-dev bc python2 libstdc++-12-dev xz-utils repo curl wget \
  python3-pip

pip install numpy pillow

# Build
git clone git@github.com:aml-streambox/yocto.git
cd yocto
git submodule update --init --recursive
source meta-meson/aml-setenv.sh
bitbake amlogic-yocto-debug
```

### Target Machines

| Machine | Description |
|---------|-------------|
| `mesont7-tvpro-5.15` | TVPro board, kernel 5.15 |
| `mesont7c-kvim4-5.15` | KVim4 board, kernel 5.15 |

### Build Output

```
build/tmp/deploy/images/CONFIGURATION/<boardname>-yocto-<date>.img
```

---

## Branch Workflow

### Before Any Code Modification

1. **Ask user for branch name:**
   - Suggest descriptive names like `feature/hdmi-vrr-fix` or `bugfix/encoder-crash`
   
2. **Create and switch to dev branch:**
   ```bash
   git checkout -b <branch-name>
   ```

3. **Make changes and commit incrementally**

4. **After testing, merge to main:**
   ```bash
   git checkout main
   git merge <branch-name>
   git push origin main
   ```

### Documentation Requirements

Every modification must be documented. Update relevant files:
- `document/custom-software/*.md` - Feature/release documentation
- `document/faq-and-roadmap/*.md` - Known issues and roadmap
- `AGENT.md` - Current development status
- `AI_CONTEXT.md` - Project-specific AI context (if exists)
- Component-specific README files

Document the direction of modification (not each line):
- What feature/bug was addressed
- Key files changed
- Testing performed

---

## Component Build Commands

### Encoder Stack

```bash
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15

# Rebuild encoder library ( Wave521 userspace)
bitbake -f -c compile libmultienc
bitbake -f -c compile gst-plugin-venc-multienc

# Rebuild capture plugin (streamboxsrc)
bitbake -f -c compile gst-plugin-vfmcap
```

### Kernel (5.15)

```bash
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15

#any kernel change require full rebuild and full image flash
```

Userspace drivers are in:
- `aml-comp/hardware/aml-5.15/` - Userspace hardware libraries
- `aml-comp/multimedia/` - GStreamer plugins and multimedia components

### Cockpit Components

```bash
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15

# Rebuild gst-manager
bitbake -f -c compile cockpit-gst-manager

# Rebuild streambox-settings
bitbake -f -c compile cockpit-streambox-settings
```
To modify the source code, you must clone to code to any location locally.
To deploy and test the code, copy the modified source code entirely to target.
Then use source code's install.sh script to apply the change to target.


### Full System Image

```bash
source meta-meson/aml-setenv.sh mesont7-tvpro-5.15
bitbake amlogic-yocto
```

---

## Deploy to Target

### SSH Access

```bash
# Default target
#ask user for target <device-ip>
ssh root@<device-ip>

# Push built components
scp build/tmp/deploy/images/mesont7-tvpro-5.15/<component> root@<device-ip>:/usr/lib/
```

### After Deploying Encoder/Plugins

```bash
ssh root@<device-ip> "rm -f ~/.cache/gstreamer-1.0/registry.*"
```

### Reboot if VPU or any hardware State is Bad

```bash
ssh root@<device-ip> "reboot"
```

---

## Device Tree & Kernel

### Kernel Location

- `aml-comp/kernel/aml-5.15/` - Kernel 5.15 sources
- `aml-comp/kernel/aml-5.4/` - Kernel 5.4 sources (legacy)

### Device Tree

Device tree files are in kernel source under `arch/arm64/boot/dts/amlogic/`.

### Kernel Modules

Drivers are in kernel source under `drivers/`. Key paths:
- `drivers/media/` - V4L2 and multimedia drivers
- `drivers/gpu/` - GPU drivers
- `drivers/amlogic/` - Amlogic-specific drivers

---

## GStreamer Plugin Development

### Key Plugins

| Plugin | Location | Purpose |
|--------|----------|---------|
| `streamboxsrc` | `aml-comp/multimedia/gst-plugin-vfmcap/` | HDMI capture source |
| `amlvenc` | `aml-comp/multimedia/gst-plugin-venc/` | Video encoder (Wave521) |
| `amlge2d` | GStreamer elements | 2D graphics operations |

### Pipeline Testing

```bash
# Basic encode test
gst-launch-1.0 streamboxsrc ! "video/x-raw,format=NV12" ! \
  amlvenc bitrate=8000 gop-pattern=1 gop=30 ! \
  "video/x-h265" ! filesink location=/tmp/out.h265

# Frame type verification
ffprobe -show_frames -select_streams v -show_entries frame=pict_type -of csv /tmp/out.h265

```

---

## Advanced Sysfs Debugging

**IMPORTANT:** Always confirm sysfs usage from corresponding driver source code first. Never make assumptions.

### Frame Lock Debug

```bash
# Enable frame lock debug print
echo 1 > /sys/module/aml_media/parameters/frame_lock_debug

#dmesg will be flooded by frame lock debug print, you must Dsiable it in a short time

# Disable
echo 0 > /sys/module/aml_media/parameters/frame_lock_debug
```

### Kernel Print Control

```bash
# Enable kernel print
echo 1 > /proc/sys/kernel/printk

# Disable
echo 0 > /proc/sys/kernel/printk
```

### VRR Mode Debug

```bash
# Force VRR mode (confirm mode number from driver code first!)
echo "mode <mode_number>" > /sys/class/aml_vrr/vrr2/debug

# Check frame lock info
cat /sys/class/amvecm/frame_lock
```

### HDMI RX Debug

```bash
# Change log level
echo log_level 0x1 > /sys/class/hdmirx/hdmirx0/param

# Check signal
cat /sys/class/hdmirx/hdmirx0/signal
cat /sys/class/hdmirx/hdmirx0/info
```

### HDMI TX Configuration

```bash
# Set output format
echo rgb,12bit > /sys/class/amhdmitx/amhdmitx0/test_attr

# Set HDR mode
echo hdr > /sys/class/amhdmitx/amhdmitx0/config

# Set display mode
echo off > /sys/class/amhdmitx/amhdmitx0/disp_mode
echo 1080p60hz > /sys/class/amhdmitx/amhdmitx0/disp_mode
```

### Display Info

```bash
cat /sys/class/display/vinfo
```

### Video Input Debug

```bash
echo state > /sys/class/vdin/vdin0/attr
```

---

## Recipe Management

### When Repo Commit Changes

If a recipe references a commit hash in a git repository, update the hash:

```bash
# In recipe file (e.g., some-package.bb):
SRCREV = "abcdef123456..."  # Update this to new commit hash
```

### Build Single Package

```bash
# Clean build
bitbake -c clean <package-name>

# Force recompile
bitbake -f -c compile <package-name>

# Build package only
bitbake <package-name>
```

---

## Key Source Locations

| Component | Path |
|-----------|------|
| Kernel 5.15 | `aml-comp/kernel/aml-5.15/` |
| Encoder userspace | `aml-comp/hardware/aml-5.15/amlogic/libencoder/` |
| GStreamer plugins | `aml-comp/multimedia/` |
| Capture plugin | `aml-comp/multimedia/gst-plugin-vfmcap/` |
| Encoder plugin | `aml-comp/multimedia/gst-plugin-venc/` |
| Cockpit gst-manager  (must explicitly clone to local first) | `cockpit-gst-manager/` |
| Cockpit settings  (must explicitly clone to local first) | `cockpit-streambox-settings/` |
| TV server (must explicitly clone to local first) | `aml_tvserver_streambox/` |
| Yocto recipes | `meta-aml-cfg/recipes-*/` |
| Machine configs | `meta-aml-cfg/conf/machine/` |

---

## Testing Checklist

After building and deploying:

1. **Clear GStreamer cache:**
   ```bash
   ssh root@<device-ip> "rm -f ~/.cache/gstreamer-1.0/registry.*"
   ```

2. **Test basic pipeline:**
   ```bash
   ssh root@<device-ip> "gst-launch-1.0 streamboxsrc ! 'video/x-raw,format=NV12' ! amlvenc ! 'video/x-h265' ! filesink location=/tmp/test.h265"
   ```

3. **Verify output:**
   ```bash
   ssh root@<device-ip> "ffprobe -show_frames /tmp/test.h265"
   ```

4. **Check kernel logs:**
   ```bash
   ssh root@<device-ip> "dmesg | tail -50"
   ```

5. **Reboot if VPU state bad:**
   ```bash
   ssh root@<device-ip> "reboot"
   ```

---

## Notes

- Minimum 150GB disk space required for full build
- Always check sysfs interface from driver source before use
- Clear GStreamer registry after deploying encoder/plugins
- Document all modifications before merging to main

# Useful additional external document
- https://dl.khadas.com/products/vim4/datasheet/a311d2-datasheet-rev-c-0.2.pdf A311D2 chip spec datasheet