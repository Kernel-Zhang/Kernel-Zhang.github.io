---
title: gstreamer插件编写指南：什么是状态
author: kernel
date: 2023-11-14 17:53:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

状态描述了元素实例是否初始化、是否准备好传输数据以及当前是否正在处理数据。 GStreamer 中定义了四种状态：

-   `GST_STATE_NULL`
    
-   `GST_STATE_READY`
    
-   `GST_STATE_PAUSED`
    
-   `GST_STATE_PLAYING`
    

从现在起，它们将被简称为 "NULL"、"READY"、"PAUSED "和 "PLAYING"。

`GST_STATE_NULL`是元素的默认状态。在这种状态下，它没有分配任何运行时资源，没有加载任何运行时库，显然也不能处理数据。

`GST_STATE_READY`是元素的下一个状态。在 "就绪 "状态下，元素已分配了所有默认资源（运行时库、运行时内存）。但是，它还没有分配或定义任何特定于流的资源。当从 NULL 状态转为 READY 状态（`GST_STATE_CHANGE_NULL_TO_READY`）时，元素应分配所有非流专用资源，并加载运行时可加载库（如果有）。反过来（从 READY 到 NULL，`GST_STATE_CHANGE_READY_TO_NULL`），元素应卸载这些库并释放所有已分配的资源。此类资源的例子包括硬件设备。请注意，文件通常是流，因此这些资源应视为特定于流的资源；因此，不应在此状态下分配这些资源。

`GST_STATE_PAUSED`是元素准备好接受和处理数据的状态。对于大多数元素来说，这种状态与 PLAYING 相同，唯一的例外是 sink 元素。sink 元素只接受一个数据缓冲区，然后阻塞。此时流水线已“预滚动（prerolled）”并准备好立即渲染数据。

`GST_STATE_PLAYING`是元素可以处于的最高状态。对于大多数元素来说，这种状态与 PAUSED（暂停）完全相同，它们接受并处理事件和数据缓冲区。只有汇元素需要区分 PAUSED 状态和 PLAYING 状态。在播放状态下，汇元素会实际呈现传入的数据，例如向声卡输出音频或向图像汇呈现视频图像。

## 管理过滤器状态

如果可能的话，您的元素应来源于一个新的基类（预制基类）。有一些现成的通用基类可用于不同类型的信号源、信号汇和滤波器/变换元素。除此之外，还有用于音频和视频元素等的专用基类。

如果使用基类，就很少需要自己处理状态变化。你只需覆盖基类的 start() 和 stop() 虚拟函数（根据基类的不同，调用方式也可能不同），基类就会为你处理好一切。

但是，如果您不是从现成的基类派生，而是从 GstElement 或其他非建立在基类之上的类派生，那么您很可能需要实现自己的状态变化函数来通知状态变化。如果您的插件是一个解多路复用器（demuxer）或一个多路复用器（muxer），这无疑是必要的，因为目前还没有多路复用器（muxer）或解多路复用器（demuxer）的基类。

元素可以通过一个虚拟函数指针通知状态变化。在该函数中，元素可以初始化元素所需的任何特定数据，也可能从一种状态转入另一种状态失败。

对于未处理的状态变化，请勿使用 `g_assert`；`GstElement` 基类会处理这些变化。

```c
static GstStateChangeReturn
gst_my_filter_change_state (GstElement *element, GstStateChange transition);

static void
gst_my_filter_class_init (GstMyFilterClass *klass)
{
  GstElementClass *element_class = GST_ELEMENT_CLASS (klass);

  element_class->change_state = gst_my_filter_change_state;
}



static GstStateChangeReturn
gst_my_filter_change_state (GstElement *element, GstStateChange transition)
{
  GstStateChangeReturn ret = GST_STATE_CHANGE_SUCCESS;
  GstMyFilter *filter = GST_MY_FILTER (element);

  switch (transition) {
	case GST_STATE_CHANGE_NULL_TO_READY:
	  if (!gst_my_filter_allocate_memory (filter))
		return GST_STATE_CHANGE_FAILURE;
	  break;
	default:
	  break;
  }

  ret = GST_ELEMENT_CLASS (parent_class)->change_state (element, transition);
  if (ret == GST_STATE_CHANGE_FAILURE)
	return ret;

  switch (transition) {
	case GST_STATE_CHANGE_READY_TO_NULL:
	  gst_my_filter_free_memory (filter);
	  break;
	default:
	  break;
  }

  return ret;
}
```

请注意，向上（NULL=>READY，READY=>PAUSED，PAUSED=>PLAYING）和向下（PLAYING=>PAUSED，PAUSED=>READY，READY=>NULL）的状态变化是在两个独立的块中处理的，向下的状态变化只有在我们链入父类的状态变化函数后才会处理。为了安全地处理多个线程的并发访问，这样做是必要的。

这样做的原因是，在状态向下变化的情况下，当插件的链函数（例如）仍在另一个线程中访问这些资源时，您不希望破坏已分配的资源。链函数是否运行取决于插件的衬底状态，而衬底状态与元素状态密切相关。衬底状态由 `GstElement` 类的状态更改函数处理，包括适当的锁定，这就是为什么在销毁已分配资源之前必须进行链式处理的原因。

