---
layout: page
title: Custom Software
permalink: /custom-software/
---

# Custom Software Components

## cockpit-gst-manager

A Cockpit plugin for managing GStreamer streaming/encoding pipelines on Amlogic A311D2.

- Repository: https://github.com/aml-streambox/cockpit-gst-manager
- Platform: Amlogic A311D2, GStreamer 1.20.7, Cockpit 220

### Features

- **Multi-instance** - Run multiple GStreamer pipelines simultaneously
- **AI Assistant** - Natural language to GStreamer CLI (BYO API key)
- **Manual Editor** - Direct CLI editing for advanced users
- **Event Triggers** - Auto-start on HDMI signal, USB plug, boot
- **Video Compositor** - OSD/overlay via ge2d hardware acceleration
- **Import/Export** - Share pipeline configurations
- **Localization** - English and Chinese UI

### Development History

- **Auto pipeline implementation** - Automatic GStreamer pipeline configuration
- **TVService integration** - Use tvserver for HDMI RX signal change detection
- **AI prompt optimization** - Refined prompts, removed recording feature
- **Boot order fix** - Service boots at last order to ensure dependencies ready

## cockpit-streambox-settings

A Cockpit plugin for managing system settings on Amlogic A311D2 Streambox.

- Repository: https://github.com/aml-streambox/cockpit-streambox-settings
- Platform: Amlogic A311D2, Cockpit 220

### Features

- **Basic Settings** - Device name, timezone, system locale
- **Network Settings** - Wired, WiFi client, WiFi AP configuration
- **TVServer Settings** - Video, audio, HDCP, debug configuration
- **Storage Settings** - SDCard management, formatting, mount points

### Requirements

- Amlogic A311D2/T7 Streambox hardware
- Yocto-built image with Cockpit
- tvservice (aml_tvserver_streambox) running
