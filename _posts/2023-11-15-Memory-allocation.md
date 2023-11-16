---
title: gstreamer插件编写指南：能力集协商
author: kernel
date: 2023-11-15 12:29:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

内存分配和管理是多媒体领域非常重要的课题。高清视频存储一帧图像需要很多兆字节。重要的是尽可能重复使用内存，而不是不断地分配和释放内存。

多媒体系统通常使用 DSP 或 GPU 等专用芯片来执行繁重的工作（尤其是视频）。这些专用芯片通常对运行内存和访问内存的方式有严格的要求。

本章将介绍 GStreamer 插件可用的内存管理功能。我们将首先讨论管理内存访问的低级`GstMemory`对象，然后继续讨论它的主要用户之一`GstBuffer`，该对象用于在元素之间以及与应用程序交换数据。我们还将讨论`GstMeta`，该对象可以放置在缓冲区上，以提供有关缓冲区及其内存的额外信息。我们还将讨论`GstBufferPool`，它可以更有效地管理相同大小的缓冲区。

在本章的最后，我们将学习`GST_QUERY_ALLOCATION`查询，该查询用于在元素之间协商内存管理选项。

## GstMemory

`GstMemory`是一个管理内存区域的对象。该内存对象指向“maxsize”内存区域。从`offset`和`size`字节开始的内存区域是可访问的内存区域。`GstMemory`创建后，其 maxsize 不能再更改，但其`offset`和`size`可以更改。

### GstAllocator

`GstMemory`对象由`GstAllocator`对象创建。大多数分配器都会使用默认的`gst_allocator_alloc()`方法，但也有一些分配器会使用不同的方法，例如，当需要额外参数来分配特定内存时。

系统内存、共享内存和由 DMAbuf 文件描述符支持的内存有不同的分配器。要实现对新内存类型的支持，必须实现一个新的分配器对象。

### GstMemory API示例

对`GstMemory`对象封装的内存进行数据访问时，必须成对使用`gst_memory_map()`和`gst_memory_unmap()`进行保护。映射内存时必须给出访问模式（读/写）。映射函数会返回一个指向有效内存区域的指针，然后可以根据请求的访问模式访问该内存区域。

下面是创建`GstMemory`对象并使用`gst_memory_map()`访问内存区域的示例。

```c
[...]

  GstMemory *mem;
  GstMapInfo info;
  gint i;

  /* allocate 100 bytes */
  mem = gst_allocator_alloc (NULL, 100, NULL);

  /* get access to the memory in write mode */
  gst_memory_map (mem, &info, GST_MAP_WRITE);

  /* fill with pattern */
  for (i = 0; i < info.size; i++)
    info.data[i] = i;

  /* release memory */
  gst_memory_unmap (mem, &info);

[...]
```

### 实现GstAllocator

待添加……

## GstBuffer

`GstBuffer`是一个轻量级对象，从上游元素传递到下游元素，包含内存和元数据。它代表下游元素推送或提取的多媒体内容。

`GstBuffer`包含一个或多个`GstMemory`对象。这些对象保存缓冲区的数据。

缓冲区中的元数据包括：

-   DTS 和 PTS 时间戳。这些时间戳代表缓冲区内容的解码和呈现时间戳，被同步元素用来调度缓冲区。在未知/未定义的情况下，这些时间戳可以是`GST_CLOCK_TIME_NONE`。
    
-   缓冲区内容的持续时间。当未知/未定义时，该持续时间可以是`GST_CLOCK_TIME_NONE`。
    
-   特定媒体的`offset`和`offset_end`值。对于视频，这是流中的帧号，对于音频，这是采样号。其他媒体可能使用不同的定义。
    
-   通过`GstMeta` 实现任意结构，见下文。
    

### 可写性

当对象的 refcount 恰好为 1 时，`GstBuffer`是可写的，这意味着只有一个对象持有缓冲区的 ref。只有当缓冲区可写时，才能修改它。这意味着在更改时间戳、偏移量、元数据或添加和删除内存块之前，需要调用`gst_buffer_make_writable()`。

### API示例

