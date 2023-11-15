---
title: gstreamer插件编写指南：查询函数
author: kernel
date: 2023-11-14 17:50:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

通过查询功能，您的元素将收到必须回复的查询。这些查询包括位置、持续时间，也包括元素支持的格式和调度模式。 查询既可以向上游也可以向下游发送，因此您可以在汇衬底上接收查询，也可以在源衬底上接收查询。

下面是一个非常简单的查询函数，我们将其安装在元素的源衬底上。

```c
static gboolean gst_my_filter_src_query (GstPad    *pad,
                                         GstObject *parent,
                                         GstQuery  *query);

[..]

static void
gst_my_filter_init (GstMyFilter * filter)
{
[..]
  /* configure event function on the pad before adding
   * the pad to the element */
  gst_pad_set_query_function (filter->srcpad,
      gst_my_filter_src_query);
[..]
}

static gboolean
gst_my_filter_src_query (GstPad    *pad,
                 GstObject *parent,
                 GstQuery  *query)
{
  gboolean ret;
  GstMyFilter *filter = GST_MY_FILTER (parent);

  switch (GST_QUERY_TYPE (query)) {
    case GST_QUERY_POSITION:
      /* we should report the current position */
      [...]
      break;
    case GST_QUERY_DURATION:
      /* we should report the duration here */
      [...]
      break;
    case GST_QUERY_CAPS:
      /* we should report the supported caps here */
      [...]
      break;
    default:
      /* just call the default handler */
      ret = gst_pad_query_default (pad, parent, query);
      break;
  }
  return ret;
}
```

对于未知查询，最好调用默认查询处理程序`gst_pad_query_default()`。默认处理程序会根据查询类型转发查询或直接取消查询。
