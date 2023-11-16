---
title: gstreamer插件编写指南：标签（元数据和流信息）
author: kernel
date: 2023-11-16 13:11:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

## 概述

标签是存储在数据流中的信息片段，它们不是内容本身，而是对内容的描述。大多数媒体容器格式都以这种或那种方式支持标签。Ogg 使用 VorbisComment，MP3 使用 ID3，AVI 和 WAV 使用 RIFF 的 INFO 列表块等。GStreamer 提供了一种通用方法，让元素从流中读取标签，并将其展示给用户。标签（至少是元数据）将成为流水线内流的一部分。这样做的结果是，只要输入和输出格式元素都支持标签，从一种格式转码到另一种格式的文件就会自动保留标签。

在 GStreamer 中，标签分为两类，尽管应用程序不会注意到这一点。第一类称为元数据（metadata），第二类称为流信息（streaminfo）。元数据是描述流内容非技术部分的标签。这些标签可以更改，而无需完全重新编码流内容。例如 "作者"、"标题 "或 "专辑"。不过，容器格式可能仍需重新编写，以适应这些标签。另一方面，Streaminfo 是技术上描述流内容的标签。要更改这些标签，就需要对流媒体重新编码。例如 "编解码器 "或 "比特率"。需要注意的是，某些容器格式（如 ID3）会将各种流信息标签作为元数据存储在文件容器中，这意味着它们可以被更改，从而不再与文件内容相匹配。不过，它们之所以被称为元数据，是因为从技术上讲，它们可以在不对整个流重新编码的情况下进行更改，尽管这样做会使它们失效。带有此类元数据标记的文件会有两次相同的标记：一次作为元数据，一次作为流媒体信息。

在 GStreamer 中，标签读取元素没有专门的名称。有些专门的元素（如 id3demux）除了读取标签外什么也不做，但任何 GStreamer 元素都可以在处理数据时提取标签，大多数解码器、解复用器和解析器都可以这样做。

标签写入器称为标签设置器（`TagSetter`）。支持这两种功能的元素可用于标签设置器，以快速更换标签（注意：在撰写本报告时，就地标签设置功能仍不完善，通常需要提取/剥离标签，并用新标签重新复用数据流）。

## 从流中读取标签

标签的基本对象是`GstTagList`。从数据流中读取标签的元素应创建一个空标签列表，并将单个标签填入其中。空标签列表可以用`gst_tag_list_new ()` 创建。然后，元素可以使用`gst_tag_list_add ()`或`gst_tag_list_add_values ()` 填充列表。需要注意的是，元素通常以字符串形式读取元数据，但标签列表中的值不一定是字符串——它们需要是标签注册时的类型（每个预定义标签的 API 文档都应包含该类型）。请务必使用`gst_value_transform ()`等函数来确保数据类型正确。读取数据后，可以通过 TAG 事件向下游发送标签。当 TAG 事件到达汇元素时，它会将 TAG 消息发布到管道的 GstBus 上，供应用程序接收。

目前，我们要求内核在使用标签之前知道它们的 GType，因此所有标签都必须先注册。您可以使用`gst_tag_register ()` 将新标签添加到已知标签列表中。如果您认为该标记不仅在您自己的元素中有用，在更多情况下也有用，那么将其添加到`gsttag.c`中可能是个好主意。如果你想在自己的元素中使用标签，最简单的方法是在类的初始函数中注册标签，最好是`_class_init ()` 函数。

```c
static void
gst_my_filter_class_init (GstMyFilterClass *klass)
{
[..]
  gst_tag_register ("my_tag_name", GST_TAG_FLAG_META,
            G_TYPE_STRING,
            _("my own tag"),
            _("a tag that is specific to my own element"),
            NULL);
[..]
}
```

## 写入标签到流

