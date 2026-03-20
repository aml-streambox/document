---
layout: page
title: 软件组件
permalink: /custom-software/
---

[🇺🇸 English Version](custom-software_en.md)

# 软件组件

## cockpit-gst-manager

适用于 Amlogic A311D2 的 GStreamer 直播/编码管线管理 Cockpit 插件。

- 仓库：https://github.com/aml-streambox/cockpit-gst-manager
- 平台：Amlogic A311D2、GStreamer 1.20.7、Cockpit 220

### 功能特性

- **多实例支持** - 同时运行多条 GStreamer 管线
- **AI 助手** - 自然语言转 GStreamer CLI（需自备 API 密钥）
- **手动编辑** - 面向高级用户的 CLI 直接编辑
- **事件触发** - HDMI 信号、USB 插入、系统启动时自动启动
- **视频合成** - 通过 ge2d 硬件加速实现 OSD/叠加
- **导入/导出** - 分享管线配置
- **本地化** - 中英文界面

### 开发历程

- **自动管线实现** - 自动 GStreamer 管线配置
- **TVService 集成** - 使用 tvserver 检测 HDMI RX 信号变化
- **AI 提示词优化** - 优化提示词，移除录制功能
- **启动顺序修复** - 服务最后启动确保依赖就绪

## cockpit-streambox-settings

适用于 Amlogic A311D2 Streambox 的系统设置管理 Cockpit 插件。

- 仓库：https://github.com/aml-streambox/cockpit-streambox-settings
- 平台：Amlogic A311D2、Cockpit 220

### 功能特性

- **基础设置** - 设备名称、时区、系统区域设置
- **网络设置** - 有线、WiFi 客户端、WiFi 热点配置
- **TVServer 设置** - 视频、音频、HDCP、调试配置
- **存储设置** - SD 卡管理、格式化、挂载点

### 系统要求

- Amlogic A311D2/T7 Streambox 硬件
- 集成 Cockpit 的 Yocto 镜像
- 运行中的 tvservice (aml_tvserver_streambox)
