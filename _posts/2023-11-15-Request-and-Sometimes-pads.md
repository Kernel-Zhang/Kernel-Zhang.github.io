---
title: gstreamer插件编写指南：请求型和暂态型衬底
author: kernel
date: 2023-11-15 10:09:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

到目前为止，我们只讨论了始终可用的衬底，但也有一些衬底仅在某些情况下创建，或者仅在应用程序请求衬底时创建。前者称为“暂态型”，后者称为“请求型”。在 pad 的模板中可以看到 pad 的可用性（始终、暂态或请求）。本章将讨论这两种垫何时有用、如何创建以及何时废弃。

## 暂态型衬底

暂态型衬底是在特定条件下创建的衬底，但并非在所有情况下都会创建。这主要取决于流的内容：解流器通常会解析流头，决定系统流中嵌入了哪些基本流（视频、音频、字幕等），然后为每个基本流创建一个暂态型pad。根据自己的选择，它还可以为每个元素实例创建一个以上的实例。唯一的限制是，每个新创建的 pad 都必须有一个唯一的名称。有时，在处理流数据时（即从 PAUSED 状态转为 READY 状态时）也会处理衬底。不应在 EOS 时处理焊衬底，因为有人可能会重新激活流水线，并搜索到流结束点之前。在 EOS 之后，流仍应保持有效，至少在流数据被处理之前是这样。在任何情况下，元素始终是该 pad 的所有者。

下面的示例代码将解析一个文本文件，其中第一行是一个数字（n）。接下来的行都以一个数字（0 至 n-1）开头，这个数字就是要发送数据的源衬底的编号。

```
3
0: foo
1: bar
0: boo
2: bye
```

解析该文件并创建动态暂态型衬底的代码如下所示：

```c
typedef struct _GstMyFilter {
[..]
  gboolean firstrun;
  GList *srcpadlist;
} GstMyFilter;

static GstStaticPadTemplate src_factory =
GST_STATIC_PAD_TEMPLATE (
  "src_%u",
  GST_PAD_SRC,
  GST_PAD_SOMETIMES,
  GST_STATIC_CAPS ("ANY")
);

static void
gst_my_filter_class_init (GstMyFilterClass *klass)
{
  GstElementClass *element_class = GST_ELEMENT_CLASS (klass);
[..]
  gst_element_class_add_pad_template (element_class,
    gst_static_pad_template_get (&src_factory));
[..]
}

static void
gst_my_filter_init (GstMyFilter *filter)
{
[..]
  filter->firstrun = TRUE;
  filter->srcpadlist = NULL;
}

/*
 * Get one line of data - without newline.
 */

static GstBuffer *
gst_my_filter_getline (GstMyFilter *filter)
{
  guint8 *data;
  gint n, num;

  /* max. line length is 512 characters - for safety */
  for (n = 0; n < 512; n++) {
    num = gst_bytestream_peek_bytes (filter->bs, &data, n + 1);
    if (num != n + 1)
      return NULL;

    /* newline? */
    if (data[n] == '\n') {
      GstBuffer *buf = gst_buffer_new_allocate (NULL, n + 1, NULL);

      gst_bytestream_peek_bytes (filter->bs, &data, n);
      gst_buffer_fill (buf, 0, data, n);
      gst_buffer_memset (buf, n, '\0', 1);
      gst_bytestream_flush_fast (filter->bs, n + 1);

      return buf;
    }
  }
}

static void
gst_my_filter_loopfunc (GstElement *element)
{
  GstMyFilter *filter = GST_MY_FILTER (element);
  GstBuffer *buf;
  GstPad *pad;
  GstMapInfo map;
  gint num, n;

  /* parse header */
  if (filter->firstrun) {
    gchar *padname;
    guint8 id;

    if (!(buf = gst_my_filter_getline (filter))) {
      gst_element_error (element, STREAM, READ, (NULL),
             ("Stream contains no header"));
      return;
    }
    gst_buffer_extract (buf, 0, &id, 1);
    num = atoi (id);
    gst_buffer_unref (buf);

    /* for each of the streams, create a pad */
    for (n = 0; n < num; n++) {
      padname = g_strdup_printf ("src_%u", n);
      pad = gst_pad_new_from_static_template (src_factory, padname);
      g_free (padname);

      /* here, you would set _event () and _query () functions */

      /* need to activate the pad before adding */
      gst_pad_set_active (pad, TRUE);

      gst_element_add_pad (element, pad);
      filter->srcpadlist = g_list_append (filter->srcpadlist, pad);
    }
  }

  /* and now, simply parse each line and push over */
  if (!(buf = gst_my_filter_getline (filter))) {
    GstEvent *event = gst_event_new (GST_EVENT_EOS);
    GList *padlist;

    for (padlist = srcpadlist;
         padlist != NULL; padlist = g_list_next (padlist)) {
      pad = GST_PAD (padlist->data);
      gst_pad_push_event (pad, gst_event_ref (event));
    }
    gst_event_unref (event);
    /* pause the task here */
    return;
  }

  /* parse stream number and go beyond the ':' in the data */
  gst_buffer_map (buf, &map, GST_MAP_READ);
  num = atoi (map.data[0]);
  if (num >= 0 && num < g_list_length (filter->srcpadlist)) {
    pad = GST_PAD (g_list_nth_data (filter->srcpadlist, num);

    /* magic buffer parsing foo */
    for (n = 0; map.data[n] != ':' &&
                map.data[n] != '\0'; n++) ;
    if (map.data[n] != '\0') {
      GstBuffer *sub;

      /* create region copy that starts right past the space. The reason
       * that we don't just forward the data pointer is because the
       * pointer is no longer the start of an allocated block of memory,
       * but just a pointer to a position somewhere in the middle of it.
       * That cannot be freed upon disposal, so we'd either crash or have
       * a memleak. Creating a region copy is a simple way to solve that. */
      sub = gst_buffer_copy_region (buf, GST_BUFFER_COPY_ALL,
          n + 1, map.size - n - 1);
      gst_pad_push (pad, sub);
    }
  }
  gst_buffer_unmap (buf, &map);
  gst_buffer_unref (buf);
}
```

