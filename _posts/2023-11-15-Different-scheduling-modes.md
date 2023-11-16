---
title: gstreamer插件编写指南：不同的调度模式
author: kernel
date: 2023-11-15 10:29:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

衬底的调度模式定义了从衬底（源）获取数据或向衬底（汇）提供数据的方式。GStreamer 可以在两种调度模式下运行，即推模式和拉模式。GStreamer 支持任何调度模式下的衬底元素，但并非所有衬底都必须在同一模式下运行。

到目前为止，我们只讨论了`_chain()` 操作的元素，即在其汇衬底（sink pad）上设置了链功能，并在其源衬底上推缓冲区的元素。我们称之为推送模式，是因为对等元素会在源衬底上使用`gst_pad_push()`，这将导致我们的`_chain()`函数被调用，进而导致我们的元素在源衬底上推送缓冲区。启动数据流的主动权发生在上游某个地方，当它推送缓冲区时，所有下游元素都会在其`_chain ()` 函数被调用时得到调度。

在解释拉模式调度之前，我们先来了解一下如何在 pad 上选择和激活不同的调度模式。

## 衬底启动阶段

在 READY（准备）->PAUSED（已启动）的元件状态变化期间，元件的衬底将被激活。这首先发生在元素的源衬底上，然后是汇衬底。GStreamer 调用衬底的`_activate()`函数。默认情况下，该函数将通过`GST_PAD_MODE_PUSH`参数调用`gst_pad_activate_mode()`函数，以推模式激活衬底。可以覆盖衬底的`_activate()`，并决定不同的调度模式。通过重写`_activate_mode ()` 函数，可以知道衬底在哪种调度模式下被激活。

GStreamer 允许元素的不同衬底以不同的调度模式运行。这就允许了许多不同的可能用例。以下是一些典型用例的概述。

-   如果元素的所有衬底都在推送模式调度中被激活，则整个元素都在推送模式下运行。对于源元素来说，这意味着它们必须启动一项任务，将源衬底上的缓冲区推送到下游元素。下游元素的数据将由上游元素使用汇衬底的`_chain ()`函数推送给它们，该函数将推送源衬底上的缓冲区。使用这种调度模式的前提条件是，使用`gst_pad_set_chain_function ()`为每个汇衬底设置了链函数，并且所有下游元素都以相同模式运行。
    
-   另外，汇衬底也可以通过在拉模式下工作成为流水线的驱动力，而元素的源衬底仍在推模式下工作。为了成为驱动力，这些衬底在激活时会启动一个`GstTask`。该任务是一个线程，将调用元素指定的函数。被调用后，通过该函数汇衬底将可以随机访问（通过`gst_pad_pull_range()`）所有的数据，并可以向源衬底推送数据，这实际上意味着该元素控制了流水线中的数据流。这种模式的前提条件是所有下游元素都能以推模式运行，并且所有上游元素都以拉模式运行（见下文）。
    
    当源衬底从 `GST_QUERY_SCHEDULING` 查询返回 `GST_PAD_MODE_PULL` 时，下游元素可以 PULL 模式激活源衬底。这种调度模式的前提条件是使用`gst_pad_set_getrange_function ()` 为源衬底设置了`getrange-function`。
    
-   最后，一个元素中的所有衬底都可以在 PULL 模式下激活，但与上述情况相反，这并不意味着它们会自行启动任务。相反，这意味着它们是下游元素的被动拉设备，必须通过它们的`_get_range ()`函数提供随机数据访问。要求使用函数`gst_pad_set_getrange_function()`在该衬底上设置`_get_range()`函数。此外，如果该元素有任何拉衬底，所有这些衬底（以及它们的同级衬底）也需要在 PULL 访问模式下运行。
    
    当 sink 元素在 PULL（拉）模式下被激活时，它应在其 sinkpad 上启动一个调用`gst_pad_pull_range()`的任务。只有当上游 SCHEDULING 查询返回支持 `GST_PAD_MODE_PULL` 调度模式时，它才能这样做。
    

