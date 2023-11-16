---
title: gstreamer插件编写指南：能力集协商
author: kernel
date: 2023-11-15 12:29:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

能力集协商是在元素之间找到一种它们可以处理的媒体格式（GstCaps）的行为。在大多数情况下，GStreamer 中的这一过程能为整个流水线找到最佳解决方案。在本节中，我们将解释其工作原理。

## 能力集协商基础知识

在 GStreamer 中，媒体格式的协商始终遵循以下简单规则：

-   下游元素在其汇衬底上建议一种格式，并将该建议置于在汇衬底上执行的 CAPS 查询结果中。 另请参阅 "执行 CAPS 查询功能"。
    
-   上游元素决定一种格式。它的源衬底通过 CAPS 事件将所选的媒体格式发送到下游。下游元素重新配置自己，以处理汇衬底上 CAPS 事件中的媒体类型。
    
-   下游元素可以通过向上游发送 RECONFIGURE 事件，通知上游它想建议一种新格式。RECONFIGURE 事件只是指示上游元素重新开始协商阶段。由于发送 RECONFIGURE 事件的元素现在建议使用另一种格式，因此流水线中的格式可能会发生变化。
    

除了 CAPS 和 RECONFIGURE 事件以及 CAPS 查询外，还有一个 `ACCEPT_CAPS` 查询，用于快速检查某个元素是否可以接受某个 CAPS。

所有协商都遵循这些简单的规则。让我们来看看一些典型的使用案例以及协商是如何进行的。

## 能力集协商使用案例

在下文中，我们将介绍推模式调度的一些用例。 拉模式调度的协商阶段在"拉模式能力集协商"中讨论过，实际上与我们将要看到的类似。

由于汇衬底只建议格式，而源衬底需要做出决定，因此最复杂的工作由源衬底完成。我们可以为源衬底确定 3 种能力集协商使用案例：

-   固定协商。一个元素只能输出一种格式。请参阅固定协商。
    
-   变换协商。元素的输入和输出格式之间有一个（固定的）转换，通常基于某些元素属性。元素产生的能力集取决于上游能力集，元素能接受的能力集取决于下游能力集。请参阅变换协商。
    
-   动态协商。一个元素可以输出多种格式。请参阅动态协商。
    

### 固定协商

在这种情况下，源衬底只能生成一种固定格式。通常这种格式是在媒体内部编码的。任何下游元素都不能要求不同的格式，源衬底重新协商的唯一方式就是元素自己决定更改能力集。

一般来说，可以（在源衬底上）执行固定能力集的元素是所有不可重新协商的元素。例如：

-   类型查找器，因为找到的类型是实际数据流的一部分，因此无法重新协商。类型查找器将查看字节流，找出类型，发送带大写字母的 CAPS 事件，然后推送该类型的缓冲区。
    
-   几乎所有的解多路复用器，因为所包含的基本数据流是在文件头中定义的，因此不可重新协商。
    
-   有些解码器的格式是嵌入在数据流中的，而不是对应解码器的一部分，解码器本身也不可重新配置。
    
-   生成固定格式的一些数据源。
    

`gst_pad_use_fixed_caps()`用于使用固定能力集的源衬底。只要 pad 未协商，默认 CAPS 查询就会返回 padtemplate 中显示的能力集。一旦 PAD 经过协商，CAPS 查询将返回协商后的能力集（而不是其他）。这些是固定能力集源衬底的相关代码片段。

```c
[..]
  pad = gst_pad_new_from_static_template (..);
  gst_pad_use_fixed_caps (pad);
[..]
```

然后，可以通过调用`gst_pad_set_caps ()` 在衬底上设置固定能力集。

```c
[..]
    caps = gst_caps_new_simple ("audio/x-raw",
        "format", G_TYPE_STRING, GST_AUDIO_NE(F32),
        "rate", G_TYPE_INT, <samplerate>,
        "channels", G_TYPE_INT, <num-channels>, NULL);
    if (!gst_pad_set_caps (pad, caps)) {
      GST_ELEMENT_ERROR (element, CORE, NEGOTIATION, (NULL),
          ("Some debug information here"));
      return GST_FLOW_ERROR;
    }
[..]
```

这类元素在输入格式和输出格式之间也没有关系，输入能力集根本不包含生成输出能力集所需的信息。

需要为格式配置的所有其他元素都应执行完整的能力集协商，这将在接下来的几节中进行说明。

### 变换协商

在这种协商技术中，元素输入能力集和输出能力集之间有一个固定的转换。这种变换可以由元素属性参数化，但不能由流的内容参数化（请参阅固定协商的用例）。

元素可接受的能力集取决于下游能力集（固定转换）。元素能生产的能力集取决于上游能力集（固定变换）。