您可以使用`gst_buffer_new ()`创建一个`GstBuffer`，然后向其中添加内存对象。也可以使用便利函数`gst_buffer_new_allocate()`，一次完成这两个操作。也可以使用`gst_buffer_new_wrapped_full()`封装现有内存，并指定释放内存时调用的函数。

访问`GstBuffer`的内存时，可以单独获取并映射`GstMemory`对象，也可以使用`gst_buffer_map（）`。后者会将所有内存合并成一个大块，然后给你一个指向它的指针。

下面举例说明如何创建缓冲区并访问其内存。

```c
[...]
  GstBuffer *buffer;
  GstMemory *mem;
  GstMapInfo info;

  /* make empty buffer */
  buffer = gst_buffer_new ();

  /* make memory holding 100 bytes */
  mem = gst_allocator_alloc (NULL, 100, NULL);

  /* add the buffer */
  gst_buffer_append_memory (buffer, mem);

[...]

  /* get WRITE access to the memory and fill with 0xff */
  gst_buffer_map (buffer, &info, GST_MAP_WRITE);
  memset (info.data, 0xff, info.size);
  gst_buffer_unmap (buffer, &info);

[...]

  /* free the buffer */
  gst_buffer_unref (buffer);

[...]
```

## GstMeta

通过`GstMeta`系统，您可以在缓冲区中添加任意结构。这些结构描述了缓冲区的额外属性，如裁剪、步长、感兴趣区域等。

元数据系统将应用程序接口规范（元数据及其API外观）和实现（如何运行）分开。这使得同一应用程序接口可以有不同的实现方式，例如，取决于你运行的硬件。

### API示例

分配新的`GstBuffer` 后，您可以使用元数据专用 API 为其添加元数据。这意味着您需要链接到定义元数据的头文件，才能使用其 API。

按照惯例，名为`FooBar`的元数据 API 应提供两个方法，即`gst_buffer_add_fooo_bar_meta ()`和`gst_buffer_get_fooo_bar_meta ()` 方法。这两个函数都应返回一个指向包含元数据字段的`FooBarMeta`结构的指针。某些`_add_*_meta ()`函数可能有额外的参数，这些参数通常用于为你配置元数据结构。

让我们来看看用于指定视频帧裁剪区域的元数据。

```c
#include <gst/video/gstvideometa.h>

[...]
  GstVideoCropMeta *meta;

  /* buffer points to a video frame, add some cropping metadata */
  meta = gst_buffer_add_video_crop_meta (buffer);

  /* configure the cropping metadata */
  meta->x = 8;
  meta->y = 8;
  meta->width = 120;
  meta->height = 80;
[...]
```

这样，元素就可以在渲染帧时使用缓冲区上的元数据：

```c
#include <gst/video/gstvideometa.h>

[...]
  GstVideoCropMeta *meta;

  /* buffer points to a video frame, get the cropping metadata */
  meta = gst_buffer_get_video_crop_meta (buffer);

  if (meta) {
    /* render frame with cropping */
    _render_frame_cropped (buffer, meta->x, meta->y, meta->width, meta->height);
  } else {
    /* render frame */
    _render_frame (buffer);
  }
[...]
```

### 实现新的GstMeta

在接下来的章节中，我们将展示如何在系统中添加新的元数据，并将其用于缓冲区。

#### 定义元数据API

首先，我们需要定义 API 的外观，并将其注册到系统中。这一点非常重要，因为当各元素协商交换何种元数据时，将使用 API 定义。API 定义还包含任意标签，这些标签会提示元数据包含哪些内容。当我们看到如何在缓冲区通过流水线时保留元数据时，这一点就非常重要了。

如果您要对现有的 API 进行新的实现，可以跳过这一步，直接进入实现。

首先，我们制作`my-example-meta.h`头文件，该文件将包含 API 的定义和元数据的结构。

```c
#include <gst/gst.h>

typedef struct _MyExampleMeta MyExampleMeta;

struct _MyExampleMeta {
  GstMeta       meta;

  gint          age;
  gchar        *name;
};

GType my_example_meta_api_get_type (void);
#define MY_EXAMPLE_META_API_TYPE (my_example_meta_api_get_type())

#define gst_buffer_get_my_example_meta(b) \
  ((MyExampleMeta*)gst_buffer_get_meta((b),MY_EXAMPLE_META_API_TYPE))
```

