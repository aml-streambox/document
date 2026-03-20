---
layout: page
title: 构建指南
permalink: /build-guide/
---

[🇺🇸 English Version]({{ '/build-guide/build-guide_en' | relative_url }})

# 构建指南

## 环境要求

推荐使用 Ubuntu 20.04/22.04 作为编译环境。

## 安装依赖

```bash
sudo apt update && sudo apt install -y git ssh make gcc libssl-dev \
liblz4-tool expect expect-dev g++ patchelf chrpath gawk texinfo chrpath \
diffstat binfmt-support qemu-user-static live-build bison flex fakeroot \
cmake gcc-multilib g++-multilib unzip device-tree-compiler ncurses-dev \
libgucharmap-2-90-dev bzip2 expat gpgv2 cpp-aarch64-linux-gnu libgmp-dev \
libmpc-dev bc python-is-python3 python2 libstdc++-12-dev xz-utils repo python3-pip

pip install numpy pillow
```

## 克隆仓库

生成 Git SSH 密钥并添加到 GitHub 账户。本项目使用 SSH 协议的 Git 子模块。

```bash
git clone git@github.com:aml-streambox/yocto.git
```

## 初始化子模块
该过程耗时较长，请耐心等待。
```bash
cd yocto
git submodule update --init --recursive
```

## 编译系统
需要超过 30G 的磁盘空间。
```bash
source meta-meson/aml-setenv.sh
# 选择目标开发板
bitbake amlogic-yocto
```

## 输出文件

编译成功后，镜像文件位于：
```
build/tmp/deploy/images/CONFIGURATION/<boardname>-yocto-<date>.img
```

镜像刷写与系统配置请参考配置指南。
