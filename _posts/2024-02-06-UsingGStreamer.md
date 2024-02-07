---
title: 常见问题：GStreamer的使用
author: kernel
date: 2024-02-06 12:31:00 +0800
categories: [gstreamer, 常见问题]
tags: [gst常见问题, gstreamer, 多媒体]
---

## 好了，我已经安装了 GStreamer。下一步该怎么做？

首先，请确认您的安装系统正常运行，并且您可以通过键入

```shell
$ gst-inspect-1.0 fakesrc
```

这会打印出有关该特定元素的大量信息。如果显示 "无此元素或插件"，说明你没有正确安装 GStreamer。请查看[如何获取 GStreamer](../GettingGStreamer/)如果出现任何其他失败信息，请提交错误报告。

是时候试一试了。首先使用 gst-launch 和两个你应该拥有的插件：fakesrc 和 fakesink。除了传递空缓冲区外，它们什么也不做。在命令行输入：

```shell
$ gst-launch-1.0 -v fakesrc silent=false num-buffers=3 ! fakesink silent=false
```

这将打印出类似下面的输出：

```log
RUNNING pipeline ...
fakesrc0: last-message = "get      ******* (fakesrc0:src)gt; (0 bytes, 0) 0x8057510"
fakesink0: last-message = "chain   ******* (fakesink0:sink)lt; (0 bytes, 0) 0x8057510"
fakesrc0: last-message = "get      ******* (fakesrc0:src)gt; (0 bytes, 1) 0x8057510"
fakesink0: last-message = "chain   ******* (fakesink0:sink)lt; (0 bytes, 1) 0x8057510"
fakesrc0: last-message = "get      ******* (fakesrc0:src)gt; (0 bytes, 2) 0x8057510"
fakesink0: last-message = "chain   ******* (fakesink0:sink)lt; (0 bytes, 2) 0x8057510"
execution ended after 5 iterations (sum 301479000 ns, average 60295800 ns, min 3000 ns, max 105482000 ns)
```

（为清晰起见，删除了部分输出）如果看起来相似，则说明 GStreamer 本身运行正常。

要显示测试视频，请尝试

```shell
$ gst-launch-1.0 videotestsrc ! videoconvert ! autovideosink
```

如果`autovideosink`不起作用，请尝试使用针对您的操作系统和窗口系统的元素，如`ximagesink`或`glimagesink`或（在 Windows 上）`d3dvideosink`。

## 我的系统能通过 GStreamer 播放声音吗？

您可以尝试播放正弦音调来测试这一点。为此，您需要将 audiotestsrc 元素链接到与硬件相匹配的输出元素上。音频输出插件（不完整）列表如下：

-   用于 Pulseaudio 输出的`pulsesink`
    
-   用于 ALSA 输出的`alsasink`
    
-   用于OSS/OSSv4输出的`osssink`和`oss4sink`
    
-   用于 JACK 输出的`jackaudiosink`
    
-   自动音频输出选择的`autoaudiosink`
    

首先，在要使用的输出插件上运行 gst-inspect-1.0，确保已安装该插件。例如，如果使用 Pulseaudio，运行：

```shell
$ gst-inspect-1.0 pulsesink
```

看看是否能打印出插件的一系列属性。

然后运行：

```shell
audiotestsrc ! audioconvert ! audioresample ! pulsesink
```

看看是否能听到声音。确保音量开得很大，但也要确保声音不会太大，而且您没有戴耳机。

## 如何查看系统中有哪些 GStreamer 插件？

为此，需要使用 GstStreamer 标配的 gst-inspect 命令行工具。调用时不需要任何参数：

```shell
$ gst-inspect-1.0
```

将打印出已安装插件的列表。要了解特定插件的更多信息，请在命令行中输入其名称。例如

```shell
$ gst-inspect-1.0 volume
```

将为您提供有关音量插件的信息。

## 我应该在哪里报告问题？

Freedesktop.org 的 Gitlab 对问题进行跟踪，网址是[https://gitlab.freedesktop.org/gstreamer](https://gitlab.freedesktop.org/gstreamer)。使用 Gitlab，您可以查看过去的问题、报告新问题、提交合并请求等。Gitlab 要求您在那里创建一个账户，这看起来可能比较麻烦，但至少可以让我们有机会联系您以获取更多信息，因为我们经常需要这样做。

## 我应该如何报告错误？

在进行错误报告时，您至少应该描述：

-   您的发行版、发行版的版本和 GStreamer 版本
    
-   您是如何安装 GStreamer 的（从 git、源代码、软件包安装？）
    
-   您是否曾经安装过GStreamer
    

如果您遇到问题的应用程序出现了断错误，请向我们提供必要的 gdb 输出。

## 如何使用 GStreamer 命令行界面？

你可以使用`gst-launch-1.0` 命令访问 GStreamer 命令行界面。例如，要播放文件，只需使用：

```shell
gst-launch-1.0 playbin uri=file:///path/to/song.mp3
```

您也可以使用`gst-play`：

```
gst-play-1.0 song.mp3
```

要解码 mp3 音频文件并通过 Pulseaudio 播放，您可以使用：

```
gst-launch-1.0 filesrc location=thesong.mp3 ! mpegaudioparse ! mpg123audiodec ! audioconvert ! pulsesink
```

要自动检测并为管道中的特定编码流选择正确的解码器，请尝试以下任何一种方法：

```shell
gst-launch-1.0 filesrc location=thesong.mp3 ! decodebin ! audioconvert ! pulsesink
```

```shell
gst-launch-1.0 filesrc location=my-random-media-file.mpeg ! decodebin ! pulsesink
```

```shell
gst-launch-1.0 filesrc location=my-random-media-file.mpeg ! decodebin ! videoconvert ！xvimagesink
```

或者更复杂一些，比如：

```shell
gst-launch-1.0 filesrc location=my-random-media-file.mpeg ! decodebin name=decoder \ 
    decoder.queue ! videoconvert ！xvimagesink \ 
    decoder.queue ! audoconvert ! pulsesink
```

在前一个例子的基础上，你可以让 GStreamer 选择一组合适的默认汇，方法是用这些自动替代元素替换特定的输出元素：

```shell
gst-launch-1.0 filesrc location=my-random-media-file.mpeg ! decodebin name=decoder \ 
    decoder.queue ! videoconvert ! autovideosink \ 
    decoder.queue ! audoconvert ! autoaudiosink
```

GStreamer 还提供`playbin`，这是一个基本的媒体播放插件，能自动处理大部分播放细节。下面的示例展示了如何播放任何格式的文件，只要其格式受支持，即安装了必要的解流和解码插件：

```shell
gst-launch-1.0 playbin uri=file:///home/joe/my-random-media-file.mpeg
```

其他示例请参见`gst-launch`手册页面。