请注意，我们在各处都使用了大量检查，以确保文件内容有效。这样做有两个目的：首先，文件可能是错误的，在这种情况下，我们要防止崩溃。第二个也是最重要的原因是——在极端情况下，文件可能被恶意使用，导致插件出现未定义的行为，从而引发安全问题。请务必假定文件可能被用来做坏事。

## 请求型衬底

请求型衬底与暂态型衬底类似，只是请求型是根据元素外部而非元素内部的要求创建的。这一概念常用于多路复用器中，在多路复用器中，每个要放入输出系统流的基本数据流都会请求一个汇衬底。它也可用于输入或输出衬底数量可变的元素，如`tee`（多输出）或`input-selector`（多输入）元素。

要实现请求型 pad，需要提供一个具有 `GST_PAD_REQUEST` 存在的 padtemplate，并在`GstElement` 中实现`request_new_pad`虚方法。要进行清理，则需要实现`release_pad`虚拟法。

```c
static GstPad * gst_my_filter_request_new_pad   (GstElement     *element,
                         GstPadTemplate *templ,
                                                 const gchar    *name,
                                                 const GstCaps  *caps);

static void gst_my_filter_release_pad (GstElement *element,
                                       GstPad *pad);

static GstStaticPadTemplate sink_factory =
GST_STATIC_PAD_TEMPLATE (
  "sink_%u",
  GST_PAD_SINK,
  GST_PAD_REQUEST,
  GST_STATIC_CAPS ("ANY")
);

static void
gst_my_filter_class_init (GstMyFilterClass *klass)
{
  GstElementClass *element_class = GST_ELEMENT_CLASS (klass);
[..]
  gst_element_class_add_pad_template (klass,
    gst_static_pad_template_get (&sink_factory));
[..]
  element_class->request_new_pad = gst_my_filter_request_new_pad;
  element_class->release_pad = gst_my_filter_release_pad;
}

static GstPad *
gst_my_filter_request_new_pad (GstElement     *element,
                   GstPadTemplate *templ,
                   const gchar    *name,
                               const GstCaps  *caps)
{
  GstPad *pad;
  GstMyFilterInputContext *context;

  context = g_new0 (GstMyFilterInputContext, 1);
  pad = gst_pad_new_from_template (templ, name);
  gst_pad_set_element_private (pad, context);

  /* normally, you would set _chain () and _event () functions here */

  gst_element_add_pad (element, pad);

  return pad;
}

static void
gst_my_filter_release_pad (GstElement *element,
                           GstPad *pad)
{
  GstMyFilterInputContext *context;

  context = gst_pad_get_element_private (pad);
  g_free (context);

  gst_element_remove_pad (element, pad);
}
```