在接下来的两节中，我们将深入探讨拉模式调度（驱动流水线的元素/衬底和提供随机存取的元素/衬底），并给出一些具体的使用案例。

## 驱动流水线的衬底

以拉模式运行的衬底，与以推模式运行的源衬底（或当它是一个汇元素时没有源衬底），可以启动一个任务来驱动流水线数据流。在这个任务函数中，你可以随机访问所有汇衬底，并通过源衬底推送数据。这对几种不同的元素都很有用：

-   解复用器、解析器和某些未解析数据的解码器（如 MPEG 音频或视频流），因为这些设备更倾向于从其输入中进行字节精确（随机）访问。不过，如果可能的话，这些元件也应准备在推送模式下运行。
    
-   某些音频输出需要对输入数据流进行控制，如 Jack 声音服务器。
    

首先需要执行 SCHEDULING 查询，检查上游元件是否支持拉模式调度。如果支持，则可以拉动模式激活 sinkpad。然后在 `activate_mode` 函数中启动任务。

```c
#include "filter.h"
#include <string.h>

static gboolean gst_my_filter_activate      (GstPad      * pad,
                                             GstObject   * parent);
static gboolean gst_my_filter_activate_mode (GstPad      * pad,
                                             GstObject   * parent,
                                             GstPadMode    mode,
                         gboolean      active);
static void gst_my_filter_loop      (GstMyFilter * filter);

G_DEFINE_TYPE (GstMyFilter, gst_my_filter, GST_TYPE_ELEMENT);
GST_ELEMENT_REGISTER_DEFINE(my_filter, "my-filter", GST_RANK_NONE, GST_TYPE_MY_FILTER);

static void
gst_my_filter_init (GstMyFilter * filter)
{

[..]

  gst_pad_set_activate_function (filter->sinkpad, gst_my_filter_activate);
  gst_pad_set_activatemode_function (filter->sinkpad,
      gst_my_filter_activate_mode);


[..]
}

[..]

static gboolean
gst_my_filter_activate (GstPad * pad, GstObject * parent)
{
  GstQuery *query;
  gboolean pull_mode;

  /* first check what upstream scheduling is supported */
  query = gst_query_new_scheduling ();

  if (!gst_pad_peer_query (pad, query)) {
    gst_query_unref (query);
    goto activate_push;
  }

  /* see if pull-mode is supported */
  pull_mode = gst_query_has_scheduling_mode_with_flags (query,
      GST_PAD_MODE_PULL, GST_SCHEDULING_FLAG_SEEKABLE);
  gst_query_unref (query);

  if (!pull_mode)
    goto activate_push;

  /* now we can activate in pull-mode. GStreamer will also
   * activate the upstream peer in pull-mode */
  return gst_pad_activate_mode (pad, GST_PAD_MODE_PULL, TRUE);

activate_push:
  {
    /* something not right, we fallback to push-mode */
    return gst_pad_activate_mode (pad, GST_PAD_MODE_PUSH, TRUE);
  }
}

static gboolean
gst_my_filter_activate_pull (GstPad    * pad,
                 GstObject * parent,
                 GstPadMode  mode,
                 gboolean    active)
{
  gboolean res;
  GstMyFilter *filter = GST_MY_FILTER (parent);

  switch (mode) {
    case GST_PAD_MODE_PUSH:
      res = TRUE;
      break;
    case GST_PAD_MODE_PULL:
      if (active) {
        filter->offset = 0;
        res = gst_pad_start_task (pad,
            (GstTaskFunction) gst_my_filter_loop, filter, NULL);
      } else {
        res = gst_pad_stop_task (pad);
      }
      break;
    default:
      /* unknown scheduling mode */
      res = FALSE;
      break;
  }
  return res;
}
```

任务一旦启动，就可以完全控制输入和输出。最简单的任务函数就是读取输入并将其推送到源衬底上。这并不十分有用，但比我们迄今为止研究的老式推模式更具灵活性。