元数据 API 定义包括对包含一个`gint`和一个字符串的结构的定义。结构中的第一个字段必须是`GstMeta`。

我们还定义了一个`my_example_meta_api_get_type ()`函数，用于注册我们的元数据 API 定义，并定义了一个`gst_buffer_get_my_example_meta ()`宏，用于使用我们的新 API 查找并返回元数据。

让我们看看`my_example_meta_api_get_type ()`函数在`my-example-meta.c`文件中是如何实现的：

```c
#include "my-example-meta.h"

GType
my_example_meta_api_get_type (void)
{
  static GType type;
  static const gchar *tags[] = { "foo", "bar", NULL };

  if (g_once_init_enter (&type)) {
    GType _type = gst_meta_api_type_register ("MyExampleMetaAPI", tags);
    g_once_init_leave (&type, _type);
  }
  return type;
}
```

如您所见，它只需使用`gst_meta_api_type_register ()`函数为 API 注册一个名称和一些标记。结果是一个新的`GType`指针，它定义了新注册的 API。

#### 实现元数据API

接下来，我们可以为已注册的元数据 API`GType` 做一个实现。

元数据 API 的实现细节保存在一个`GstMetaInfo`结构中，通过`my_example_meta_get_info ()`函数和一个方便的`MY_EXAMPLE_META_INFO`宏，您可以向元数据 API 实现的用户提供该结构。您还提供了将元数据实现添加到`GstBuffer` 的方法。您的`my-example-meta.h`头文件需要添加这些内容：

```c
[...]

/* implementation */
const GstMetaInfo *my_example_meta_get_info (void);
#define MY_EXAMPLE_META_INFO (my_example_meta_get_info())

MyExampleMeta * gst_buffer_add_my_example_meta (GstBuffer      *buffer,
                                                gint            age,
                                                const gchar    *name);
```

让我们看看`my-example-meta.c`文件是如何实现这些函数的。

```c
[...]

static gboolean
my_example_meta_init (GstMeta * meta, gpointer params, GstBuffer * buffer)
{
  MyExampleMeta *emeta = (MyExampleMeta *) meta;

  emeta->age = 0;
  emeta->name = NULL;

  return TRUE;
}

static gboolean
my_example_meta_transform (GstBuffer * transbuf, GstMeta * meta,
    GstBuffer * buffer, GQuark type, gpointer data)
{
  MyExampleMeta *emeta = (MyExampleMeta *) meta;

  /* we always copy no matter what transform */
  gst_buffer_add_my_example_meta (transbuf, emeta->age, emeta->name);

  return TRUE;
}

static void
my_example_meta_free (GstMeta * meta, GstBuffer * buffer)
{
  MyExampleMeta *emeta = (MyExampleMeta *) meta;

  g_free (emeta->name);
  emeta->name = NULL;
}

const GstMetaInfo *
my_example_meta_get_info (void)
{
  static const GstMetaInfo *meta_info = NULL;

  if (g_once_init_enter (&meta_info)) {
    const GstMetaInfo *mi = gst_meta_register (MY_EXAMPLE_META_API_TYPE,
        "MyExampleMeta",
        sizeof (MyExampleMeta),
        my_example_meta_init,
        my_example_meta_free,
        my_example_meta_transform);
    g_once_init_leave (&meta_info, mi);
  }
  return meta_info;
}

MyExampleMeta *
gst_buffer_add_my_example_meta (GstBuffer   *buffer,
                                gint         age,
                                const gchar *name)
{
  MyExampleMeta *meta;

  g_return_val_if_fail (GST_IS_BUFFER (buffer), NULL);

  meta = (MyExampleMeta *) gst_buffer_add_meta (buffer,
      MY_EXAMPLE_META_INFO, NULL);

  meta->age = age;
  meta->name = g_strdup (name);

  return meta;
}
```

`gst_meta_register ()`注册了实现细节，比如你实现的 API 和元数据结构的大小，以及初始化和释放内存区域的方法。您还可以实现一个变换函数，当对缓冲区执行某种变换（由夸克和夸克特定数据标识）时，该函数将被调用。

