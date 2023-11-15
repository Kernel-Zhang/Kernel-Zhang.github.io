---
title: gstreamer插件编写指南：添加属性
author: kernel
date: 2023-11-15 08:53:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

控制元素行为的最主要和最重要的方式是通过 GObject 属性。GObject 属性是在`_class_init ()`函数中定义的。元素可选择实现`_get_property ()`和`_set_property ()`函数。如果应用程序更改或请求某个属性的值，这些函数将收到通知，然后可以填入该属性的值，或采取必要的措施使该属性的值在内部发生变化。

您可能还想保留一个实例变量，其中包含您在 get 和 set 函数中使用的当前配置值作为属性。请注意，`GObject`不会自动将您的实例变量设置为默认值，您必须在元素的`_init ()`函数中进行设置。

```c
/* properties */
enum {
  PROP_0,
  PROP_SILENT
  /* FILL ME */
};

static void gst_my_filter_set_property  (GObject      *object,
                         guint         prop_id,
                         const GValue *value,
                         GParamSpec   *pspec);
static void gst_my_filter_get_property  (GObject      *object,
                         guint         prop_id,
                         GValue       *value,
                         GParamSpec   *pspec);

static void
gst_my_filter_class_init (GstMyFilterClass *klass)
{
  GObjectClass *object_class = G_OBJECT_CLASS (klass);

  /* define virtual function pointers */
  object_class->set_property = gst_my_filter_set_property;
  object_class->get_property = gst_my_filter_get_property;

  /* define properties */
  g_object_class_install_property (object_class, PROP_SILENT,
    g_param_spec_boolean ("silent", "Silent",
              "Whether to be very verbose or not",
              FALSE, G_PARAM_READWRITE | G_PARAM_STATIC_STRINGS));
}

static void
gst_my_filter_set_property (GObject      *object,
                guint         prop_id,
                const GValue *value,
                GParamSpec   *pspec)
{
  GstMyFilter *filter = GST_MY_FILTER (object);

  switch (prop_id) {
    case PROP_SILENT:
      filter->silent = g_value_get_boolean (value);
      g_print ("Silent argument was changed to %s\n",
           filter->silent ? "true" : "false");
      break;
    default:
      G_OBJECT_WARN_INVALID_PROPERTY_ID (object, prop_id, pspec);
      break;
  }
}

static void
gst_my_filter_get_property (GObject    *object,
                guint       prop_id,
                GValue     *value,
                GParamSpec *pspec)
{
  GstMyFilter *filter = GST_MY_FILTER (object);

  switch (prop_id) {
    case PROP_SILENT:
      g_value_set_boolean (value, filter->silent);
      break;
    default:
      G_OBJECT_WARN_INVALID_PROPERTY_ID (object, prop_id, pspec);
      break;
  }
}
```

以上是一个非常简单的属性使用示例。图形应用程序将使用这些属性，并将显示一个用户可控的窗口小部件，通过它可以更改这些属性。 这意味着，为了使属性尽可能方便用户使用，您应该尽可能精确地定义属性。 不仅要定义有效属性的范围（整数、浮点数等），还要在属性定义中使用描述性很强的字符串（最好是国际化字符串），如果可能，还要使用枚举和标志来代替整数。GObject 文档以非常完整的方式对这些进行了描述，下面我们将举一个简短的例子来说明这在哪些方面是有用的。请注意，在这里使用整数可能会使用户完全混淆，因为它们在这种情况下没有任何意义。该示例来自 videotestsrc。

```c
typedef enum {
  GST_VIDEOTESTSRC_SMPTE,
  GST_VIDEOTESTSRC_SNOW,
  GST_VIDEOTESTSRC_BLACK
} GstVideotestsrcPattern;

[..]

#define GST_TYPE_VIDEOTESTSRC_PATTERN (gst_videotestsrc_pattern_get_type ())
static GType
gst_videotestsrc_pattern_get_type (void)
{
  static GType videotestsrc_pattern_type = 0;

  if (!videotestsrc_pattern_type) {
    static GEnumValue pattern_types[] = {
      { GST_VIDEOTESTSRC_SMPTE, "SMPTE 100% color bars",    "smpte" },
      { GST_VIDEOTESTSRC_SNOW,  "Random (television snow)", "snow"  },
      { GST_VIDEOTESTSRC_BLACK, "0% Black",                 "black" },
      { 0, NULL, NULL },
    };

    videotestsrc_pattern_type =
    g_enum_register_static ("GstVideotestsrcPattern",
                pattern_types);
  }

  return videotestsrc_pattern_type;
}

[..]

static void
gst_videotestsrc_class_init (GstvideotestsrcClass *klass)
{
[..]
  g_object_class_install_property (G_OBJECT_CLASS (klass), PROP_PATTERN,
    g_param_spec_enum ("pattern", "Pattern",
               "Type of test pattern to generate",
                       GST_TYPE_VIDEOTESTSRC_PATTERN, GST_VIDEOTESTSRC_SMPTE,
                       G_PARAM_READWRITE | G_PARAM_STATIC_STRINGS));
[..]
}
```
