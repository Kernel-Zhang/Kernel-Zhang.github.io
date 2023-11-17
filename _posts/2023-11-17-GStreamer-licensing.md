---
title: gstreamer插件编写指南：GStreamer 许可
author: kernel
date: 2023-11-17 09:53:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

## 如何为您的GStreamer代码添加许可

GStreamer 是一个基于插件的框架，采用 LGPL 许可。之所以选择这种授权方式，是为了确保每个人都能使用 GStreamer 构建应用程序，并使用自己选择的授权方式。

为了保持这一政策的可行性，GStreamer 社区为 GStreamer 内核或 GStreamer 官方模块（如我们的插件包）中的代码制定了一些许可规则。我们要求所有进入核心软件包的代码都必须使用 LGPL 协议。对于插件代码，我们要求所有从零开始编写的插件或链接到外部库的插件使用 LGPL。唯一的例外是当插件包含使用更自由许可证（如 MPL 或 BSD）的旧代码时。它们可以使用这些许可证，但仍会被考虑纳入。我们不接受 GPL 代码添加到我们的插件模块中，但我们接受使用外部 GPL 库的 LGPL 许可插件。要求插件使用 LGPL 许可（即使使用 GPL 库）的原因是，其他开发者可能希望将插件代码链接到非 GPL 库的插件模块。

我们还计划最终将使用 GPL 库的插件拆分成一个单独的软件包，并实施一套系统，确保应用程序无法访问这些插件，除非使用一些特殊代码来访问。这样做的目的不是阻止使用和开发 GPL 许可的插件，而是确保人们不会无意中违反这些插件的 GPL 许可。

本公告是更大公告的一部分，附有常见问题解答，可在[GStreamer 网站](https://gstreamer.freedesktop.org/documentation/licensing.html)上查阅
