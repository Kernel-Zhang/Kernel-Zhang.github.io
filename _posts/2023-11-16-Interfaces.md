---
title: gstreamer插件编写指南：接口
author: kernel
date: 2023-11-16 13:01:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

在前面的“添加属性”一章中，我们介绍了控制元素行为的 GObject 属性概念。这个概念非常强大，但它有两个很大的缺点：首先，它过于通用；其次，它不是动态的。

第一个缺点与终端用户接口的可定制性有关，该接口将用于控制元素。有些属性比其他属性更重要。一些整数属性最好用旋转按钮部件来显示，而另一些则最好用滑块部件来表示。这些都是不可能的，因为用户接口在应用程序中没有实际意义。表示比特率属性的用户接口部件与表示视频大小的用户接口部件是一样的，只要两者都是相同的`GParamSpec`类型。另一个问题是，参数分组、函数分组或参数耦合等功能实际上都无法实现。

参数的第二个问题是它们不是动态的。在许多情况下，属性的允许值并不是固定的，而是取决于运行时才能检测到的因素。例如，只有当我们打开设备时，源元素才能才能从video4linux的内核驱动程序中获取电视卡的输入名称；只有当元素进入“就绪”（READY）状态时，才能获取输入名称。这意味着我们无法创建一个枚举属性类型来向用户显示这一点。

解决这些问题的办法是为某些常用控件创建非常专门的控件类型。我们使用接口的概念来实现这一点。这一切的基础是通用的`GTypeInterface`类型。对于我们认为有用的每种情况，我们都创建了接口，元素可以按照自己的意愿来实现这些接口。

有一点很重要：接口不能取代属性。相反，接口应建在属性旁边。这有两个重要原因。首先，属性更容易被内省。其次，属性可以在命令行中指定（`gst-launch-1.0`）。

## 如何实现接口

接口的实现是通过元素的`_get_type ()`启动的。在注册了类型本身后，还可以注册一个或多个接口。有些接口依赖于其他接口，或者只能由特定类型的元素注册。在使用元素时，如果你注册错误，你会收到通知：它将以失败断言退出，并解释出错的原因。如果出现这种情况，你需要先注册对该接口的支持，然后再注册对你想要支持的接口的支持。下面的示例说明了如何为一个简单的接口添加支持，而无需其他依赖项。

```c
static void gst_my_filter_some_interface_init   (GstSomeInterface *iface);

GType
gst_my_filter_get_type (void)
{
  static GType my_filter_type = 0;

  if (!my_filter_type) {
    static const GTypeInfo my_filter_info = {
      sizeof (GstMyFilterClass),
      NULL,
      NULL,
      (GClassInitFunc) gst_my_filter_class_init,
      NULL,
      NULL,
      sizeof (GstMyFilter),
      0,
      (GInstanceInitFunc) gst_my_filter_init
    };
    static const GInterfaceInfo some_interface_info = {
      (GInterfaceInitFunc) gst_my_filter_some_interface_init,
      NULL,
      NULL
    };

    my_filter_type =
    g_type_register_static (GST_TYPE_ELEMENT,
                "GstMyFilter",
                &my_filter_info, 0);
    g_type_add_interface_static (my_filter_type,
                 GST_TYPE_SOME_INTERFACE,
                                 &some_interface_info);
  }

  return my_filter_type;
}

static void
gst_my_filter_some_interface_init (GstSomeInterface *iface)
{
  /* here, you would set virtual function pointers in the interface */
}
```

或者更方便些：

```c
static void gst_my_filter_some_interface_init   (GstSomeInterface *iface);

G_DEFINE_TYPE_WITH_CODE (GstMyFilter, gst_my_filter,GST_TYPE_ELEMENT,
     G_IMPLEMENT_INTERFACE (GST_TYPE_SOME_INTERFACE,
            gst_my_filter_some_interface_init));

GST_ELEMENT_REGISTER_DEFINE(my_filter, "my-filter", GST_RANK_NONE, GST_TYPE_MY_FILTER);
```

## URI 接口

待添加……

## 色彩平衡接口

待添加……

## 视频叠加接口

`GstVideoOverlay`接口主要有两个用途：

-   抓取视频汇元素将要呈现的窗口。要做到这一点，要么需要了解视频汇元素生成的窗口标识符，要么需要强制视频汇元素使用特定的窗口标识符进行呈现。
    
-   强制重新绘制窗口上显示的最新视频帧。事实上，如果`GstPipeline`处于`GST_STATE_PAUSED`状态，移动窗口会损坏其内容。应用程序开发人员需要自行处理 Expose 事件，并强制视频汇元素刷新窗口内容。

在视频窗口中绘制视频输出的插件需要在某个阶段拥有该窗口。简单地说，被动模式就是在该阶段之前没有给插件提供窗口，所以插件自己创建了窗口。在这种情况下，插件有责任在不再需要该窗口时将其销毁，并且必须告诉应用程序已经创建了一个窗口，以便应用程序可以使用它。插件可以通过`gst_video_overlay_got_window_handle`方法发布`have-window-handle`消息。

正如你可能已经猜到的那样，激活模式只是意味着向插件发送一个视频窗口，以便视频输出到该窗口。这需要使用`gst_video_overlay_set_window_handle`方法来完成。

在任何时候都有可能从一种模式切换到另一种模式，因此实现该接口的插件必须处理所有情况。插件编写者必须实现的方法只有 2 个，它们很可能看起来像 ：

```c
static void
gst_my_filter_set_window_handle (GstVideoOverlay *overlay, guintptr handle)
{
  GstMyFilter *my_filter = GST_MY_FILTER (overlay);

  if (my_filter->window)
    gst_my_filter_destroy_window (my_filter->window);

  my_filter->window = handle;
}

static void
gst_my_filter_xoverlay_init (GstVideoOverlayClass *iface)
{
  iface->set_window_handle = gst_my_filter_set_window_handle;
}
```

您还需要使用接口方法在需要时发布信息，例如在接收到 CAPS 事件时，您需要知道视频的几何形状并创建窗口。

```c
static MyFilterWindow *
gst_my_filter_window_create (GstMyFilter *my_filter, gint width, gint height)
{
  MyFilterWindow *window = g_new (MyFilterWindow, 1);
  ...
  gst_video_overlay_got_window_handle (GST_VIDEO_OVERLAY (my_filter), window->win);
}

/* called from the event handler for CAPS events */
static gboolean
gst_my_filter_sink_set_caps (GstMyFilter *my_filter, GstCaps *caps)
{
  gint width, height;
  gboolean ret;
  ...
  ret = gst_structure_get_int (structure, "width", &width);
  ret &= gst_structure_get_int (structure, "height", &height);
  if (!ret) return FALSE;

  gst_video_overlay_prepare_window_handle (GST_VIDEO_OVERLAY (my_filter));

  if (!my_filter->window)
    my_filter->window = gst_my_filter_create_window (my_filter, width, height);

  ...
}
```

## 导航接口

待添加……
