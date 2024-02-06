---
title: gstreamer插件编写指南：支持动态参数
author: kernel
date: 2023-11-16 13:00:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

有时对象的属性参数不足以控制影响元素行为。在这种情况下，你可以将这些参数标记为“可控”。应用程序可以有意识地使用控制器子系统随时间动态调整属性值。

> 警告，本部分描述的是 0.10 版本，已经过时。

## 入门

控制器子系统包含在`gstcontroller`库中。您需要在元素源文件中包含该头文件：

```c
...
#include <gst/gst.h>
#include <gst/controller/gstcontroller.h>
...
```

尽管`gstcontroller`库可能已链接到主应用程序中，但仍应确保在`plugin_init`函数中对其进行初始化：

```c
  static gboolean
  plugin_init (GstPlugin *plugin)
  {
    ...
    /* initialize library */
    gst_controller_init (NULL, NULL);
    ...
  }
```

对所有 GObject 参数进行实时控制是没有意义的，因此下一步是对可控参数进行标记。在`_class_init`方法中设置 GObject 参数时，可以使用特殊标志`GST_PARAM_CONTROLLABLE`来标记可控参数。

```c
  g_object_class_install_property (gobject_class, PROP_FREQ,
      g_param_spec_double ("freq", "Frequency", "Frequency of test signal",
          0.0, 20000.0, 440.0,
          G_PARAM_READWRITE | GST_PARAM_CONTROLLABLE | G_PARAM_STATIC_STRINGS));
```

## 数据处理循环

在上一节中，我们学习了如何将 GObject 参数标记为可控参数。应用程序开发人员可以为这些参数的更改排队。控制器子系统采用的方法是让插件负责将更改生效。这只需要一个操作：

```c
gst_object_sync_values(element,timestamp);
```

该调用通过调整元素的 GObject 属性，使给定时间戳的所有参数变化都处于激活状态。同步率由元素决定。

### 视频元素的数据处理回路

对于视频处理元素，最好是每帧都同步。这意味着需要在元素的数据处理函数中添加上一节所述的`gst_object_sync_values()`调用。

### 音频元素的数据处理回路

音频处理元素的情况不像视频处理元件那么容易。问题在于音频的处理速度要高得多。对于 PAL 制式视频，例如每秒可处理 25 个全帧，但对于标准音频，则需要 44100 个采样点。很少有必要如此频繁地同步可控参数。最简单的解决方案也是在每个缓冲处理过程中只调用一次同步。这样，控制速率就取决于缓冲区的大小。

需要特定控制速率的元素需要中断数据处理循环，每 n 个采样点同步一次。
