---
title: 常见问题：GStreamer的获取
author: kernel
date: 2024-02-06 11:32:00 +0800
categories: [gstreamer, 常见问题]
tags: [gst常见问题, gstreamer, 多媒体]
---

## 如何获取 GStreamer？

一般来说，您有三个选项，从易到难：

-   适用于 Windows、macOS、iOS 和 Android 的二进制软件包
    
-   特定发行版软件包
    
-   源代码压缩包
    
-   git
    

## 0.10 和 1.0 版本之间有什么区别？

_GStreamer 似乎有不同的版本，比如 0.10 和 1.0？这是怎么回事？_

GStreamer-0.10 和 GStreamer-1.0 是目前使用的主要版本 "系列"。就所有实际用途而言，你应该将它们视为两个完全不同的库，只是碰巧名字相似而已。它们可以并行安装，完全独立。

自 2012 年以来，GStreamer 1.x 一直是主要的稳定版系列。GStreamer 0.10 不再维护。

对于 0.10 版本，您需要 0.10 插件和绑定（gst-plugins 0.10.x、gst-ffmpeg 0.10.x、gst-python 0.10.x 等），而对于 1.0 版本，您需要 1.0 插件和绑定（即 gst-plugins-base 1.0.x、gst-plugins-good 1.0.x、gst-plugins-ugly 1.0.x、gst-plugins-bad 1.0.x、gst-ffmpeg 1.0.x、gst-python 1.0.x）。每个主版本的微型版本不必完全匹配，只需主要版本相同即可（例如，当前的 gst-plugins-good 版本可能是 1.0.6，而当前的 GStreamer 核心版本可能是 1.0.13）。GStreamer-1.0 将不会看到或使用任何 GStreamer-0.10 插件，反之亦然。

所有 GStreamer 命令行工具都以主版本为后缀，例如`gst-launch-1.0`和`gst-inspect-1.0`。

应用程序将使用 GStreamer-1.0 或 GStreamer-0.10，因为 1.0 和 0.10 API/ABI 不兼容。

奇数版本，如 0.9.x、0.11.x、1.3.x、1.5.x、1.7.x 等，都是不稳定的开发者版本，一般不应使用。

### 那么，我应该购买哪个版本的 GStreamer 呢？

您应下载 GStreamer 1.x。GStreamer-0.10 已停止维护。

## 如何从源代码安装 GStreamer？

我们在自己的网站[https://gstreamer.freedesktop.org/src/](https://gstreamer.freedesktop.org/src/)上提供了发布版本的压缩包。

从源代码编译时，请确保正确设置`PKG_CONFIG_PATH`环境变量，将其设置为安装前缀 libdir 的 pkgconfig 子目录，以确保在针对 GStreamer 编译时，新安装的 GStreamer 版本能被识别。例如，如果你使用默认前缀（即`/usr/local`）配置 GStreamer，那么你需要

```shell
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig
```

然后再构建 GStreamer 插件模块。在 Linux 上从源代码安装 GStreamer 后，运行`sudo ldconfig`以确保能找到新的库。

请注意，从源代码构建 GStreamer 并不容易，因为它有来自多个模块的许多相互关联的部分，这些模块必须全部正确安装，而且必须以正确的版本找到彼此。

如果你已经从软件包中安装了 GStreamer，强烈建议你寻找更新的软件包，而不是从源码安装，或者升级到有更新软件包的发行版。从源代码安装到一个前缀，而发行版软件包安装到另一个前缀，如果操作不当，可能会导致问题，而且很难有人能为这种设置提供帮助。

## 有预制二进制文件吗？

是的，我们目前为 Windows、OS/X、Android 和 iOS 提供[预编译软件包](https://gstreamer.freedesktop.org/pkg/)。

我们目前不为 Linux 发行版提供软件包，而是依靠发行版来提供。GStreamer 软件包应适用于所有主要（和次要）发行版。

## 为什么不为 XY 发行版提供预制二进制文件？

GStreamer 由志愿者运营。所提供的软件包都是由无偿人员利用自己的时间制作的。我们提供二进制文件支持的发行版都是由我们的志愿者制作的二进制文件。如果你有兴趣为其他发行版或Unices维护GStreamer二进制文件，我们将非常欢迎。请通过 GStreamer-devel 邮件列表联系我们。

## 我的 LFS 安装在编译 GStreamer 时遇到困难，为什么？

如果您使用的是 LFS，我们的基本观点是您应该有足够的知识来自行解决任何构建问题。 作为志愿者，我们当然不能承诺为任何人提供支持，但如果您使用的是 LFS，请将自己视为额外的不支持者。我们既不能也不想知道您的系统是如何配置的，因此无法为您提供帮助。不过，如果您访问 irc.openprojects.net 上的 #gstreamer 频道，我们或许可以给您一些一般性的提示和指点。

## 如何通过 git 获取 GStreamer？

请参阅此页面：[https://gstreamer.freedesktop.org/dev/](https://gstreamer.freedesktop.org/dev/)获取 git 访问（匿名和开发人员）。