最后，实现`gst_buffer_add_*_meta()`，将元数据实现添加到缓冲区，并设置元数据的值。

## GstBufferPool

`GstBufferPool`对象为管理可重复使用的缓冲区列表提供了一个方便的基类。该对象的关键在于所有缓冲区都具有相同的属性，如大小、填充、元数据和对齐方式。

`GstBufferPool`可配置为管理特定大小缓冲区的最小和最大数量。还可以将其配置为使用特定的`GstAllocator`来分配缓冲区的内存。缓冲池还支持启用缓冲池特定选项，例如向缓冲池的缓冲区添加`GstMeta`或在缓冲区内存中启用特定填充。

`GstBufferPool`可以处于非激活或激活状态。在非激活状态下，可以配置缓冲池。在激活状态下，不能再更改配置，但可以从缓冲池中获取和释放缓冲区。

下面我们将介绍如何使用`GstBufferPool`。

### API示例

`GstBufferPool`可以有多种不同的实现；它们都是`GstBufferPool`基类的子类。在本示例中，我们将假定自己可以访问缓冲池，这可能是因为我们自己创建了缓冲池，也可能是因为我们通过`ALLOCATION`查询得到了一个缓冲池，下面我们将看到这一点。

`GstBufferPool`最初处于非活动状态，以便我们对其进行配置。尝试配置未处于非活动状态的`GstBufferPool`将失败。同样，尝试激活未配置的缓冲池也会失败。

```c
  GstStructure *config;

[...]

  /* get config structure */
  config = gst_buffer_pool_get_config (pool);

  /* set caps, size, minimum and maximum buffers in the pool */
  gst_buffer_pool_config_set_params (config, caps, size, min, max);

  /* configure allocator and parameters */
  gst_buffer_pool_config_set_allocator (config, allocator, &params);

  /* store the updated configuration again */
  gst_buffer_pool_set_config (pool, config);

[...]
```

`GstBufferPool`的配置保存在一个通用的`GstStructure`中，可以通过`gst_buffer_pool_get_config()`获取。 有一些方便的方法可以获取和设置该结构中的配置选项。更新该结构后，将通过`gst_buffer_pool_set_config()`再次将其设置为`GstBufferPool`中的当前配置。

可以在`GstBufferPool` 上配置以下选项：

-   要分配的缓冲区上限。
    
-   缓冲区的大小。这是建议的池中缓冲区大小。缓冲池可能会分配更大的缓冲区，以增加缓冲。
    
-   缓冲池中缓冲区的最小和最大数量。当最小值设置为大于0时，缓冲池将预先分配该数量的缓冲区。当最大值不为 0 时，bufferpool 将分配最大值的缓冲区。
    
-   要使用的分配器和参数。某些缓冲池可能会忽略分配器，而使用其内部分配器。
    
-   通过`gst_buffer_pool_get_options()`，bufferpool 会列出支持的选项；通过`gst_buffer_pool_has_option()`，可以询问是否支持某个选项。使用`gst_buffer_pool_config_add_option()`将选项添加到配置结构中，即可启用该选项。这些选项用于启用一些功能，例如让池在缓冲区上设置元数据，或者为填充添加额外的配置选项。
    

在缓冲池上设置配置后，可以使用`gst_buffer_pool_set_active (pool, TRUE)` 激活缓冲池。从那时起，你就可以使用`gst_buffer_pool_acquire_buffer ()`从缓冲池中获取缓冲区，就像下面这样：

```c
  [...]

  GstFlowReturn ret;
  GstBuffer *buffer;

  ret = gst_buffer_pool_acquire_buffer (pool, &buffer, NULL);
  if (G_UNLIKELY (ret != GST_FLOW_OK))
    goto pool_failed;

  [...]
```

检查 acquire 函数的返回值非常重要，因为它有可能失败：当您的元素关闭时，它将停用缓冲池，然后所有对 acquire 的调用都将返回`GST_FLOW_FLUSHING`。

所有从缓冲池中获取的缓冲区，其缓冲池成员都将设置为原始缓冲池。当缓冲区的最后一个 ref 被递减时，GStreamer 会自动调用`gst_buffer_pool_release_buffer()`，将缓冲区释放回池中。你（或任何其他下游元素）不需要知道缓冲区是否来自池，只需取消引用即可。

