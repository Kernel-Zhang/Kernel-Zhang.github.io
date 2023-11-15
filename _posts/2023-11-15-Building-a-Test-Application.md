---
title: gstreamer插件编写指南：构建测试应用程序
author: kernel
date: 2023-11-15 10:03:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

通常，您希望在尽可能小的环境中测试新编写的插件。通常，`gst-launch-1.0`是测试插件的第一步。如果没有将插件安装在 GStreamer 可以搜索到的目录中，则需要设置插件路径。 可以将 `GST_PLUGIN_PATH` 设置为包含插件的目录，或者使用命令行选项 --gst-plugin-path 来设置。如果你的插件是基于 gst-plugin 模板开发的，那么看起来就像`gst-launch-1.0 --gst-plugin-path=$HOME/gst-template/gst-plugin/src/.libs TESTPIPELINE`。不过，你通常需要比 gst-launch-1.0 更多的测试功能，比如搜索、事件、交互等。编写自己的小型测试程序是最简单的方法，本节将简要介绍如何编写测试程序。如需完整的应用程序开发指南，请参阅《应用程序开发手册》。

开始时，需要调用`gst_init ()` 来初始化 GStreamer 核心库。你也可以调用`gst_init_get_option_group()`，它会返回一个指向 GOptionGroup 的指针。然后就可以使用 GOption 来处理初始化，这样就完成了 GStreamer 的初始化。

你可以使用`gst_element_factory_make()` 创建元素，其中第一个参数是你要创建的元素类型，第二个参数是一个自由格式的名称。末尾的示例使用了一个简单的文件源-解码器-声卡输出流水线，但如果有必要，你也可以使用特定的调试元素。例如，可以在流水线中间使用一个`identity`元素，作为数据到应用程序的传送器。这可用于在测试应用程序中检查数据的错误行为或正确性。此外，还可以在流水线末端使用`fakesink`元素将数据转储到 stdout（为此，请将`dump`属性设置为 TRUE）。最后，还可以使用 valgrind 检查内存错误。

在链接过程中，您的测试程序可以使用过滤衬底来驱动特定类型的数据输入或输出您的元素。这是检查元素中多种输入和输出类型的一种非常简单而有效的方法。

请注意，在运行过程中，您至少应监听总线和插件/元素上的 "error "和 "eos "信息，以检查处理是否正确。此外，您还应将事件添加到流水线中，并确保您的插件能正确处理这些事件（如时钟、内部缓存等）。

千万不要忘记清理插件或测试程序中的内存。 当进入 NULL 状态时，您的元素应清理已分配的内存和缓存。此外，它还应关闭对可能的支持库的任何引用。您的应用程序应`unref()`流水线，并确保它不会崩溃。

```c
#include <gst/gst.h>

static gboolean
bus_call (GstBus     *bus,
      GstMessage *msg,
      gpointer    data)
{
  GMainLoop *loop = data;

  switch (GST_MESSAGE_TYPE (msg)) {
    case GST_MESSAGE_EOS:
      g_print ("End-of-stream\n");
      g_main_loop_quit (loop);
      break;
    case GST_MESSAGE_ERROR: {
      gchar *debug = NULL;
      GError *err = NULL;

      gst_message_parse_error (msg, &err, &debug);

      g_print ("Error: %s\n", err->message);
      g_error_free (err);

      if (debug) {
        g_print ("Debug details: %s\n", debug);
        g_free (debug);
      }

      g_main_loop_quit (loop);
      break;
    }
    default:
      break;
  }

  return TRUE;
}

gint
main (gint   argc,
      gchar *argv[])
{
  GstStateChangeReturn ret;
  GstElement *pipeline, *filesrc, *decoder, *filter, *sink;
  GstElement *convert1, *convert2, *resample;
  GMainLoop *loop;
  GstBus *bus;
  guint watch_id;

  /* initialization */
  gst_init (&argc, &argv);
  loop = g_main_loop_new (NULL, FALSE);
  if (argc != 2) {
    g_print ("Usage: %s <mp3 filename>\n", argv[0]);
    return 01;
  }

  /* create elements */
  pipeline = gst_pipeline_new ("my_pipeline");

  /* watch for messages on the pipeline's bus (note that this will only
   * work like this when a GLib main loop is running) */
  bus = gst_pipeline_get_bus (GST_PIPELINE (pipeline));
  watch_id = gst_bus_add_watch (bus, bus_call, loop);
  gst_object_unref (bus);

  filesrc  = gst_element_factory_make ("filesrc", "my_filesource");
  decoder  = gst_element_factory_make ("mad", "my_decoder");

  /* putting an audioconvert element here to convert the output of the
   * decoder into a format that my_filter can handle (we are assuming it
   * will handle any sample rate here though) */
  convert1 = gst_element_factory_make ("audioconvert", "audioconvert1");

  /* use "identity" here for a filter that does nothing */
  filter   = gst_element_factory_make ("my_filter", "my_filter");

  /* there should always be audioconvert and audioresample elements before
   * the audio sink, since the capabilities of the audio sink usually vary
   * depending on the environment (output used, sound card, driver etc.) */
  convert2 = gst_element_factory_make ("audioconvert", "audioconvert2");
  resample = gst_element_factory_make ("audioresample", "audioresample");
  sink     = gst_element_factory_make ("pulsesink", "audiosink");

  if (!sink || !decoder) {
    g_print ("Decoder or output could not be found - check your install\n");
    return -1;
  } else if (!convert1 || !convert2 || !resample) {
    g_print ("Could not create audioconvert or audioresample element, "
             "check your installation\n");
    return -1;
  } else if (!filter) {
    g_print ("Your self-written filter could not be found. Make sure it "
             "is installed correctly in $(libdir)/gstreamer-1.0/ or "
             "~/.gstreamer-1.0/plugins/ and that gst-inspect-1.0 lists it. "
             "If it doesn't, check with 'GST_DEBUG=*:2 gst-inspect-1.0' for "
             "the reason why it is not being loaded.");
    return -1;
  }

  g_object_set (G_OBJECT (filesrc), "location", argv[1], NULL);

  gst_bin_add_many (GST_BIN (pipeline), filesrc, decoder, convert1, filter,
                    convert2, resample, sink, NULL);

  /* link everything together */
  if (!gst_element_link_many (filesrc, decoder, convert1, filter, convert2,
                              resample, sink, NULL)) {
    g_print ("Failed to link one or more elements!\n");
    return -1;
  }

  /* run */
  ret = gst_element_set_state (pipeline, GST_STATE_PLAYING);
  if (ret == GST_STATE_CHANGE_FAILURE) {
    GstMessage *msg;

    g_print ("Failed to start up pipeline!\n");

    /* check if there is an error message with details on the bus */
    msg = gst_bus_poll (bus, GST_MESSAGE_ERROR, 0);
    if (msg) {
      GError *err = NULL;

      gst_message_parse_error (msg, &err, NULL);
      g_print ("ERROR: %s\n", err->message);
      g_error_free (err);
      gst_message_unref (msg);
    }
    return -1;
  }

  g_main_loop_run (loop);

  /* clean up */
  gst_element_set_state (pipeline, GST_STATE_NULL);
  gst_object_unref (pipeline);
  g_source_remove (watch_id);
  g_main_loop_unref (loop);

  return 0;
}
```
