---
title: 手动编写VAPI
author: kernel
date: 2023-11-23 13:34:00 +0800
categories: [Vala语言, VAIP]
tags: [Vala语言, Vala使用技巧, 语言交互]
---

本文档旨在介绍如何编写将现有 C 库绑定到 Vala。如果该库使用的是 GLib，请勿遵循本文档。请阅读[使用 GObject 自省生成 VAPI](https://wiki.gnome.org/Projects/Vala/Bindings)。库可能并不完全遵循 GLib 的编码实践，但修复库以便与 GObject 自省一起工作比编写手动绑定更好。

C 程序员是一群相当自由的人，某些程序可以根据程序员的心情以多种方式完成，而 Vala 则受到更多限制。本指南不可能涵盖 C 程序员编写的不同 API 的所有可能情况。您的工作就是理解 C API，并以对 Vala 友好的语义来呈现它。

本文件包含大量资料，一开始可能难以理解。学习本教程的实用方法是：

1.  首先绑定一个枚举，因为枚举很容易测试。
一旦测试得到了预期的结果，你就知道构建过程成功了。这意味着需要学习“入门”部分和“枚举和标志”子部分。绑定枚举还引入了一个概念，即从 C 到 Vala 并没有一个直接的映射关系。
    
2.  接下来绑定紧凑类的创建和销毁。
这意味着要学习“使用 Vala 的自动内存管理”部分，并开始理解 C 语言中的结构体在 Vala 中可以绑定为简单类型、结构体或紧凑类。可以通过查看 Vala 中的一行 C 代码（如`new MyBoundCompactClass ();` ）来测试绑定。
    
3.  绑定紧凑类的方法。
这是您的绑定开始变得有用的时候，也是对本文档进行概述的时候。一旦你有了一个概览，本文档将成为解决更多棘手的函数绑定问题的参考资料。
    

以上假设库是用面向对象的 C 语言编写的。但 C 绑定只是由结构体和函数组成，因此，足够详细地了解这一点才是本方法的目的。

## 先决条件

要编写绑定程序，请收集以下内容：

-   带头文件的函数库副本
-   库的文档，如果有的话
-   源代码，如果可能的话
-   可用作绑定测试的示例或教程

如果库是用 C++ 编写的，则无法将其绑定到 Vala，除非 C++ 库有单独的 C 绑定（如 LLVM）。

如果您使用的是 vim，不妨在 .vimrc 中添加以下内容：

```
:noremap <F8> "gyiwO[CCode (cname = "<ESC>"gpa")]<ESC>
```

它允许您插入一个属性，以便在光标位于符号上时按 F8 键，更轻松地重命名函数。

## 入门

### VAPI 文件

检查程序库的开发包是否安装了[pkg-config](https://www.freedesktop.org/wiki/Software/pkg-config/)文件（扩展名为 .pc 文件）。如果有，请为您的 VAPI 文件取相同的名称。例如，libfoo.pc 应将 VAPI 命名为 libfoo.vapi。这样就可以自动获取库文件的详细信息，并将其传递给 C 编译器和链接器。

在开发 VAPI 时，针对绑定建立测试的典型命令是

```shell
valac --vapidir . --pkg libfoo program_using_libfoo.vala
```

在`--vapidir`后面的点告诉`valac`在查找 VAPI 文件时包含当前目录。通过`--pkg libfoo`命令行开关，valac会查找名为`libfoo.vapi` 的 VAPI。请注意，`.vapi`后缀被去掉了。如果 VAPI 也与`.pc`文件同名，`valac`将查找并使用`.pc`文件提取相关库的详细信息，然后传递给 C 编译器和链接器。

VAPI 文件示例可在 Vala git 仓库的[vapi 目录](https://git.gnome.org/browse/vala/tree/vapi)中找到。声明由`vapigen`生成的文件是通过 GObject 自省生成的，并非手动编写的绑定示例。

一旦您有了可运行的 VAPI 文件，即使它只是库功能的一个子集，也请考虑共享该文件。请参阅[Vala额外的VAPI](https://gitlab.gnome.org/GNOME/vala-extra-vapis)和[为什么要在上游分发绑定](https://wiki.gnome.org/Projects/Vala/UpstreamGuide)。

### 署名和许可证

如果要通过主要资源库之一发布 VAPI，则需要版权声明。在开始编写绑定程序时处理这一手续可能更容易一些。

版权声明应包括署名和许可证副本。署名是你的姓名和电子邮件地址。这表明您是 VAPI 的作者，同时也是万一第三方违反许可证规定使用绑定文件时的联系人。自由软件和开放源码许可证允许复制 VAPI 文件，只要符合许可证条款即可。许可证应与库的许可证相同。这可确保绑定与程序库之间的兼容性。

版权声明应放在多行注释之间，而不是文档注释之间：

```vala
/*
 * Copyright (c) 2016 My Name <my_email@my_address.com>
 * 
 * This library is free software...[or whichever license is used by the library]
 *
 */
```

文档注释的开头有一个星号，即`/**`，会被`valadoc` 接收。关于`valadoc`的使用将在后面介绍。

### CCode 属性

Vala 以特定风格生成 C 代码，例如 Vala 遵循自身的命名约定和自动生成参数的排序。`CCode`属性可对 Vala 生成的 C 代码进行精细控制，在使用自身约定绑定 C 库时被广泛使用。

CCode属性将用于：

-   包括一个 C 头文件
-   从 Vala 命名规范转换为库命名规范
-   将库与 Vala 的辅助内存管理绑定
-   控制函数调用参数的位置，尤其是 Vala 生成的参数
-   克服各种边界条件

这些属性将在本教程的相关部分进行介绍。有关单个参考资料，请参阅[《Vala 手册》](https://wiki.gnome.org/Projects/Vala/Manual/Attributes)的属性部分。

### 创建根命名空间

通常，一个库的所有绑定都放在一个根命名空间中。例如，libfoo 或 foolib 最好放在名为 Foo 的命名空间中。这符合上述命名约定。例如，初始 VAPI 可以是：

```vala
namespace Foo {
    // bindings
}
```

然后，可以在 Vala 程序中使用该绑定，方法是在命名空间前加上前缀，例如：

```vala
void main () {
    Foo.library_function();
}
```

或将 VAPI 命名空间纳入文件范围：

```vala
using Foo;

void main () {
    library_function ();
}
```

命名空间还提供了一种方便的函数分组方式。通常，对于基于 GLib 的库，`x_y_foo`模式可以直接转换为`x.y.foo` 命名空间。由于大多数 C 语言库并不遵循这些约定，因此情况会稍微复杂一些。作为一般经验法则，请尝试以下方法：

-   如果全局变量、函数、常量、枚举、标志和委托定义只与类和结构体定义有关，则将它们移到类和结构体定义中。例如，将`enum FooOptions`移入`class Foo`中，使其成为简单的`Options`。请注意，结构体不能包含枚举、标记或委托定义，只能包含常量和静态方法。
    
-   以头文件和目录为参考。如果头文件存放在`foo-2.0/db/{handle,transaction,row}.h`或`foo-2.0/db_{handle,transaction,row}.h` 中，或者`foo-2.0/db.h`包含了对`foo_handle`、`foo_tx` 和`foo_row` 的定义，那么创建命名空间Db很有可能是一个逻辑分组。
    
-   为大组相关常量创建命名空间。有时，常量集合无法转换为枚举，在这种情况下，将它们分组到命名空间中更易于管理。

### 包含 C 头文件

`CCode`属性`cheader_filename`通过逗号分隔定义了要包含在生成的 C代码中的头文件列表。例如：

```vala
[CCode (cheader_filename = "libfoo/foo.h")]
namespace Foo {
    // bindings
}
```

尽量将头文件应用于命名空间或包含的类型。将其应用于外部上下文，可以避免在内部上下文中重复使用。

一个库通常会有一个包含多个子头文件的头文件。例如，请参见[glib/glib.h](https://git.gnome.org/browse/glib/tree/glib/glib.h)头文件。在这种情况下，只需包含主头文件。

### 符号名称翻译

Vala 具有从 Vala 到 C 的符号名称翻译规则。默认规则遵循 GLib 命名约定，但对于绑定，名称翻译可通过`lower_case_cprefix`、`cprefix`和`cname` 的CCode属性进行自定义。

下面的示例说明了默认的符号名称翻译规则。Vala 的名称翻译规则适用于 Vala 程序和绑定。使用`valac --ccode name_conversion_example.vala`编译以下示例程序，然后检查 Vala 符号名称是如何翻译的：

```vala
void main () {
    Foo.Bar a = new Foo.Bar ();
    a.test ();
    var b = Foo.Bar.UNCHANGING;
}

namespace Foo {
    [Compact]
    class Bar {
        public const int UNCHANGING = 42;
        public void test () {
        }
    }
}
```

使用`[Compact]`属性使 C 代码更简单，因此更容易阅读，但名称翻译规则也适用于完整的 Vala 类。下面的表格总结了示例中的翻译：

| Vala标识符 | C标识符 | 说明 |
| --- | --- | --- | 
| `Foo.Bar` | `FooBar` | 这是数据类型 |
| `new Foo.Bar ()` | `foo_bar_new ()` | 这是构造函数 |
| `a.test ()` | `foo_bar_test (a)` | 这是一个作用于实例的函数 |
| `Foo.Bar.UNCHANGING` | `FOO_BAR_UNCHANGING` | 类型定义的常数 |


绑定程序库时，Vala 符号名应遵循以下约定，然后可使用`lower_case_cprefix`、`cprefix`和`cname`来确保 C 符号名与程序库相匹配：

| Vala语义 | Vala惯例 | 到C的默认转换 | 通过CCode进行修改 |
| --- | --- | --- | --- |
| 类 | `TitleCase` |  | |
| 常数 | `UPPER_SNAKE_CASE` | 
| 委托 | `TitleCase` | 
| 枚举和标志 | `TitleCase` |
| 字段 | `lower_snake_case` |
| 方法 | `lower_snake_case` |
| 命名空间 | `TitleCase` | `title_case_`<br> `TITLE_CASE_`<br> `TitleCase` | `lower_case_cprefix`<br> `lower_case_cprefix`<br>`cprefix` |
| 属性 | `lower_snake_case` |
| 结构体 | `TitleCase` |
| 动态类型（泛型） | `T`（单个大写字母）。对于map键和值首选`K`、`V`。 |

在适当的情况下，将隐晦的 C 语言名称扩展为更容易理解的 Vala 语言名称（例如，将`Tx`扩展为`Transaction`）。Vala 通常比 C 语言更简洁，因此我们愿意做出不同的选择，比 C 语言程序员更倾向于可读性而非简洁性。特别是，`var`可以节省大量冗长类型名的编写，import 则有助于更好地使用前缀。

请注意以下几点：

-   在名称空间中使用`cprefix`和`lower_case_cprefix`
    
-   在使用`cprefix`和`lower_case_cprefix`时，类的优先级高于命名空间
    
-   在函数和常量中使用`cname`
    

### 代码格式约定

-   用于缩进的制表符
-   括号前空格，括号后无空格
-   等号两边有空格
-   逗号前无空格，逗号后有空格

### 文档和 Valadoc.org

[Valadoc.org](https://valadoc.org/)通常是 Vala 开发人员在寻求如何使用绑定时访问的第一个网站。通过将 VAPI 添加到 Valadoc.org 上的[下载软件包列表](https://github.com/Valadoc/valadoc-org/blob/master/documentation/packages.xml)并向 Valadoc.org[资源库](https://github.com/Valadoc/valadoc-org)提交拉取请求，即可将新的 VAPI 提交到[Vala Extra VAPI](https://gitlab.gnome.org/GNOME/vala-extra-vapis)中。请参阅[libcolumbus 拉取](https://github.com/Valadoc/valadoc-org/pull/39)作为示例。

Valadoc.org 经常重新生成。Valadoc.org 重新生成时，会从`vala-extra-vapis`中调入 VAPI 并从中生成文档。如果某个 VAPI 没有关联文档注释，Valadoc.org 将只显示 VAPI 中的符号。

在 VAPI 中的符号前添加文档注释。文档注释是带有星号的 C 语言多行注释：

```vala
/**
 * Brief description of class Foo
 *
 * Long description of class Foo, which can include an example
 */
[CCode (cname = "foo", ref_function = "foo_retain", unref_function = "foo_release")]
[Compact]
public class Foo {
    // Details of binding
}
```

注释可以包括附加标记。详情请参阅[Valadoc 注释标记](https://valadoc.org/markup)。

可以在本地生成文档，以测试文档的显示效果。首先，下载并构建valadoc：

```shell
git clone git://git.gnome.org/valadoc
cd valadoc
./autogen.sh
make
make install
```

第二，生成文档：

```vala
cd my_binding_directory
valadoc --directory docs --force --package-name mybinding mybinding.vapi
```

这将在`docs`目录中生成 HTML 文档。`valadoc`希望`docs`目录不存在，但`--force`会覆盖这一点。`--package-name mybinding`将在`docs`中创建名为`mybinding`的子目录，其中包含为`mybinding.vapi` 生成的文档。

本地生成的文档将具有与valadoc.org 相同的结构，但视觉样式可能有所不同。

### 版本属性

Vala 符号可使用`[Version]`属性进行注释。这样就可以将一个符号标记为experimental、deprecated，并标明版本信息。例如：

```vala
namespace Test {
        [Version (experimental = true)]
        public void test_function_1 ();

        [Version (deprecated = true)]
        public void test_function_2 ();

        [Version (deprecated_since = "2.0")]
        public void test_function_3 ();

        [Version (deprecated = true, deprecated_since = "2.0", replacement = "test_function_5", since = "1.0")]
        public void test_function_4 ();

        [Version (since = "1.0")]
        public void test_function_5 ();
}
```

## 使用Vala的自动内存管理

在编写 Vala 代码（包括使用 C 库的代码）时，内存管理由 Vala 编译器处理。通常无需手动申请和释放内存。不过，在编写绑定代码时，准确指导 Vala 编译器如何使用 C 库的内存管理调用是整个过程的重要部分。这是一项一次性工作，意味着使用绑定的任何人都可以利用绑定的优势，更容易编写代码。

Vala 的内存分配和类型比大多数语言都要复杂。在 Python 中，所有东西都是动态类型的对象，它们都是先被分配，然后再被垃圾回收。在 C 语言中，内存分配主要由用户处理，类型只是在编译时对内存的描述。Vala 则试图同时覆盖所有这些基础。重要的是，Vala 中的类型暗示了内存管理的一些内容。

Vala 共有 4 种内存管理方案：

| 类型 | 内存管理者 | 需要处理？ | 复制开销 |
| --- | --- | --- | --- |
| 值类型 | C编译器 | 不需要 | 小 |
| 派生类型 | C编译器 | 需要 | 大 |
| 单所有权 | 堆分配器 | 需要 | 大 |
| 引用计数 | 堆分配器 | 需要 | 小 |

### C 语言中的指针

星号`*` 是 C 语言中的间接运算符。间接操作符表示标识符包含指向内存位置的指针。通常还会指明内存位置的数据类型。例如，`int*identifier`表示一个`int`保存在`identifier` 指向的内存位置。不过，数据类型不必指定，可以使用“通用”类型：`void *identifier`。

可以有多个间接层次，例如`char **identifier`。

取地址运算符是双引号，即`&`。

间接操作符和地址操作符的使用与绑定函数签名有关，这将在后面的章节中介绍。有关 C 语言指针的全面解释，请参阅[《C 语言指针须知》](https://boredzo.org/pointers/)。

目前，我们只需了解指针并不说明所指向的内存是如何管理的。从 C 代码中的指针无法得知内存是常量、栈分配还是堆分配。

### C 语言中的常量、栈和堆

在 C 语言中，可以使用一种机制来分配数据，这种机制可以阻止数据在程序运行期间被更改。这些数据被称为常量。数据还可以通过另外两种方式分配：栈和堆。在编写绑定程序时，需要了解这三种方案。这主要是为了让 Vala 代码在使用绑定时能正确分配和释放堆内存，同时也是为了确保绑定不会将堆规则应用到其他两种方案中。

为了更好地理解这三种方案，我们可以分四个阶段来分析它们是如何处理内存的：

1.  声明
2.  分配
3.  初始化
4.  释放

声明告知编译器需要多少内存。例如，`uint8`会让编译器知道至少需要 8 位（一个字节），或者`double`可能比`float` 需要更多内存。每种类型的确切大小取决于平台，将由编译器决定。

分配是专门保留内存区域的过程。内存从哪里分配取决于内存方案。

内存分配完毕后，需要将内存初始化为所需值。例如，`inta = 128;`将把为`int`保留的内存设置为`128`。

取消分配内存意味着程序的其他部分可以再次使用该内存。

| 条目 | 声明 | 分配 | 初始化 | 释放 |
| --- | --- | --- | --- | --- |
| 常数 | 编译时 | 编译器 | 编译时 | 程序退出 |
| 栈 | 编译时 | 编译器 | 运行时 | 编译器 |
| 堆 | 编译时 | 程序员 | 运行时 | 程序员 |

C 编译器会进行一些内存管理。当项目被放置到栈或其他结构中时，编译器会创建存放这些对象所需的空间。否则，将使用`malloc`和`free`从堆中分配空间。如果任何实例包含对其他实例的引用，则需要辅助函数来分配和取消这些引用。

### Vala中的所有权概念

#### 绑定到 C 堆处理程序

Vala 的独特功能之一是同时拥有单一所有实例和引用计数实例。引用计数实例可以存储在不同的位置，并通过计算引用的数量来进行内存管理；当不再有对该实例的引用时，就会销毁该实例。单独拥有的实例只有一个所有权引用，当该引用被销毁时，实例也随之销毁。因此，引用计数的对象可以通过增加引用计数来“复制”，而单个拥有的实例则无法在不复制其中实际数据的情况下进行复制（如果可能的话）。

虽然这主要是对象的问题，但 Vala 中的所有实例都必须采用其中一种内存管理方案。不同类型的对象可以遵循不同的方案，某些类型还可以根据声明中的细微差别采用不同的方案。

| Vala类型 | 分类 | C类型 | 需要内存管理绑定吗？ |
| --- | --- | --- | --- |
| 枚举和标志 | 值 | int | 不需要 |
| 委托`(has_target = false)` | 值 | 函数指针 | 不需要 |
| 委托`(has_target = true)` | 值 | 函数指针和Void指针 | 不需要 |
| 委托`(has_target = true)` | 单所有权 | 函数指针和Void指针 | 需要，使用`free_function` |
| 简单类型结构体 | 值 | 各种基本类型或一个结构体 | 不需要 |
| 结构体 | 值 | 结构体，但是作为结构体指针传递 | 不需要 |
| 结构体 | 派生类型 | 结构体，但是作为结构体指针传递 | 需要，使用`destroy_function` |
| 紧凑类 | 单所有权 | 指向结构体的指针 | 需要，使用`free_function` |
| 紧凑类 | 引用计数 | 指向结构体的指针 | 需要，使用`ref_function`和`unref_function` |
| 指针 | 值 | 指向内容的指针 | 不需要 |
| 数组 | 单所有权 | 元素类型的指针 （包括int类型的长度） | 需要，使用 `free_function` |

## 识别 C 代码中的 Vala 语义

C 和 Vala 之间的一个重要区别是 Vala 在语义上更具表现力。例如，在 C 语言中`char*`有多种含义。它可以是一个字符串、一个数组、一个指向单个字符的指针、一个返回字符的输出参数、一个指向将被例程修改的字符的指针。这个指针是否可以为空也完全不清楚。Vala 在语法上表达了这些差异，因此编写绑定代码需要理解原始代码的意图。

最简单的方法是先查看头文件，确定所有需要绑定的重要类型。为每个类型查找分配函数、复制函数和清理函数。从中可以推断出正确的绑定策略。

### 常数

本小节介绍

-   C 语言中的`#define`预处理器指令
    
-   Vala编译器遵循的阶段

常量在程序运行过程中不会变化，必须是一个简单的类型或字符串。举例来说，如果 C 库通过`#define`语句定义了一个常量：

```c
#define CUSTOM_PI 3.14159265358979323846

```

`#define`是预处理器的简单文本替换。因此，在编译 C 代码之前，C 预处理器会将`CUSTOM_PI`的相关位置替换为`3.14159265358979323846`。这就是没有给出类型信息的原因。此外，由于这是在编译前进行的，因此隐含了该值是常量。

在将其绑定到 Vala 时，类型信息和它是常量的信息都是明确的：

```vala
public const double CUSTOM_PI;
```

需要注意的一点是，值不绑定，只绑定标识符。Vala 会在生成的 C 代码中使用标识符，然后 C 预处理器会在编译前将其替换为值。

### 枚举和标志

虽然 C 语言支持枚举，但 C 程序员通常不使用枚举，而是使用 `#defines`。这两种结构都可以绑定到 Vala 枚举。

第一个示例是 C 枚举和 Vala 枚举之间的直接映射。C：

```c
typedef enum {
    FOO_A,
    FOO_B,
    FOO_C,
} foo_e;
```

和Vala绑定：

```
[CCode (cname = "foo_e", cprefix = "FOO_", has_type_id = false)]
public enum Foo {
    A,
    B,
    C
}
```

请注意上例中`cprefix`是如何在生成 C 时将`FOO_`前置到所有 Vala 值中的。

第二个示例展示了如何将 C 语言中的一系列常量定义映射到 Vala 枚举中：

```c
#define BAR_X 1
#define BAR_Y 2
#define BAR_Z 3


```

```vala
[CCode (cname = "int", cprefix = "BAR_", has_type_id = false)]
public enum Bar {
    X,
    Y,
    Z
}
```

检查使用枚举的地方，以确定正确的类型，不过`int`和`unsigned int`是最常见的类型。

使用可组合位模式也是一种常见用法。这些可转换为 Vala 标志枚举。

```
#define FOO_READ (1<<0)
#define FOO_WRITE (1<<1)
#define FOO_CREATE (1<<2)

```

```vala
[CCode (cname = "int", cprefix = "FOO_", has_type_id = false)]
[Flags]
public enum Foo {
    READ,
    WRITE,
    CREATE
}
```

在 Vala 中，枚举和标志可以有成员函数。尤其是类似 `strerr` 的函数最好转换为成员函数。

枚举也可以继承，因此如果一组标志是另一组标志的超集，但它们在逻辑上是独立的，就可以使用继承来实现。

```c
#define FOO_A 1
#define FOO_B 2
#define FOO_C 3
#define FOO_D 4
/* takes FOO_A or B only */
void do_something(int);
/* takes any FOO_ value */
void do_something_else(int);
```

```vala
[CCode (cname = "int", cprefix = "FOO_", has_type_id = false)]
public enum Foo { A, B }
[CCode (cname = "int", cprefix = "FOO_", has_type_id = false)]
public enum FooExtended : Foo { C, D }
```

### 简单类型结构体

C 语言库经常为数字、大小和偏移定义新类型。要将这些类型转换到 VAPI 文件中，只需在结构体中使用SimpleType属性，并从 C 头文件中继承相同的简单类型即可。

举个例子：

```
typedef uint32_t people_inside;
```

将在 VAPI 文件中定义为

```vala
    [SimpleType]
    [CCode (cname = "people_inside", has_type_id = false)]
    public struct PeopleInside : uint32 {
    }
```

从现有类型继承时，所有方法都将沿用。对于大小和偏移量，这可能是可取的；但对于句柄，这可能是不可取的。例如，UNIX 文件描述符是以整数形式存储的，但将两个文件句柄相加或相乘就没有意义了。在这种情况下，最好不要从数值类型继承，而是添加`IntegerType (rank=X)`，这样 Vala 编译器就能在需要时自动将类型转换为适当大小的整数（例如，从整数常数初始化）。

XCB 的一个例子：

```
typedef uint32_t xcb_atom_t;
```

将在 VAPI 文件中定义为

```vala
    [SimpleType]
    [IntegerType (rank = 9)]
    [CCode (cname = "xcb_atom_t", has_type_id = false)]
    public struct AtomT {
    }
```

在`glib-2.0.vapi`和`posix.vapi`文件中定义的常用类型的级别是：

| Rank | glib-2.0的类型 | 其他应用 |
| --- | --- | --- |
| 1 | gint8<br>gfloat |  |
| 2 | gchar<br>gdouble |  |
| 3 | guchar <br>guint8 | Posix.cc_t |
| 4 | gshort <br>gint16 |  |
| 5 | gushort <br>guint16 |  |
| 6 | gint <br>gint32 | Posixpid_t |
| 7 | guint <br>guint32 <br>gunichar | Posix.speed_t <br>Posix.tcflag_t |
| 8 | glong <br>gssize <br>time_t | Posix.clock_t |
| 9 | gulong <br>gsize | Posix.nfds_t <br>Posix.key_t <br>Posix.fsblkcnt_t <br>Posix.fsfilcnt_t <br>Posix.off_t <br>Posix.uid_t <br>Posix.gid_t <br>Posix.mode_t <br>Posix.dev_t <br>Posix.ino_t <br>Posix.nlink_t <br>Posix.blksize_t <br>Posix.blkcnt_t |
| 10 | gint64 |  |
| 11 | guint64 |  |

### 结构体

请注意，C 语言侧的结构体可以包含与 Vala 等同的对象实例数据，因此可以绑定到 Vala 中的紧凑类。这将在后面的章节中介绍。本节将介绍如何将 C 结构和 C 基元绑定到 Vala 结构。

C 语言中的一种常见模式是父代结构，就像下面这样：

```c
typedef struct {
    int a;
    int *b;
} foo_t;
void foo_init(foo_t*);
void foo_free(foo_t*);
```

正确的绑定方法是：

```vala
[CCode (cname = "foo_t", destroy_function = "foo_free", has_type_id = false)]
public struct Foo {
    int a;
    int *b; // We can do better later
    [CCode (cname = "foo_init")]
    public Foo ();
}
```

它的最大陷阱在于命名：`foo_free`并不释放传递给它的指针。除了阅读`foo_free` 的实现之外，我们可能无法确定这一点。对于结构体来说，必须给出`Foo`的完整结构。对于结构紧凑的类，它可能是不透明的（即不提供结构的内容），但也不一定。

下一个示例说明了空`destroy_function`的使用，并为结构体设置了默认值：

```c
typedef struct {
    int x;
    int y;
} bar_t;
#define BAR_INITIALIZER {0, 1}

```

```vala
[CCode (cname = "bar_t", destroy_function = "", default_value = "BAR_INITIALIZER", has_type_id = false)]
public struct Bar {
    int x;
    int y;
}
```

值得注意的是，如果一个结构体没有指定 destroy 函数，Vala 会在结构体中存在任何看起来需要去分配的字段的情况下生成一个 destroy 函数，根据上下文的不同，其行为可能是正确的，也可能是不正确的。一个空的`destroy_function`将保持生成代码的正确性，并防止 Vala 生成一个析构函数。

### 紧凑型类

Vala 有[三种类型的类](https://wiki.gnome.org/Projects/Vala/Manual/Classes)：GObject 子类、GType 类和紧凑类。非基于 GLib 的 C 库的结构体可以绑定到 Vala 的紧凑类中。

#### 单所有权类

最常见的情况是单所有权的紧凑型类，它遵循其中一种模式：

```c
typedef struct foo Foo;
/* Create a new Foo handle. */
Foo *foo_make(void);
/* Make a copy of a Foo. */
Foo *foo_dup(Foo*);
/* Free a Foo handle. */
void foo_free(Foo*);

typedef struct bar *Bar;
/* Open a new Bar from a file, NULL if an error occurs. */
Bar bar_open(const char *filename);
/* Dispose of a Bar when finished. */
void bar_close(Bar);
```

它们都应绑定为紧凑型类。`foo_make`和`bar_open`函数将分配内存，并创建一个新的类型实例（这时文档就很有用了）。这两个函数之间有一个重要的细微差别：指针在哪里被提及。在`Foo` 的例子中，指针在每个函数中都会被提及，而`Bar`则是在类型定义中就有了。Vala 总是会添加一个星号，因此`Bar`实际上是使用`struct bar` 绑定的。

第二个区别是构造函数：`Foo` 的构造函数不会失败，但`Bar` 的构造函数可能会失败。Vala 构造函数不允许返回 null。`Bar` 的构造函数最好绑定为静态方法，因为静态方法可以返回 `null`。

```vala
[CCode (cname = "Foo", free_function = "foo_free")]
[Compact]
public class Foo {
    [CCode (cname = "foo_make")]
    public Foo ();

    [CCode (cname = "foo_dup")]
    public Foo dup ();
}

[CCode (cname = "struct bar", free_function = "bar_close", has_type_id = false)]
[Compact]
public class Bar {
    [CCode (cname = "bar_open")]
    public static Bar? open (string filename);
}
```

如果需要显式复制，可在成员函数中加入一个名为`dup()` 的复制函数。

#### 引用计数类

引用计数类通常有如下模式：

```c
typedef struct foo Foo;
Foo *foo_new();
Foo *foo_retain(Foo*);
void foo_release(Foo*);
```

并应绑定为：

```vala
[CCode (cname = "foo", ref_function = "foo_retain", unref_function = "foo_release")]
[Compact]
public class Foo {
    [CCode (cname = "foo_new")]
    public Foo ();
    [CCode (cname = "foo_retain")]
    public void @ref ();
    [CCode (cname = "foo_release")]
    public void unref ();
}
```

提供`ref`和`unref`函数是为了方便用户在难以满足要求情况下手动更改引用计数。

### 函数

这些函数可单独工作，无需事先调用其他函数即可使用。这是`sync`系统调用的 Posix VAPI 文件的一个简单示例：

```vala
[CCode (cname = "sync")]
void sync();
```

ccode 属性`cname` 指定了要使用的 C 名称。这样可以避免valac将当前命名空间追加到函数名称中，确保在 vala 中调用`Posix.sync()`，将映射到在 C 中调用`sync()`，而不是`posix_sync()`。

### 委托

C 语言允许定义函数指针，即指向符合特定签名的代码指针，这些代码可以被执行。这样做的主要问题是，它不能通过库将信息从被调者传递到调用者。在其他语言中，闭包是对代码和状态的封装。C 程序员有时会通过传递一个“用户数据”或“上下文”的 void 指针来模拟这种行为，该指针充当闭包的状态部分。

Vala 支持这两种模式：委托可以是有目标的（即闭包），也可以是无目标的（即函数指针）。这由 `has_target` 值控制，默认值为 true。目标位置被假定为参数列表中的最后一个值，这也是大多数 C 程序的典型位置，不过偶尔也会放在第一个。

```
typedef int(*compute_func)(int a, int b);
typedef double(*analyze_func)(int a, int b, void *userdata);
```

```
[CCode (cname = "compute_func", has_target = false)]
public delegate int ComputeFunc (int a, int b);
[CCode (cname = "analyze_func")]
public delegate double AnalyzeFunc (int a, int b);
```

如果上下文的位置不是最后一个参数，请按照[更改参数](https://wiki.gnome.org/Projects/Vala/ManualBindings#Changing_the_Position_of_Parameters)位置的方法设置 CCode 属性`delegate_target_pos`。

C 程序员通常不会为函数指针创建`typedef`，而是直接将其包含在内。创建一个委托，不要设置`cname`。如果可能，请为函数库提供一个补丁，以创建一个`typedef`。

## 绑定C函数的基本原理

函数签名包括函数的参数和任何返回值。

### 输出和引用参数及返回值

C 语言大量使用 out 参数作为返回的替代方法。不幸的是，由于这种返回系统的非统一性，它相当令人困惑。

对于除结构体以外的所有类型，在返回时都会按照惯例返回实例。任何补充信息（委托目标、数组长度）都会作为 out 参数悄悄附加。要返回另一个值，可以将一个参数声明为out。Vala 将假定函数接受指向该值的指针，并在返回时填充该指针。ref参数与之类似，但必须在调用函数前初始化该参数，并且函数可以操作其值。

请考虑以下几点：

```c
int div_and_mod(int a, int b, int *mod) {
    *mod = a % b;
    return a / b;
}
```

```vala
public int div_and_mod (int a, int b, out int mod);
```

这对类类型参数也同样适用：

```c
int open_file_and_fd(const char *filename, FILE **file) {
    FILE *f = fopen(filename, "r");
    if (file)
    *file = f;
    return (f == NULL) ? -1 : fileno(f);
}
```

```vala
public int open_file_and_fd (string filename, out FileStream file);
```

对于数组和委托，这意味着同时返回参数及其相关参数：

```c
void do_approximation(int *input_array, int input_length, int **output_array, int *output_length);
```

```vala
public void do_approximation (int[] input, out int[] output);
```

需要注意的是，当你想到“输出数组”时，实际上你想要的可能只是一个缓冲区。如果调用者分配了内存，你需要的是缓冲区，而不是输出数组。

返回结构体则截然不同。因为结构体的内存是由调用者分配的，所以结构体的 out 参数与普通指针没有区别。此外，返回结构体实际上意味着包含一个隐藏的 out 参数。

```vala
public struct Foo { ... }
public Foo get_foo (int x);
public void get_foo2 (int x, out Foo f);
```

```c
void get_foo(int x, foo *ret);
void get_foo2(int x, foo *ret);
```

如果要直接返回结构体，问号操作符会将其框住，使其看起来像是堆分配的：

```vala
public Foo? get_foo (int x);
public int make_foo (int y, out Foo? f);
```

```c
foo *get_foo(int x);
int make_foo(int y, foo **f);
```

所有权适用以下规则。

### 所有权

除非使用owned关键字标记，否则所有参数默认为非所有。除非使用unowned关键字标记，否则所有返回值、ref和out参数默认为所有。上述基本类型没有所有权，因为它们可以随意复制。

函数通常会返回其中一个输入值，尤其是在填充缓冲区时。所有权的正确性至关重要。如果处理不当，Vala 会获取它认为必须释放的指针的第二个副本，并释放同一块内存两次，从而导致在 Valgrind 中耗费大量时间。

**如果所有权语义不正确，要么是编写了内存泄漏程序，要么是编写了双重释放程序**。通常情况下，我们需要阅读源代码才能绝对确定所有权语义是正确的。

通常情况下，C程序员会将返回值标记为 `const`，但它们并不属于自己。

另请参见[依赖类型所有权](https://wiki.gnome.org/Projects/Vala/ManualBindings#Dependently_Typed_Ownership)

### 无效性

对于大多数类型来说，添加问号允许类型为空。一般来说，C 程序员在表达特定参数是否为空方面做得很差。对于任何指针类型（数组、精简类、数组和委托），空值不会改变 C 语言的类型。也就是说，如果Foo是一个类，那么`Foo foo`和`Foo? foo`具有相同的 C 语言签名。对于简单类型、枚举和标志，添加 nullability 会将类型提升为指针。也就是说，`bool b`的 C 语言类型是`gboolean b`，`而bool? b`的 C 语言类型是`gboolean *b`。父结构体是一种特殊情况。当作为参数传递时，它们总是作为指针传递，因此空值性只带来语义上的区别；当作为返回值时，空值性会改变行为，这将在下文中讨论。

Vala 总是假定输出参数可以为空。例如

```vala
public delegate void ComputeFunc (int x);
public void get_compute_func (double epsilon, out ComputeFunc func);

ComputeFunc f;
get_compute_func (3.14158, out f);
f (3); // f should never be a null pointer.
get_compute_func (2.72, null); // This is perfectly okay according to Vala.

```

需要注意的是，空值性指的是参数的类型，而不是参数的处理方式。许多 C 语言库在访问 out 参数之前不会检查该参数是否为空，从而导致段错误。Vala 中没有导致这种情况发生的语法。

### 静态方法

枚举、标志、简单类型、结构体和类都可以包含函数。当 Vala 编译器生成 C 函数调用时，数据结构将作为第一个参数包含在内。为防止自动生成参数，请在 VAPI 中的函数定义中使用static关键字。

事实上，绑定静态方法比绑定成员方法更简单，因为没有实例。应注意将静态方法组织到符合逻辑的位置：有些应放在包含命名空间中，有些应放在类型定义中。一般来说，产生类型实例的方法（即像构造函数一样可能会失败的方法）属于类型定义。

### 更改生成参数的位置

Vala 的默认行为是保持 Vala 调用器中参数的位置与 C 函数调用者中参数的位置相同。在 Vala 端未明确参数（例如实例数据）的情况下，Vala 会假定参数位于特定位置。实例数据被假定为 C 函数的第一个参数，但可以通过`instance_posCCode` 属性将其更改为任何位置。Vala 位置系统可用于实例位置(`instance_pos`)、数组长度位置 (`array_length_pos`)、委托目标位置 (`delegate_target_pos`)，甚至可用于重新排列参数(pos)。

Vala 的位置系统一开始有点令人困惑，因此需要解释一下。 假设我们有一个如下的 Vala 函数：

```vala
public class Foo {
    public delegate int Transform (double a);
    public int[] compute (int x, Transform t);
}
```

生成的`compute`签名将是

```c
int *foo_compute(Foo self, int x(position = 1), foo_transform t(position = 2), void *t_userdata, int *array_len);
```

我在 Vala 中逐字出现的参数上标注了它们的位置。同样，`t_userdata`必须大于 2，`array_len`必须大于`t_userdata`，这样排序才有意义。Vala 允许用浮点数值来描述这种排序。可以把self想象成位置 0，把t 的上下文想象成位置 2.1，把返回的数组长度想象成位置 2.2。这只是一组可能的值。它也可以分别是 0.9、2.5、2.8，并产生相同的结果。

默认情况下，Vala 会将实例设置为 0，将任何数组长度设置为数组位置加 0.1，将任何委托的目标设置为委托位置加 0.1，将任何自有委托的析构函数设置为委托位置加 0.2。

如果顺序与 C 功能不符，可以使用适当的值重新排序，但必须在头脑中保持总顺序的整洁。

### 默认值和更改参数位置

由于 C 语言没有默认参数，因此有时会有重复的 C 语言函数以这种方式运行：

```c
int foo_compute(Foo *f, int base_height);
int foo_compute_ex(Foo *f, int base_height, Table *t, struct opts *opts);
```

由于 Vala 确实有默认参数，因此只绑定扩展版本可能会有好处，但前提是默认值不太可能改变。这通常适用于文档中写明“设置为空值以自动确定”的情况。如果不确定，最好两种都绑定。

```vala
[CCode (cname = "foo_compute_ex")]
public compute (int base_height, Table t = null, opts? opts = null);
```

### 用 Vala 封装器更改签名

使用 Vala 编写的封装函数可以调整现有函数签名。这可以使签名对 Vala 更友好。

通常的做法是将 C 绑定设置为私有，并让封装器调用私有方法。封装器也写入 VAPI 文件。

### 变长参数（又称`...`）

C 语言的变长参数系统非常诡谲，有很多潜在的破解方法。不幸的是，Vala 继承了它们。Vala 增加了一些安全措施，但也带来了一些新问题。

其中一个安全措施是，如果方法的`CCode`属性包含`sentinel = "X"`，那么 X 将始终作为最后一个参数被追加。由于列表通常以特殊值（通常为空）结束，因此这可以防止变量参数超限。

此外，Vala 还可以通过添加`PrintfFunction`或`ScanfFunction`属性，对printf-like 和scanf-like 函数进行类型检查。不过，如果格式字符串被修改为包含特殊值，这些格式标记将无法正常工作。

附加到函数末尾的返回值（如数组长度返回值和委托上下文）通常会与变量参数产生不良交互，因为 Vala 编译器会错误地将参数置于定义中的`...`之后。在处理变量函数时，最好明确指定所有位置。

### 不返回的函数

如果函数永远不会返回，`NoReturn`属性可以让编译器的分析器知道，在该语句之后执行的任何代码都不会被执行。这种情况很少见，但对调用`abort`或`exit`的语句很有用。

### 更改实例引用的方法

有时，方法会返回一个指向实例的新指针（想想 realloc）。在 VAPI 中声明函数返回 void，并添加属性`ReturnsModifiedPointer`。

```c
typedef struct table Table;
Table *table_grow(Table *t, size_t object_count);
```

```vala
[Compact]
[CCode (cname = "Table")]
public class Table {
    [ReturnsModifiedPointer]
    public void grow (size_t object_count);
}
```

### 销毁实例引用的方法

如果一个方法要销毁实例（即释放实例），可以用`DestroysInstance`属性来标记。该方法必须返回 void。尽管在大多数情况下，这种方法会被绑定为紧凑类的`free_function`。

如果函数会销毁一个实例，但提供了一个可用的返回值，则应将其绑定为一个静态方法，该方法会获取一个实例的自有变量：

```c
typedef struct transaction Transaction;
Transaction begin_tx(Database *db);
void transaction_abort(Transaction *tx);
void transaction_commit(Transaction *tx);
bool transaction_try_commit(Transaction *tx);
```

```vala
[Compact]
[CCode (cname = "Transaction", free_function = "transaction_abort")]
public class Transaction {
    public Transaction (Database db);
    [DestroysInstance]
    public void commit ();
    public static bool try_commit (owned Transaction tx);
}
```

## 添加Vala友好语义

所有绑定的方法都应该是公共的，除非是在某些尴尬的情况下。Vala 编译器不尊重 VAPI 文件中的可见性，因此定义私有方法只是防止它们出现在[Valadoc](https://wiki.gnome.org/Projects/Valadoc) 中，而不是防止它们被访问。

Vala 有一些特殊的方法名称，允许使用 Vala 语法。可以使用CCode属性捕捉 C 和 Vala 之间的差异。

### to\_string () 方法

将方法绑定为`to_string ()`将允许在 Vala 字符串模板中使用该方法，而无需在 Vala 源代码中写入方法标识符。

### 属性

Vala 允许紧凑型类具有属性，这些属性是 `get` 和 `set` 方法对的语法糖。通常，具有不透明实现的 C 对象会提供一系列函数来查询实例的状态。这些函数可以通过以下方式转换为属性：

-   `get`方法的签名是`T get(I self)`，`set`方法的签名是`void set(I self, T val)`。它们实际上不必成对出现。
    
-   `get`方法不会产生用户无法察觉的副作用。
    
-   `get`方法的调用成本很低。
    
-   `set`方法不会返回错误信息。
    

与大多数返回类型不同，除非明确标示为`owned`，否则`get`方法的返回被假定为非拥有。

考虑一下

```c
typedef struct foo Foo;
int foo_item_count(Foo f);
int foo_max_items(Foo f);
void foo_set_max_items(Foo f);
```

```vala
public class Foo {
    public int item_count {
        [CCode (cname = "foo_item_count")] get;
    }
    public int max_items {
        [CCode (cname = "foo_max_items")] get;
        [CCode (cname = "foo_set_max_items")] set;
    }
}
```

所有常见的CCode属性都可以应用于`get;`和`set;`，而owned属性则可以应用于更改`get;` 的默认所有权。请注意，改变属性的所有权是错误的，除非实例实际上并不拥有`set;` 提供给它的值。

### 集合

Vala 有几个标准方法名，这些方法名是为与 Vala 语法（如`foreach`）配合使用而设计的。

Vala 使用`get ()`方法实现方括号索引语法。例如，一个list实例的get方法返回一个`list`项，即`list.get (index)`，也可以写成`list[index]`。

在下一个示例中，C 函数签名返回集合中的一个项目：

```c
blkid_partition
blkid_partlist_get_partition (blkid_partlist ls,
                              int n);
```

这可以在 VAPI 中绑定为：

```vala
[Compact]
[CCode (cname = "blkid_partlist")]
public class ListOfPartitions {
    [CCode (cname = "blkid_partlist_get_partition")]
    public unowned Partition get (int index);
}
```

请注意，`[CCode (cname = "blkid_partlist_get_partition")]`用于将 Vala 方法名称`get`更改为 C 语言所需的名称：

```vala
var partition = partitions [count];
```

`set`是Vala 用来替换集合中某一项目的方法。`set`必须返回`void`。

`get`和`set`这两种索引方法可以和 C 函数一样接受多个参数，从而可以绑定多维索引。在 Vala 中使用`set`时，最后一个参数必须是新值。

通过将 Vala 中的size属性绑定到用 C 语言返回集合大小的函数上，就可以在集合中使用 Vala 的`foreach`关键字。同时还需要使用获取索引方法。下面的示例延续了上面的 PartitionList 示例。获取列表大小的 C 函数签名是：

```c
int
blkid_partlist_numof_partitions (blkid_partlist ls);
```

这将绑定为：

```vala
[Compact]
[CCode (cname = "blkid_partlist")]
public class ListOfPartitions {
    [CCode (cname = "blkid_partlist_get_partition")]
    public unowned Partition get (int index);
    public int size { [CCode (cname = "blkid_partlist_numof_partitions")] get; }
}
```

请注意，CCode的`cname`译名位于属性主体内部。

现在可以在 Vala 代码中使用foreach 绑定：

```vala
foreach (var partition in partitions) { /* do something with the partition */ }
```

如果集合为非所有，则 Vala 会给出错误信息：
`duplicating ListOfPartitions instance, use unowned variable or explicitly invoke copy method.`（正在复制 ListOfPartitions 实例，请使用非所有变量或显式调用复制方法。）
请参见[Bug 661876](https://bugzilla.gnome.org/show_bug.cgi?id=661876)。

对于非自有集合，for循环仍然有效：

```vala
for (int count = 0; count < partitions.size; count++) {
    var partition = partitions [count];
    /* do something with the partition */
}
```

对于容器实例，Vala 提供了语法糖，可将某些操作转换为方法调用：

```vala
x in a -> a.contains (x)
a[x, y] -> a.get (x, y)
a[x, y] = z -> a.set (x, y, z);
foreach (var x in a) { ... } -> var x; var i = a.iterator (); while ((x = i.next_value ()) != null) {...}
foreach (var x in a) { ... } -> var i = a.iterator (); while (i.next ()) { var x = i.get (); ... }
```

如果合适，提供与这些原型相匹配的方法将允许使用这些语法糖。

`contains`必须返回`bool`。

迭代器需要一个中间对象来保持迭代状态。该类必须实现一个 `next_value` 函数，返回下一个值，如果要停止迭代，则返回空值；或者它可以有一个`next`方法，其签名为`bool next ()`，用于移动到下一个元素，如果有，则返回 true；还可以有一个`T get ()`方法，用于获取迭代器的当前值。C 程序很少有这样的接口。

请根据自己的判断来决定是否使用这些约定。这虽然是对接口的修改，但确实会使生成的接口更易于使用。

## 绑定 C 函数的参数和返回类型

### 基本类型

最基本的类型（`int`、`double`、`size_t`）可以简单地翻译。有些类型有不同的版本（例如，`uint32_t`和`u_int32_t`是相同的，但在不同的头文件中定义），但在绑定时都可以统一。

### 结构体

大多数库都接收通过引用传递的结构体，Vala 的默认行为也是通过引用传递结构体。因此，要在函数或方法调用中将结构体作为参数传递，只需指定结构类型和变量名称即可。例如 C 代码

```c
typedef struct foo {
    int x;
    int y;
};
void compute_foo(foo *f);
```

将被绑定为：

```vala
[CCode (cname = "foo", has_type_id = false)]
public struct Foo {
    public int x;
    public int y;
};
void compute_foo(Foo f);
```

很少有 C 库函数是为了接收通过值而非引用传递的结构体而编写的。您会在 C 函数的参数中看到struct关键字。您还可能看到const struct。为了让 Vala 按值传递结构体，需要在结构体的 Vala 绑定中添加`[SimpleType]`注解。下面是 C 语言中的模式：

```c
typedef struct foo {
    int x;
    int y;
};
void compute_foo(struct foo f);
```

将被绑定为：

```
[CCode (cname = "foo", has_type_id = false)]
[SimpleType]
public struct Foo {
    public int x;
    public int y;
}
void compute_foo(Foo f);
```

### 数组

Vala 数组的设计符合大多数 C 数组语义。由于 C 数组通常没有明确的长度，Vala 需要特殊的提示才能知道该怎么做。关于数组的长度，有以下几种情况。对于参数，附加在该参数上的CCode属性控制数组的绑定。对于返回值，**方法的** CCode属性控制数组的绑定。

#### 数组长度作为参数传递

默认情况下，Vala 假设是第一种情况，并进行以下转换：

```vala
void foo (double[] array);
double[] foo (float f);
```

```c
void foo(double *array, int array_length);
double *foo(float f, int *array_length);
```

如果 C 代码这样做，仍有两个潜在的不匹配：参数的顺序和数组长度的类型。通常情况下，数组长度是`size_t`或`unsigned int`。`array_length_pos`可以移动数组长度参数的位置，参见[改变参数生成位置](https://wiki.gnome.org/Projects/Vala/ManualBindings#Changing_the_Position_of_Generated_Arguments)。`array_length_type`指定一个字符串，表示数组的 C 类型（例如`size_t`）。

#### 数组为空终止

`array_null_terminated`将假定数组像字符串一样以空值结束，并通过遍历数组中的项目自动设置数组长度。由于 Vala 总是以最后一个元素为空在数组中分配填充，因此传递一个 Vala 声明的数组并不涉及以任何方式修改数组。

#### 数组长度是一个常量表达式

`array_length_cexpr`可以设置为填充数组值的 C 表达式。它不能访问数组、被调用对象的实例或任何其他上下文。它必须是一个无上下文的表达式。

#### 数组长度未知

如果数组长度未知，在CCode属性中设置`array_length = false`将导致 Vala 将数组的.length属性设置为-1，并在作为参数使用时不传递长度。

#### 通过一些笨拙的方法了解数组长度

这只适用于返回的数组。如果数组的长度可以确定，但不是绝对的，则可以包含一个封装函数，将数组的.length 属性设置为正确的值。请参阅数组[长度](https://wiki.gnome.org/Projects/Vala/ManualBindings#Array_Lengths)。

### 字符串和缓冲区

在 C 语言中，字符串和缓冲区通常被当作数组处理，但在 Vala 中可能需要稍微精细一些。在 Vala 中，字符串是UTF-8 数据的空端列表，不可更改。如果使用情况并非如此，则处理该数据的首选方式是uint8数组。

函数经常使用一个缓冲区，将字符串填充其中，然后返回缓冲区或空值（例如realpath(3)）。通常情况下，缓冲区应为`uint8[]`，返回值应为`unowned string?`。

再次，彻底检查返回字符串的所有权。通常情况下，调用者不会释放字符串，尤其是在标记为常量的情况下。

### 函数指针

C 语言中的函数指针在 Vala 中绑定为委托。委托是一种声明函数指针应具有的函数签名的类型。函数指针还可以有一个相关的数据参数，称为目标。

对于没有目标的委托，可以简单地将其视为简单类型。

对于目标委托，必须包含目标。默认情况下，Vala 假定目标位置在函数指针本身之后，但可以通过`delegate_target_pos` 进行调整。接收目标的位置是在委托的定义中定义的，而不是在调用函数中定义的。

具有目标的委托不能被简单复制，因为目标也必须被复制。因此，目标委托的处理方式很像单个拥有的类，它们可以被重新分配，但不能被多重引用。

如果方法要保留对委托的引用，那么它需要一个辅助函数来在委托完成后销毁委托。该位置位于目标之后，但可以通过`delegate_target_destroy_notify_pos` 设置。

如果返回的是委托（这种情况比较少见），则目标和销毁通知符被假定为 out 参数。

```c
typedef void (*foo_func)(int x, void *context);

void call_foo(foo_func f, void *context);
void call_foo_later(foo_func f, void *context, void(*free_context)(void*));
foo_func get_foo(void **context);
foo_func make_foo(void **context, void(**free_context)(void*));
```

```vala
[CCode (cname = "foo_func", has_target = true)]
public delegate void FooFunc (int x);

public void call_foo (FooFunc f);
public void call_foo_later (owned FooFunc f);
public unowned FooFunc get_foo ();
public FooFunc make_foo ();
```

### 变量类型参数（泛型）

Vala 的泛型可以应用于使用 void 指针作为泛型值参数的 C 函数。内存管理和泛型往往相处得不好，因此尽可能避免这种情况可能会有好处。特别是，拥有泛型实例的泛型结构体可能会表现奇怪。此外，将拥有的结构体放入泛型集合中也会导致崩溃。

在开始之前，请确定类型变量的作用域：它适用于方法还是类？泛型与委托配对。按如下方式绑定委托：

```c
typedef int (*foo_func)(void *a, void *b, void* context);
```

```vala
[CCode (cname = "foo_func", simple_generics = true)]
public delegate int FooFunc<T> (T a, T b);
```

#### 泛型方法

通常，泛型变量的上下文是一个方法。只需将`simple_generics`应用于CCode属性即可：

```c
void sort(void **array,int array_length,foo_func compare,void*context);
```

```vala
[CCode(simple_generics=true)]
public void sort<T>(T[] array, FooFunc<T> compare);
```

有时，这不是一个 C 函数，而是一个使用类型名称（如`va_arg`）的类函数宏，在这种情况下，将`generic_type_pos`设为参数的位置：

```c
#define sort(array, type, compare, context) ...

```

```vala
[CCode(generic_type_pos=1.1)]
public void sort<T>(T[] array,FooFunc<T> compare);
```

#### 泛型类和结构体

如果数据结构类似于容器，那么就有可能使用泛型来绑定结构。然而，Vala 对泛型结构的假设相当僵化，因此这可能是不可能的。

-   在类上创建一个类型变量。
-   用 `simple_generics` 装饰所有使用类型变量的方法。
-   如果提供了`simple_generics`，类的构造函数应将析构函数作为参数。如果构造函数不带参数，则使用`simple_generics`将所有构造函数转换为静态方法。
    
-   验证所有的所有权。当 Vala 输出自有变量的 `simple_generics`代码时，总是会传递析构函数。在编写 C 程序时，经常会在构造函数中传递一次析构函数。在这种情况下，应将析构函数设置为空，并坚持所有值都不属于自己。
    

#### 用户指针情形

通常情况下，C 语言程序库会为一些与对象相关的用户数据设置一个指针，而这些数据完全由用户掌握。这很容易绑定。

```c
typedef struct foo Foo;
void *foo_get_userptr(Foo*);
void foo_set_userptr(Foo*,void*);
```

```vala
public class Foo<T> {
    public unowned T? user_data {
        [CCode (cname = "foo_get_userptr", simple_generics = true)] get;
        [CCode (cname = "foo_set_userptr", simple_generics = true)] set;
    }
}
```

唯一需要注意的是，这种绑定方式很有感染力：在其他上下文中使用`Foo`的所有方法，包括该对象的数组和包含该类型的其他类，都必须应用`simple_generics`属性。为了避免这种情况，另一种绑定方法是：

```vala
public class Foo {
   [CCode (simple_generics = true)]
   public void set_user_ptr<T> (T value);
   [CCode (simple_generics = true)]
   public T get_user_ptr<T> ();
}
```

不过，这种方案的类型安全性较低。

### 指针

如果你已经做了这么多，但这个东西似乎仍然需要是一个指针，那么它就是一个指针，但这是一个耻辱的徽章。

## 绑定 C 结构体的字段

紧凑型类、结构体和简单类型结构体可能有字段。通常情况下，类是不透明的；也就是说，类的内容没有任何信息。如果是这样，请跳过本节。绑定字段时，首先要检查是否存在同名的 getter/setter 函数（请参阅 "[属性](https://wiki.gnome.org/Projects/Vala/ManualBindings#Properties)"）。通常情况下，结构的细节都在头文件中，但并不打算公开；请避免绑定不应被访问的变量。请查阅文档。

### 结构体

任何简单类型（int、double、enum或同一绑定中的简单类型）都可以通过在前面加上public 来绑定。这也适用于任何非直接指向的父代结构体（即它们是`foo f;`，而不是`foo *f;`）。

### 结构体指针

作为指针引用的任何字段都稍显复杂。

如果类型是父结构或字段可能为空，则在类型后添加问号。

```c
foo_t *myfoo;
```

```vala
public foo? myfoo;
```

接下来的问题是：该引用是否被拥有？如果值被覆盖，是否应该调用析构函数？如果答案是否定的，那么就使用unowned 作为前缀。对于有父指针的树形结构来说，情况往往如此。

```c
foo_t *parent;
```

```vala
public unowned Foo parent;
```

如果缺少 unowned，则在字段被覆盖时会发生重复释放事件。如果在不需要时将其包含在内，则会出现内存泄漏。

### 数组

在 C 语言中，数组有两种类型：分配内存的指针或包含在结构的指针。Vala 也遵循类似的约定：

```c
int foo[20];
int *bar;
```

```vala
public int foo[20];
public int[] bar;
```

请注意 Vala 版本中方括号的位置。对于固定长度的数组，Vala 希望方括号（以及长度）跟在变量名后面，而对于动态大小的数组，Vala 希望方括号跟在类型后面（不包含长度）。

同样，如果数组可能为空，则在类型后加上问号。

Vala 数组有与之相关的长度。通常，C 程序员也会这样做：

```c
int *foo;
size_t foo_count;
```

其绑定方式为：

```vala
[CCode(array_length_cname="foo_count",array_length_type="size_t")]
public int[] foo;
```

通常情况下，数组将以空值结束，因此不会包含大小：

```vala
[CCode(array_null_terminated=true)]
public Foo[] foos;
```

有时，长度不包括在内，而是在其他地方定义，如常数：

```vala
[CCode(array_length_cexpr="FOO_COUNT")]
public Foo[] foos;
```

由于 Vala 只允许使用数值作为数组长度，因此如果数组长度会随库的新版本而变化，使用`array_length_cexpr`可能会比较方便。

Vala 并不能真正实现 C 风格的堆叠数组（又称锯齿状多维数组），因此，如果没有额外的 C 代码，将它们绑定为数组几乎是不可能的。由于 Vala 的指针语义相同，因此可以将它们视为指针。

### 函数指针

函数指针字段的复杂程度取决于所有权和目标。如果委托是无目标的，那么它就可以被视为一个简单类型，不需要考虑所有权问题。

如果委托有一个目标，那么 C 结构必须有一个目标的持有者：

```c
typedef void(*foo_func)(int a, void *userdata);

typedef struct {
    foo_func callback;
    void *callback_context;
} foo;
```

```vala
[CCode (cname = "foo_func")]
public delegate void FooFunc(int a);

public struct Foo {
    [CCode (delegate_target_cname = "callback_context")]
    public unowned FooFunc callback;
}
```

按照惯例检查无效性。

所有权稍微复杂一些，因为必须有一个字段来保存释放上下文数据的函数。用 GLib 术语来说，这就是销毁通知。

```c
typedef void(*foo_func)(int a, void *userdata);

typedef struct {
    foo_func callback;
    void *callback_context;
    void(*callback_free)(void*);
} foo;
```

```vala
[CCode (cname = "foo_func")]
public delegate void FooFunc(int a);

public struct Foo {
    [CCode (delegate_target_cname = "callback_context", delegate_target_destroy_notify_cname = "callback_free")]
    public FooFunc callback;
}
```

如果函数指针将被精确调用一次，并且调用它将导致上下文销毁，则使用`scope = "async"`。

```c
typedef void(*start_job)(int priority, void *context);

void threadpool_queue_job(Pool *p, start_job j, void *context);
```

```
[CCode (scope = "async", cname = "start_job")]
public delegate void StartJob (int priority);

public class ThreadPool {
        public void queue_job (StartJob j);
}
```

### 共用体

Vala 不理解共用体，但共用体中的名称可以作为 cname 的一部分。

```c
typedef struct {
    bool which_one;
    union {
        double d;
        int i;
    } data;
} foo_t;
```

```vala
public struct Foo {
    public bool which_one;
    [CCode (cname = "data.d")]
    public double data_d;
    [CCode (cname = "data.i")]
    public int data_i;
}
```

## 额外提示

你可以多次绑定一个方法。特别是，使用父对象的构造函数通常既可以作为子对象的构造函数，也可以作为父对象实例的方法。

有时，类的`cname ="void"`可以绕过糟糕的类型定义，但绝对不能用于委托，因为将 void 指针转换为函数指针不符合 C 语言的规定。

如果能在类定义中添加有用的方法，可以让类更像 Vala。

## 尴尬局面

有一些尴尬的情况经常出现，需要得到解答。

### 数组长度

有时，带有返回数组的函数会包含长度，但与 Vala 期望的方式不同。最常见的两种情况是：

```c
int get_array(foo**out_array_p);

struct {
foo *data;
int size;
} array_with_length;
void get_data(array_with_length *output);
```

可以绑定为

```vala
[CCode (cname = "get_array")]
private int _get_array ([CCode (array_length = false)] out foo[] a);
[CCode (cname = "vala_get_array")]
public foo[] get_array () {
    foo[] temp;
    var len = _get_array (out temp);
    temp.length = len;
    return (owned) temp;
}

[CCode (cname = "array_with_length", destroy_function = "")]
private struct array_with_length {
    [CCode (array_length_name = "size")]
    foo[] data;
}
[CCode (cname = "get_data")]
private void _get_data (out array_with_length a);
[CCode (cname = "vala_get_data")]
public foo[] get_data () {
    array_with_length temp;
    _get_data (out temp);
    return (owned) a.data;
}
```

### 独立类型所有权

函数可以有条件地获取对象的所有权。这取决于参数或返回值。在有参数的情况下，函数可以这样绑定：

```c
void somefunc(foo *data,bool free_when_done);
```

```vala
[CCode (cname = "somefunc")]
private _somefunc(Foo data, bool free_when_done);
[CCode (cname = "")]
private _sink_foo (owned Foo foo);
[CCode (cname = "vala_somefunc")]
public somefunc (Foo data) {
    _somefunc(data, false);
}
[CCode (cname = "vala_somefunc_owned")]
public somefunc_owned (owned Foo data) {
    _somefunc (data, true);
    _sink_foo ((owned) foo);
}
```

当返回代码是依赖类型的来源时，这种情况就比较尴尬。一种选择如下：

```c
/* foo is freed if return value is 3. */
int awkward(foo*);
```

```vala
[CCode (cname = "")]
private void _sink_foo (owned Foo f);
[CCode (cname = "awkward")]
private int _awkward (Foo f);
[CCode (cname = "vala_awkward")]
public int awkward (ref Foo f) {
    var ret = _awkward (f);
    if (ret == 3)
        _sink_foo ((owned)f);
    return ret;
}
```

### 成员长度

在处理原始内存访问时，有一种常见的模式：

```c
void foo(void *data, size_t size, size_t nmemb);
```

在这种情况下，通常最好将数据类型固定为uint8，并使用合适的大小作为默认参数：

```vala
public void foo([CCode (array_length_pos = 2.1)] uint8[] data, size_t size = 1);
```

### 无主对象的有主数组

Vala 没有方便的方法来表达由非自有对象组成的自有数组。参见[bug 571486](https://bugzilla.gnome.org/show_bug.cgi?id=571486)。

### 共享上下文委托

当传递多个委托时，它们有时会共享一个上下文指针：

```c
void foo(void *context, void(*x)(int a, void *context), void(*y)(double a, void *context));
```

在这里，x和y共享上下文，但 Vala 没有办法表达这一点。不过，有一个解决方法：

```vala
[CCode (simple_generics = true, has_target = false)]
public void X<T> (int a, T context);
[CCode (simple_generics = true, has_target = false)]
public void Y<T> (double a, T context);
[CCode (simple_generics = true);]
public void foo<T> (T context, X<T> x, Y<T> y);
```

这样就不容易传递 lambda，但传递类或结构体却很实用。
