---
layout: page
title: 项目概览
---

[🇺🇸 English Version](index_en.md)

# 项目概览

## 1. 项目简介

Streambox 是一个基于 Amlogic A311D2 SoC 的直播专用设备项目，支持视频采集与处理功能，专为流媒体应用场景设计。

核心特性：
- 支持 4K60fps HDR 视频从 HDMI RX 到 HDMI TX 的透传，具备超低延迟和 VRR 支持
- 支持 4K60 视频的网络采集、编码与传输（SDR YUV420 与 HDR 10-bit）
- HDR10 直播支持，集成 GPU 加速 Vulkan 计算着色器管线，提供完整的 HDR10 VUI 信号
- 支持 H.264/H.265 硬件编码
- 支持 SRT/RTMP 流媒体协议

## 2. 项目背景

本项目最初旨在通过单板计算机（SBC）以 1080p120fps 的规格直播街机游戏。选用 A311D2 芯片基于以下考量：
- 支持 4K60 + VRR 的 HDMI RX/TX
- 硬件编码器最高支持 4K60fps
- 强劲的 CPU 性能
- 支持 WiFi 6 等无线网络（通过 Khadas VIM4 开发板）

这些规格远超原始需求，使 A311D2 成为构建专业直播主机的理想选择。

## 3. 功能概述

本项目提供驱动修复与精简软件栈，用于视频直播：
- 定制版 aml_tvserver 用于视频管线控制
- 定制版 GStreamer v4l2src 插件用于视频采集
- Cockpit 插件用于 GStreamer 管线管理
- 基于 Cockpit 的轻量级 Web 界面

**注意**：系统稳定性有限，可能出现绿屏、黑屏或系统崩溃等情况。遇到问题时可能需要重启设备。

## 4. 硬件兼容性

任何搭载 A311D2 芯片且具备 HDMI RX、HDMI TX、以太网和 WiFi 功能的设备均可兼容。

已验证设备：
- Khadas VIM4（主要开发平台）
- TVPro 4G（基于 A311D2 核心板的系统）

## 5. 使用指南

### 构建系统镜像

参考[构建指南](build-guide/build-guide_cn.md)自行编译镜像。本项目不提供预编译镜像。

### 刷写镜像

1. 编译完成后，镜像位于 `build/tmp/deploy/images/CONFIGURATION/<boardname>-yocto-<date>.img`
2. 使用 AmlBurn Tool V3 直接刷写至设备 eMMC

### 首次启动

刷写完成后：
- 可通过串口（波特率 921600）或 SSH 连接
- 默认登录：root / Streamb0x
- 以太网自动通过 DHCP 获取 IP
- WiFi 热点默认开启

### 访问 Cockpit Web 界面

连接设备 WiFi 热点或通过以太网，访问：
```
http://192.168.172.1:9090（WiFi 热点）
或
http://<DHCP 分配的 IP>:9090（以太网或 WiFi 客户端模式）
```

### HDMI 透传

HDMI 透传默认启用。HDMI TX 必须连接至有效显示设备（电视或显示器），视频采集功能才能正常工作。

### 视频采集与直播

默认配置下，GStreamer Manager 插件采集 HDMI TX 的输出（因此 HDMI TX 必须连接），并在 8888 端口提供 SRT 串流。可在 Cockpit GStreamer Manager 选项卡中调整直播设置。

## 6. 文档索引

- [项目概览](index/index_cn.md) - 本文档
- [构建指南](build-guide/build-guide_cn.md) - 镜像编译说明
- [配置指南](setup-guide/setup-guide_cn.md) - 镜像刷写与系统使用
- [驱动改动](driver-modifications/driver-modifications_cn.md) - 内核与驱动修改详情
- [软件组件](custom-software/custom-software_cn.md) - Cockpit 插件说明
- [源码参考](source-code-reference/source-code-reference_cn.md) - 许可证与代码来源
- [常见问题与路线图](faq-and-roadmap/faq-and-roadmap_cn.md) - 已知问题、FAQ 与开发计划

## 7. 开发方式

本项目完全通过**氛围编程（Vibe Coding）**方式开发，使用了 Claude Opus、Kimi 2.5、GLM-4.7、Gemini 3 Pro 等多种 AI 模型。

**重要提示**：作者**完全不理解 AI 生成的代码工作原理**（但代码确实能跑）。

### 使用须知

- 未经过正式代码审查
- AI 生成的代码可能存在意外行为或"有趣"的逻辑
- 项目功能可用，但实现方式可能非主流
- 欢迎贡献代码与提交 Issue

## 8. 许可证

本项目采用 [MIT License](../LICENSE) 授权。

### 原始代码许可

本项目基于 Khadas VIM4 Yocto 版本。原始组件继承上游项目的相应许可证（内核及多数组件为 GPL/LGPL）。详情请参考[源码参考](source-code-reference/source-code-reference_cn.md)。

### 署名声明（感谢但非强制）

商业使用场景中，提及本项目作为参考是受欢迎的，但许可证不作强制要求。

### 道德使用

作者强烈反对将本项目用于任何形式的作弊行为，包括但不限于学术不端、游戏作弊或考试作弊。请负责任地使用。

---

*本项目通过氛围编程与 AI 辅助开发完成。详见[开发方式](#7-开发方式)。*