```c
#define BLOCKSIZE 2048

static void
gst_my_filter_loop (GstMyFilter * filter)
{
    GstFlowReturn ret;
    guint64 len;
    GstBuffer *buf = NULL;

    if (!gst_pad_query_duration (filter->sinkpad, GST_FORMAT_BYTES, &len)) {
    GST_DEBUG_OBJECT (filter, "failed to query duration, pausing");
    goto stop;
    }

    if (filter->offset >= len) {
    GST_DEBUG_OBJECT (filter, "at end of input, sending EOS, pausing");
    gst_pad_push_event (filter->srcpad, gst_event_new_eos ());
    goto stop;
    }

    /* now, read BLOCKSIZE bytes from byte offset filter->offset */
    ret = gst_pad_pull_range (filter->sinkpad, filter->offset,
        BLOCKSIZE, &buf);

    if (ret != GST_FLOW_OK) {
    GST_DEBUG_OBJECT (filter, "pull_range failed: %s", gst_flow_get_name (ret));
    goto stop;
    }

    /* now push buffer downstream */
    ret = gst_pad_push (filter->srcpad, buf);

    buf = NULL; /* gst_pad_push() took ownership of buffer */

    if (ret != GST_FLOW_OK) {
    GST_DEBUG_OBJECT (filter, "pad_push failed: %s", gst_flow_get_name (ret));
    goto stop;
    }

    /* everything is fine, increase offset and wait for us to be called again */
    filter->offset += BLOCKSIZE;
    return;

stop:
    GST_DEBUG_OBJECT (filter, "pausing task");
    gst_pad_pause_task (filter->sinkpad);
}
```

## 提供随机访问

在上一节中，我们谈到了使用自己的任务来驱动流水线的元素（或衬底）必须在其汇衬底上使用拉模式调度。这意味着与这些衬底链接的所有衬底都需要以拉模式激活。以拉模式激活的源衬底必须使用`gst_pad_set_getrange_function()` 实现`_get_range()`函数，当配对的衬底使用`gst_pad_pull_range ()` 请求某些数据时，该函数将被调用。然后，该元素负责寻找正确的偏移量，并提供所请求的数据。有几个元素可以实现随机存取：

-   数据源，如文件源，可从任何偏移量以合理的低延迟提供数据。
    
-   希望在整个流水线中提供拉模式调度的过滤器。
    
-   有些解析器可以通过跳过一小部分输入来轻松实现这一功能，因此基本上是“转发”getrange 请求，而无需进行任何处理。这方面的例子包括标签阅读器（如 ID3）或单一输出解析器（如 WAVE 解析器）。
    

下面的示例将展示如何在源元素中实现`_get_range ()` 函数：

```c
#include "filter.h"
static GstFlowReturn
        gst_my_filter_get_range (GstPad     * pad,
                     GstObject  * parent,
                     guint64      offset,
                     guint        length,
                     GstBuffer ** buf);

G_DEFINE_TYPE (GstMyFilter, gst_my_filter, GST_TYPE_ELEMENT);
GST_ELEMENT_REGISTER_DEFINE(my_filter, "my-filter", GST_RANK_NONE, GST_TYPE_MY_FILTER);


static void
gst_my_filter_init (GstMyFilter * filter)
{

[..]

  gst_pad_set_getrange_function (filter->srcpad,
      gst_my_filter_get_range);

[..]
}

static GstFlowReturn
gst_my_filter_get_range (GstPad     * pad,
             GstObject  * parent,
             guint64      offset,
             guint        length,
             GstBuffer ** buf)
{

  GstMyFilter *filter = GST_MY_FILTER (parent);

  [.. here, you would fill *buf ..]

  return GST_FLOW_OK;
}
```

在实践中，许多理论上可以进行随机存取的元素，由于没有下游元素可以启动自己的任务，因此实际上可能经常在推模式调度中被激活。因此，在实践中，这些元素应同时实现`_get_range ()` 函数和`_chain ()` 函数（针对过滤器和解析器），或实现`_get_range ()` 函数，并准备通过提供`_activate* ()` 函数（针对源元素）来启动自己的任务。