### 实现新的GstBufferPool

待添加……

## GST\_QUERY\_ALLOCATION

`ALLOCATION`查询用于在元素之间协商`GstMeta`、`GstBufferPool`和`GstAllocator`。分配策略的协商总是在协商格式后、决定推送缓冲区前由 srcpad 发起和决定。sinkpad 可以提出分配策略建议，但最终还是由源衬底根据下游 sinkpad 的建议做出决定。

源衬底将以协商好的能力为参数执行`GST_QUERY_ALLOCATION`。这样下游设备才能知道正在处理的媒体类型。下游汇衬底可通过以下结果回应分配查询：

-   可支持的`GstBufferPool`数组，包含建议的缓冲区大小、最小和最大数量。
    
-   `GstAllocator`对象数组，以及建议的分配参数，如标志、前缀、对齐方式和填充。如果缓冲池支持，也可以在缓冲池中配置这些分配器。
    
-   受支持的`GstMeta`实现数组以及元数据的特定参数。上游元素在将元数据放到缓冲区之前，必须知道下游支持哪种元数据。
    

当`GST_QUERY_ALLOCATION`返回时，源衬底将从可用的缓冲池、分配器和元数据中选择如何分配缓冲区。

### ALLOCATION 查询示例

下面是`ALLOCATION`查询的示例。

```c
#include <gst/video/video.h>
#include <gst/video/gstvideometa.h>
#include <gst/video/gstvideopool.h>

  GstCaps *caps;
  GstQuery *query;
  GstStructure *structure;
  GstBufferPool *pool;
  GstStructure *config;
  guint size, min, max;

[...]

  /* find a pool for the negotiated caps now */
  query = gst_query_new_allocation (caps, TRUE);

  if (!gst_pad_peer_query (scope->srcpad, query)) {
    /* query failed, not a problem, we use the query defaults */
  }

  if (gst_query_get_n_allocation_pools (query) > 0) {
    /* we got configuration from our peer, parse them */
    gst_query_parse_nth_allocation_pool (query, 0, &pool, &size, &min, &max);
  } else {
    pool = NULL;
    size = 0;
    min = max = 0;
  }

  if (pool == NULL) {
    /* we did not get a pool, make one ourselves then */
    pool = gst_video_buffer_pool_new ();
  }

  config = gst_buffer_pool_get_config (pool);
  gst_buffer_pool_config_add_option (config, GST_BUFFER_POOL_OPTION_VIDEO_META);
  gst_buffer_pool_config_set_params (config, caps, size, min, max);
  gst_buffer_pool_set_config (pool, config);

  /* and activate */
  gst_buffer_pool_set_active (pool, TRUE);

[...]
```

这种特殊的实现方式会创建一个自定义的`GstVideoBufferPool`对象，专门用于分配视频缓冲区。您还可以启用池，在池中的缓冲区上放置`GstVideoMeta`元数据：

```c
gst_buffer_pool_config_add_option (config, GST_BUFFER_POOL_OPTION_VIDEO_META)
```

### 基类中的 ALLOCATION 查询

在许多基类中，你都会看到以下用于影响分配策略的虚方法：

-   `propose_allocation ()`应该为上游元素建议分配参数。
    
-   `decide_allocation ()`应根据从下游收到的建议来决定分配参数。
    

这些方法的实现者应通过更新池选项和分配选项来修改给定的`GstQuery`对象。

### 协商视频缓冲区的精确布局

硬件元素可能对其输入缓冲区的布局有特定限制，要求在其平面上添加垂直或水平填充。 如果生产者能够创建满足这些要求的缓冲区，我们就可以在开始生产缓冲区之前对其驱动程序进行相应配置，以确保零拷贝。

在 Linux 上的这种设置中，我们通常使用 dmabuf 来交换缓冲区，以减少内存拷贝。生产者可以向消费者导出缓冲区（dmabuf export），也可以从消费者导入缓冲区（dmabuf import）。

在本节中，我们将概述消费者如何在导入和导出用例中通知生产者其预期的缓冲区布局的步骤。 让我们考虑一下`v4l2src`（生产者）向`v4l2h264enc`（消费者）提供缓冲区进行编码的情况。

