---
title: gstreamer插件编写指南：编写插件
author: kernel
date: 2023-11-14 16:34:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

现在，你已经准备好学习如何制作插件了。在本部分指南中，你将学习如何应用基本的 GStreamer 编程概念来编写一个简单的插件。本指南的前几部分没有明确的示例代码，可能会让人觉得有些抽象和难以理解。相比之下，本部分将通过开发一个名为 "MyFilter "的音频过滤器插件示例，同时介绍应用程序和代码。

示例滤波器元素一开始只有一个输入衬底和一个输出衬底。起初，过滤器只是简单地将媒体和事件数据从其汇衬底传输到源 衬底，而不做任何修改。但在本部分内容结束时，您将学会添加一些更有趣的功能，包括属性和信号处理器。阅读本指南的下一部分 "高级过滤器概念"后，您将能为插件添加更多功能。

本章主要由以下几部分组成

- [构建模板](../boiler)

- [指定衬底](../Specifying-the-pads)

- [链函数](../The-chain-function)

- [事件函数](../The-event-function)

- [查询函数](../The-query-function)

- [什么是状态？](../What-are-states)

- [添加属性](../Adding-Properties)

- [信号](../Signals)

- [构建测试应用程序](../Building-a-Test-Application)