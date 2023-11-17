---
title: gstreamer插件编写指南：编写管理器
author: kernel
date: 2023-11-17 09:41:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

管理器是增加功能或统一其他（一系列）元素功能的元素。管理器通常是一个带有一个或多个衬底的`GstBin`。其中包含的是实际的重要元素。有几种情况下这是有用的。例如：

-   在另一个元素中添加对带有自定义事件处理功能的私有事件的支持。
    
-   为另一个元素添加对自定义衬底`_query ()`或`_convert ()`处理的支持。
    
-   在另一个元素的数据处理函数（通常是其`_chain ()`函数）之前或之后添加自定义数据处理。
    
-   将一个元素或一系列元素嵌入到某个东西中，使其在外界看来就像一个简单的元素。这对于实现具有多个衬底的源和汇元素尤为方便。
    

制作管理器非常简单。您可以从`GstBin` 派生，在大多数情况下，您可以在`_init ()`中嵌入所需的元素，包括设置衬底。如果需要自定义数据处理器，可以连接信号或嵌入由自己控制的第二个元素。