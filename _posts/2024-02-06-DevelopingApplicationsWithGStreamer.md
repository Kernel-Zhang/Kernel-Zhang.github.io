---
title: 常见问题：使用GStreamer开发应用
author: kernel
date: 2024-02-06 13:51:00 +0800
categories: [gstreamer, 常见问题]
tags: [gst常见问题, gstreamer, 多媒体]
---

## 如何编译使用 GStreamer 的程序？

这取决于您的开发环境和目标操作系统。以下主要针对 Linux/unix 设置。

GStreamer 使用`pkg-config`工具为应用程序提供正确的编译器和链接器标志。`pkg-config`是一个标准的编译工具，在 unix 系统中广泛用于定位库和检索编译设置。如果你已经熟悉了它，那么基本上就可以使用了。

如果你不熟悉`pkg-config`，要编译和链接一个小的单文件程序，可向`pkg-config` 传递`--cflags`和`--libs`参数。 对于仅有 gstreamer 的程序来说，以下参数就足够了：

```shell
libtool --mode=link gcc `pkg-config --cflags --libs gstreamer-1.0` -o myprog myprog.c
```

如果您的应用程序也使用 GTK+ 3.0，您可以使用

```shell
libtool --mode=link gcc `pkg-config --cflags --libs gstreamer-1.0 gtk+-3.0` -o myprog myprog.c
```

这些是反引号（在美国键盘上与 “~” 键在同一个键上），而不是单引号。

对于大型项目，应在 Makefile 中集成`pkg-config`的使用，或使用 pkg.m4 宏（提供`PKG_CONFIG_CHECK`）与 autoconf 结合使用。

## 如何在开发环境中针对 GStreamer 副本进行开发？

你可以在开发环境中根据 GStreamer 及其插件的副本进行开发和编译，例如，根据 git 检出进行开发和编译。 这样，你就可以测试 GStreamer 的最新版本，而不会影响整个系统的安装。请参阅 "[使用 meson 从源代码构建](https://gstreamer.freedesktop.org/documentation/installing/building-from-source-using-meson.html?gi-language=c)"文档。

## 如何使用 GConf 获取全系统默认值？

GStreamer 曾经有基于 GConf 的元素，但在 2011 年，`GConf`本身被弃用，转而使用`GSettings` 之后，这些元素也被移除。

如果需要自动音频/视频汇，可考虑使用`autovideosink`和`autoaudiosink`元素。

当你在开发环境中使用 libtool 将程序与 GStreamer 链接时，会制作有趣的 shell 脚本来修改共享对象搜索路径，然后运行你的程序。例如，要调试`gst-launch`，请尝试

```shell
libtool --mode=execute gdb /path/to/gst-launch
```

如果不起作用，可能是使用的 libtool 版本有问题。

如果使用 Meson 编译系统构建 GStreamer，就不能使用 libtool，这不是问题。你可以直接在 Meson 在构建目录下创建的二进制文件上运行`gdb`、`valgrind`或其他调试工具。

## 为什么 gstreamer-devel 上的邮件流量这么低？

我们协调和讨论的主要平台是 IRC 和 Gitlab，而不是邮件列表。请在 irc.oftc.net 上的[`#gstreamer`](irc://irc.oftc.net/#gstreamer)中加入我们，我们也有[网络聊天界面](https://webchat.oftc.net/?channels=%23gstreamer)。如果有更大范围的问题或想从更多人那里获得更多意见，向 gstreamer-devel 邮件列表发送邮件也不失为一个好主意。

## GStreamer 使用哪种版本控制方案？

对于公开发布的版本，GStreamer 采用标准的 MAJOR.MINOR.MICRO 版本方案。如果发布的版本主要包含错误修复或增量更改，则递增 MICRO 版本。如果版本包含较大改动，则会递增 MINOR 版本。MAJOR 版本的变更表示不兼容的 API 或 ABI 变更，这种情况极少发生（最近一次可追溯到 2012 年）。这也被称为[语义版本控制](https://semver.org/)。

MINOR 数字是偶数表示稳定版本：1.0.x、1.2.x、1.4.x、1.6.x、1.8.x 和 1.10.x 是我们的稳定版本系列。奇数 MINOR 编号用于不稳定的开发版本和先行版本，这些版本只能暂时用于测试；非常感谢您帮助测试这些压缩包和软件包！

在开发周期中，GStreamer 还会使用第四个数字或 NANO 数字。如果这个数字是 1，那么它就是 git 开发版本。任何 Nano 编号为 1 的压缩包或软件包都是从 git 制作的，因此不受支持。此外，如果你不是从 GStreamer 团队获得这个软件包或压缩包的，就不要对它寄予厚望了。

## GStreamer 代码的编码风格是什么？

基本上，核心模块和几乎所有插件模块都使用 K&R，缩进为 2 格。只需遵循已有的规则即可。我们只要求代码文件缩进，标题可以手动缩进以提高可读性。请使用空格缩进，而不是制表符，即使在头文件中也是如此。

gst-plugins-\* 中的单个插件或您希望考虑添加到这些模块中的插件应使用相同的样式。如果所有内容都保持一致，就会更容易。当然，一致性是我们的目标。

确保您遵循我们的编码风格的一种方法是使用以下选项通过 GNU Indent 运行您的代码（记住，只运行`*.c`文件，不运行头文件）：

```shell
indent \ 
    --braces-on-if-line \ 
    --case-brace-indentation0 \ 
    --case-indentation2 \ 
    --braces-after-struct-decl-line \ 
    --line-length80 \ 
    --no-tabs \ 
    --cuddle-else \ 
    --dont-line-up-parentheses \
    --continuation-indentation4 \
    --honour-newlines \ 
    ---tab-size8 \ 
    --indent-level2
```

工具目录下的 GStreamer 核心源代码树中还有一个`gst-indent`脚本，它封装了 GNU Indent 并使用正确的选项。

获得正确缩进的最简单方法可能是根据 git 检出进行开发。本地 git 提交钩子将确保正确的缩排。

注释应使用`/* ANSI C 注释样式 */`，代码一般应与 ANSI C89 兼容，因此请在代码块开头声明所有变量等。

合并请求最好针对 git 主版本或最近发布的版本。 请不要将补丁发送到邮件列表。它们很可能会在那里丢失。

更多详情，请参阅[如何提交补丁](https://gstreamer.freedesktop.org/documentation/contribute/index.html?gi-language=c#how-to-submit-patches)。

## 如何收录我的译文？

我将模块中的一个 .po 文件翻译成了一种新语言。如何将其包含在内？

GStreamer 翻译由[翻译项目](https://translationproject.org/)统一管理。有关如何加入翻译项目团队和提交新译文的说明，请访问 https://translationproject.org/html/translators.html。

通过翻译项目提交的新译文，会由维护者在准备新版本时通过在不同模块中运行`make download-po`定期合并到 git 中。

我们不会直接合并新的翻译或翻译修正，一切都必须通过翻译项目进行。