当接收到 CAPS 事件时，这类元素通常可以通过`_event()`函数在其源衬底上设置能力集。 这意味着能力集转换函数可以将一个固定能力集转换为另一个固定能力集。这类元素的例子包括：

-   视频框。它可根据对象属性在视频框周围添加可配置的边框。
    
-   身份元素。所有不改变数据格式，只改变内容的元素。视频和音频效果就是一个例子，其他例子还包括检查数据流的元素。
    
-   一些解码器和编码器，其输出格式由输入格式定义，如 mulawdec 和 mulawenc。这些解码器通常没有定义数据流内容的标头。它们通常更像是转换元素。
    

下面是一个典型转换元素的协商步骤示例。在汇衬底 CAPS 事件处理程序中，我们计算源衬底的能力集并进行设置。

```c
[...]

static gboolean
gst_my_filter_setcaps (GstMyFilter *filter,
               GstCaps *caps)
{
  GstStructure *structure;
  int rate, channels;
  gboolean ret;
  GstCaps *outcaps;

  structure = gst_caps_get_structure (caps, 0);
  ret = gst_structure_get_int (structure, "rate", &rate);
  ret = ret && gst_structure_get_int (structure, "channels", &channels);
  if (!ret)
    return FALSE;

  outcaps = gst_caps_new_simple ("audio/x-raw",
      "format", G_TYPE_STRING, GST_AUDIO_NE(S16),
      "rate", G_TYPE_INT, rate,
      "channels", G_TYPE_INT, channels, NULL);
  ret = gst_pad_set_caps (filter->srcpad, outcaps);
  gst_caps_unref (outcaps);

  return ret;
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
    {
      GstCaps *caps;

      gst_event_parse_caps (event, &caps);
      ret = gst_my_filter_setcaps (filter, caps);
      break;
    }
    default:
      ret = gst_pad_event_default (pad, parent, event);
      break;
  }
  return ret;
}

[...]
```

### 动态协商

最后一种协商方法是最复杂、最强大的动态协商。

与变换协商中的变换协商一样，动态协商将对下游/上游上限执行转换。与转换协商不同的是，这种转换会将固定能力集转换为非固定能力集。 这意味着汇衬底输入能力集可转换为非固定（多种）格式。源衬底必须从所有可能的格式中选择一种。源衬底通常希望选择一种最省力的格式，但并非必须如此。格式的选择也应取决于下游可接受的能力集（参见实现能力集查询功能中的`QUERY_CAPS`功能）。

典型的流程是这样的：

-   能力集位于元素的汇衬底上。
    
-   如果元素希望以直通模式运行，则通过 `ACCEPT_CAPS` 查询检查下游是否接受能力集。如果接受，我们就可以完成协商，并以直通模式运行。
    
-   计算源衬底可能的能力。
    
-   查询下游对对应衬底，以获取可能的能力集列表。
    
-   从下游列表中选择您可以转换到的第一个能力集，并将其设置为输出能力集。您可能需要将能力集固定为一些合理的默认值，以构建固定的能力集。
    

这类元素的例子包括：

-   转换器元素，如视频转换、音频转换、音频采样、视频缩放、...
    
-   源元素，如 audiotestsrc、videotestsrc、v4l2src、pulssesrc...

让我们以一个可以在采样率之间进行转换的元素为例，这样输入和输出的采样率就不必相同：

```c
static gboolean
gst_my_filter_setcaps (GstMyFilter *filter,
               GstCaps *caps)
{
  if (gst_pad_set_caps (filter->srcpad, caps)) {
    filter->passthrough = TRUE;
  } else {
    GstCaps *othercaps, *newcaps;
    GstStructure *s = gst_caps_get_structure (caps, 0), *others;

    /* no passthrough, setup internal conversion */
    gst_structure_get_int (s, "channels", &filter->channels);
    othercaps = gst_pad_get_allowed_caps (filter->srcpad);
    others = gst_caps_get_structure (othercaps, 0);
    gst_structure_set (others,
      "channels", G_TYPE_INT, filter->channels, NULL);

    /* now, the samplerate value can optionally have multiple values, so
     * we "fixate" it, which means that one fixed value is chosen */
    newcaps = gst_caps_copy_nth (othercaps, 0);
    gst_caps_unref (othercaps);
    gst_pad_fixate_caps (filter->srcpad, newcaps);
    if (!gst_pad_set_caps (filter->srcpad, newcaps))
      return FALSE;

    /* we are now set up, configure internally */
    filter->passthrough = FALSE;
    gst_structure_get_int (s, "rate", &filter->from_samplerate);
    others = gst_caps_get_structure (newcaps, 0);
    gst_structure_get_int (others, "rate", &filter->to_samplerate);
  }

  return TRUE;
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
    {
      GstCaps *caps;

      gst_event_parse_caps (event, &caps);
      ret = gst_my_filter_setcaps (filter, caps);
      break;
    }
    default:
      ret = gst_pad_event_default (pad, parent, event);
      break;
  }
  return ret;
}

static GstFlowReturn
gst_my_filter_chain (GstPad    *pad,
             GstObject *parent,
             GstBuffer *buf)
{
  GstMyFilter *filter = GST_MY_FILTER (parent);
  GstBuffer *out;

  /* push on if in passthrough mode */
  if (filter->passthrough)
    return gst_pad_push (filter->srcpad, buf);

  /* convert, push */
  out = gst_my_filter_convert (filter, buf);
  gst_buffer_unref (buf);

  return gst_pad_push (filter->srcpad, out);
}
```

