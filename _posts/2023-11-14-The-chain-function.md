---
title: gstreamer插件编写指南：链函数
author: kernel
date: 2023-11-14 17:30:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

链函数是进行所有数据处理的函数。在简单过滤器的情况下，`_chain ()`函数大多是线性函数——因此每输入一个缓冲区，也会输出一个缓冲区。下面是一个非常简单的链函数实现：

```c
static GstFlowReturn gst_my_filter_chain (GstPad    *pad,
                                          GstObject *parent,
                                          GstBuffer *buf);

[..]

static void
gst_my_filter_init (GstMyFilter * filter)
{
[..]
  /* configure chain function on the pad before adding
   * the pad to the element */
  gst_pad_set_chain_function (filter->sinkpad,
      gst_my_filter_chain);
[..]
}

static GstFlowReturn
gst_my_filter_chain (GstPad    *pad,
                     GstObject *parent,
             GstBuffer *buf)
{
  GstMyFilter *filter = GST_MY_FILTER (parent);

  if (!filter->silent)
    g_print ("Have data of size %" G_GSIZE_FORMAT" bytes!\n",
        gst_buffer_get_size (buf));

  return gst_pad_push (filter->srcpad, buf);
}
```

很明显，上述操作没有什么用处。通常情况下，你需要在缓冲区中处理数据，而不是打印已输入的数据。但请记住，缓冲区并不总是可写的。

在更高级的元素（进行事件处理的元素）中，您可能需要额外指定一个事件处理函数，在发送流事件（如衬底、流结束、新片段、标记等）时调用该函数。

```c
static void
gst_my_filter_init (GstMyFilter * filter)
{
[..]
  gst_pad_set_event_function (filter->sinkpad,
      gst_my_filter_sink_event);
[..]
}



static gboolean
gst_my_filter_sink_event (GstPad    *pad,
                  GstObject *parent,
                  GstEvent  *event)
{
  GstMyFilter *filter = GST_MY_FILTER (parent);

  switch (GST_EVENT_TYPE (event)) {
    case GST_EVENT_CAPS:
      /* we should handle the format here */
      break;
    case GST_EVENT_EOS:
      /* end-of-stream, we should close down all stream leftovers here */
      gst_my_filter_stop_processing (filter);
      break;
    default:
      break;
  }

  return gst_pad_event_default (pad, parent, event);
}

static GstFlowReturn
gst_my_filter_chain (GstPad    *pad,
             GstObject *parent,
             GstBuffer *buf)
{
  GstMyFilter *filter = GST_MY_FILTER (parent);
  GstBuffer *outbuf;

  outbuf = gst_my_filter_process_data (filter, buf);
  gst_buffer_unref (buf);
  if (!outbuf) {
    /* something went wrong - signal an error */
    GST_ELEMENT_ERROR (GST_ELEMENT (filter), STREAM, FAILED, (NULL), (NULL));
    return GST_FLOW_ERROR;
  }

  return gst_pad_push (filter->srcpad, outbuf);
}
```

在某些情况下，元素对输入数据速率的控制可能也很有用。在这种情况下，你可能需要编写一个所谓的基于循环的元素。源元素（只有源衬底）也可以是基于获取的元素。这些概念将在本指南的高级章节以及专门讨论源衬底的章节中进行解释。
