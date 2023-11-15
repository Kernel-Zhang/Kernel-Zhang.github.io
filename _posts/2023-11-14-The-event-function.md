---
title: gstreamer插件编写指南：事件函数
author: kernel
date: 2023-11-14 17:40:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

事件功能可通知您数据流中发生的特殊事件（如衬底、流结束、新段、标记等）。事件既可在上游也可在下游传播，因此您既可以在汇衬底上接收事件，也可以在源衬底上接收事件。

下面是一个非常简单的事件函数，我们将其安装在元素的汇衬底上。

```c
static gboolean gst_my_filter_sink_event (GstPad    *pad,
                                          GstObject *parent,
                                          GstEvent  *event);

[..]

static void
gst_my_filter_init (GstMyFilter * filter)
{
[..]
  /* configure event function on the pad before adding
   * the pad to the element */
  gst_pad_set_event_function (filter->sinkpad,
      gst_my_filter_sink_event);
[..]
}

static gboolean
gst_my_filter_sink_event (GstPad    *pad,
                  GstObject *parent,
                  GstEvent  *event)
{
  gboolean ret;
  GstMyFilter *filter = GST_MY_FILTER (parent);

  switch (GST_EVENT_TYPE (event)) {
    case GST_EVENT_CAPS:
      /* we should handle the format here */

      /* push the event downstream */
      ret = gst_pad_push_event (filter->srcpad, event);
      break;
    case GST_EVENT_EOS:
      /* end-of-stream, we should close down all stream leftovers here */
      gst_my_filter_stop_processing (filter);

      ret = gst_pad_event_default (pad, parent, event);
      break;
    default:
      /* just call the default handler */
      ret = gst_pad_event_default (pad, parent, event);
      break;
  }
  return ret;
}
```

对于未知事件，调用默认事件处理程序`gst_pad_event_default ()`是个好主意。根据事件类型，默认处理程序将转发事件或直接取消引用。CAPS 事件默认不转发，因此我们需要在事件处理程序中自行处理。
