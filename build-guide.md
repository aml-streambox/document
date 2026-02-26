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

Please refer setup guide for OS flashing and Streambox setup
