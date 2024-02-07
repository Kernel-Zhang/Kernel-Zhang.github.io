---
title: gstreamer插件编写指南：编写元素时需要检查的事项
author: kernel
date: 2023-11-17 09:52:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

本章随机选取了一些编写元素时需要注意的事项。至于你能在多大程度上遵守这些准则，这取决于你自己。不过，请记住，当你编写一个元素，并希望将其纳入主流 GStreamer 发行版时，它必须满足这些要求。我们将尽可能解释这些要求的原因。

## 关于状态

-   确保元素的状态在变为`NULL` 时被重置。 理想情况下，应将所有对象属性设置为其原始状态。该函数也应在 `_init` 中调用。
    
-   确保元素从`PAUSED`状态转为`READY`状态时，忘记其包含的流的所有信息。在`READY`状态下，所有数据流状态都会被重置。从`PAUSED`到`READY`再返回`PAUSED`的元素应重新从头开始读取流。
    
-   使用`gst-launch`进行测试的人往往不关心清理工作。这是错误的。应使用各种应用程序对元素进行测试，测试不仅意味着“确保它不会崩溃”，还意味着使用`valgrind` 等工具测试是否存在内存泄漏。元素在重置后必须能在管道中重复使用。
    

## 调试

-   元素绝不能使用其标准输出进行调试（使用`printf ()`或`g_print ()` 等函数）。相反，元素应使用 GStreamer 提供的日志功能，即 `GST_DEBUG（）`、`GST_LOG（）`、`GST_INFO（）`、`GST_WARNING（）`和`GST_ERROR（）`。各种日志级别可在运行时打开或关闭，因此可用于解决出现的问题。您也可以使用`GST_LOG_OBJECT()` 来代替 `GST_LOG()`（比如）打印日志输出的对象。
    
-   理想情况下，元素应使用自己的调试类别。大多数元素都使用以下代码来实现这一点：

```c
GST_DEBUG_CATEGORY_STATIC (myelement_debug);
#define GST_CAT_DEFAULT myelement_debug

[..]

static void
gst_myelement_class_init (GstMyelementClass *klass)
{
[..]
  GST_DEBUG_CATEGORY_INIT (myelement_debug, "myelement",
               0, "My own element");
}
```
在运行时，可以使用命令行选项`--gst-debug=myelement:5` 打开调试功能。
    
-   例如，在设置 pad 函数或覆盖元素类方法时，元素应使用 `GST_DEBUG_FUNCPTR`：
    
```c
gst_pad_set_event_func (myelement->srcpad,
    GST_DEBUG_FUNCPTR (my_element_src_event));
```
    
这使得调试输出更容易阅读。
    
-   旨在纳入 GStreamer 模块的元素应确保元素名称、结构和函数名称的命名一致。例如，如果元素类型是 GstYellowFooDec，函数前缀应为 `gst_yellow_foo_dec_`，元素应注册为 "yellowfoodec"。在此方案中，单独的词应该是分开的，因此应该是 GstFooDec 和 `gst_foo_dec`，而不是 `GstFoodec` 和 `gst_foodec`。
    

## 查询、事件等

-   它所适用的所有元素（源、汇、解复用器）都应在其衬底上实现查询功能，以便应用程序和邻近元素可以请求当前位置、数据流长度（如果已知）等。
    
-   元素应确保使用 `gst_pad_event_default (pad, parent, event)` 转发它们不处理的事件，而不是直接丢弃它们。除非有特别意图，否则绝不能丢弃事件。
    
-   元素应确保使用 `gst_pad_query_default (pad, parent, query)` 转发它们不需要处理的查询，而不是直接丢弃它们。
    

## 测试您的元素

-   `gst-launch`并不是一个显示您的元素已完成的好工具，Rhythmbox 和 Totem（用于 GNOME）或 AmaroK（用于 KDE）等应用程序才是。`gst-launch`无法测试重置时的正确清理、事件处理、查询等各种问题。
    
-   解析器和解码器应确保检查其输入。输入不可信。防止可能出现的缓冲区溢出等情况，并在出现无法恢复的流错误时敢于报错。使用诸如`breakmydata`（包含在 gst-plugins 中）之类的流破坏元素测试你的解多路复用器。它会随机插入、删除和修改数据流中的字节，因此是对稳健性的良好测试。如果在添加该元素时出现崩溃，说明你的元素需要修复。如果它能正常出错，那就足够好了。理想情况下，它能继续工作并尽可能多地转发数据。
    
-   解码器不应假定搜索是有效的。也要做好处理不可搜索输入流（如网络信号源）的准备。
    
-   源和汇应准备好被分配到另一个时钟，而不是他们自己暴露的时钟。始终使用提供的时钟进行同步，否则会出现 A/V 同步问题。
