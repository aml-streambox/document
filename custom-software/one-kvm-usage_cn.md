---
layout: page
title: One-KVM 使用说明
permalink: /custom-software/one-kvm-usage_cn/
---

# One-KVM 使用说明

One-KVM 在 StreamBox 镜像中作为默认 GStreamer Manager 工作流之外的另一种
可选应用提供。它提供基于浏览器的 KVM 访问，并支持键盘/鼠标输入和 USB
大容量存储等 USB gadget 功能。

> **状态说明：** StreamBox v7.0 中的 One-KVM 仍属于实验功能。视频、HID 和
> MSD 基本功能已经可用，但稳定性尚不建议用于生产环境。

## 默认行为

StreamBox 默认启动 GStreamer Manager 应用，也就是设备开机后会进入由
`gst-manager` 管理的常规直播/编码工作流。

如果要使用 One-KVM，需要把当前活动应用从 GStreamer Manager 切换到 One-KVM。

One-KVM 默认监听 `8080` 端口：

```text
http://<设备IP>:8080
```

Cockpit 仍然使用 `9090` 端口：

```text
http://<设备IP>:9090
```

## 通过 Cockpit 切换

推荐方式：

1. 在浏览器中打开 Cockpit：`http://<设备IP>:9090`
2. 使用 `root` 登录
3. 打开 StreamBox Settings 页面
4. 找到活动应用设置
5. 将活动应用从 GStreamer Manager 切换为 One-KVM
6. 应用设置，并在提示时重启所选应用
7. 打开 One-KVM：`http://<设备IP>:8080`

也可以在同一页面把活动应用从 One-KVM 切回 GStreamer Manager。

## 通过 systemd 切换

高级用户也可以直接使用 `systemctl` 手动切换应用。

停止并禁用 GStreamer Manager：

```bash
systemctl disable --now gst-manager.service
```

启用并启动 One-KVM：

```bash
systemctl enable --now one-kvm.service
```

查看服务状态：

```bash
systemctl status one-kvm.service
```

切回 GStreamer Manager：

```bash
systemctl disable --now one-kvm.service
systemctl enable --now gst-manager.service
```

## HID 和 MSD

One-KVM 支持以下 USB gadget 功能：

- HID：通过浏览器会话向被控主机发送键盘和鼠标输入
- MSD：支持用于虚拟介质工作流的 USB 大容量存储设备

当 USB gadget 路径处于活动状态并已连接时，被控主机应能识别到 StreamBox
提供的 USB HID 设备。MSD 是否可用取决于 One-KVM 中的存储/介质配置。

## 注意事项和限制

- One-KVM 仍为实验功能，可能崩溃或需要手动重启服务。
- 不要同时把 GStreamer Manager 和 One-KVM 作为活动视频应用运行。
- 如果切换应用后视频采集失败，请重启当前所选服务，或直接重启设备。
- 对于生产直播场景，仍建议使用 GStreamer Manager 作为活动应用。
