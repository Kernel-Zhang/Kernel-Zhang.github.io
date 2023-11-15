---
title: gstreamer插件编写指南：指定衬底
author: kernel
date: 2023-11-14 17:20:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

如前所述，pad 是数据进出元素的端口，因此在元素创建过程中非常重要。在模板代码中，我们已经看到静态 pad 模板是如何在元素类中注册 pad 模板的。在这里，我们将看到如何创建实际的元素，如何使用`_event ()` 函数为特定格式进行配置，以及如何注册函数让数据在元素中流动。

在元素`_init ()`函数中，通过在`_class_init ()`函数中向元素类注册的 pad 模板创建 pad。创建 pad 后，必须设置一个`_chain ()`函数指针，用于接收和处理 sinkpad 上的输入数据。还可以选择设置`_event ()`函数指针和`_query()` 函数指针。另外，sinkpad 也可以在循环模式下运行，这意味着它们可以自己获取数据。 这方面的内容稍后详述。然后，必须将 pad 注册到元素中。注册过程如下：

```c
static void
gst_my_filter_init (GstMyFilter *filter)
{
  /* pad through which data comes in to the element */
  filter->sinkpad = gst_pad_new_from_static_template (
    &sink_template, "sink");
  /* pads are configured here with gst_pad_set_*_function () */



  gst_element_add_pad (GST_ELEMENT (filter), filter->sinkpad);

  /* pad through which data goes out of the element */
  filter->srcpad = gst_pad_new_from_static_template (
    &src_template, "src");
  /* pads are configured here with gst_pad_set_*_function () */



  gst_element_add_pad (GST_ELEMENT (filter), filter->srcpad);

  /* properties initial value */
  filter->silent = FALSE;
}
```