标签写入器与标签读取器正好相反。标签写入器只考虑元数据标签，因为这是唯一必须写入数据流的标签类型。标签写入器可以通过三种方式接收标签：内部标签、应用标签和流水线标签。内部标签是由元素本身读取的标签，这意味着标签写入器在这种情况下也是一个标签读取器。应用标签是通过标签设定器接口（这只是一个层）提供给元素的标签。流水线标签是从流水线中提供给元素的标签。元素通过`GST_EVENT_TAG`事件接收此类标签，这意味着标签写入器应实现一个事件处理程序。标签写入器负责将上述三者合并成一个列表，并写入输出流。

下面的示例将接收来自应用程序和流水线的标签，将它们合并并写入输出流。它实现了标签设置器，因此应用程序可以设置标签，并从传入事件中检索流水线标签。

> 警告：此示例已过时，不再适用于 1.0 版本的 GStreamer。

```c
GType
gst_my_filter_get_type (void)
{
[..]
    static const GInterfaceInfo tag_setter_info = {
      NULL,
      NULL,
      NULL
    };
[..]
    g_type_add_interface_static (my_filter_type,
                 GST_TYPE_TAG_SETTER,
                 &tag_setter_info);
[..]
}

static void
gst_my_filter_init (GstMyFilter *filter)
{
[..]
}

/*
 * Write one tag.
 */

static void
gst_my_filter_write_tag (const GstTagList *taglist,
             const gchar      *tagname,
             gpointer          data)
{
  GstMyFilter *filter = GST_MY_FILTER (data);
  GstBuffer *buffer;
  guint num_values = gst_tag_list_get_tag_size (list, tag_name), n;
  const GValue *from;
  GValue to = { 0 };

  g_value_init (&to, G_TYPE_STRING);

  for (n = 0; n < num_values; n++) {
    guint8 * data;
    gsize size;

    from = gst_tag_list_get_value_index (taglist, tagname, n);
    g_value_transform (from, &to);

    data = g_strdup_printf ("%s:%s", tagname,
        g_value_get_string (&to));
    size = strlen (data);

    buf = gst_buffer_new_wrapped (data, size);
    gst_pad_push (filter->srcpad, buf);
  }

  g_value_unset (&to);
}

static void
gst_my_filter_task_func (GstElement *element)
{
  GstMyFilter *filter = GST_MY_FILTER (element);
  GstTagSetter *tagsetter = GST_TAG_SETTER (element);
  GstData *data;
  GstEvent *event;
  gboolean eos = FALSE;
  GstTagList *taglist = gst_tag_list_new ();

  while (!eos) {
    data = gst_pad_pull (filter->sinkpad);

    /* We're not very much interested in data right now */
    if (GST_IS_BUFFER (data))
      gst_buffer_unref (GST_BUFFER (data));
    event = GST_EVENT (data);

    switch (GST_EVENT_TYPE (event)) {
      case GST_EVENT_TAG:
        gst_tag_list_insert (taglist, gst_event_tag_get_list (event),
                 GST_TAG_MERGE_PREPEND);
        gst_event_unref (event);
        break;
      case GST_EVENT_EOS:
        eos = TRUE;
        gst_event_unref (event);
        break;
      default:
        gst_pad_event_default (filter->sinkpad, event);
        break;
    }
  }

  /* merge tags with the ones retrieved from the application */
  if ((gst_tag_setter_get_tag_list (tagsetter)) {
    gst_tag_list_insert (taglist,
             gst_tag_setter_get_tag_list (tagsetter),
             gst_tag_setter_get_tag_merge_mode (tagsetter));
  }

  /* write tags */
  gst_tag_list_foreach (taglist, gst_my_filter_write_tag, filter);

  /* signal EOS */
  gst_pad_push (filter->srcpad, gst_event_new (GST_EVENT_EOS));
}
```

请注意，在处理标记之前，元素通常不会读取整个数据流。相反，它们会从每个 sinkpad 读取数据，直到收到数据为止（因为标签通常在第一个数据缓冲区之前收到），然后再进行处理。
