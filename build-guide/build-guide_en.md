---
layout: page
title: Build Guide
permalink: /build-guide/
---

[🇨🇳 中文版]({{ '/build-guide/build-guide_cn' | relative_url }})

# Build Guide

## Prerequisites

Ubuntu 20.04/22.04 is recommended for compilation.

## Method 1: Docker Build Environment (Recommended)

Using Docker ensures a consistent build environment across different host systems.

### Prerequisites for Docker

- Docker installed on your host system
- At least 50GB free disk space for the build

### Build Steps

1. **Clone the repository and initialize submodules**

   ```bash
   git clone git@github.com:aml-streambox/yocto.git
   cd yocto
   git submodule update --init --recursive
   ```

2. **Build the Docker image**

   ```bash
   ./build_docker_env.sh
   ```

   This script will build a Docker image named `streambox-builder` with all required dependencies.

   - Use `-f` flag to force rebuild: `./build_docker_env.sh -f`

3. **Start the Docker container and build**

   ```bash
   # Run the container with your yocto directory mounted
   docker run -it --rm \
     -v $(pwd):/yocto \
     -w /yocto \
     streambox-builder bash
   
   # Inside the container
   source meta-meson/aml-setenv.sh
   # Select your board
   bitbake amlogic-yocto
   ```

## Method 2: Native Build

If you prefer to build directly on your host system without Docker.

### Install Dependencies

```bash
sudo apt update && sudo apt install -y git ssh make gcc libssl-dev \
liblz4-tool expect expect-dev g++ patchelf chrpath gawk texinfo chrpath \
diffstat binfmt-support qemu-user-static live-build bison flex fakeroot \
cmake gcc-multilib g++-multilib unzip device-tree-compiler ncurses-dev \
libgucharmap-2-90-dev bzip2 expat gpgv2 cpp-aarch64-linux-gnu libgmp-dev \
libmpc-dev bc python-is-python3 python2 libstdc++-12-dev xz-utils repo python3-pip

pip install numpy pillow
```

### Clone Repository

Create a Git SSH key and add it to your GitHub account. This project uses Git submodules with SSH links.

```bash
git clone git@github.com:aml-streambox/yocto.git
```

### Initialize Submodules

It will take a while.

```bash
cd yocto
git submodule update --init --recursive
```

### Build

It will need more than 30G disk space.

```bash
source meta-meson/aml-setenv.sh
# Select your board
bitbake amlogic-yocto
```

## Output

After a successful build, find the image at:

```
build/tmp/deploy/images/CONFIGURATION/<boardname>-yocto-<date>.img
```

Please refer to the [Setup Guide]({{ '/setup-guide/setup-guide_en' | relative_url }}) for OS flashing and Streambox setup.
