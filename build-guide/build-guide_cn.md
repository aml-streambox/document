---
layout: page
title: 构建指南
---

[🇺🇸 English Version]({{ '/build-guide/build-guide_en' | relative_url }})

# 构建指南

## 预编译镜像（推荐）

项目提供预编译镜像下载，可直接下载使用，无需自行编译：

**下载地址：** https://streambox-dl.cosmiccat.net/

下载适合您设备的镜像后，请参考[配置指南]({{ '/setup-guide/setup-guide_cn' | relative_url }})进行刷写。

---

## 从源码编译

如果您希望自行编译镜像，请参考以下说明。

### 环境要求

推荐使用 Ubuntu 20.04/22.04 作为编译环境。

### 方法一：Docker 编译环境

使用 Docker 可以确保在不同主机系统上保持一致的编译环境。

### Docker 环境要求

- 主机系统已安装 Docker
- **至少 150GB 可用磁盘空间**用于编译

### 编译步骤

1. **克隆仓库并初始化子模块**

   ```bash
   git clone git@github.com:aml-streambox/yocto.git
   cd yocto
   git submodule update --init --recursive
   ```

2. **构建 Docker 环境并进入容器**

   ```bash
   ./build_docker_env.sh
   ```

   该脚本将执行以下操作：
   - 构建名为 `streambox-builder` 的 Docker 镜像，包含所有必需的依赖
   - 自动启动容器并挂载当前目录
   - 直接进入容器内的 shell

   - 使用 `-f` 参数强制重新构建：`./build_docker_env.sh -f`

3. **在容器内编译**

   进入 Docker 容器后，执行以下命令：

   ```bash
   source meta-meson/aml-setenv.sh
   # 选择目标开发板
   bitbake amlogic-yocto
   ```

## 方法二：本地编译

如果你希望直接在主机系统上编译而不使用 Docker。

### 环境要求

- **至少 150GB 可用磁盘空间**

### 安装依赖

```bash
sudo apt update && sudo apt install -y git ssh make gcc libssl-dev \
liblz4-tool expect expect-dev g++ patchelf chrpath gawk texinfo chrpath \
diffstat binfmt-support qemu-user-static live-build bison flex fakeroot \
cmake gcc-multilib g++-multilib unzip device-tree-compiler ncurses-dev \
libgucharmap-2-90-dev bzip2 expat gpgv2 cpp-aarch64-linux-gnu libgmp-dev \
libmpc-dev bc python-is-python3 python2 libstdc++-12-dev xz-utils repo python3-pip

pip install numpy pillow
```

### 克隆仓库

生成 Git SSH 密钥并添加到 GitHub 账户。本项目使用 SSH 协议的 Git 子模块。

```bash
git clone git@github.com:aml-streambox/yocto.git
```

### 初始化子模块

该过程耗时较长，请耐心等待。

```bash
cd yocto
git submodule update --init --recursive
```

### 编译系统

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

镜像刷写与系统配置请参考[配置指南]({{ '/setup-guide/setup-guide_cn' | relative_url }})。
