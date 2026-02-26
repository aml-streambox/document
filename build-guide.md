---
layout: page
title: Build Guide
permalink: /build-guide/
---

# Build Guide

## Prerequisites

Ubuntu 20.04/22.04 is recommended for compilation.

## Install Dependencies

```bash
sudo apt update && sudo apt install -y git ssh make gcc libssl-dev \
liblz4-tool expect expect-dev g++ patchelf chrpath gawk texinfo chrpath \
diffstat binfmt-support qemu-user-static live-build bison flex fakeroot \
cmake gcc-multilib g++-multilib unzip device-tree-compiler ncurses-dev \
libgucharmap-2-90-dev bzip2 expat gpgv2 cpp-aarch64-linux-gnu libgmp-dev \
libmpc-dev bc python-is-python3 python2 libstdc++-12-dev xz-utils repo python3-pip

pip install numpy pillow
```

## Clone Repository

Create a Git SSH key and add it to your GitHub account. This project uses Git submodules with SSH links.

```bash
git clone git@github.com:aml-streambox/yocto.git
```

## Initialize Submodules
It will take a while.
```bash
cd yocto
git submodule update --init --recursive
```

## Build
It wil need more than 30G disk space.
```bash
source meta-meson/aml-setenv.sh
# select your board
bitbake amlogic-yocto
```

## Output

After a successful build, find the image at:
```
build/tmp/deploy/images/CONFIGURATION/<boardname>-yocto-<date>.img
```

## Flashing System Image

### Khadas VIM4

Just flash the image like a normal OS image.

Please refer VIM4's official guide [boot-into-upgrade-mode](https://docs.khadas.com/products/sbc/vim4/install-os/boot-into-upgrade-mode) to boot into upgrade mode then using [flash os using usb tool](https://docs.khadas.com/products/sbc/vim4/install-os/install-os-into-emmc-via-usb-tool)

[download USB upgrade tool](https://dl.khadas.com/products/vim4/tools/aml-burn-tool-v3.2.0.zip)

### TVPro Device
Pretty much same as VIM4 for Serial Mode.

For Key Mode, press the reset key and plug in power supply without releasing the reset key.


**Note**: Flashing methods may vary based on the specific board revision and bootloader version. Always ensure you have the correct image for your device variant.
