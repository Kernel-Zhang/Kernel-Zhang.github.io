---
title: gintro——GTK4和GTK3高级绑定
author: kernel
date: 2023-11-17 13:34:00 +0800
categories: [Nim语言, 包使用]
tags: [Nim语言, Nim包应用, 语言交互]
---

[Nim](https://nim-lang.org/)是一种现代通用编程语言。

[GTK](https://www.gtk.org/) 也称为`Gimp`工具包，现在有时也称为`Gnome`工具包，是一个图形用户界面库。

虽然 GTK 最初是作为跨平台图形用户界面工具包而设计和宣传的，但它目前主要用于Linux和其他类似Unix的操作系统。 大多数 Linux 发行版都包含 GTK，一些发行版还将 GTK 用作默认桌面环境，通常与 Gnome 环境或其他窗口管理器一起使用。 虽然 GTK2 应用程序（如GIMP）仍在Windows上使用，但目前似乎只有极少数 GTK3 应用程序用于 Windows 或 MacOSX 。 当您为 Linux 或其他类似 Unix 的操作系统开发自由开放源码软件(FOSS) 时，GTK3 对您来说是一个不错的选择。只要稍加努力，您甚至可以将应用程序移植到专有的 Windows 或 MacOSX 操作系统上。但是，如果您的主要目标平台是 Windows 和 MacOSX，而且您希望获得真正的本地外观和感觉，那么您可能会在 Nim 软件仓库中找到更合适的软件包。 此外，如果您只需要在 Windows 和 MacOSX 上非常容易安装的最小限制图形用户界面，那么您可能会在 Nim 软件包仓库中找到更合适的软件包。GTK 目前完全不支持 Android 操作系统。

> 建议：
您可以在 GIntro README 网站上获取本文档的更详细的副本，其中包含深色源代码背景。 本文档主要介绍了这些绑定在 GTK3 中的使用。对于 GTK4，您也可以查阅 <https://ssalewski.de/gtkprogramming.html> 上的 Nim GTK4 书籍预印本。

> 警告：
请不要使用本页示例中的代码，而是使用 <https://github.com/StefanSalewski/gintro/tree/master/examples> 中的实际代码。github readme.adoc 不允许插入文件中的代码，所以我不得不手动插入，因此 html 页面中的代码可能无法编译到最新的 gintro 版本。我计划在其他地方创建一个新页面，直接从文件中插入代码。

> 备注：
稍后，我们将在这个位置插入一张漂亮的 Nim GTK3 图形用户界面图片。但这样的图片并不能真正证明图形用户界面工具包的质量——具体的例子可能看起来不错，但工具包在其他环境下看起来要差得多，而且远远不能满足现实生活中的所有需要。

> 提示：
正如用户 zetashift 最近在#24中报告的那样，至少在 Windows 10 中安装 GTK3 库似乎并不难：

```
Sketch of GTK3 install for Windows 10:
For the GTK libs I did according these instructions(https://www.gtk.org/download/windows.php):
Install MSYS2
In the msys2 cmd I entered:
pacman -S mingw-w64-x86_64-gtk3
Then for some other necessary depencies(girepository.dll) you need to do:
pacman -S mingw-w64-x86_64-python3-gobject

Additional, you have to install the separate GtkSourceView lib in a similar manner from
https://github.com/Alexpux/MINGW-packages/blob/master/mingw-w64-gtksourceview3/
```

虽然 GTK3 的低级 Nim 绑定早在几年前就已可用，但本产品试图提供真正的高级绑定，具有完全的类型安全性、完全的垃圾收集器(GC) 支持和惯用的应用程序编程接口(API)。

目前，至少有 3 个 Nim GTK3 绑定源：

-   [https://github.com/ngtk3](https://github.com/ngtk3)（过时，已删除）
    
-   [https://github.com/StefanSalewski/oldgtk3](https://github.com/StefanSalewski/oldgtk3)
    
-   [https://github.com/StefanSalewski/gintro](https://github.com/StefanSalewski/gintro)
    

ngtk3 是为 Nim 提供 GTK3 支持的首次尝试。它包含所有 GTK 相关库的单一软件源，但不受 nimble 软件包管理器的支持。它由 GTK 3.20 头文件创建，现已废弃。

oldgtk3 是 ngtk3 移植到 GTK 3.22 的版本 - 加入了所有库并提供敏捷支持。有些人可能仍然喜欢使用 oldgtk3。由于它是使用 Nim 工具 c2nim 直接从 C 头文件生成的，无需太多人工干预，因此应该是完整的，包含的 bug 也不多。缺少垃圾回收器支持通常不是什么大问题，因为部件通常被放入容器中，而且由于 GTK 的引用计数，它们会连同其父级部件一起被自动删除。

尽管如此，我们仍然需要一些真正的高级绑定，因此本 gintro 软件库尝试提供这些绑定。

高级 GTK3 绑定（可用于C++、Python、Ruby或D等其他编程语言）具有这些优势：

-   完全支持垃圾回收器或析构器 - 您无需手动释放资源
    
-   部件是 Nim 对象，因此可以使用继承和子类化功能
    
-   全类型安全--无需强制转换或其他不安全和危险的操作
    

这些高级绑定基于`GObject-Introspection`，这是一种基于XML数据库的接口描述。与C语言头文件相比，这种描述为我们提供了更多更深入的数据类型和函数调用信息，例如对象的所有权转移和过程变量的进出方向，这使得编写粘合代码变得更加容易。它只需进行极少的修改就可用于即将推出的 GTK4。

遗憾的是，它也有一些缺点：

-   应用程序编程接口（API）与C语言 API 不同，因此使用C语言示例或C语言教程并不简单
    
-   高级源代码将不同于现有的C示例，因此对教程的需求很大
    
-   我们需要大量的胶水代码，而这些代码有很大的漏洞空间。因此有必要进行大量测试。
    
-   间接调用会产生一些开销，导致代码量增加，但性能损失极小。
    

> 备注：
新的软件包名称是gintro，是`GObject-Introspection`的缩写。以前的名称是nim-gi，但软件包名称中的连字符和 nim 前缀都已废弃。

## 这些绑定的当前状态

我们仍处于早期阶段，但它已不仅仅是一个概念验证。GTK 和相关库有成千上万的可调用函数和几乎同样多的数据类型。对于一个资源有限的小团队来说，测试所有这些几乎是不可能的。 最初的方法是生成低级绑定，这看起来与`c2nim`工具从C头文件生成的绑定类似。之后，我们将所有C结构和GObject数据类型与 Nim 代理对象关联起来。这些代理对象与低级C数据类型之间定义明确的关系应能确保完全自动的垃圾回收。这可以通过智能类型转换来实现，例如，`glib`库返回的C字符串会分配给新创建的 Nim 字符串，而C字符串的内存则会自动释放。在大多数情况下，这似乎是可行的。但也有一些更复杂的情况，例如函数可能返回整个C字符串数组或其他非基本数据类型，或者函数参数或结果可能是所谓的glists，即`glib`库的列表结构。这些情况无法自动处理，需要手动仔细调查。可能还会有函数和数据类型丢失的情况：GObject-Introspection 查询为我们提供了成千上万行的 Nim 接口代码，但是否有遗漏以及遗漏了什么并不十分明显。 一些函数和数据类型肯定是遗漏了——至少是一些低级函数和数据类型，`GObject-Introspection` 认为这些函数和数据类型对于高级绑定来说是不需要的。 但也许还有更多的遗漏，我们必须对此进行调查。到目前为止，这些绑定仅在使用 GTK 3.24 的 64 位 Linux 系统上进行了测试。

这些基本库已经过部分测试：

Gtk、Gdk、GLib、GObject、Gio、GdkPixbuf、GtkSource、Pango、PangoCairo、PangoFT2、GModule、Rsvg、fontconfig、freetype2、xlib、Atk、Vte、cairo

在最好的情况下，应该可以在这个列表中添加更多基于 GObject 的库，而无需对生成器源代码进行较大的修改。不幸的是，`GObject-Introspection` 提供的cairo绘图库的绑定只是一个最小的存根，我们必须手动扩展它。

## 如何试用

当然，您需要安装一个可正常运行的 Nim，并使用最新版本的编译器，还必须确保系统中已安装 GTK 和相关库。对于某些主要提供预编译软件的 Linux 发行版，您可能还需要安装一些 GTK 相关的开发人员文件。

使用最新的 nimble 版本（>= v0.8.10），只需在 shell 窗口中键入即可：

```shell
nimble install gintro
```

> 备注：
最新版本的 gintro 软件包使用 oldgtk3 软件包中的一些文件进行引导。我们假设 gintro 的用户一般对低级的 oldgtk3 软件包不感兴趣，因此我们尝试只从 oldgtk3 软件包中下载 3 个文件。如果有 wget 或 nimgrab 可执行文件，这样做应该可行。如果失败，你会得到一个较长的错误信息，这可能有助于你解决问题。

> 备注：
Nimble准备时间约 20 秒，编译并执行生成器程序`gen.nim`。遗憾的是，我们不能保证生成器命令能够真正生成所有需要的模块。编译过程在很大程度上取决于操作系统和安装的 GTK 版本。对于安装了 GTK 3.24 和所有必需依赖项的 64 位 Linux 系统，应该可以正常工作。对于从未发布过的 GTK 版本，如果该 GTK 版本引入了新的未知数据类型（如数组容器），则可能会失败。基于 GObject-Introspection 的生成过程会根据执行生成器的操作系统生成定制的绑定，因此对于较旧的 GTK 版本或 32 位系统会生成不同的文件。稍后，我们也会为不同的操作系统和 GTK 版本提供预生成文件，但在可能的情况下，我们更倾向于在本地生成

## 几个基本例子

> 备注
目前，我们没有安装示例程序。如果您想试用它们，必须将示例程序的源代码从<https://github.com/StefanSalewski/gintro/tree/master/examples>复制到本地计算机，也许可以复制到 `/tmp/gintro/examples` 目录。

然后，您可以在 shell 中编译和运行它们，命令如下：

```shell
cd /tmp/gintro/examples/
nim c app0.nim
./app0
```

或在您最喜欢的 Nim IDE 或编辑器中打开源代码文件。但不建议从这份说明文件中获取源代码，因为这些源代码列表可能不是最新版本。

GTK3 程序仍然可以使用旧的GTK2设计，即首先初始化 GTK 库，然后创建部件，最后进入 GTK 主循环。 这种风格仍然在许多教程中使用，如[Zetcode 教程](https://zetcode.com/gui/gtk2/)或 A. Krause 的 GTK 书籍。 或者，您也可以使用新的 GTK3 App 风格，这通常是较新的 GTK 原始文档所推荐的。 不幸的是，GTK3 原始文档大多仅限于 GTK3 API 文档，该文档通常非常好，但对于初学者来说，开始使用 GTK 并不容易。这里有 API 文档和一些基本介绍：

-   [https://www.gnome.org/](https://www.gnome.org/)
    
-   [https://www.gtk.org/](https://www.gtk.org/)
    
-   [https://developer.gnome.org/](https://developer.gnome.org/)
    
-   [https://developer.gnome.org/gtk3/stable/](https://developer.gnome.org/gtk3/stable/)
    
-   [https://developer.gnome.org/gtk3/stable/ch01s04.html#id-1.2.3.12.5](https://developer.gnome.org/gtk3/stable/ch01s04.html#id-1.2.3.12.5)
    
-   [https://developer.gnome.org/gnome-devel-demos/stable/c.html.en](https://developer.gnome.org/gnome-devel-demos/stable/c.html.en)
    

> 提示：
如果您决定继续使用 GTK 开发软件，那么可以考虑安装所谓的`devhelp`工具。它能让您方便快捷地访问 GTK API 文档。例如，如果您想在 GUI 中使用`Button Widget`，并想了解更多相关函数和信号的信息，只需在该工具中输入`Button`，就会引导您查阅所有相关信息。

我们从一个最简单的传统旧式例子开始，这应该是我们大多数人都熟悉的：

t0.nim

```nim
# nim c t0.nim
import gintro/[gtk, gobject]

proc bye(w: Window) =
  mainQuit()
  echo "Bye..."

proc main =
  gtk.init()
  let window = newWindow()
  window.title = "First Test"
  window.connect("destroy", bye)
  window.showAll
  gtk.main()

main()
```

这是 GTK2 程序的传统布局。使用这种布局时，重要的是在一开始就调用`gtk.init()`来初始化 GTK 库。然后我们创建所需的部件、连接信号、显示所有部件，最后通过调用`gtk.main` 进入 GTK 主循环。关于连接信号，我们很快就会学到更多，现在唯一重要的是，我们必须在这里连接 destroy 信号，以便用户通过点击窗口关闭按钮来终止程序的执行。

现在是一个真正简约但完整的 App 风格示例，显示一个空窗口。

> 备注：
所有这些示例的源代码都包含在 examples 目录中。遗憾的是，github似乎不允许将源代码直接包含在本文档中，因此此处显示的源代码与 examples 目录中的源代码可能存在微小差异。

app0.nim

```nim
# app0.nim -- minimal application style example
# nim c app0.nim
import gintro/[gtk, glib, gobject, gio]

proc appActivate(app: Application) =
  let window = newApplicationWindow(app)
  window.title = "GTK3 & Nim"
  window.defaultSize = (200, 200)
  showAll(window)

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", appActivate)
  discard run(app)

main()
```

在`main proc`中，我们创建一个新的应用程序，并将激活信号连接到我们的`activate proc`，然后创建并显示仍然为空的窗口。

> 备注：
我们正在导入 gtk 和 gio 模块。最初，这两个模块都有一个名为`Application`的数据类型（`gtk.Application` 事实上扩展了 `gio.Application`），因此我们必须使用模块名称前缀，或者我们可以只从 gio 导入真正需要的内容（from gio import ...... ），或者使用排除方式（import gio exept ...... ）。不过，由于 gio.Application 一般不常用，所以我们没有将 gio.Application 重命名为 GApplication。不再有名称冲突。

支持多种设置 widget 参数的方法 - 数字 1 至 6 指的是下面的注释：

```nim
setDefaultSize(window, 200, 200) # (1)
gtk.setDefaultSize(window, 200, 200) # (2)
window.setDefaultSize(200, 200) # (3)
window.setDefaultSize(width = 200, height = 200) # (4)
window.defaultSize = (200, 200) # (5)
window.defaultSize = (width: 200, height: 200) # (6)
```

1.  proc 调用语法
    
2.  可选限定，带模块名前缀
    
3.  方法调用语法
    
4.  命名参数
    
5.  元组赋值
    
6.  指定成员名称的元组赋值

空空如也的窗口实在没什么意思。GTK 和 Gnome 团队在<https://developer.gnome.org/gnome-devel-demos/> 上提供了一些 GTK 示例。 C 演示似乎是最实际、最完整的，而且很容易移植到 Nim 上。因此，我们从这些开始，但如果您熟悉其他列出的语言，也可以尝试将它们移植到 Nim 中。让我们从<https://developer.gnome.org/gnome-devel-demos/3.22/button.c.html.en>开始，因为它仍然简短易懂，但已经展示了一些有趣的主题。

![NimGTK3Button](/assets/img/2023-11-17/NimGTK3Button.png)

C代码如下所示:

button.c

```c
#include <gtk/gtk.h>

/*This is the callback function. It is a handler function which
reacts to the signal. In this case, it will cause the button label's
string to reverse.*/
static void
button_clicked (GtkButton *button,
                gpointer   user_data)
{
  const char *old_label;
  char *new_label;

  old_label = gtk_button_get_label (button);
  new_label = g_utf8_strreverse (old_label, -1);

  gtk_button_set_label (button, new_label);
  g_free (new_label);
}

static void
activate (GtkApplication *app,
          gpointer        user_data)
{
  GtkWidget *window;
  GtkWidget *button;

  /*Create a window with a title and a default size*/
  window = gtk_application_window_new (app);
  gtk_window_set_title (GTK_WINDOW (window), "GNOME Button");
  gtk_window_set_default_size (GTK_WINDOW (window), 250, 50);

  /*Create a button with a label, and add it to the window*/
  button = gtk_button_new_with_label ("Click Me");
  gtk_container_add (GTK_CONTAINER (window), button);

  /*Connecting the clicked signal to the callback function*/
  g_signal_connect (GTK_BUTTON (button),
                    "clicked",
                    G_CALLBACK (button_clicked),
                    G_OBJECT (window));

  gtk_widget_show_all (window);
}

int
main (int argc, char **argv)
{
  GtkApplication *app;
  int status;

  app = gtk_application_new ("org.gtk.example", G_APPLICATION_FLAGS_NONE);
  g_signal_connect (app, "activate", G_CALLBACK (activate), NULL);
  status = g_application_run (G_APPLICATION (app), argc, argv);
  g_object_unref (app);

  return status;
}
```

只需掌握一些基本的C语言和 Nim 知识，就可以直接将其转换为 Nim，而且 Nim 并不强迫我们将其形状转换为纯面向对象（OO）语言的所有已知类。我们可以使用 Nim 工具`c2nim`来帮助我们完成转换，也可以手动完成转换。事实上，`c2nim`可以帮助我们将C语言源代码转换为 Nim 语言。大多数时候它都能很好地工作。就我个人而言，我通常会对C语言文件进行预处理，例如删除过于奇怪的宏和宏定义，或将奇怪的结构（如C的for循环替换为更简单的while循环）。然后，我将`c2nim`应用于C文件，最后手动逐行比较结果并微调 Nim 代码。不过，对于这个简短的源代码文本，我们可以手动完成所有这些工作，最后得到这样的结果：

button.nim

```nim
# nim c button.nim
import gintro/[gtk, glib, gobject, gio]

proc buttonClicked (button: Button) =
  button.label = utf8Strreverse(button.label, -1)

proc appActivate (app: Application) =
  let window = newApplicationWindow(app)
  window.title = "GNOME Button"
  window.defaultSize = (250, 50)
  let button = newButton("Click Me")
  window.add(button)
  button.connect("clicked",  buttonClicked)
  window.showAll

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", appActivate)
  discard app.run

main()
```

同样，我们已经从app0.nim示例中了解了基本形状：`Main proc`创建应用程序，连接到激活信号，最后运行应用程序。当 GTK 启动应用程序并发出`activate`信号时，我们的激活过程就会被调用，它将创建一个包含按钮部件的主窗口。该按钮再次与一个信号相连，在本例中名为`clicked`。当鼠标点击该按钮时，GTK 就会发出该信号，并调用我们提供的`buttonClicked()`过程。与信号相连的过程被称为回调，通常会将发出信号的部件作为第一个参数。它们还可以获得任意类型的第二个可选参数——我们将在后面的示例中看到这一点。这里的回调只获取按钮本身作为参数，它的任务是反转按钮显示的文本。这并不是什么有趣的事情，但我们确实使用了glib函数`utf8Strreverse()`来完成这项任务。在C语言中，我们必须释放返回`cstring` 的内存，而在我们的 Nim 示例中，Nim 的垃圾回收器会自动完成这项工作。当你仔细比较我们的示例和C代码时，你可能会注意到一个不同之处。C代码将包含按钮的窗口作为附加参数传递给回调函数，但该参数实际上并没有被使用。在下面的示例中，您将了解如何以类型安全的方式传递（几乎）任意参数。 另一个不同之处是，C代码向操作系统返回了`g_application_run()`返回的整数状态值。我们也可以通过使用 Nim系统模块的`quit() proc`来完成同样的操作，但由于这不会给我们带来额外的好处，所以我们干脆忽略它。

> 提示：
`nim c sourcetext.nim`命令会生成一个包含运行时检查和调试代码的可执行文件，这会增加可执行文件的大小并降低性能。 在仔细测试过您的软件后，您可以使用附加参数`-d:release`来避免这种情况。对于`gcc`后端，可以额外启用链接时优化（LTO），这将进一步减少可执行文件的大小。要启用 LTO，您可以在源代码目录下放入一个`nim.cfg`文件，内容如下：

```
path:"$projectdir"
nimcache:"/tmp/$projectdir"
gcc.options.speed = "-march=native -O3 -flto -fstrict-aliasing"
```

经过优化后，可执行文件的大小应仅在 50 kB 左右！

## 可选的，类型安全的回调参数

下一个示例展示了我们如何向连接程序传递（几乎）任意参数。 我们传递一个字符串、一个堆栈中的对象、一个指向堆上分配对象的引用，最后传递一个部件（在本例中是应用程序窗口本身，您也可以尝试传递另一个按钮）。由于主窗口本身是一个所谓的 GTK `bin`，只能包含一个子窗口部件，因此我们要创建一个容器部件（本例中是一个垂直方框），在方框中填充一些按钮，然后将方框添加到窗口中。

从命令行编译并启动该示例，然后观察点击按钮时会发生什么。

connect\_args.nim

```nim
# nim c connect_args.nim
import gintro/[gtk, glib, gobject, gio]

type
  O = object
    i: int

proc b1Callback(button: Button; str: string) =
  echo str

proc b2Callback(button: Button; o: O) =
  echo "Value of field i in object o = ", o.i

proc b3Callback(button: Button; r: ref O) =
  echo "Value of field i in ref to object O = ", r.i

proc b4Callback(button: Button; w: ApplicationWindow) =
  if w.title == "Nim with GTK3":
    w.title = "GTK3 with Nim"
  else:
    w.title = "Nim with GTK3"

proc appActivate (app: Application) =
  var o: O
  var r: ref O
  new r
  o.i = 1234567
  r.i = 7654321
  let window = newApplicationWindow(app)
  let box = newBox(Orientation.vertical, 0)
  window.title = "Parameters for callbacks"
  let b1 = newButton("Nim with GTK3")
  let b2 = newButton("Passing an object from stack")
  let b3 = newButton("Passing an object from heap")
  let b4 = newButton("Passing a Widget")
  b1.connect("clicked",  b1Callback, "is much fun.")
  b2.connect("clicked",  b2Callback, o)
  b3.connect("clicked",  b3Callback, r)
  b4.connect("clicked",  b4Callback, window)
  box.add(b1)
  box.add(b2)
  box.add(b3)
  box.add(b4)
  window.add(box)
  window.showAll

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", appActivate)
  discard app.run

main()
```

为了证明类型安全，我们可以修改其中一个回调过程，并观察编译器的输出结果：

```nim
proc b1Callback(button: Button; str: int) =
  discard # echo str
```

```
connect_args.nim(37, 5) template/generic instantiation from here
gtk.nim(-15021, 10) Error: type mismatch: got (ref Button:ObjectType, string)
but expected one of:
proc b1Callback(button: Button; str: int)
```

编译器想告诉我们什么可能并不总是很明显，但至少告诉我们它得到了一个字符串，而预期得到的是一个 int。

目前，连接功能由一个 Nim 类型安全宏实现。Connect 接受两个或三个参数——部件、信号名称和可选参数。如果可选参数是 ref（对堆上对象的引用），则以引用形式传递，否则传递参数的深度副本。在上述代码中，这意味着`r`和`window`变量是作为引用传递的，而字符串和堆栈对象则是深度复制的。目前无法再次释放已传递参数的内存。这应该不是什么大问题，因为在大多数情况下根本不会传递参数，即使传递了参数，它们的大小一般也很小，比如普通数字或字符串，或者可能是部件的引用，而这些部件是图形用户界面的一部分，根本无法释放。以后我们可能会添加更多连接宏的变体。

> 备注：
对于初学者来说，导航可能很难。您可能已经掌握了 GTK 的基本知识，并希望为自己的应用程序创建一个图形用户界面。但如何找到所需的内容呢？目前，我们没有提供单独的自动生成 API 文档，因为这实际上没有什么帮助。在大多数情况下，只需猜测 Nim 符号名称、proc 参数和所有其他信息即可。使用具有良好`nimsuggest`支持的智能编辑器可以进一步支持导航，例如，当我们将光标移动到一个过程名称上时，`NEd`会显示所有需要的过程参数，或者我们按`Ctrl+W`跳转到该符号的定义。对于未定义的东西，原始的C函数名通常是一个很好的起点。 假设您对 GTK 的按钮不甚了解，但您知道您想在 GUI 应用程序中添加一个按钮。GTK 通常提供名称中包含`new`字符串的生成器函数，因此很容易猜到存在一个名为`gtk_button_new` 的C函数。这个名字也包含在绑定文件中，本例中是在`gtk.nim`中。因此，我们用文本编辑器打开该文件并搜索该术语。这样就很容易找到相关程序和数据类型的起始点。记住 GTK `devhelp`工具，并使用`grep`或`nimgrep`变体。

## 扩展和子类化 Widget

我们可能希望通过扩展或子类的方式为 GTK widget 附加额外的信息。为此，我们不仅要为每个部件类提供相应的 new() 过程（返回新创建的部件），还要提供 init() 过程（获取（扩展）部件类型的未初始化变量作为参数，并用新创建的 GTK 部件初始化该变量）。 要做到这一点，可以为每个部件类提供一个额外的 new() 过程，它将类型描述符作为第一个参数，就像下面示例中的`newButton(CountButton, "Counting down from 100 by 5")`。添加字段的初始化由用户单独完成。下面的代码显示了一个带有计数器成员字段的 GTK 按钮。每次点击按钮，计数器都会减少。减少的数量（5）作为 int 参数传递给回调。

最近的测试证明，提供自定义析构函数实际上已不再需要，请参见 <https://forum.nim-lang.org/t/7360#46632>。

自 gintro 0.7.1 版起，当使用编译选项`--gc:arc`时，我们支持析构函数。要销毁子类部件，我们必须创建一个`=destroy()`过程，如下代码所示。这看起来可能有点冗长，但这只是为了避免在程序执行过程中多次创建和销毁部件时出现内存泄露。大多数部件都是在启动时创建的，并一直存活到程序终止，因此即使没有匹配的 destroy，也不会出现明显的内存泄漏。（在`examples/gtk3`中有一个名为`subclassArcDestructorTest.nim`的扩展文件，用于测试析构行为）。

count\_button.nim

```nim
# nim c count_button.nim
import gintro/[gtk, gobject, gio]

type
  CountButton = ref object of Button
    counter: int

when defined(gcDestructors):
  proc `=destroy`(x: var typeof(CountButton()[])) =
    gtk.`=destroy`(typeof(Button()[])(x))

proc buttonClicked (button: CountButton; decrement: int) =
  dec(button.counter, decrement)
  button.label = "Counter: " & $button.counter
  echo "Counter is now: ", button.counter

proc appActivate (app: Application) =
  #var button: CountButton
  let window = newApplicationWindow(app)
  window.title = "Count Button"
  #initButton(button, "Counting down from 100 by 5") # deprecated
  let button = newButton(CountButton, "Counting down from 100 by 5")
  button.counter = 100
  window.add(button)
  button.connect("clicked", buttonClicked, 5)
  window.showAll

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", appActivate)
  discard app.run

main()
```

在本例中，我们必须首先定义新的 widget 类型，然后声明该类型的变量，并将该变量传递给 init() 过程。

## CSS 样式、GErrors 和异常

![NimGTK3Label](/assets/img/2023-11-17/NimGTK3Label.png)

GTK 初学者经常会问如何在 GTK widget 上应用自定义样式，例如自定义颜色。 虽然在大多数情况下，使用自定义颜色只会带来难看的结果，因为自定义颜色通常无法与默认配色方案很好地匹配，但了解如何做到这一点还是很有好处的。在 GTK3 中，样式是通过层叠样式表（CSS）应用于部件的。您可以找到类似的 C 示例代码：

label.c

```c
// https://stackoverflow.com/questions/30791670/how-to-style-a-gtklabel-with-css
// gcc `pkg-config gtk+-3.0 --cflags` test.c -o test `pkg-config --libs gtk+-3.0`
#include <gtk/gtk.h>
int main(int argc, char *argv[]) {
    gtk_init(&argc, &argv);
    GtkWidget *window = gtk_window_new(GTK_WINDOW_TOPLEVEL);
    GtkWidget *label = gtk_label_new("Label");
    GtkCssProvider *cssProvider = gtk_css_provider_new();
    char *data = "label {color: green;}";
    gtk_css_provider_load_from_data(cssProvider, data, -1, NULL);
    gtk_style_context_add_provider(gtk_widget_get_style_context(window),
                                   GTK_STYLE_PROVIDER(cssProvider),
                                   GTK_STYLE_PROVIDER_PRIORITY_USER);
    g_signal_connect(window, "destroy", G_CALLBACK(gtk_main_quit), NULL);
    gtk_container_add(GTK_CONTAINER(window), label);
    gtk_widget_show_all(window);
    gtk_main();
}
```

将其转换为 Nim 也很简单：

label.nim

```nim
# nim c label.nim
import gintro/[gtk, glib, gobject, gio]

proc appActivate(app: Application) =
  let window = newApplicationWindow(app)
  let label = newLabel("Yellow text on green background")
  let cssProvider = newCssProvider()
  let data = "label {color: yellow; background: green;}"
  #discard cssProvider.loadFromPath("doesnotexist")
  discard cssProvider.loadFromData(data)
  let styleContext = label.getStyleContext
  assert styleContext != nil
  addProvider(styleContext, cssProvider, STYLE_PROVIDER_PRIORITY_USER)
  window.add(label)
  showAll(window)

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", appActivate)
  discard run(app)

main()
```

在本示例中，我们创建了一个带有文字的普通标签部件。要对其进行着色，我们需要生成一个 CssProvider，并将所需颜色的文字说明加载其中。然后，我们从标签中提取样式上下文，并将 CssProvider 添加到其中。

C函数 `gtk_css_provider_load_from_data()` 的最后一个参数是 GError 类型，可在C代码中用于检测运行时错误。对于 Nim，我们将_GError参数映射为异常。要测试在 Nim 中当 GError 报告错误条件时会发生什么，你可以取消上面代码中函数 loadFromPath() 的注释。由于指定的路径不存在，我们应该会得到一个异常，并有一条信息告诉我们问题所在。当然，在您的实际代码中，您可以使用 Nim 的`try:`块捕获此类异常。（你也可以将上面的 data 变量修改为非法的 CSS 语句——如果语句严重错误，那么你就会从 loadFromData() 中得到异常）。

## 旋转按钮

该部件用于输入数值。我们可以用键盘输入数值、点击 +/- 符号或使用鼠标滚轮。该示例还显示了我们可以为该 widget 使用垂直或水平方向，以及如何使用 bindProperty() 将一个 widget 的属性绑定到另一个 widget。 在这里，我们使用一个按钮来控制旋转按钮的包装行为。

spinbutton.nim

```nim
##  https://github.com/GNOME/gtk/blob/gtk-3-24/tests/testspinbutton.c
##  gcc `pkg-config gtk+-3.0 --cflags` spinbutton.c -o spinbutton `pkg-config --libs gtk+-3.0`

import gintro/[gtk, gdk, glib, gobject]

var numWindows: int

proc onDeleteEvent(w: gtk.Window; event: gdk.Event): bool =
  dec(numWindows)
  if numWindows == 0:
    gtk.mainQuit()
  return EVENT_PROPAGATE # false

proc prepareWindowForOrientation(orientation: gtk.Orientation) =
  let window = newWindow()
  discard connect(window, "delete_event", onDeleteEvent)
  let mainbox = gtk.newBox(if orientation == gtk.Orientation.horizontal: Orientation.vertical else: Orientation.horizontal, 2)
  window.add(mainbox)
  let wrapButton = newToggleButtonWithLabel("Wrap")
  mainbox.add(wrapButton)
  var max = 0
  while max <= 999999999:
    let adj = newAdjustment(max.float, 1, max.float, 1, (max.float + 1) * 0.1, 0)
    let spin = newSpinButton(adj, 1, 0)
    spin.setOrientation(orientation)
    spin.setHalign(gtk.Align.center)
    discard bindProperty(wrapButton, "active", spin, "wrap", {BindingFlag.syncCreate})
    let hbox = newBox(gtk.Orientation.horizontal, 2)
    hbox.packStart(spin, false, false, 2)
    mainbox.add(hbox)
    max = max * 10 + 9
  window.showAll()
  inc(numWindows)

proc main =
  gtk.init()
  prepareWindowForOrientation(gtk.Orientation.horizontal)
  prepareWindowForOrientation(gtk.Orientation.vertical)
  gtk.main()

main()
```

## GTK生成器——使用glade工具创建的用户界面

由于 C 代码可能非常冗长，有些人更喜欢将 GUI 布局外包给 XML 文件，这些文件可以用 glade GUI 创建程序来创建和修改。 对于 Python 或 Nim 这样的高级语言，程序源代码一般都比较简短和干净，因此使用 XML 文件的好处可能不大。当然，我们可以使用 Nim 的 GTK 生成器。我们沿用<https://developer.gnome.org/gtk3/stable/ch01s03.html>中的示例，但对其进行修改以使用新的 GTK3 应用程序样式：在 XML 文件中，我们只需将 class="GtkWindow" 改为 class="GtkApplicationWindow"。我们的 Nim 程序具有众所周知的应用程序形状，但有一点需要补充：我们必须明确地为主窗口设置应用程序。当然，你也可以在 Nim 和 Builder 中使用传统的程序结构，在这种情况下，你可以直接按照链接页面或其他示例进行操作。以下是 XML 文件和 Nim 代码：

builder.ui

```xml
<interface>
  <object id="window" class="GtkApplicationWindow">
    <property name="visible">True</property>
    <property name="title">Grid</property>
    <property name="border-width">10</property>
    <child>
      <object id="grid" class="GtkGrid">
        <property name="visible">True</property>
        <child>
          <object id="button1" class="GtkButton">
            <property name="visible">True</property>
            <property name="label">Button 1</property>
          </object>
          <packing>
            <property name="left-attach">0</property>
            <property name="top-attach">0</property>
          </packing>
        </child>
        <child>
          <object id="button2" class="GtkButton">
            <property name="visible">True</property>
            <property name="label">Button 2</property>
          </object>
          <packing>
            <property name="left-attach">1</property>
            <property name="top-attach">0</property>
          </packing>
        </child>
        <child>
          <object id="quit" class="GtkButton">
            <property name="visible">True</property>
            <property name="label">Quit</property>
          </object>
          <packing>
            <property name="left-attach">0</property>
            <property name="top-attach">1</property>
            <property name="width">2</property>
          </packing>
        </child>
      </object>
      <packing>
      </packing>
    </child>
  </object>
</interface>
```

builder.nim

```nim
# https://developer.gnome.org/gtk3/stable/ch01s03.html
# builder.nim -- application style example using builder/glade xml file for user interface
# nim c builder.nim
import gintro/[gtk, glib, gobject, gio]

proc hello(b: Button; msg: string) =
  echo "Hello", msg

proc quitApp(b: Button; app: Application) =
  echo "Bye"
  quit(app)

proc appActivate(app: Application) =
  let builder = newBuilder()
  discard builder.addFromFile("builder.ui")
  let window = builder.getApplicationWindow("window")
  window.setApplication(app)
  var button = builder.getButton("button1")
  button.connect("clicked", hello, "")
  button = builder.getButton("button2")
  button.connect("clicked", hello, " again...")
  button = builder.getButton("quit")
  button.connect("clicked", quitApp, app)
  #showAll(window)

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", appActivate)
  discard run(app)

main()
```

对于每个构建器组件，gintro 都提供了一个类型安全的访问过程，例如本例中的 getApplicationWindow() 和 getButton()。

一般来说，可以使用与可执行程序合并的资源文件，而不是外部 XML 文件，我们必须研究如何在 Nim 中做到这一点。 也许可以在 XML 文件中将信号处理器与处理程序连接起来——这也是正在进行的工作...

## GAction

GAction 表示单个命名的动作，是 GTK3 实现用户交互的首选方式。GAction 可与按钮、菜单和键盘快捷键一起使用。

下面的示例基于<https://wiki.gnome.org/HowDoI/GAction>：

gaction.nim

```nim
# https://wiki.gnome.org/HowDoI/GAction
# nim c gaction.nim
import gintro/[gtk, glib, gobject, gio]

proc saveCb(action: SimpleAction; v: Variant) =
  echo "saveCb"

proc appActivate(app: Application) =
  let window = newApplicationWindow(app)
  let action = newSimpleAction("save")
  discard action.connect("activate", saveCB)
  window.actionMap.addAction(action)
  let button = newButton()
  button.label = "Save"
  window.add(button)
  button.setActionName("win.save")
  setAccelsForAction(app, "win.save", "<Control><Shift>S")
  showAll(window)

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", appActivate)
  discard run(app)

main()
```

GtkApplicationWindow 为 GActionMap 提供了一个接口。由于接口本身和接口提供者定义在不同的模块中，自动转换是不可能的，因此我们必须将 ApplicationWindow 转换为 ActionMap。(我们可以使用转换器来完成转换，但由于这种转换非常罕见，而且 gintro 至今仍未使用转换器，因此我们使用了显式过程）。在 setAccelsForAction() 过程中使用 cstringArray 作为第三个参数有点难看，我们稍后必须解决这个问题。

## 带有 GActions 的 GMenu

下面的示例展示了如何定义 GActions 并将其绑定到菜单、按钮和键盘快捷键。示例中提供了无状态操作（退出）、切换操作（拼写检查）和有状态操作（文本校正）。

请注意，以下代码并非直接翻译现有示例，而是收集了各种来源的信息，因此可能包含错误或不完全优化的代码。

menubar.nim

```nim
# https://developer.gnome.org/glib/stable/glib-GVariant.html
# https://developer.gnome.org/glib/stable/glib-GVariantType.html
# https://wiki.gnome.org/HowDoI/GMenu
# https://wiki.gnome.org/HowDoI/GAction
# nim c menubar.nim
import gintro/[gtk, glib, gobject, gio]
from strutils import `%`, format

# https://github.com/GNOME/glib/blob/master/gio/tests/gapplication-example-actions.c
proc activateToggleAction(action: SimpleAction; parameter: Variant; app: Application) =
  app.hold # hold/release taken over from C example, there may be reasons...
  block:
    echo format("action $1 activated", action.name)
    let state: Variant = action.state
    let b = state.getBoolean
    action.state = newVariantBoolean(not b)
    echo format("state change $1 -> $2", b, not b)
  app.release

proc activateStatefulAction(action: SimpleAction; parameter: Variant; app: Application) =
  app.hold
  block:
    echo format("action $1 activated", action.name)
    let state: Variant = action.state
    var l: uint64
    let oldState = state.getString(l) # yes uint64 parameter is a bit ugly
    let newState = parameter.getString(l)
    action.state = newVariantString(newState)
    echo format("state change $1 -> $2", oldState, newState)
  app.release

proc quitProgram(action: SimpleAction; parameter: Variant; app: Application) =
  quit(app)

proc appStartup(app: Application) =
  let quit = newSimpleAction("quit") # here we create the actions for whole app
  connect(quit, "activate", quitProgram, app)
  app.addAction(quit)

  let menu = gio.newMenu() # root of all menus
  block: # plain stateless menu
    let subMenu = gio.newMenu()
    menu.appendSubMenu("Application", submenu)
    # let section = gio.newMenu() # no separating section needed here
    # submenu.appendSection(nil, section)
    # section.append("Quit", "app.quit")
    submenu.append("Quit", "app.quit")

  block: #stateful menu with radio items
    let subMenu = gio.newMenu()
    menu.appendSubMenu("Layout", submenu)
    let subMenu2 = gio.newMenu()
    submenu.appendSubMenu("justify", submenu2)
    let section = gio.newMenu()
    submenu2.appendSection(nil, section)
    section.append("left", "win.justify::left")
    section.append("center", "win.justify::center")
    section.append("right", "win.justify::right")

  block: # and finally a toggle menu
    let subMenu = gio.newMenu()
    menu.appendSubMenu("Spelling", submenu)
    let section = gio.newMenu()
    submenu.appendSection(nil, section)
    section.append("Check", "win.toggleSpellCheck")
   # finally add the menubar
    setMenuBar(app, menu)

proc appActivate(app: Application) =
  let window = newApplicationWindow(app)
  window.title = "GTK3 App with Menubar"
  window.defaultSize = (500, 200)
  window.position = WindowPosition.center
  block: # creat the window related actions
    let v = newVariantBoolean(true)
    let spellCheck = newSimpleActionStateful("toggleSpellCheck", nil, v)
    connect(spellCheck, "activate", activateToggleAction, app)
    window.actionMap.addAction(spellCheck)
  block:
    let v = newVariantString("left") # default value and
    let vt = newVariantType("s") # string (value type)
    let justifyAction = newSimpleActionStateful("justify", vt, v)
    connect(justifyAction, "activate", activateStatefulAction, app)
    window.actionMap.addAction(justifyAction)
  let button = newButton()
  button.label = "Justify Center"
  #window.add(button) # do not add it here already: (menubar:10010): Gtk-WARNING **:
  # 22:00:33.230: actionhelper: action win.justify can't be activated due to
  # parameter type mismatch (parameter type s, target type NULL)
  button.setDetailedActionName("win.justify::center")
  #button.setActionName("app.quit") # for a stateless action
  setAccelsForAction(app, "win.justify::right", "<Control><Shift>R")
  window.add(button)
  showAll(window)

proc main =
  let app = newApplication("app.example")
  connect(app, "startup", appStartup)
  connect(app, "activate", appActivate)
  echo "GTK Version $1.$2.$3" % [$majorVersion(), $minorVersion(), $microVersion()]
  let status = run(app)
  quit(status)

main()
```

我们可以轻松修改上述示例，通过标题栏和“齿轮”菜单按钮获得更现代的外观：

gearsmenu.nim

```nim
# https://developer.gnome.org/glib/stable/glib-GVariant.html
# https://developer.gnome.org/glib/stable/glib-GVariantType.html
# https://wiki.gnome.org/HowDoI/GMenu
# https://wiki.gnome.org/HowDoI/GAction
# https://developer.gnome.org/gnome-devel-demos/stable/menubutton.c.html.en
# nim c gearsmenu.nim
import gintro/[gtk, glib, gobject, gio]
import strformat

# https://github.com/GNOME/glib/blob/master/gio/tests/gapplication-example-actions.c
proc activateToggleAction(action: SimpleAction; parameter: Variant; app: Application) =
  app.hold # hold/release taken over from C example, there may be reasons...
  block:
    echo fmt"action {action.name} activated"
    let state: Variant = action.state
    let b = state.getBoolean
    action.state = newVariantBoolean(not b)
    echo fmt"state change {b} -> {not b}"
  app.release

proc activateStatefulAction(action: SimpleAction; parameter: Variant; app: Application) =
  app.hold
  block:
    echo fmt"action {action.name} activated"
    let state: Variant = action.state
    var l: uint64
    let oldState = state.getString(l) # yes uint64 parameter is a bit ugly
    let newState = parameter.getString(l)
    action.state = newVariantString(newState)
    echo fmt"state change {oldState} -> {newState}"
  app.release

proc quitProgram(action: SimpleAction; parameter: Variant; app: Application) =
  quit(app)

proc appStartup(app: Application) =
  echo "appStartup"
  let quit = newSimpleAction("quit") # here we create the actions for whole app
  connect(quit, "activate", quitProgram, app)
  app.addAction(quit)

proc appActivate(app: Application) =
  echo "appActivate"
  let window = newApplicationWindow(app)
  # window.title = "GTK3 App with Headerbar and Gears Menu" # unused due to HeaderBar
  window.defaultSize = (500, 200)
  window.position = WindowPosition.center

  let menu = gio.newMenu() # root of all menus
  block: # plain stateless menu
    let subMenu = gio.newMenu()
    menu.appendSubMenu("Application", submenu)
    # let section = gio.newMenu() # no separating section needed here
    # submenu.appendSection(nil, section)
    # section.append("Quit", "app.quit")
    submenu.append("Quit", "app.quit")

  block: #stateful menu with radio items
    let subMenu = gio.newMenu()
    menu.appendSubMenu("Layout", submenu)
    let subMenu2 = gio.newMenu()
    submenu.appendSubMenu("justify", submenu2)
    let section = gio.newMenu()
    submenu2.appendSection(nil, section)
    section.append("left", "win.justify::left")
    section.append("center", "win.justify::center")
    section.append("right", "win.justify::right")

  block: # and finally a toggle menu
    let subMenu = gio.newMenu()
    menu.appendSubMenu("Spelling", submenu)
    let section = gio.newMenu()
    submenu.appendSection(nil, section)
    section.append("Check", "win.toggleSpellCheck")

  let headerBar = newHeaderBar()
  headerBar.setShowCloseButton
  headerBar.setTitle("Title")
  headerBar.setSubtitle("Subtitle")
  window.setTitlebar (headerBar)

  let menubar = newMenuButton()
  # menubar.setDirection(ArrowType.none) # show the gears Icon
  # let image = newImageFromIconName("open-menu-symbolic", IconSize.menu.ord)
  let image = newImageFromIconName("document-save", IconSize.dialog.ord) # dialog is really big!
  menubar.setImage(image) # this is only an example for a custom image
  # menubar.setIconName("open-menu-symbolic") # only gtk4
  headerBar.packEnd(menubar)
  menubar.setMenuModel(menu)

  block: # creat the window related actions
    let v = newVariantBoolean(true)
    let spellCheck = newSimpleActionStateful("toggleSpellCheck", nil, v)
    connect(spellCheck, "activate", activateToggleAction, app)
    window.actionMap.addAction(spellCheck)
  block:
    let v = newVariantString("left") # default value and
    let vt = newVariantType("s") # string (value type)
    let justifyAction = newSimpleActionStateful("justify", vt, v)
    connect(justifyAction, "activate", activateStatefulAction, app)
    window.actionMap.addAction(justifyAction)
  let button = newButton()
  button.label = "Justify Center"
  button.setDetailedActionName("win.justify::center")
  #button.setActionName("app.quit") # for a stateless action
  setAccelsForAction(app, "win.justify::right", "<Control><Shift>R")
  window.add(button)
  showAll(window)

proc main =
  let app = newApplication("app.example")
  connect(app, "startup", appStartup)
  connect(app, "activate", appActivate)
  echo fmt"GTK Version {majorVersion()}.{minorVersion()}.{microVersion()}"
  let status = run(app)
  quit(status)

main()
```

在上一个示例中，我们只在 proc appStartup() 中为所有应用程序窗口创建了一个菜单实例，而在这里，我们在 proc appActivate() 中为所有实例创建了一个新菜单。这样做似乎没有问题，所以我认为是正确的。

## 使用GTK生成器创建GMenu和GAction

下面是<https://github.com/GNOME/gtk/blob/mainline/tests/>中的一个示例，该示例结合使用了 gaction 和 gmenu 以及 GTK builder XML 文件来描述菜单。

gaction2.nim

```nim
# nim c gaction2.nim
# https://github.com/GNOME/gtk/blob/mainline/tests/testgaction.c
# gcc -Wall gaction.c -o gaction `pkg-config --cflags --libs gtk4`
import gintro/[gtk, glib, gobject, gio]

const menuData = """
<interface>
  <menu id="menuModel">
    <section>
      <item>
        <attribute name="label">Normal Menu Item</attribute>
        <attribute name="action">win.normal-menu-item</attribute>
      </item>
      <submenu>
        <attribute name="label">Submenu</attribute>
        <item>
          <attribute name="label">Submenu Item</attribute>
          <attribute name="action">win.submenu-item</attribute>
        </item>
      </submenu>
      <item>
        <attribute name="label">Toggle Menu Item</attribute>
        <attribute name="action">win.toggle-menu-item</attribute>
      </item>
    </section>
    <section>
      <item>
        <attribute name="label">Radio 1</attribute>
        <attribute name="action">win.radio</attribute>
        <attribute name="target">1</attribute>
      </item>
      <item>
        <attribute name="label">Radio 2</attribute>
        <attribute name="action">win.radio</attribute>
        <attribute name="target">2</attribute>
      </item>
      <item>
        <attribute name="label">Radio 3</attribute>
        <attribute name="action">win.radio</attribute>
        <attribute name="target">3</attribute>
      </item>
    </section>
  </menu>
</interface>
"""

proc changeLabelButton(action: SimpleAction; v: Variant; label: Label) =
  label.setLabel("Text set from button")

proc normalMenuItem(action: SimpleAction; v: Variant; label: Label) =
  label.setLabel("Text set from normal menu item")

proc toggleMenuItem(action: SimpleAction; v: Variant; label: Label) =
  label.setLabel("Text set from toggle menu item")

proc submenuItem(action: SimpleAction; v: Variant; label: Label) =
  label.setLabel("Text set from submenu item")

proc radio(action: SimpleAction; parameter: Variant; label: Label) =
  var l: uint64
  let newState: Variant = newVariantString(getString(parameter, l))
  let str: string = "From Radio menu item " & getString(newState, l)
  label.setLabel(str)

proc bye(w: Window) =
  mainQuit()
  echo "Bye..."

proc main =
  gtk.init()
  let
    window = newWindow()
    box = newBox(Orientation.vertical, 12)
    menubutton = newMenuButton()
    button1 = newButton("Change Label Text")
    label = newLabel("Initial Text")
    actionGroup = newSimpleActionGroup()

  window.connect("destroy", gtk.mainQuit)
  #window.connect("destroy", bye)

  var action = newSimpleAction("change-label-button")
  discard action.connect("activate", changeLabelButton, label)
  actionGroup.addAction(action)

  action = newSimpleAction("normal-menu-item")
  discard action.connect("activate", normalMenuItem, label)
  actionGroup.addAction(action)

  var v = newVariantBoolean(true)
  action = newSimpleActionStateful("toggle-menu-item", nil, v)
  discard action.connect("activate", toggleMenuItem, label)
  actionGroup.addAction(action)

  action = newSimpleAction("submenu-item")
  discard action.connect("activate", subMenuItem, label)
  actionGroup.addAction(action)

  v = newVariantString("1")
  let vt = newVariantType("s")
  action = newSimpleActionStateful("radio", vt, v)
  discard action.connect("activate", radio, label)
  actionGroup.addAction(action)

  insertActionGroup(window, "win", actionGroup)

  label.setMarginTop(12)
  label.setMarginBottom(12)
  box.add(label)
  menubutton.setHAlign(Align.center)
  let builder: Builder = newBuilderFromString(menuData)
  let menuModel = builder.getMenuModel("menuModel")
  let menu = newMenuFromModel(menuModel)
  menuButton.setPopup(menu)
  box.add(menubutton)
  button1.setHalign(Align.center)
  button1.setActionName("win.change-label-button")
  box.add(button1)
  window.add(box)
  window.showAll
  gtk.main()

main()
```

## GSettings

GSettings 提供了一种永久存储配置数据并将其绑定到 widget 属性的便捷方法。

您可以在<https://blog.gtk.org/2017/05/01/first-steps-with-gsettings/>上阅读介绍。

要在自己的程序中使用 GSettings，首先要创建一个 XML 文件，定义每个配置项的名称和类型，并提供默认值和说明。此类 XML 文件的文件名必须以".gschema.xml "结尾。 下面的示例中只有一个名为 like-nim 的布尔类型字段 (b)。对于真正的应用程序，我们需要在计算机上安装配置文件——不幸的是，我们需要 root 访问权限。我们可以这样做：

```
# For making gsettings available system wide one method is, as root
# https://developer.gnome.org/gio/stable/glib-compile-schemas.html
# echo $XDG_DATA_DIRS
# /usr/share/gnome:/usr/local/share:/usr/share:/usr/share/gdm
# cd /usr/local/share/glib-2.0/schemas
# cp test.gschema.xml .
# glib-compile-schemas .
#
```

对于测试，有一种更简便的方法：

创建一个目录，将 xml 文件和下面的测试程序复制到其中。

那就像普通用户一样：

```shell
glib-compile-schemas .
nim c gsettings.nim
GSETTINGS_SCHEMA_DIR="." ./gsettings
```

这是 xml 文件和测试程序：

test.gschema.xml

```xml
<schemalist>
  <schema path="/org/gnome/recipes/"
         id="org.gnome.Recipes">
    <key type="b" name="like-nim">
      <default>false</default>
      <summary>I like Nim</summary>
      <description>
        I like or like not
        the Nim programming language.
      </description>
    </key>
  </schema>
</schemalist>
```

gsettings.nim

```nim
# gsettings.nim -- basic use of gsettings
# nim c gsettings.nim
# https://blog.gtk.org/2017/05/01/first-steps-with-gsettings/
# https://mail.gnome.org/archives/gtk-list/2016-December/msg00003.html
import gintro/[gtk, glib, gobject, gio]

# unused
proc toggle(b: CheckButton) =
  echo b.active
  let s = newSettings("org.gnome.Recipes")
  discard s.setBoolean("like-nim", b.active)

proc appActivate(app: Application) =
  let window = newApplicationWindow(app)
  window.title = "GTK3, Nim and GSettings"
  window.defaultSize = (200, 200)
  let b = newCheckButton()
  b.halign = Align.center
  b.label = "I like Nim"
  #b.connect("toggled", toggle) # we don't need this for plain binding!
  let s = newSettings("org.gnome.Recipes")
  if s.getBoolean("like-nim"):
    echo "I like Nim language"
  `bind`(s, "like-nim", b, "active", {SettingsBindFlag.get, SettingsBindFlag.set})
  window.add(b)
  showAll(window)

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", appActivate)
  discard run(app)

main()
```

命令`glib-compile-schemas .`会编译当前目录下的所有schemas。而`GSETTINGS_SCHEMA_DIR="." ./gsettings` 启动我们的测试程序，环境变量 `GSETTINGS_SCHEMA_DIR` 指向包含编译模式的当前目录。

请注意，有一个与我们的测试程序同名的系统工具——它可以用来获取或设置配置数据——例如，你可以用以下命令查询 "like-nim "字段的当前状态

```shell
gsettings --schemadir "." get org.gnome.Recipes like-nim
```

或者说，测试程序首先创建了一个带有检查按钮的窗口。然后打开设置文件，打印布尔变量的当前值。然后，绑定过程将窗口部件的活动属性（复选标记状态）与设置文件中的 "like-nim "条目绑定。绑定的结果是，我们的校验标记状态将自动变成持久状态，也就是说，当我们终止并重新启动测试程序时，校验标记将再次保持上次的状态。

这些绑定适用于布尔值、整数、浮点数和字符串。widget 属性的类型必须与设置 xml 文件中相应条目的类型一致。

在 Linux 系统中，可以通过添加下面的语句到你的`.bashrc`文件中来永久设置 gsetting 目录，当然要先用实际路径替换 pathToMyProg。

```shell
export GSETTINGS_SCHEMA_DIR="pathToMyProg"
```

有关 gsettings 的更多信息，请参阅：

<https://developer.gnome.org/gio/stable/GSettings.html>

<https://developer.gnome.org/gio/stable/running-gio-apps.html>

## 使用Cairo图形库绘图

下一个示例展示了如何使用 cairo 图形库在 DrawingArea widget 上绘图，同时使用glib的timeoutAdd() 函数创建一个定时器，定时调用绘图函数来创建一些动画。该代码基于最近在 cairo 邮件列表中发布的一篇帖子，显示了一个持续向左移动的正弦波。

> 备注：
由 gobject-introspection 生成的 cairo 模块只是一个最小的存根，因为 cairo 库并不真正支持内省。现在，我们使用 c2nim 工具直接从 cairo C 头文件生成的 cairo 模块，然后进行修改以支持高级 API。

cairo\_anim.nim

```nim
# https://lists.cairographics.org/archives/cairo/2016-October/027791.html
# Nim version of that plain cairo animation example

import gintro/[gtk, glib, gobject, gio, cairo]
import math

const
  NumPoints = 1000
  Period = 100.0

proc invalidateCb(w: Widget): bool =
  queueDraw(w)
  return SOURCE_CONTINUE

proc sineToPoint(x, width, height: int): float =
  math.sin(x.float * math.TAU / Period) * height.float * 0.5 + height.float * 0.5

proc drawingAreaDrawCb(widget: DrawingArea; context: Context): bool =
  var redrawNumber {.global.} : int
  let width = getAllocatedWidth(widget)
  let height = getAllocatedHeight(widget)
  for i in 1 ..< NumPoints:
    context.lineTo(i.float , sineToPoint(i + redrawNumber, width, height))
  context.stroke
  inc(redrawNumber)
  return true # TRUE to stop other handlers from being invoked for the event. FALSE to propagate the event further.

proc appActivate(app: Application) =
  let window = newApplicationWindow(app)
  window.title = "Drawing example"
  window.defaultSize = (400, 400)
  let drawingArea = newDrawingArea()
  window.add(drawingArea)
  showAll(window)
  discard timeoutAdd(1000 div 60, invalidateCb, drawingArea)
  connect(drawingArea, "draw", drawingAreaDrawCb)

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", appActivate)
  discard run(app)

main()
```

## 一个简单的 ListView 示例

![NimGTK3ListView](/assets/img/2023-11-17/NimGTK3ListView.png)

最近有人报告说在将 GTK2 应用程序移植到 Nim GTK3 时遇到了一些问题，因此我将举一个使用 ListViews 和 TreeViews 的小例子，也许会有所帮助。这两种部件类型是 GTK 中最复杂的部件类型——我记得几年前使用 Ruby-GTK 时也遇到过一些问题。由于我现在不记得使用 ListView widget 的细节，我决定从[zetcode.com](https://zetcode.com/gui/gtk2/gtktreeview/)上的示例代码作为起点。当然，移植是很简单的，但当我尝试编译结果时，我发现了一些 bug 和当前 gintro 软件包的限制。当然这并不奇怪，因为软件包还没有经过真正的测试。第一个问题是，我们在 TreeView 中存储了一个 ListStore 作为模型，我们需要从 TreeView 中提取 ListStore 进行一些操作。但目前 gtk.nim 模块只提供了一个提取模型本身的函数，而模型本身的类型是 TreeModel。 在 C 代码中，我们使用了向上转换的方法来从提取的 TreeModel 中获取 ListStore。为了避免在我们的 Nim 代码中进行转换，我只是复制了 getModel() 过程，并将其修改为返回 ListStore。第二个问题是，gio 模块也导出了 ListStore 数据类型，为了避免所有 ListStore 类型都使用 gtk 前缀，我在导入列表中排除了 gio.ListStore。最后还有一个真正的 bug：程序 newListStore() 目前的最后一个参数是一个普通指针，而我们知道它应该是一个 GTypes 列表的地址。 因此，我们现在不得不使用一个丑陋的转换。目前我们使用 GValues 来填充 ListStore。这不是很方便，为此我们需要正确的字符串列表 GType。在 C 语言中，我们可以使用宏`G_TYPE_STRING`，但 gobject-introspection 并没有提供这个宏。因此，我们使用 typeFromName() 来获取正确的 GType，当我们知道字符串名称是 "gchararray "时，这个方法就很好用了。 稍后，我们将为这一过程提供更高级别的函数。

我稍后会尝试给出更多更好解释的 ListView 和 TreeView 示例...

listview.nim

```nim
# https://zetcode.com/gui/gtk2/gtktreeview/
# dynamiclistview.c

import gintro/[glib, gobject, gtk]
import gintro/gio except ListStore

const
  LIST_ITEM = 0
  N_COLUMNS = 1

var list: TreeView

# this is copied from gtk.nim
#proc getModel*(self: TreeView): TreeModel =
#  new(result)
#  result.impl = gtk_tree_view_get_model(cast[ptr TreeView00](self.impl))

proc getListStore(self: TreeView): ListStore =
  new(result)
  result.impl = gtk_tree_view_get_model(cast[ptr TreeView00](self.impl))

proc appendItem(widget: Button; entry: Entry) =
  var
    val: Value
    iter: TreeIter
  let store = getListStore(list)
  let gtype = typeFromName("gchararray")
  discard gValueInit(val, gtype)
  gValueSetString(val, entry.text)
  store.append(iter)
  store.setValue(iter, LIST_ITEM, val)
  entry.text = ""

proc removeItem(widget: Button; selection: TreeSelection) =
  var
    ls: ListStore
    iter: TreeIter
  let store = getListStore(list)
  if not store.getIterFirst(iter):
      return
  if getSelected(selection, ls, iter):
    discard store.remove(iter)

proc onRemoveAll(widget: Button; selection: TreeSelection) =
  var
    iter: TreeIter
  let store = getListStore(list)
  if not store.getIterFirst(iter):
    return
  clear(store)

proc initList(list: TreeView) =
  let renderer = newCellRendererText()
  let column = newTreeViewColumn()
  column.title = "List Item"
  column.packStart(renderer, true)
  column.addAttribute(renderer, "text", LIST_ITEM)
  discard list.appendColumn(column)
  let gtype = typeFromName("gchararray")
  let store = newListStore(N_COLUMNS, cast[pointer]( unsafeaddr gtype)) # cast due to bug in gtk.nim
  list.setModel(store)

proc appActivate(app: Application) =
  let
    window = newApplicationWindow(app)
    sw = newScrolledWindow()
    hbox = newBox(Orientation.horizontal, 5)
    vbox = newBox(Orientation.vertical, 0)
    add = newButton("Add")
    remove = newButton("Remove")
    removeAll = newButton("Remove All")
    entry = newEntry()
  window. title = "List view"
  window.position = WindowPosition.center
  window.borderWidth = 10
  window.setSizeRequest(370, 270)
  list = newTreeView()
  sw.add(list)
  sw.setPolicy(PolicyType.automatic, PolicyType.automatic)
  sw.setShadowType(ShadowType.etchedIn)
  list.setHeadersVisible(false)
  vbox.packStart(sw, true, true, 5)
  entry.setSizeRequest(120, -1)
  hbox.packStart(add, false, true, 3)
  hbox.packStart(entry, false, true, 3)
  hbox.packStart(remove, false, true, 3)
  hbox.packStart(removeAll, false, true, 3)
  vbox.packStart(hbox, false, true, 3)
  window.add(vbox)
  initList(list)
  let selection = getSelection(list)
  connect(add, "clicked", listview.appendItem, entry)
  connect(remove, "clicked", listview.removeItem, selection)
  connect(removeAll, "clicked", listview.onRemoveAll, selection)
  showAll(window)

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", appActivate)
  discard run(app)

main()
```

## 带 CSS 样式的 ListView 示例

最近，C. Eric Cashon 在 <https://discourse.gnome.org/t/gtk-treeview-cell-color-change/1750/3> 上提供了这样一个例子

我也将在这里展示他的原始代码，以便我们可以更好地对 Nim 版本进行比较。 我们可以看到，Nim 代码目前仍然存在一些缺点，例如，我们没有实现 varargs procs，因此属性和属性的设置需要使用 GValues 来完成，虽然类型安全，但并不紧凑。这还不算太糟，但我们可以考虑创建宏，以支持更密集但仍然类型安全的方式，类似于 C 的 varargs 函数。

cell\_color1.c

```c
// gcc -Wall cell_color1.c -o cell_color1 `pkg-config --cflags --libs gtk+-3.0`
// https://discourse.gnome.org/t/gtk-treeview-cell-color-change/1750/4
// C. Eric Cashon

#include<gtk/gtk.h>

enum
{
   ID,
   PROGRAM,
   COLOR1,
   COLOR2,
   COLUMNS
};

int main(int argc, char *argv[])
  {
    gtk_init(&argc, &argv);

    GtkWidget *window=gtk_window_new(GTK_WINDOW_TOPLEVEL);
    gtk_window_set_title(GTK_WINDOW(window), "Select Cell");
    gtk_window_set_position(GTK_WINDOW(window), GTK_WIN_POS_CENTER);
    gtk_window_set_default_size(GTK_WINDOW(window), 500, 500);
    gtk_container_set_border_width(GTK_CONTAINER(window), 20);
    g_signal_connect(window, "destroy", G_CALLBACK(gtk_main_quit), NULL);

    GtkTreeIter iter;
    GtkListStore *store=gtk_list_store_new(COLUMNS, G_TYPE_UINT, G_TYPE_STRING, G_TYPE_STRING, G_TYPE_STRING);
    gtk_list_store_append(store, &iter);
    gtk_list_store_set(store, &iter, ID, 0, PROGRAM, "Gedit", COLOR1, "DarkCyan", COLOR2, "cyan", -1);
    gtk_list_store_append(store, &iter);
    gtk_list_store_set(store, &iter, ID, 1, PROGRAM, "Gimp", COLOR1,  "LightSlateGray", COLOR2, "cyan", -1);
    gtk_list_store_append(store, &iter);
    gtk_list_store_set(store, &iter, ID, 2, PROGRAM, "Inkscape", COLOR1, "DarkCyan", COLOR2, "cyan", -1);
    gtk_list_store_append(store, &iter);
    gtk_list_store_set(store, &iter, ID, 3, PROGRAM, "Firefox", COLOR1, "LightSlateGray", COLOR2, "cyan", -1);
    gtk_list_store_append(store, &iter);
    gtk_list_store_set(store, &iter, ID, 4, PROGRAM, "Calculator", COLOR1, "DarkCyan", COLOR2, "cyan", -1);
    gtk_list_store_append(store, &iter);
    gtk_list_store_set(store, &iter, ID, 5, PROGRAM, "Devhelp", COLOR1, "LightSlateGray", COLOR2, "cyan", -1);

    GtkWidget *tree=gtk_tree_view_new_with_model(GTK_TREE_MODEL(store));
    gtk_widget_set_hexpand(tree, TRUE);
    gtk_widget_set_vexpand(tree, TRUE);
    g_object_set(tree, "activate-on-single-click", TRUE, NULL);

    GtkTreeSelection *selection=gtk_tree_view_get_selection(GTK_TREE_VIEW(tree));
    gtk_tree_selection_set_mode(selection, GTK_SELECTION_SINGLE);

    GtkCellRenderer *renderer1=gtk_cell_renderer_text_new();
    g_object_set(renderer1, "editable", FALSE, NULL);

    GtkCellRenderer *renderer2=gtk_cell_renderer_text_new();
    g_object_set(renderer2, "editable", TRUE, NULL);

    //Bind the COLOR column to the "cell-background" property.
    GtkTreeViewColumn *column1=gtk_tree_view_column_new_with_attributes("ID", renderer1, "text", ID, "cell-background", COLOR1, NULL);
    gtk_tree_view_append_column(GTK_TREE_VIEW(tree), column1);
    GtkTreeViewColumn *column2 = gtk_tree_view_column_new_with_attributes("Program", renderer2, "text", PROGRAM, "cell-background", COLOR2, NULL);
    gtk_tree_view_append_column(GTK_TREE_VIEW(tree), column2);

    GtkWidget *grid=gtk_grid_new();
    gtk_grid_attach(GTK_GRID(grid), tree, 0, 0, 1, 1);

    gtk_container_add(GTK_CONTAINER(window), grid);

    gchar *css_string=g_strdup("treeview{background-color: rgba(0,255,255,1.0); font-size:30pt} treeview:selected{background-color: rgba(255,255,0,1.0); color: rgba(0,0,255,1.0);}");
    GError *css_error=NULL;
    GtkCssProvider *provider=gtk_css_provider_new();
    gtk_css_provider_load_from_data(provider, css_string, -1, &css_error);
    gtk_style_context_add_provider_for_screen(gdk_screen_get_default(), GTK_STYLE_PROVIDER(provider), GTK_STYLE_PROVIDER_PRIORITY_APPLICATION);
    if(css_error!=NULL)
      {
        g_print("CSS loader error %s\n", css_error->message);
        g_error_free(css_error);
      }
    g_object_unref(provider);
    g_free(css_string);

    gtk_widget_show_all(window);

    gtk_main();
    return 0;
  }
```

这是用 c2nim 和一些手动调整创建的 Nim 版本：

css\_colored\_listview.nim

```nim
# nim c css_colored_listview.nim
import gintro/[gtk, glib, gobject]
import gintro/gdk except Window # there is a problem with gdk.Window -- we have to investigate!
const # maybe we should use Nim's enum here?
  Id = 0
  Program = 1
  Color1 = 2
  Color2 = 3
  Columns = 4

proc bye(w: Window) =
  mainQuit()
  echo "Bye..."

proc toStringVal(s: string): Value =
  let gtype = typeFromName("gchararray")
  discard init(result, gtype)
  setString(result, s)

proc toUIntVal(i: int): Value =
  let gtype = typeFromName("guint")
  discard init(result, gtype)
  setUint(result, i)

proc toBoolVal(b: bool): Value =
  let gtype = typeFromName("gboolean")
  discard init(result, gtype)
  setBoolean(result, b)

# we need the following two procs for now -- later we will not use that ugly cast...
proc typeTest(o: gobject.Object; s: string): bool =
  let gt = g_type_from_name(s)
  return g_type_check_instance_is_a(cast[ptr TypeInstance00](o.impl), gt).toBool

proc listStore(o: gobject.Object): gtk.ListStore =
  assert(typeTest(o, "GtkListStore"))
  cast[gtk.ListStore](o)

proc updateRow(renderer: CellRendererText; path: cstring; newText: cstring; tree: TreeView) =
  var iter: TreeIter
  var value: Value
  let gtype = typeFromName("gchararray")
  discard init(value, gtype)
  let store = listStore(tree.getModel())
  value.setString(newText)
  let treePath = newTreePathFromString(path)
  discard store.getIter(iter, treePath)
  store.setValue(iter, 1, value)

# we use the old gtk style with init() as is used in the C original -- maybe better use modern app sytle
proc main() =
  gtk.init()
  let window = newWindow()
  window.title = "Select Cell"
  window.position = WindowPosition.center
  window.defaultSize = (500, 500)
  window.borderWidth = 20
  connect(window, "destroy", bye)
  var iter: TreeIter
  var h = [typeFromName("guint"), typeFromName("gchararray"), typeFromName("gchararray"),
    typeFromName("gchararray")]
  var store = newListStore(Columns,  cast[pointer]( unsafeaddr h)) # cast is ugly, we should fix it in bindings.
  let progNames = ["Gedit", "Gimp", "Inkscape", "Firefox", "Calculator", "Devhelp"]
  for i, n in progNames:
    store.append(iter) # currently we have to use setValue() as there is no varargs proc as in C original
    store.setValue(iter, Id, toUIntVal(i))
    store.setValue(iter, Program, toStringVal(n))
    store.setValue(iter, Color1, toStringVal(if (i and 1) != 0: "LightSlateGray" else: "DarkCyan"))
    store.setValue(iter, Color2, toStringVal("cyan"))
  var tree  = newTreeViewWithModel(store)
  tree.setHexpand
  tree.setVexpand
  setProperty(tree, "activate-on-single-click", toBoolVal(true))
  var selection = tree.getSelection()
  selection.setMode(SelectionMode.single)
  var renderer1 = newCellRendererText()
  setProperty(renderer1, "editable", toBoolVal(false))
  var renderer2 = newCellRendererText()
  setProperty(renderer2, "editable", toBoolVal(true))
  connect(renderer2, "edited", updateRow, tree)
  ## Bind the Color column to the "cell-background" property.
  var column1 = newTreeViewColumn()
  column1.setTitle("ID")
  column1.packStart(renderer1, true)
  column1.addAttribute(renderer1, "text", Id)
  column1.addAttribute(renderer1, "cell-background", Color1)
  discard tree.appendColumn(column1)
  var column2  = newTreeViewColumn()
  column1.setTitle("Program")
  column1.packStart(renderer2, true)
  column1.addAttribute(renderer2, "text", Program)
  column1.addAttribute(renderer2, "cell-background", Color2)
  discard tree.appendColumn(column2)
  var grid = newGrid() # only one occupied cell makes no sense -- but so we can add more widgets later
  grid.attach(tree, 0, 0, 1, 1)
  window.add(grid)
  const cssString = # note: big font selected intentionally
    """treeview{background-color: rgba(0,255,255,1.0); font-size:30pt} treeview:selected{background-color:
    rgba(255,255,0,1.0); color: rgba(0,0,255,1.0);}"""
  var provider  = newCssProvider()
  discard provider.loadFromData(cssString)
  addProviderForScreen(getDefaultScreen(), provider, STYLE_PROVIDER_PRIORITY_APPLICATION)
  window.showAll
  gtk.main()

main()
```

当你使用`nim c -d:release -d:danger --passC:-flto css_colored_listview.nim`进行编译时，你将得到一个 80k 大小的可执行文件，与 C 语言版本的 20k 相比是大了点，但也不算太差。你可能会注意到我添加了 updateRow() 过程，这对于永久编辑程序名称条目是必要的。该过程需要 cstring 参数，这可能令人惊讶，因为我们通常使用 Nim 字符串。问题不大，也许是有意为之，我们可能需要检查一下 gimpl.nim 中的 connect() 宏。

## 再看一个列表视图示例——自定义cairo绘图

该示例也是 C. Eric Cashon 在 <https://discourse.gnome.org/t/gtk-how-to-draw-on-top-of-gtktreeview/1783/2> 上提供的 C 示例的 Nim 版本。

它在选定的列表视图单元格上绘制一个矩形框。为此，需要使用 connectAfter() 来确保自定义cairo绘制在 GTK 部件绘制之后进行。

overlay\_tree1.nim

```nim
# nim c overlayTree1.nim
import gintro/[gtk, gdk, glib, gobject, cairo]
import strformat
from strutils import parseInt
const
  Id = 0
  Program = 1
  Color = 2
  Color2 = 3
  Columns = 4

var
  rowG = 0
  columnG = 1

proc bye(w: gtk.Window) =
  mainQuit()
  echo "Bye..."

proc toStringVal(s: string): Value =
  let gtype = typeFromName("gchararray")
  discard init(result, gtype)
  setString(result, s)

proc toUIntVal(i: int): Value =
  let gtype = typeFromName("guint")
  discard init(result, gtype)
  setUint(result, i)

proc toBoolVal(b: bool): Value =
  let gtype = typeFromName("gboolean")
  discard init(result, gtype)
  setBoolean(result, b)

proc selectCell(treeView: TreeView; path: TreePath; column: TreeViewColumn) =
  let str = toString(path)
  echo fmt"{str} {getTitle(column)}"
  rowG = parseInt(str)
  queueDraw(treeView)

proc drawRectangle(overlay: Overlay; cr: cairo.Context; treeView: TreeView): bool =
  echo fmt"Draw Rectangle {rowG} {columnG}"
  let path = newTreePathFromIndices(@[rowG.int32])
  echo path.toString
  let column = getColumn(treeView, columnG)
  var rect: gdk.Rectangle
  var x, y: int
  treeView.convertBinWindowToWidgetCoords(0, 0, x, y)
  cr.save
  cr.translate(x.float, y.float)
  cr.setLineWidth(2)
  cr.setSource(0, 0, 0, 1)
  treeView.getCellArea(path, column, rect)
  cr.rectangle(rect.x.float + 1, rect.y.float + 1, rect.width.float - 1, rect.height.float - 1)
  cr.stroke
  cr.restore
  return EVENT_PROPAGATE # false

proc main =
  gtk.init()
  let window = newWindow()
  window.setTitle("Overlay Tree")
  window.setPosition(WindowPosition.center)
  window.setDefaultSize(500, 500)
  window.setBorderWidth(20)
  window.connect("destroy", bye)
  var iter: TreeIter
  let h = [typeFromName("guint"), typeFromName("gchararray"), typeFromName("gchararray"),
    typeFromName("gchararray")]
  let store = newListStore(Columns, cast[pointer](unsafeaddr h)) # cast is ugly, we should fix it in bindings.
  let progNames = ["Gedit", "Gimp", "Inkscape", "Firefox", "Calculator", "Devhelp"]
  for i, n in progNames:
    store.append(iter) # currently we have to use setValue() as there is no varargs proc as in C original
    store.setValue(iter, Id, toUIntVal(i))
    store.setValue(iter, Program, toStringVal(n))
    store.setValue(iter, Color, toStringVal("SpringGreen"))
    store.setValue(iter, Color2, toStringVal("cyan"))
  let tree = newTreeViewWithModel(store)
  tree.setHexpand
  tree.setVexpand
  tree.setProperty("activate-on-single-click", toBoolVal(true))
  let selection = tree.getSelection
  selection.setMode(SelectionMode.single)
  let renderer1 = newCellRendererText()
  renderer1.setProperty("editable", toBoolVal(false))
  let renderer2 = newCellRendererText()
  renderer2.setProperty("editable", toBoolVal(true))
  tree.connect("row-activated", selectCell)
  ## Bind the COLOR column to the "cell-background" property.
  let column1 = newTreeViewColumn()
  column1.setTitle("ID")
  column1.packStart(renderer1, true)
  column1.addAttribute(renderer1, "text", Id)
  column1.addAttribute(renderer1, "cell-background", Color)
  discard tree.appendColumn(column1)
  let column2 = newTreeViewColumn()
  column2.setTitle("Program")
  column2.packStart(renderer2, true)
  column2.addAttribute(renderer2, "text", Program)
  column2.addAttribute(renderer2, "cell-background", Color2)
  discard tree.appendColumn(column2)
  ## For drawing the outline of the cell.
  let overlay = newOverlay()
  overlay.setHexpand
  overlay.setVexpand
  overlay.setAppPaintable
  overlay.addOverlay(tree)
  overlay.setOverlayPassThrough(tree, true)
  overlay.connectAfter("draw", drawRectangle, tree)
  let grid = newGrid()
  grid.attach(overlay, 0, 0, 1, 1)
  window.add(grid)
  const cssString =
    """treeview{background-color: rgba(0,255,255,1.0);
      font-size:30pt} treeview:selected{background-color:rgba(0,255,255,1.0);
      color: rgba(0,0,255,1.0);}"""
  let provider = newCssProvider()
  discard provider.loadFromData(cssString)
  getDefaultScreen().addProviderForScreen(provider, STYLE_PROVIDER_PRIORITY_APPLICATION)
  window.showAll
  gtk.main()

main() # 123 lines
```

## 使用 CellDataFunction 的 Listview 示例

本示例展示了如何使用 CellDataFunction 自定义树形视图或列表视图的单元格。

celldatafunction.nim

```nim
# This example shows how to apply a CellDataFunc to a GtkTreeView
# C example code was provided by A.Krause in chapter 8 of his book
import gintro/[gtk, gobject, glib]

const
  Color = 0
  Columns = 1
  clr = ["00", "33", "66", "99", "CC", "FF"]

proc bye(w: Window) =
  mainQuit()
  echo "Bye..."

proc toStringVal(s: string): Value =
  let gtype = gStringGetType() # typeFromName("gchararray")
  discard init(result, gtype)
  setString(result, s)

proc toBoolVal(b: bool): Value =
  let gtype = gBooleanGetType() # typeFromName("gboolean")
  discard init(result, gtype)
  setBoolean(result, b)

# our Nim function
proc cellDataFuncN(column: TreeViewColumn; renderer: CellRenderer;
                  model: TreeModel; iter: TreeIter, data: TreeViewColumn) =
  ##  Get the color string stored by the column and make it the foreground color.
  # for testing that optional args work, we pass a TreeViewColumn and echo its title
  echo data.title
  var val: Value
  model.getValue(iter, Color, val)
  let text = val.getString
  val.unset # is this necessary?
  setProperty(renderer, "foreground", toStringVal("#FFFFFF"))
  setProperty(renderer, "foreground-set", toBoolVal(true))
  setProperty(renderer, "background", toStringVal(text))
  setProperty(renderer, "background-set", toBoolVal(true))
  setProperty(renderer, "text", toStringVal(text))

##  Add three columns to the GtkTreeView. All three of the columns will be
##  displayed as text, although one is a gboolean value and another is
##  an integer.
proc setupTreeView(treeview: TreeView) =
  let renderer = gtk.newCellRendererText()
  let column = newTreeViewColumn()
  column.title = "Standard Colors"
  column.packStart(renderer, expand = true)
  column.addAttribute(renderer, "text", Color)
  discard treeview.appendColumn(column)
  column.setCellDataFunc(renderer, cellDataFuncN, column)
  column.setCellDataFunc(renderer) # test unsetting!
  column.setCellDataFunc(renderer, nil)
  column.unsetCellDataFunc(renderer)
  column.setCellDataFunc(renderer, cellDataFuncN, column)

proc main =
  var iter: TreeIter
  gtk.init()
  let window = newWindow()
  window.setTitle("Color List")
  window.setBorderWidth(10)
  window.setSizeRequest(250, 175)
  window.connect("destroy", bye)
  let treeview = newTreeView()
  setupTreeView(treeview)
  let gtype = typeFromName("gchararray")
  let store = newListStore(Columns, cast[pointer](unsafeaddr gtype)) # ugly cast
  ##  Add all of the products to the GtkListStore.
  for i in 0 ..< 6:
    for j in 0 ..< 6:
      for k in 0 ..< 6:
        let color: string = "#" & clr[i] & clr[j] & clr[k]
        store.append(iter)
        store.setValue(iter, Color, toStringVal(color))
  treeView.setModel(store)
  let scrolledWin = newScrolledWindow(nil, nil)
  scrolledWin.setPolicy(PolicyType.automatic, PolicyType.automatic)
  scrolledWin.add(treeview)
  window.add(scrolledWin)
  window.showAll
  gtk.main()

main()
```

## 带缩放、平移和滚动功能的cairo绘图高级示例

以下代码是我几年前用 Ruby (<https://ssalewski.de/PetEd-Demo.html.en>) 编写的绘图演示的纯 Nim 版本。cairo曲面目前是手动释放的，因为 GC 可能会产生过大的延迟。

您可以调整窗口大小，并使用鼠标滚轮放大窗口。放大时会出现滚动条。您可以在移动鼠标时按住鼠标中键进行平移，也可以按住鼠标左键并移动鼠标，首先绘制一个矩形选区，然后在松开鼠标键时放大该矩形选区。

在示例目录中还有一个简化版本，名为`simpledrawingarea.nim`，它在绘制回调中完成所有绘制，而不对表面使用缓冲。对于普通应用程序来说，这通常是更可取的。

drawingarea.nim

```nim
# Plain demo for zooming, panning, scrolling with GTK DrawingArea
# (c) S. Salewski, 21-DEC-2010 (initial Ruby version)
# Nim version April 2019
# License MIT

# This version of the demo program uses a separate proc paint()
# which allocates a custom surface for buffered drawing.
# That may be not really necessary, for simple drawings doing all
# the drawing in the "draw" call back is easier and faster. But for
# more complicated drawing operations, for example when using a
# background grid, which is a bit larger than the window size and
# is reused when scrolling, a custom surface may be useful.
# And finally that custom surface and custom cairo context is an
# important test for the language bindings.

# https://discourse.gnome.org/t/problem-with-gtkscrollbar-gtk-window-resize-and-gtk-adjustment-set-value/1081

import gintro/[gtk, gdk, glib, gobject, gio, cairo]

const
  ZoomFactorMouseWheel = 1.1
  ZoomFactorSelectMax = 10 # ignore zooming in tiny selection
  ZoomNearMousepointer = true # mouse wheel zooming -- to mouse-pointer or center
  SelectRectCol = [0.0, 0, 1, 0.5] # blue with transparency

discard """
Zooming, scrolling, panning...

|-------------------------|
|<-------- A ------------>|
|                         |
|  |---------------|      |
|  | <---- a ----->|      |
|  |    visible    |      |
|  |---------------|      |
|                         |
|                         |
|-------------------------|

a is the visible, zoomed in area == darea.allocatedWidth
A is the total data range
A/a == userZoom >= 1
For horizontal adjustment we use
hadjustment.setUpper(darea.allocatedWidth * userZoom) == A
hadjustment.setPageSize(darea.allocatedWidth) == a
So hadjustment.value == left side of visible area

Initially, we set userZoom = 1, scale our data to fit into darea.allocatedWidth
and translate the origin of our data to (0, 0)

Zooming: Mouse wheel or selecting a rectangle with left mouse button pressed
Scrolling: Scrollbars
Panning: Moving mouse while middle mouse button pressed
"""

# drawing area and scroll bars in 2x2 grid (PDA == Plain Drawing Area)

type
  PosAdj = ref object of Adjustment
    handlerID: uint64

proc newPosAdj: PosAdj =
  initAdjustment(result, 0, 0, 1, 1, 10, 1)

type
  PDA_Data* = object
    draw*: proc (cr: Context)
    extents*: proc (): tuple[x, y, w, h: float]
    windowSize*: tuple[w, h: int]

type
  PDA = ref object of Grid
    zoomNearMousepointer: bool
    selecting: bool
    userZoom: float
    surf: Surface
    pattern: Pattern
    cr: cairo.Context
    darea: DrawingArea
    hadjustment: PosAdj
    vadjustment: PosAdj
    hscrollbar: Scrollbar
    vscrollbar: Scrollbar
    fullScale: float
    dataX: float
    dataY: float
    dataWidth: float
    dataHeight: float
    lastButtonDownPosX: float
    lastButtonDownPosY: float
    lastMousePosX: float
    lastMousePosY: float
    zoomRectX1: float
    zoomRectY1: float
    oldSizeX: int
    oldSizeY: int
    drawWorld: proc (cr: Context)
    extents: proc (): tuple[x, y, w, h: float]

proc drawingAreaDrawCb(darea: DrawingArea; cr: Context; this: PDA): bool =
  if this.pattern.isNil: return
  cr.setSource(this.pattern)
  cr.paint
  if this.selecting:
    cr.rectangle(this.lastButtonDownPosX, this.lastButtonDownPosY,
      this.zoomRectX1 - this.lastButtonDownPosX, this.zoomRectY1 - this.lastButtonDownPosY)
    cr.setSource(0, 0, 1, 0.5) # SELECT_RECT_COL) # 0, 0, 1, 0.5
    cr.fillPreserve
    cr.setSource(0, 0, 0)
    cr.setLineWidth(2)
    cr.stroke
  return gdk.EVENT_STOP # EVENT_PROPAGATE
  #return true # TRUE to stop other handlers from being invoked for the event. FALSE to propagate the event further.

# clamp to correct values, 0 <= value <= (adj.upper - adj.pageSize), block calling onAdjustmentEvent()
proc updateVal(adj: PosAdj; d: float) =
  adj.signalHandlerBlock(adj.handlerID)
  adj.setValue(max(0.0, min(adj.value + d, adj.upper - adj.pageSize)))
  adj.signalHandlerUnblock(adj.handlerID)

proc updateAdjustments(this: PDA; dx, dy: float) =
  this.hadjustment.setUpper(this.darea.allocatedWidth.float * this.userZoom)
  this.vadjustment.setUpper(this.darea.allocatedHeight.float * this.userZoom)
  this.hadjustment.setPageSize(this.darea.allocatedWidth.float)
  this.vadjustment.setPageSize(this.darea.allocatedHeight.float)
  updateVal(this.hadjustment, dx)
  updateVal(this.vadjustment, dy)

proc paint(this: PDA) =
  # echo "paint"
  this.cr.save
  this.cr.translate(this.hadjustment.upper * 0.5 - this.hadjustment.value, # our origin is the center
    this.vadjustment.upper * 0.5 - this.vadjustment.value)
  this.cr.scale(this.fullScale * this.userZoom, this.fullScale * this.userZoom)
  this.cr.translate(-this.dataX - this.dataWidth * 0.5, -this.dataY - this.dataHeight * 0.5)
  this.drawWorld(this.cr) # call the user provided drawing function
  this.cr.restore

proc dareaConfigureCallback(darea: DrawingArea; event: EventConfigure; this: PDA): bool =
  (this.dataX, this.dataY, this.dataWidth,
    this.dataHeight) = this.extents() # query user defined size
  this.fullScale = min(this.darea.allocatedWidth.float / this.dataWidth,
      this.darea.allocatedHeight.float / this.dataHeight)
  if this.surf != nil:
    destroy(this.surf) # manually destroy surface -- GC would do it for us, but GC is slow...
  this.surf = this.darea.window.createSimilarSurface(Content.color,
      this.darea.allocatedWidth, this.darea.allocatedHeight)
  if this.pattern != nil:
    patternDestroy(this.pattern)
  if this.cr != nil:
    destroy(this.cr)
  this.pattern = patternCreateForSurface(this.surf) # pattern now owns the surface!
  this.cr = newContext(this.surf) # this function references target!
  this.paint
  return gdk.EVENT_STOP

proc hscrollbarSizeAllocateCallback(s: Scrollbar; r: gdk.Rectangle; pda: PDA) =
  pda.hadjustment.setUpper(r.width.float * pda.userZoom)
  pda.hadjustment.setPageSize(r.width.float)
  if pda.oldSizeX != 0: # this fix is not exact, as fullScale can ...
    updateVal(pda.hadjustment, (r.width - pda.oldSizeX).float * 0.5)
  pda.oldSizeX = r.width

proc vscrollbarSizeAllocateCallback(s: Scrollbar; r: gdk.Rectangle; pda: PDA) =
  pda.vadjustment.setUpper(r.height.float * pda.userZoom)
  pda.vadjustment.setPageSize(r.height.float)
  if pda.oldSizeY != 0: # ... change when window is rezized. But it's good enough!
    updateVal(pda.vadjustment, (r.height - pda.oldSizeY).float * 0.5)
  pda.oldSizeY = r.height

proc updateAdjustmentsAndPaint(this: PDA; dx, dy: float) =
  this.updateAdjustments(dx, dy)
  this.paint
  this.darea.queueDrawArea(0, 0, this.darea.allocatedWidth, this.darea.allocatedHeight)

# event coordinates to user space
proc getUserCoordinates(this: PDA; eventX, eventY: float): (float, float) =
  ((eventX - this.hadjustment.upper * 0.5 + this.hadjustment.value) / (
      this.fullScale * this.userZoom) + this.dataX + this.dataWidth * 0.5,
   (eventY - this.vadjustment.upper * 0.5 + this.vadjustment.value) / (
       this.fullScale * this.userZoom) + this.dataY + this.dataHeight * 0.5)

proc onMotion(darea: DrawingArea; event: EventMotion; this: PDA): bool =
  let state = getState(event)
  let (x, y) = event.getCoords
  if state.contains(button1): # selecting
    this.selecting = true
    this.zoomRectX1 = x
    this.zoomRectY1 = y
    this.darea.queueDrawArea(0, 0, this.darea.allocatedWidth, this.darea.allocatedHeight)
  elif button2 in state: # panning
    this.updateAdjustmentsAndPaint(this.lastMousePosX - x, this.lastMousePosY - y)
  else:
    return gdk.EVENT_PROPAGATE
  this.lastMousePosX = x
  this.lastMousePosY = y
  return gdk.EVENT_STOP
  #event.request # request more motion events ?

# zooming with mouse wheel -- data near mouse pointer should not move if possible!
# hadjustment.value + event.x is the position in our zoomed_in world, (userZoom / z0 - 1)
# is the relative movement caused by zooming
# In other words, this is the delta-move d of a point at position P from zooming:
# d = newPos - P = P * scale - P = P * (z/z0) - P = P * (z/z0 - 1). We have to compensate for this d.
proc scrollEvent(darea: DrawingArea; event: EventScroll; this: PDA): bool =
  let z0 = this.userZoom
  case getScrollDirection(event)
  of ScrollDirection.up:
    this.userZoom *= ZoomFactorMouseWheel
  of ScrollDirection.down:
    this.userZoom /= ZoomFactorMouseWheel
    if this.userZoom < 1:
      this.userZoom = 1
  else:
    return gdk.EVENT_PROPAGATE
  if this.zoomNearMousepointer:
    let (x, y) = event.getCoords
    this.updateAdjustmentsAndPaint((this.hadjustment.value + x) * (this.userZoom / z0 - 1),
      (this.vadjustment.value + y) * (this.userZoom / z0 - 1))
  else: # zoom to center
    this.updateAdjustmentsAndPaint((this.hadjustment.value +
        this.darea.allocatedWidth.float * 0.5) * (this.userZoom / z0 - 1),
        (this.vadjustment.value + this.darea.allocatedHeight.float * 0.5) * (this.userZoom / z0 - 1))
  return gdk.EVENT_STOP

proc buttonPressEvent(darea: DrawingArea; event: EventButton; this: PDA): bool =
  var (x, y) = event.getCoords
  this.lastMousePosX = x
  this.lastMousePosY = y
  this.lastButtonDownPosX = x
  this.lastButtonDownPosY = y
  echo "buttonPressEvent", x, " ", y
  (x, y) = this.getUserCoordinates(x, y)
  echo "User coordinates: ", x, ' ', y, "\n" # to verify getUserCoordinates()
  return gdk.EVENT_STOP

# zoom into selected rectangle and center it
# math: we first center the selection rectangle, and then compensate for translation due to scale
proc buttonReleaseEvent(darea: DrawingArea; event: EventButton; this: PDA): bool =
  let (x, y) = event.getCoords
  let b = getButton(event)
  if b == 1:
    this.selecting = false
    let z1 = min(this.darea.allocatedWidth.float / (this.lastButtonDownPosX - x).abs,
      this.darea.allocatedHeight.float / (this.lastButtonDownPosY - y).abs)
    if z1 < ZoomFactorSelectMax: # else selection rectangle will persist, we may output a message...
      this.userZoom *= z1
      this.updateAdjustmentsAndPaint(
        ((x + this.lastButtonDownPosX) * z1 - this.darea.allocatedWidth.float) * 0.5 + this.hadjustment.value * (z1 - 1),
        ((y + this.lastButtonDownPosY) * z1 - this.darea.allocatedHeight.float) * 0.5 + this.vadjustment.value * (z1 - 1))
    return gdk.EVENT_STOP
  return gdk.EVENT_PROPAGATE

proc onAdjustmentEvent(this: PosAdj; pda: PDA) =
  pda.paint
  pda.darea.queueDrawArea(0, 0, pda.darea.allocatedWidth, pda.darea.allocatedHeight)

proc newPDA: PDA =
  initGrid(result)
  let da = newDrawingArea()
  result.darea = da
  da.setHExpand
  da.setVExpand
  da.connect("draw", drawingAreaDrawCb, result)
  da.connect("configure-event", dareaConfigureCallback, result)
  da.addEvents({EventFlag.buttonPress, EventFlag.buttonRelease,
      EventFlag.scroll, button1Motion, button2Motion, pointerMotionHint})
  da.connect("motion-notify-event", onMotion, result)
  da.connect("scroll_event", scrollEvent, result)
  da.connect("button_press_event", buttonPressEvent, result)
  da.connect("button_release_event", buttonReleaseEvent, result)
  result.zoomNearMousepointer = ZoomNearMousepointer # mouse wheel zooming
  result.userZoom = 1.0
  result.hadjustment = newPosAdj()
  result.hadjustment.handlerID = result.hadjustment.connect("value-changed", onAdjustmentEvent, result)
  result.vadjustment = newPosAdj()
  result.vadjustment.handlerID = result.vadjustment.connect("value-changed", onAdjustmentEvent, result)
  result.hscrollbar = newScrollbar(Orientation.horizontal, result.hadjustment)
  result.vscrollbar = newScrollbar(Orientation.vertical, result.vadjustment)
  result.hscrollbar.setHExpand
  result.vscrollbar.setVExpand
  result.hscrollbar.connect("size-allocate", hscrollbarSizeAllocateCallback, result)
  result.vscrollbar.connect("size-allocate", vscrollbarSizeAllocateCallback, result)
  result.attach(result.darea, 0, 0, 1, 1)
  result.attach(result.vscrollbar, 1, 0, 1, 1)
  result.attach(result.hscrollbar, 0, 1, 1, 1)

proc appStartup(app: Application) =
  echo "appStartup"

proc appActivate(app: Application; initData: PDA_Data) =
  let window = newApplicationWindow(app)
  window.title = "Drawing example"
  # window.defaultSize = initData.windowSize
  window.defaultSize = (initData.windowSize[0], initData.windowSize[1])
  let pda = newPDA()
  pda.drawWorld = initData.draw
  pda.extents = initData.extents
  window.add(pda)
  showAll(window)

proc newDisplay*(initData: PDA_Data) =
  let app = newApplication("org.gtk.example")
  connect(app, "startup", appStartup)
  connect(app, "activate", appActivate, initData)
  discard run(app)

when isMainModule:

  const # arbitrary locations for our data
    DataX = 150.0
    DataY = 250.0
    DataWidth = 200.0
    DataHeight = 120.0

  # we need two user defined functions -- one gives the extent of the graphics,
  # and the other does the cairo drawing using a cairo context.

  # bounding box of user data -- x, y, w, h -- top left corner, width, height
  proc worldExtents(): (float, float, float, float) =
    (DataX, DataY, DataWidth, DataHeight) # current extents of our user world

  # draw to cairo context
  proc drawWorld(cr: cairo.Context) =
    cr.setSource(1, 1, 1)
    cr.paint
    cr.setSource(0, 0, 0)
    cr.setLineWidth(2)
    var i = 0.0
    while min(DataWidth - 2 * i, DataHeight - 2 * i) > 0:
      cr.rectangle(DataX + i, DataY + i, DataWidth - 2 * i, DataHeight - 2 * i)
      i += 10
    cr.stroke

  proc test =
    let data = PDA_Data(draw: drawWorld, extents: worldExtents, windowSize: (800, 600))
    newDisplay(data)

  test() # 337 lines
```

我们可以轻松地将该模块作为一个库使用，并获得这个完全支持缩放和滚动的简单绘图工具：

darea\_test.nim

```nim
import gintro/cairo
import drawingarea
from math import PI

proc extents(): (float, float, float, float) =
  (0.0, 0.0, 100.0, 100.0) # ugly float literals

# draw to cairo context
proc draw(cr: cairo.Context) =
  cr.setSource(1, 1, 1) # set background color and paint
  cr.paint
  cr.setSource(0, 0, 0) # forground color
  cr.arc(20, 30, 10, 0, 5) # nearly a circle
  cr.newSubPath # do not join the two arcs
  cr.arc(70, 60, 20, 0, math.PI)
  cr.stroke # finally do it

proc main =
  var data: PDA_Data
  data.draw = draw
  data.extents = extents
  data.windowSize = (800, 600)
  newDisplay(data)

main()
```

## 再举一个cairo的例子

最近，C. Eric Cashon 先生提供了一个处理大型位图图像的示例代码。 他的示例将图像写入磁盘，再次加载并显示图像，允许缩放和转换。由于现在很少有示例，而且该示例也不是很大，因此我使用 c2nim 将其转换为 Nim。以下是经过手动修正的代码。请注意，目前已发布的 cairo.nim 模块包含一个断言语句，它会阻止运行这个示例。 如果你真的想运行这段代码，你必须修正 cairo.nim 中的那一行。我还需要对 cairo 模块进行更多修正，最终可能会发布一个新版本。这个例子非常低级，因为直接使用了 alloc()。

cairoImage.nim

```nim
# https://discourse.gnome.org/t/proper-zoom-pan-image-approach-for-large-images/1497/6
# Nim version of the C example of C. Eric Cashon
import gintro/[gtk, gobject, glib, cairo]
from math import TAU
import strutils

const
  Width = 5000
  Height = 5000
  CFormat = cairo.Format.argb32

var
  Key: cairo.UserDataKey
  translateX: float
  translateY: float
  scale = 1.0
  ## Store data from file.
  bigSurfaceData*: ptr cuchar = nil

proc translateXSpinChanged(spinButton: SpinButton; data: DrawingArea) =
  translateX = spinButton.value
  data.queueDraw

proc translateYSpinChanged(spinButton: SpinButton; data: DrawingArea) =
  translateY = spinButton.getValue
  data.queueDraw

proc scaleSpinChanged(spinButton: SpinButton; data: DrawingArea) =
  scale = spinButton.value
  data.queueDraw

proc saveBigSurface =
  ## Use gdk_cairo_surface_create_from_pixbuf() to read in a pixbuf. Try a test surface here.
  let bigSurface = imageSurfaceCreate(CFormat, Width, Height)
  let cr = newContext(bigSurface)
  ## Paint the background.
  cr.setSource(1, 1, 1)
  cr.paint
  ## Draw a circle.
  cr.setSource(0, 0, 1)
  cr.arc(250, 250, 50, 0, math.TAU)
  cr.fill
  ## Draw some test grid lines.
  cr.setSource(0, 1, 0)
  for i in countup(0, 4900, 100):
    cr.moveTo(0, i.float)
    cr.lineTo(5000, i.float)
    cr.stroke
  for i in countup(0, 4900, 100):
    cr.moveTo(i.float, 0)
    cr.lineTo(i.float, 5000)
    cr.stroke
  cr.setSource(0, 0, 1)
  cr.setLineWidth(10)
  for i in 0 ..< 10:
    cr.moveTo(0, i.float * 500.0)
    cr.lineTo(5000, i.float * 500.0)
    cr.stroke
  for i in 0 ..< 10:
    cr.moveTo(i.float * 500.0, 0)
    cr.lineTo(i.float * 500.0, 5000)
    cr.stroke
  ## Outside box.
  cr.setLineWidth(20)
  cr.setSource(1, 0, 1)
  cr.rectangle(0, 0, 5000, 5000)
  cr.stroke
  ## Save surface data to file.
  let f: File = open("big_surface.s", fmWrite)
  let p: ptr cuchar = cairo_image_surface_get_data(bigSurface.impl)
  let len = writeBuffer(f, p, cairo_format_stride_for_width(CFormat, Width) * Height)
  echo("write $1\n" % $len)
  close(f)

proc myDealloc(data: pointer) {.cdecl.} =
  system.dealloc(data)

proc getBigSurface(): Surface =
  let f: File = open("big_surface.s", fmRead)
  # setFilePos(f, 0)
  # https://www.cairographics.org/manual/cairo-Image-Surfaces.html#cairo-format-stride-for-width
  let stride = cairo_format_stride_for_width(CFormat, Width)
  bigSurfaceData = cast[ptr cuchar](malloc((stride * Height).uint64))
  var len = readBuffer(f, bigSurfaceData, stride * Height)
  echo("read $1" % $len)
  close(f)
  let bigSurface: Surface = new Surface # this is a temporary fix, we will support this later in cairo modul
  bigSurface.impl = cairo_image_surface_create_for_data(bigSurfaceData, CFormat, Width, Height, stride)
  discard setUserData(bigSurface, addr(Key), bigSurfaceData, myDealloc) # automatic deallocation
  # flush(bigSurface)
  echo("open $1" % bigSurface.status.statusToString)
  return bigSurface

proc daDrawing*(da: DrawingArea; cr: Context; bigSurface: Surface): bool =
  var
    width = da.getAllocatedWidth.float
    height = da.getAllocatedHeight.float
    originX = translateX
    originY = translateY
  ## Some constraints.
  if translateX > 5000.0 - width:
    originX = 5000.0 - width / scale
  if translateY > 5000.0 - height:
    originY = 5000.0 - height / scale
  cr.setSource(0, 0, 0)
  cr.paint
  ## Partition the big surface.
  var littleSurface: Surface = cairo.surfaceCreateForRectangle(bigSurface,
      originX, originY, width / scale, height / scale)
  cr.scale(scale, scale)
  cr.setSourceSurface(littleSurface, 0, 0)
  setFilter(getSource(cr), cairo.Filter.bilinear)
  cr.paint
  return true

proc bye(w: Window) =
  mainQuit()
  echo "Bye..."

proc main =
  gtk.init()
  let window = newWindow()
  window.setTitle("Big Surface2")
  window.setDefaultSize(500, 500)
  window.setPosition(gtk.WindowPosition.center)
  window.connect("destroy", bye)
  ## Get a test surface.
  saveBigSurface()
  let bigSurface = getBigSurface()
  let da: DrawingArea = newDrawingArea()
  da.setHexpand
  da.setVexpand
  da.connect("draw", daDrawing, bigSurface)
  let
    translateXAdj = newAdjustment(0, 0, 5000, 20, 0, 0)
    translateYAdj = newAdjustment(0, 0, 5000, 20, 0, 0)
    scaleAdj = newAdjustment(1, 1, 5, 0.1, 0, 0)
    translateXLabel = newLabel("translate x")
    translateXSpin= newSpinButton(translateXAdj, 50, 1)
  connect(translateXSpin, "value-changed", translateXSpinChanged, da)
  let translateYLabel = newLabel("translate y")
  let translateYSpin = newSpinButton(translateYAdj, 50, 1)
  connect(translateYSpin, "value-changed", translateYSpinChanged, da)
  let scaleLabel = newLabel("Scale")
  let scaleSpin = newSpinButton(scaleAdj, 0.2, 1)
  connect(scaleSpin, "value-changed", scaleSpinChanged, da)
  let grid = newGrid()
  grid.attach(da, 0, 0, 3, 1)
  grid.attach(translateXLabel, 0, 1, 1, 1)
  grid.attach(translateYLabel, 1, 1, 1, 1)
  grid.attach(scaleLabel, 2, 1, 1, 1)
  grid.attach(translateXSpin, 0, 2, 1, 1)
  grid.attach(translateYSpin, 1, 2, 1, 1)
  grid.attach(scaleSpin, 2, 2, 1, 1)
  add(window, grid)
  showAll(window)
  gtk.main()

main()
```

## 一个 VTE 例子

下面的代码展示了如何使用 vte 库创建普通 shell 窗口：

vte.nim

```nim
# https://vincent.bernat.im/en/blog/2017-write-own-terminal
import gintro/[gtk, glib, gobject, gio, vte]

proc appActivate(app: Application) =
  let window = newApplicationWindow(app)
  window.title = "GTK3 & Nim"
  window.defaultSize = (600, 200)
  let terminal = newTerminal()
  let environ = getEnviron()
  let command = environ.environGetenv("SHELL")
  var pid = 0
  echo terminal.spawnSync({}, nil, [command], [], {SpawnFlag.leaveDescriptorsOpen}, nil, nil, pid, nil)
  window.add(terminal)
  showAll(window)

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", appActivate)
  discard run(app)

main()
```

## GStreamer 示例

最近有人询问 gintro 是否支持 gstreamer，请参见[#59](https://github.com/StefanSalewski/gintro/issues/59)。 因此我们添加了它。我对 gstreamer 不甚了解，但有人告诉我，Python 和最近的 Rusts 绑定都使用了它，因此可能会有一些用例。

gstBasicTutorial1.nim

```nim
# https://gstreamer.freedesktop.org/documentation/tutorials/basic/hello-world.html?gi-language=c
# nim c gstBasicTutorial1.nim
import gintro/gst

proc main =
  var pipeline: gst.Element
  var bus: gst.Bus
  var msg: gst.Message
  ##  Initialize GStreamer
  gst.init()
  ##  Build the pipeline
  pipeline = gst.parseLaunch("playbin uri=https://www.freedesktop.org/software/gstreamer-sdk/data/media/sintel_trailer-480p.webm")
  ##  Start playing
  discard gst.setState(pipeline, gst.State.playing)
  ##  Wait until error or EOS
  bus = gst.getBus(pipeline)
  msg = gst.timedPopFiltered(bus, gst.Clock_Time_None, {gst.MessageFlag.error, gst.MessageFlag.eos})
  echo msg.getType
  discard gst.setState(pipeline, gst.State.null) # is this necessary?

main()
```

## 线程示例

GTK GUI 的一个常见问题是，在 GTK 主线程中运行较长时间的函数会阻塞 GUI 的显示更新或用户输入。我们可以通过创建一个单独的线程在后台运行这些函数来解决这个问题。

GTK 图形用户界面并非真正的线程安全，因此我们无法直接从其他线程更新图形用户界面。解决这个问题的简单方法是使用 glib 函数 g-idle-add() 或 g-timeout-add()。

glib 函数 g-timeout-add() 在 GTK 主循环中添加了一个用户定义的函数，定期调用；而函数 g-idle-add() 则在 GTK 主循环中添加了一个用户定义的函数，在 GTK 不忙其他事情时调用。这两个函数都在 GTK 主线程中执行，因此都可以更新 GUI 部件或调用其他 GTK 相关函数。

在第一个示例中，我们创建了一个 Nim 工作线程，通过 Nim 通道向主线程发送数据。我们使用 timeoutAdd()定期调用一个函数，从通道读取数据并更新图形用户界面。在本示例中，工作线程只进行简单的倒计时，而在实际工作中，工作线程会在后台进行繁重的工作，例如运行国际象棋引擎寻找下一步最佳棋步或对大型数据库进行排序。函数 updateGUI() 接收当前的计数器值并更新按钮的标签。

thread1.nim

```nim
# https://nim-lang.org/docs/channels.html
# nim c --threads:on --gc:arc -r thread1.nim

import gintro/[gtk, glib, gobject, gio]
from  os import sleep

var channel: Channel[int]
var workThread: system.Thread[void]

proc workProc =
  var countdown {.global.} = 25
  while countdown > 0:
    sleep(1000)
    dec(countdown)
    channel.send(countdown)

proc updateGUI(b: Button): bool =
  let msg = channel.tryRecv()
  if msg.dataAvailable:
    b.label = $msg.msg
    if msg.msg == 0:
      workThread.joinThread
      channel.close
      return SOURCE_REMOVE
  return SOURCE_CONTINUE

proc buttonClicked (b: Button) =
  b.label = utf8Strreverse(b.label, -1)

proc activate (app: Application) =
  let window = newApplicationWindow(app)
  window.title = "Countdown"
  window.defaultSize = (250, 50)
  let button = newButton("Click Me")
  window.add(button)
  button.connect("clicked",  buttonClicked)
  window.showAll
  channel.open
  createThread(workThread, workProc)
  discard timeoutAdd(1000 div 60, updateGUI, button)

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", activate)
  discard app.run

main()
```

下一个示例使用工作线程内的函数 g-idle-add() 调用用户定义的函数，然后更新 GUI 部件：

thread2.nim

```nim
# nim c --threads:on --gc:arc -r thread2.nim

import gintro/[gtk, glib, gobject, gio]
from  os import sleep

var workThread: system.Thread[void]
var button: Button

proc idleFunc(i: int): bool =
  button.label = $i
  return SOURCE_REMOVE

proc workProc =
  var countdown {.global.} = 25
  while countdown > 0:
    sleep(1000)
    dec(countdown)
    idleAdd(idleFunc, countdown)

proc buttonClicked(button: Button) =
  button.label = utf8Strreverse(button.label, -1)

proc activate(app: Application) =
  let window = newApplicationWindow(app)
  window.title = "Countdown"
  window.defaultSize = (250, 50)
  button = newButton("Click Me")
  window.add(button)
  button.connect("clicked",  buttonClicked)
  window.showAll
  createThread(workThread, workProc)

proc main =
  let app = newApplication("org.gtk.example")
  connect(app, "activate", activate)
  discard app.run

main()
```

## GTK4 的 HeaderBar 示例

下面的代码是 GTK4 示例的 C 语言 Nim 版本。它只有在您已经安装了 GTK4 的情况下才能以未经修改的形式运行——我的 GTK4 位于 /opt/gtk，我必须先键入这些命令才能使用 GTK4 程序，如 gtk4-demo：

```shell
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/gtk/lib64/
export GSETTINGS_SCHEMA_DIR=/opt/gtk/share/glib-2.0/schemas /opt/gtk/bin/gtk4-demo
#export PKG_CONFIG_PATH="/opt/gtk/lib64/pkgconfig/" # only when compiling C code directly
```

这个示例有点复杂，因为要创建一个自定义的（假的）标题栏，这涉及到 C 和 Nim 代码的转换。 示例展示了如何使用默认的标题栏、如何打开文件对话框以及如何使用图片和 CSS 样式。 我想大部分代码在 GTK3 中也可以使用，但我还没有尝试制作 GTK3 版本，甚至还没有理解所有的代码。请注意，在这个例子中，destroy() 并没有连接到窗口关闭按钮，而是在 gtk4.main() 之后调用的。这确实有点奇怪，请参阅 E.Bassi 先生的论坛注释。

headerbar.nim

```nim
# https://github.com/GNOME/gtk/blob/mainline/tests/testheaderbar.c
# this is still based on the original testheaderbar.c, which was recently replaced by testheaderbar2.c
# and testheaderbar.c is not a very good Nim GTK4 example unfortunately -- too comlicated and strange code.
# nim c headerbar.nim
import gintro/[gtk4, glib, gobject]

const
  Css = """
 .main.background {
 background-image: linear-gradient(to bottom, red, blue);
 border-width: 0px;
 }
 .titlebar.backdrop {
 background-image: none;
 background-color: @bg_color;
 border-radius: 10px 10px 0px 0px;
 }
 .titlebar {
 background-image: linear-gradient(to bottom, white, @bg_color);
 border-radius: 10px 10px 0px 0px;
 }
"""

# we try to avoid use of global header variable as done in C code
type
  MyWindow = ref object of gtk4.Window
    header: gtk4.Widget

proc response(d: gtk4.FileChooserDialog; responseID: int) = gtk4.destroy(d)

proc onBookmarkClicked(button: Button; data: MyWindow) =
  let window = gtk4.Window(data)
  let chooser = newFileChooserDialog("File Chooser Test", window,
      FileChooserAction.open)
  discard chooser.addButton("_Close", gtk4.ResponseType.close.ord)
  chooser.connect("response", response)
  chooser.show

#proc changeSubtitle(button: Button; w: MyWindow) =
#  if w.header.subtitle == "":
#    w.header.setSubtitle("(subtle subtitle)")
#  else:
#    w.header.setSubtitle("") # can we pass nil?

proc toggleFullscreen(button: Button; window: MyWindow) =
  var fullscreen {.global.}: bool
  if fullscreen:
    window.unfullscreen
    fullscreen = false
  else:
    window.fullscreen
    fullscreen = true

proc toIntVal(i: int): Value =
  let gtype = typeFromName("gint")
  discard init(result, gtype)
  setInt(result, i)

var done = false

proc quit_cb(b: Button) = # we can not pass a var parameter
 #gtk4.mainQuit()
 done = true
 wakeup(defaultMainContext()) # g_main_context_wakeup (NULL);

proc changeHeader(button: ToggleButton; window: MyWindow) =
  if button != nil and button.getActive:
    window.header = (newBox(gtk4.Orientation.horizontal,
        10))
    addCssClass(window.header, "titlebar")
    addCssClass(window.header, "header-bar")
    #window.header.setProperty("margin_start", toIntVal(10))
    #window.header.setProperty("margin_end", toIntVal(10))
    #window.header.setProperty("margin_top", toIntVal(10))
    #window.header.setProperty("margin_bottom", toIntVal(10))
    window.header.setMarginStart(10)
    window.header.setMarginEnd(10)
    window.header.setMarginTop(10)
    window.header.setMarginBottom(10)
    let label = newLabel("Label")
    gtk4.Box(window.header).append(label)
    let levelBar = newLevelBar()
    levelBar.setValue(0.4)
    levelBar.setHexpand
    gtk4.Box(window.header).append(levelBar)
  else:
    window.header = newHeaderBar()
    #addClass(getStyleContext(window.header), "titlebar")
    addCssClass(window.header, "titlebar")
    #window.header.setTitle("Example header")
    var button = newButton("_Close")
    button.setUseUnderline
    addCssClass(button, "suggested-action")
    button.connect("clicked", quit_cb)
    gtk4.HeaderBar(window.header).packEnd(button)
    button = newButton()
    let image = newImageFromIconName("bookmark-new-symbolic")
    button.connect("clicked", onBookmarkClicked, window)
    button.setChild(image)
    gtk4.HeaderBar(window.header).packStart(button)
  window.setTitlebar(window.header)

proc main =
  gtk4.init()
  #var window: MyWindow
  let window = newWindow(MyWindow)
  addCssClass(window, "main") # gtk_widget_add_css_class (window, "main");
  let provider = newCssProvider()
  provider.loadFromData(Css)
  addProviderForDisplay(getDisplay(window), provider, STYLE_PROVIDER_PRIORITY_USER)
  changeHeader(nil, window)
  let box = newBox(Orientation.vertical, 0)
  window.setChild(box) # gtk_window_set_child (GTK_WINDOW (window), box);
  #window.add(box)
  let content = newImageFromIconName("start-here-symbolic")
  content.setPixelSize(512)
  content.setVexpand
  box.append(content)
  let footer = newActionBar()
  footer.setCenterWidget(newCheckButtonWithLabel("Middle"))
  let button = newToggleButtonWithLabel("Custom")
  button.connect("clicked", changeHeader, window)
  footer.packStart(button)
  #var button1 = newButton("Subtitle")
  #button1.connect("clicked", changeSubtitle, window)
  #footer.packEnd(button1)
  var button1 = newButton("Fullscreen")
  footer.packEnd(button1)
  button1.connect("clicked", toggleFullscreen, window)
  box.append(footer)
  window.show
  #gtk4.main()
  while not done:
    discard iteration(defaultMainContext(), true) # g_main_context_iteration (NULL, TRUE);
  destroy(window) # this is special for this example, see  https://discourse.gnome.org/t/tests-testgaction-c/2232/6

main() # 137 lines
```

说明：相关参考<https://github.com/jdmansour/nim-smartgi>

## 备注

> 备注： 这项工作部分基于 J. Mansour 的早期工作，并得到了 E. Bassi 和其他GTK/Gnome开发人员的支持。`combinatorics`模块由 R. Behrends 慷慨提供。

> 备注： 由于我们的版本已经达到 0.9.9，我们暂时停止了版本发布。因此，您应该使用 "nimble uninstall gintro; nimble install gintro@#head "进行 #head 安装，以获得最新版本。

> 备注： 这终于是 gintro Nim GTK 绑定的 0.9.9 版本了。它包含了我们在过去几个月中应用的一些小修正。请注意，gintro 的版本号是无符号整数，因此会缠绕在一起，所以无论何时再有新版本发布，都可能会再次获得 0.1 的标记（更严重的是，我们犹豫是否已经将下一个 gintro 版本称为 1.0，因为 1.0 表示某种最终状态，而 gintro 永远无法将其归档。所有与 GTK 相关的库都非常复杂，几乎不可能创建完美的绑定。）但由于认真使用 gintro 的用户数量很少，我们可能会在 2022 年底之前将 gintro 从 GitHub 完全移除。这将为我们省去一些持续修复 bug 的工作，用户也可以从其他 20 个 Nim GUI 工具包中选择一个。gintro 能否在 Nim 2.0 或 GTK 5.0 上运行确实是个问题。在为 gintro 工作了 1600 多个小时之后，现在退休也许是个不错的决定。

 > 备注： gintro Nim GTK 绑定的 0.9.8 版包含一些小的修正，其中包括对 gobject 中最新的 uref/unref 名称更改的修正。

> 备注： 在 0.9.7 版的 gintro Nim GTK 绑定中，我们尝试修复 mconnect() 宏生成的代码中使用不带模块前缀的符号导致的问题。 参见<a href="https://github.com/StefanSalewski/gintro/issues/188" data-dl-uid="1223" data-dl-original="true" data-dl-translated="true">#188</a>。不幸的是，这一修复可能会破坏某些现有项目。 我们努力使改动尽可能小：我们尽量让生成模块的内容保持不变，只修改了 gimplgob.nim 和 gen.nim 文件，以及生成的 gisup4.nim 和 gisup3.nim 文件。 示例程序仍能编译并运行。

> 备注： 在 0.9.6 版本的 gintro Nim GTK 绑定中，我们尝试修复了一个与 libsoup 3.0 的出现有关的 bug。 当 gobject-introspection 首次处理 libnice 时，它会加载旧的 libsoup 2.4，然后处理 linsoup 3.0 时，名称冲突会导致错误信息和安装过程挂起。现在，我们总是先处理 libsoup 2.4 和 3.0 版本，然后再处理 libnice，这似乎行得通。对于 libsoup 2.4，我们像以前一样生成一个名为 libsoup.nim 的模块，而 libsoup 3.0 模块现在称为 libsoup3.nim。

> 备注： 在 0.9.5 版 gintro Nim GTK 绑定中，我们添加了一个补丁以支持 Debian Buster 等非常老的 GTK3 版本，更新了 Rectangle、TextIter 等不需要 Nim 代理对象的 "轻型符号 "列表，并最终修复了最近的一个 glib 采购名称冲突问题。

> 备注： 0.9.4 版的 gintro Nim GTK 绑定包含大量修复。 在 0.9.4 之前，由于从 ref 对象到 RootRef 的错误转换，一些 GtkDrawingArea 示例在最新的 Nim 编译器下会出现崩溃。这个问题已经解决，现在绘图区域示例无需手动释放 cairo 资源（如 Context、Surface 和 Pattern）也能正常工作。我们没有更新该示例，因此它仍然包含手动内存管理，但如果你愿意，可以将其注释掉。当你使用 --gc:arc 编译代码时，你确实不需要关心释放 cairo 资源的问题。如果你仍然使用默认的 refc GC，你可以手动释放cairo资源，因为 GC 有延迟，在 GC 激活并释放cairo资源之前，可能会先分配一些 GB。请注意，所有 cairo 函数中只有一小部分已经过测试，因此可能仍然存在错误。 字符串现在以 cstrings 的形式传递给 GTK 函数，因此你可以传递 nil 和空字符串。对于 GTK 而言，nil 值在大多数情况下都有特殊含义，它不同于空的""字符串。对于某些函数参数，nil 是 cstrings 的默认值。 最后，我们现在支持命名为空的标志集，因此您可以使用 BindingFlagsDefault 或 BindingFlags.default 这样的名称，而不是将空集作为 {} 传递给默认标志集值。

> 备注： gintro Nim GTK 绑定的 0.9.2 版本只是修复了 gitlab 最新版 gstreamer 的问题，请参见<a href="https://github.com/StefanSalewski/gintro/issues/138" data-dl-uid="1236" data-dl-original="true" data-dl-translated="true">#138</a>。

> 备注： 0.9.1 版主要是对<a href="https://github.com/StefanSalewski/gintro/issues/133" data-dl-uid="1239" data-dl-original="true" data-dl-translated="true">#133</a> 问题的简单修复。 对于带有方向 in 和传输 full 的 GObject proc 参数，我们必须避免 Nim 内存管理在 Nim 代理对象销毁时销毁 GObject。目前的解决方法是在这种情况下添加一个简单的".ignoreFinalizer = true"，这在大多数情况下都有效。但也许更好的解决办法是改用 ref() gtk 对象。也许这对于 gobjects 来说没有什么区别，但对于像 GtkExpression 这样会被进一步使用的实体来说就有区别了，参见问题<a href="https://github.com/StefanSalewski/gintro/issues/137" data-dl-uid="1240" data-dl-original="true" data-dl-translated="true">#137</a>。不过我们会把这个问题留到下一个 0.9.3 版本。

> 备注： 应用户要求，我们在 0.8.9 版中添加了对 adwaita 库的支持。 该库是 GTK4 的 libhandy 变体，旨在支持移动设备上的 GTK。遗憾的是，我们还不能提供该库的示例程序。git 源中的 C 示例并不小，因此移植到 Nim 至少需要几个小时，因为我们对该库还完全不了解。也许我们明年可以提供一个示例，或者那位用户最终会提供一些东西？

> 备注： 从 0.8.8 版的 gintro Nim GTK 绑定开始，对于像 getStartIter() 这样没有结果但带有 var out 参数的程序，我们创建了一个重载版本，其中的 var out 参数将作为结果返回。因此，我们现在可以写 "let startIter = buffer.getStartIter()"。 我们还尝试修复了 g_file_new_tmp() 中的 out gobject 参数问题。

> 备注： 在 0.8.7 版的 gintro Nim GTK 绑定中，我们对 gen.nim 生成器脚本进行了较大的内部清理，但仍试图生成与 0.8.6 版相同的输出模块。在下一个版本 v0.8.8 中，生成的模块会有一些变化。 另外，在 0.8.7 版中，我们支持 webkit2 for GTK4。

> 备注： 0.8.6 版的 gintro Nim GTK 绑定现在包含了所有标准的 gst 模块，由于最近的请求，还包含了 gtklayershell.nim 模块。由于某些 gst 模块的问题，我们现在会在使用 gobject-introspection 之前调用 gst 模块的 init()。 我们对 GTK3 和 GTK4 也采取了同样的做法，因为它们也提供了 init() 过程。根据最近与 GTK 核心开发人员的讨论，init() 调用是必要的。调用是通过使用 dlopen() 来完成的，为此我们需要提供动态库的名称，这对于 Windows 和 Mac 来说是需要猜测的。

> 备注： 在 0.8.5 版的 gintro Nim GTK 绑定中，我们添加了 webkitgtk 支持。 遗憾的是，由于最新的 webkitgtk 软件包 2.30.4 无法编译到稳定的 GTK4，因此仍然只能支持 GTK3。但我们应该会在几周内获得 GTK4 的 webkitgtk。现在的问题仍然是如何命名 GTK4 版本。GTK3 版本的官方名称是 webkit2，那么我们应该称 GTK4 版本为 webkitgtk4 还是 webkit4 呢？或者不同的名称？

> 备注： 从 0.8.4 版开始，我们支持 Linux 和 Windows 下的 libnice 模块。我们还添加了 gtksourceview5，以支持 GTK4 使用 gtksourceview。该模块之所以被称为 gtksourceview5，是因为底层 C 库已经有了 Mayor 5 版本。因此，您现在可以编写自己的 GTK4 Nim 编辑器了。现在还支持额外的可选 var out 参数，GtkGestures 也应能正常工作。

> 备注：gintro v0.8.3 版本将进行一些内部清理，删除临时数组类型。过去我们使用名称来表示数组参数，效果不错，但使用名称来表示数据类型仍然很难看。因此我们修复了这一问题。我们还为 GList 参数添加了更好的支持。GLists 现在可以转换为 Nim seqs，反之亦然。由于 GList 参数大多数情况下仅在特殊函数中使用，因此这种转换尚未经过测试。另一个变化是，应一位日本用户的请求，我们支持 libnice。由于他打算在仅安装了 glib 和 gobject 但未安装 gtk 的 Windows 8.1 上使用 libnice，我们不得不将模块 gimpl.nim 分解为两个单元，分别称为 gimplgobj.nim 和 gimplgtk.nim，并添加了一个称为 dummygtk.nim 的模块，以便导入该模块。如果这些改动会破坏普通 gtk 用户的使用，请告诉我们。 为了测试 libnice，我们添加了从 C 示例转换而来的 sdp_example.nim，它似乎可以编译和运行。但我们还没有测试过不同计算机之间的实际通信，也没有在 Windows 上测试过。如果您有 libnice 的用例，可以自行测试，可能存在的问题包括 Nim 的字符串 vs seq[uint8] vs plain ptr char，或 C 示例中的 uint 数据类型，我们已将其替换为 Nim 首选的 plain int。调试应该不难，如果你无法自行调试，可以在 github 上提交问题。v0.8.3 版还使用了一些更轻量级的实体，如 gtk.TextIter 或 glib.Mutex，这些实体通常在堆栈上分配，并且只初始化为普通二进制 0。这些类型不需要代理符号。遗憾的是，使用 gobject-introspection 发现这些实体的方法并不可靠，因此我们使用了一个手动创建的列表，其中可能包含错误。 最后，在 v0.8.3 版中，GTK4 书籍<a href="https://ssalewski.de/gtkprogramming.html" rel="nofollow" data-dl-uid="1261" data-dl-original="true" data-dl-translated="true">(https://ssalewski.de/gtkprogramming.html)</a> 中的示例应该可以正常工作。对于 GTK3 示例，我们必须进行一些最基本的修正，因此您可能也需要对自己的代码进行类似的修正。

> 备注： gintro 0.8.2 版本只是对最近的 gstreamer 1.18 进行了修复，该版本添加了一些不常用的 gobject-introspection 内容，导致一些用户无法安装 0.8.1 版本。我们还无法验证基于 gst C tutorial1 的 gst 示例是否能在 gstreamer 1.18 中运行。

> 备注： 应 FedericoCeratto 的要求，我们在 0.8.1 版 gintro 中添加了 libhandy 支持。 Libhandy 尚不能用于 GTK4，但似乎可以用于 gtk+-3.24.22。由于 Gentoo Linux 上的 libhandy 仍然很老，我们使用<a href="https://gitlab.gnome.org/GNOME/libhandy/" rel="nofollow" data-dl-uid="1266" data-dl-original="true" data-dl-translated="true">https://gitlab.gnome.org/GNOME/libhandy/</a>上的最新版本进行了测试。目前唯一可用的示例是从 example.py 转换而来的 /examples/gtk3/handy.nim。

> 备注： 如果你使用的是 Arch Linux，可能仍需确保 curl 或 wget 可用，请参阅<a href="https://github.com/StefanSalewski/gintro/issues/83" data-dl-uid="1269" data-dl-original="true" data-dl-translated="true">#83</a>。 下一个版本将支持 GList proc 参数和结果，但提供 GList proc 参数和结果并非易事，可能会破坏其他一些东西。请注意，模块 gio 的文件类型现在称为 GFile，以避免与 Nims 自带的文件类型名称冲突。

> 备注： 从 0.7.4 版开始，我们支持最新的 GTK 3.98.3，该版本可能在 2020 年底成为 GTK 4。

> 备注： 从 v0.7.3 版开始，我们允许将类型描述符作为 new() 程序的第一个参数，例如<code data-dl-uid="1278" data-dl-original="true" data-dl-translated="true">newButton(MyButtonSubclass)</code>，以支持以 OOP 风格对 Widget 进行子类化和扩展。 请参阅<a href="https://github.com/StefanSalewski/gintro/blob/master/README.adoc#extending-or-sub-classing-widgets" data-dl-uid="1279" data-dl-original="true" data-dl-translated="true">扩展或子类化 Widget</a>的完整示例。以前用于此任务的 init() procs 现已过时，稍后将被移除。这种新方法一般可节省一行源代码，允许使用<code data-dl-uid="1280" data-dl-original="true" data-dl-translated="true">let</code>代替<code data-dl-uid="1281" data-dl-original="true" data-dl-translated="true">var</code>，而且 procs 的命名也更加一致。

> 备注： 从 v0.7.1 版开始，我们在使用<code data-dl-uid="1284" data-dl-original="true" data-dl-translated="true">--gc:arc</code> 编译时添加了析构函数支持，因此子类化对象不会再有内存泄漏问题，请参阅<a href="https://github.com/StefanSalewski/gintro/blob/master/README.adoc#extending-or-sub-classing-widgets" data-dl-uid="1285" data-dl-original="true" data-dl-translated="true">扩展或子类化小部件</a>部分的示例。 在使用默认 (refc) GC 编译时，析构函数将一如既往地使用。对于在 gobject-introspection 中标记了<code data-dl-uid="1286" data-dl-original="true" data-dl-translated="true">nullable</code>标签的对象，当 C 库返回 NULL 时，我们现在会返回代理对象的 nil 值。 因此，根据 C API 文档，您可以检查少数可能返回 NULL 的函数的 nil 结果。 GTK Builder 返回的 Nil 对象现在会导致程序终止（通过 assert()），因为在这种情况下，nil 应表示编程错误。

> 备注： 从 v0.7.0 版开始，我们支持名为 ARC 的新 Nim 内存管理，请参见<a href="https://forum.nim-lang.org/t/5734" rel="nofollow" data-dl-uid="1289" data-dl-original="true" data-dl-translated="true">https://forum.nim-lang.org/t/5734</a>。只需使用<code data-dl-uid="1290" data-dl-original="true" data-dl-translated="true">--gc:arc</code> 编译程序即可。其主要优势在于 ARC 是确定性的，因此更容易发现绑定或程序中的 bug。手动释放资源，就像我们以前释放某些cairo数据结构一样，现在应该不需要了，因为释放时不会出现 GC 延迟。一般来说，这个版本应该更加稳定：对于相同的数据类型，不带选项<code data-dl-uid="1291" data-dl-original="true" data-dl-translated="true">--gc:arc</code>的 Nim 在编译 new() 时，会默默调用带或不带 finalizer proc 参数的函数，但 finalizer 总是会被调用。Nim 手册中已经说明了这种行为，但人们很容易忘记这种奇怪的行为，因此可能会无意中调用终结器。 带有 --gc:arc 选项的 Nim 至少能检测到其中一些错误，现在我们努力避免在 gintro 中混合调用这些终结器。 一般来说，我们会指定一个终结器，并在必要时使用 ref 对象中的一个字段来忽略调用。 对于一个类型，必须始终使用相同的终结器（或始终不使用），而且终结器必须与对象类型本身定义在同一模块中。为了可靠地实现这一点，我们通常必须用模块前缀来限定 finalizer proc。所有这些都需要对 gen.nim 生成器脚本进行更大规模的重写，并有可能引入错误。我们还没怎么测试 v0.7，但 gtk3 目录中的示例至少可以编译并启动。我们仍需更仔细地检查 gimpl.nim 中的宏 - 我们不得不用纯拷贝替换了 deepcopy，并删除了一个（错误的？）GC_ref()。一般来说，我们必须调查可能的内存泄漏。有一个泄漏是不可避免的：如果我们对 Widgets 进行子类化，那么终结器就不会应用于 GTK 对象，因此会导致内存泄漏。请参阅<a href="https://forum.nim-lang.org/t/5825#36241" rel="nofollow" data-dl-uid="1292" data-dl-original="true" data-dl-translated="true">https://forum.nim-lang.org/t/5825#36241</a>。但这应该不是一个太严重的问题，子类化对象通常在程序中只分配一次，而且只要程序在运行，它们就会一直存在。在 gintro 的下一个版本中，我们会考虑只使用析构函数而不使用终结符，参见<a href="https://forum.nim-lang.org/t/5854" rel="nofollow" data-dl-uid="1293" data-dl-original="true" data-dl-translated="true">https://forum.nim-lang.org/t/5854 和</a> <a href="https://forum.nim-lang.org/t/5786" rel="nofollow" data-dl-uid="1294" data-dl-original="true" data-dl-translated="true">https://forum.nim-lang.org/t/5786</a>。这可能会简化代码并使子类化的 GTK 对象释放其内存，但需要再次重写 gen.nim。但这样我们就必须始终使用<code data-dl-uid="1295" data-dl-original="true" data-dl-translated="true">--gc:arc</code>了。也许我们可以通过指定一些有条件的<code data-dl-uid="1296" data-dl-original="true" data-dl-translated="true">when</code>表达式将两者结合起来--我们将拭目以待。 如果您认为安装或编译 v0.7 不可行，那么请在 github issue tracker 上报告问题，并暂时继续使用 v0.6.1。

> 备注： 从 v.0.6.0 版开始，我们支持 gstreamer (gst)。同时，我们将 cairo 模块拆分为 gobject-introspection 基本部分和手动创建部分。不幸的是，gobject-introspection 无法用于非常老的 GTK/cairo 库，因此安装可能会失败。 在这种情况下，请使用 v0.5.5。此外，我们现在支持 gBoxed 类型，这被认为运行良好，但尚未经过充分测试。 安装旧版本的命令应类似于<code data-dl-uid="1299" data-dl-original="true" data-dl-translated="true">nimble uninstall gintro</code>，然后是 nimble install<code data-dl-uid="1300" data-dl-original="true" data-dl-translated="true">gintro@v0.5.5</code>。

> 备注： 从 0.5.3 版开始，我们不再为对象生成字段条目，也不再生成类结构体和私有对象。此外，我们也不再导出类似 gtk_button_new()这样的低级函数。对于真正的高级绑定，我们不需要这些函数。如果这对您来说是一个严重的限制，那么请暂时使用 0.5.2 版，并在 github 上为您的用例创建一个问题，我们会尝试修复它，也许会撤销这些更改。此外，从 0.5.3 版开始，我们将尝试支持 TargetEntryArray、PageRangeArray 和 KeymapKeyArray 等数组参数。使用这些数组参数的情况很少，如果您要使用带有这些参数的函数，可以先检查源代码，因为这些代码是自动生成的，尚未经过测试。

> 备注： 从 0.5.0 版开始，我们也支持 GTK4。GTK4 仍在开发过程中，目前还不面向最终用户，但可以用于迁移测试。对于 GTK4，我们有一个新模块 gsk，以及新版本的模块 gtk、gdk 和 gdkX11，它们与旧版的 GTK3 并不向后兼容。GTK3 和 GTK4 可以同时使用其他模块。因此，我们使用了一个可用于 GTK3 和 GTK4 开发的单一 nimble 软件包。为了归档，我们将新模块命名为 gtk4、gdk4 和 gdkX114，而旧模块仍命名为 gtk、gdk 和 gdkX11。因此，现有的 GTK3 软件无需修改代码。 对于 GTK4，我们提供了一个示例--它现在导入的是 gtk4 而不是 gtk，并且需要 window.showAll() 而不是 window.show()。最终可能会有更多 GTK4 示例，请参见<a href="https://developer.gnome.org/gtk4/stable/gtk-migrating-3-to-4.html" rel="nofollow" data-dl-uid="1305" data-dl-original="true" data-dl-translated="true">https://developer.gnome.org/gtk4/stable/gtk-migrating-3-to-4.html</a> 上的 GTK4 迁移页面。如果 GTK4 在本地计算机上可用，gintro 软件包会尝试安装 GTK4 模块，如果不可用，则会跳过。 要成功检测到 GTK4，必须找到 typelibs。例如，如果您按照<a href="https://developer.gnome.org/gtk4/3.96/gtk-building.html" rel="nofollow" data-dl-uid="1306" data-dl-original="true" data-dl-translated="true">https://developer.gnome.org/gtk4/3.96/gtk-building.html</a> 中的描述从 /opt/gtk 上的源代码安装了 GTK4，那么在执行 "niminate install gintro "之前，您可能需要在 shell 中执行 "export GI_TYPELIB_PATH=/opt/gtk/lib64/girepository-1.0"。目前，gtksourceview 和 vte 还不能用于 GTK4。GTK4 提供了一个名为 gtk4-demo 的官方测试程序--当然，在考虑使用 GTK4 测试 Nim 之前，该程序应该可以在你的电脑上正常运行。