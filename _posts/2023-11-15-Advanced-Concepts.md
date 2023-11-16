---
title: gstreamer插件编写指南：高级概念
author: kernel
date: 2023-11-15 10:09:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

现在，你应该能够创建可以接收和发送数据的基本过滤元素了。这就是 GStreamer 所代表的简单模式。但 GStreamer 能做的远不止这些！本章将讨论各种高级主题，如调度、特殊 pad 类型、时钟、事件、接口、标记等。这些主题是使 GStreamer 在应用程序中如此易于使用的 "糖"。

该部分由以下几个部分构成：

[请求型和暂态型衬底](../Request-and-Sometimes-pads/)

[不同的调度模式](../Different-scheduling-modes/)

[能力集协商](../Caps-negotiation/)

[内存分配](../Memory-allocation/)

[媒体类型和属性](../Media-Types-and-Properties/)

[事件：搜索、导航及更多](../Events-Seeking-Navigation-and-More/)

[时钟](../Clocking/)

[服务质量（QoS）](../QualityOfService/)

[支持动态参数](../SupportingDynamicParameters/)

[接口](../Interfaces/)

[标记（元数据和流信息）](../Tagging-MetadataandStreaminfo/)