## 上游能力集（重新）协商

上游协商的主要用途是将已协商好的流水线（部分）重新协商为新格式。一些实际例子包括选择不同的视频尺寸，因为视频窗口的尺寸发生了变化，而视频输出本身不能重新缩放，或者因为音频通道配置发生了变化。

通过向上游发送 `GST_EVENT_RECONFIGURE` 事件来请求上游能力集重新协商。这样做的目的是指示上游元素重新配置其能力集，方法是对允许的能力集进行新的查询，然后选择新的能力集。发送 RECONFIGURE 事件的元素将通过其 `GST_QUERY_CAPS` 查询功能返回新的首选能力集，从而影响新能力集的选择。RECONFIGURE 事件将在其经过的所有 PAD 上设置 `GST_PAD_FLAG_NEED_RECONFIGURE`。

这里需要注意的是，不同的元素在这里实际上承担着不同的责任：

-   想要向上游提出新格式的要素需要首先通过` ACCEPT_CAPS` 查询检查上游是否接受新的能力集。然后发送 RECONFIGURE 事件，并准备好用新的首选格式回答 CAPS 查询。需要注意的是，如果没有上游元素能够（或愿意）重新协商，元素就需要使用当前配置的格式。
    
-   根据“转换协商”（Transform negotiation）进行转换协商的元素会向上游发送 RECONFIGURE 事件。由于这些元素只是根据上游能力集进行固定变换，因此它们需要向上游发送事件，以便选择新格式。
    
-   在固定协商（Fixednegotiation）模式下运行的元素会放弃 RECONFIGURE 事件。这些元素无法重新配置，其输出能力集也不依赖于上游能力集，因此可以放弃该事件。
    
-   可以在源衬底上重新配置的元素（在动态协商中执行动态协商的源衬底）应使用`gst_pad_check_reconfigure ()`检查其 `NEED_RECONFIGURE` 标志，并在函数返回 TRUE 时开始重新协商。

## 实现 CAPS 查询功能

当一个配对元素想知道该 PAD 支持哪些格式以及优先顺序时，就会调用 `GST_QUERY_CAPS` 查询类型的`_query ()` 函数。返回值应是该元素支持的所有格式，并考虑到更下游或上游配对元素的限制，按优先级排序，优先级最高者优先。

```c
static gboolean
gst_my_filter_query (GstPad *pad, GstObject * parent, GstQuery * query)
{
  gboolean ret;
  GstMyFilter *filter = GST_MY_FILTER (parent);

  switch (GST_QUERY_TYPE (query)) {
    case GST_QUERY_CAPS
    {
      GstPad *otherpad;
      GstCaps *temp, *caps, *filt, *tcaps;
      gint i;

      otherpad = (pad == filter->srcpad) ? filter->sinkpad :
                                           filter->srcpad;
      caps = gst_pad_get_allowed_caps (otherpad);

      gst_query_parse_caps (query, &filt);

      /* We support *any* samplerate, indifferent from the samplerate
       * supported by the linked elements on both sides. */
      for (i = 0; i < gst_caps_get_size (caps); i++) {
        GstStructure *structure = gst_caps_get_structure (caps, i);

        gst_structure_remove_field (structure, "rate");
      }

      /* make sure we only return results that intersect our
       * padtemplate */
      tcaps = gst_pad_get_pad_template_caps (pad);
      if (tcaps) {
        temp = gst_caps_intersect (caps, tcaps);
        gst_caps_unref (caps);
        gst_caps_unref (tcaps);
        caps = temp;
      }
      /* filter against the query filter when needed */
      if (filt) {
        temp = gst_caps_intersect (caps, filt);
        gst_caps_unref (caps);
        caps = temp;
      }
      gst_query_set_caps_result (query, caps);
      gst_caps_unref (caps);
      ret = TRUE;
      break;
    }
    default:
      ret = gst_pad_query_default (pad, parent, query);
      break;
  }
  return ret;
}
```

## 拉模式能力集协商

待完善：拉模式协商机制尚未完全明了。

如果有疑问，请查看我们 git 仓库中的其他同类型元素，了解它们是如何完成你想做的事情的。