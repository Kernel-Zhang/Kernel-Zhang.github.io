---
title: 常见问题：依赖关系
author: kernel
date: 2024-02-06 11:31:00 +0800
categories: [gstreamer, 常见问题]
tags: [gst常见问题, gstreamer, 多媒体]
---

## 为什么有这么多依赖关系？

制作一个全功能的媒体框架本身就是一项艰巨的任务。通过使用他人的工作，我们既减少了多余的工作，又能腾出时间来研究架构本身，而不是研究底层的东西。如果我们不重复使用他人编写的代码，那就太愚蠢了。

不过，请务必认识到，您绝不可能被迫安装所有依赖项。没有一个核心开发者会安装所有依赖项。GStreamer 只有几个必须安装的依赖项：GLib 2.0、liborc，以及一些非常常见的东西，如 glibc、C 编译器等等。所有其他依赖项都是可选的。

最后，让我们把问题改成："你为什么要给我这么多的选择和这么丰富的环境？

## GStreamer 独立于 X11 吗？可以无头使用吗？

是的，我们的任何模块都不硬性依赖 X11 或任何其他窗口系统。许多 GStreamer 应用程序无需显示服务器或窗口系统即可正常运行，例如流媒体服务器、转码应用程序或不输出任何视频的音频应用程序。其他应用程序会将视频输出到帧缓冲器、定制的输出硬件或通过 wayland 输出。

## GStreamer 对 LADSPA 或 LV2 等工作的立场是什么？

GStreamer 积极努力地支持，对于[LADSPA](https://en.wikipedia.org/wiki/LADSPA)或[LV2](https://lv2plug.in/)，我们已经有了封装插件。这些封装插件可在运行时检测系统中的 LADSPA/LV2 插件中找到，并将其作为 GStreamer 元素提供。

## GStreamer 支持 MIDI 吗？

GStreamer 支持一些基本的 MIDI，但还不够完善。

GStreamer 架构应能很好地支持 MIDI 应用程序的需求，但目前仍未完全实现。 如果你是开发人员，有兴趣为 GStreamer 添加 MIDI 支持，请与我们联系，我们一定会很感兴趣。

至于现在的情况：[`alsamidisrc`](https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gst-plugins-base-plugins/html/gst-plugins-base-plugins-alsamidisrc.html)元素可用于获取 ALSA MIDI 音序器事件，并将其提供给能理解`audio/x-midi-events`格式的元素。

MIDI 播放功能由`midiparse`、`fluiddec`、`wildmidi`和`timidity` 等插件提供。

## GStreamer 依赖于 GNOME 还是 GTK+？

都不，这只是因为许多 GStreamer 应用程序（包括我们的一些示例程序）碰巧是 GNOME 或 GTK+ 应用程序，但使用 Qt 工具包或为 Mac OS/X、Windows、Android 或 iOS 编写的应用程序也同样多。

我们的目标是提供一个与工具包无关的应用程序接口，使 GStreamer 可以在任何工具包、桌面环境或操作系统中使用。