#### v4l2src 从 v4l2h264enc 导入缓冲区

1.  v4l2h264enc：查询硬件的需求并创建相应的`GstVideoAlignment`。
2.  v4l2h264enc：在其缓冲池`alloc_buffer`实现中，调用`gst_buffer_add_video_meta_full()`，然后在返回的元上调用`gst_vide_meta_set_alignment()`，并请求对齐。对齐方式将被添加到元数据中，从而允许`v4l2src`在尝试导入缓冲区之前配置其驱动程序。

```c
      meta = gst_buffer_add_video_meta_full (buf, GST_VIDEO_FRAME_FLAG_NONE,
          GST_VIDEO_INFO_FORMAT (&pool->video_info),
          GST_VIDEO_INFO_WIDTH (&pool->video_info),
          GST_VIDEO_INFO_HEIGHT (&pool->video_info),
          GST_VIDEO_INFO_N_PLANES (&pool->video_info), offset, stride);

      gst_video_meta_set_alignment (meta, align);
```

3.  v4l2h264enc：在回复`ALLOCATION`查询`(propose_allocation())` 时，向生产者提议其池。
4.  v4l2src：当收到`ALLOCATION`查询（`decide_allocation()`）的回复时，从建议的池中获取单个缓冲区，并使用`GstVideoMeta.stride` 和`gst_vide_meta_get_plane_height()`检索其布局。
5.  v4l2src：如果可能，配置其驱动程序以生成符合这些要求的数据，然后尝试导入缓冲区。 如果不行，`v4l2src`将无法从`v4l2h264enc`导入，因此会退回到向`v4l2h264enc`发送自己的缓冲区，而`v4l2h264enc`将不得不复制每个输入缓冲区以满足其要求。

#### v4l2src 向 v4l2h264enc 导出缓冲区

1.  v4l2h264enc：查询硬件的需求并创建相应的`GstVideoAlignment`。
2.  v4l2h264enc：创建名为`video-meta`的`GstStructure`，将对齐方式序列化：

```c
params = gst_structure_new ("video-meta",
    "padding-top", G_TYPE_UINT, align.padding_top,
    "padding-bottom", G_TYPE_UINT, align.padding_bottom,
    "padding-left", G_TYPE_UINT, align.padding_left,
    "padding-right", G_TYPE_UINT, align.padding_right,
    NULL);
```

3.  v4l2h264enc：在处理`ALLOCATION`查询（`propose_allocation()`）时，添加`GST_VIDEO_META_API_TYPE`元数据时将此结构作为参数传递：

```c
gst_query_add_allocation_meta (query, GST_VIDEO_META_API_TYPE, params);
```

4.  v4l2src：在收到`ALLOCATION`查询（`decide_allocation()`）的回复时，检索`GST_VIDEO_META_API_TYPE`参数，以计算预期的缓冲区布局：

```
guint video_idx;
GstStructure *params;

if (gst_query_find_allocation_meta (query, GST_VIDEO_META_API_TYPE, &video_idx)) {
  gst_query_parse_nth_allocation_meta (query, video_idx, &params);

  if (params) {
    GstVideoAlignment align;
    GstVideoInfo info;
    gsize plane_size[GST_VIDEO_MAX_PLANES];

    gst_video_alignment_reset (&align);

    gst_structure_get_uint (s, "padding-top", &align.padding_top);
    gst_structure_get_uint (s, "padding-bottom", &align.padding_bottom);
    gst_structure_get_uint (s, "padding-left", &align.padding_left);
    gst_structure_get_uint (s, "padding-right", &align.padding_right);

    gst_video_info_from_caps (&info, caps);

    gst_video_info_align_full (&info, align, plane_size);
  }
}
```

5.  v4l2src：使用`GstVideoInfo.stride`和`GST_VIDEO_INFO_PLANE_HEIGHT()`获取请求的缓冲区布局。
6.  v4l2src：配置其驱动程序，以便尽可能生成符合这些要求的数据。如果不符合，驱动程序将使用自己的布局生成缓冲区，但`v4l2h264enc`必须复制每个输入缓冲区，以满足其要求。
