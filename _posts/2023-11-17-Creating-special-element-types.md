---
title: gstreamer插件编写指南：创建特殊元素类型
author: kernel
date: 2023-11-17 09:11:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

到目前为止，我们已经了解了几乎所有可以嵌入 GStreamer 元素的功能。其中大部分都相当低级，让我们深入了解 GStreamer 的内部工作原理。幸运的是，GStreamer 包含一些更易于使用的接口来创建此类元素。为此，我们将仔细研究 GStreamer 提供基类（源、汇和转换元素）的元素类型。我们还将仔细研究一些不需要特定编码（如调度交互或数据传递），但需要特定流水线控制（如 N 对 1 元素和管理器）的元素类型。

本部分分成以下几部分进行介绍：

[预制基类](../Pre-made-base-classes/)

[编写解复用器和解析器](../Writing-a-Demuxer-or-Parser/)

[编写多对一元素和多路复用器](../Writing-a-N-to-1-Element-or-Muxer/)

[编写管理器](../Writing-a-Manager/)

