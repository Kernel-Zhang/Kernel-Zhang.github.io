---
title: 编译Vala程序和库
author: kernel
date: 2023-11-22 14:34:00 +0800
categories: [Vala语言, 教程]
tags: [Vala语言, Vala官方教程, Vala入门]
---

Meson 支持编译用[Vala](https://vala-project.org/)和[Genie](https://wiki.gnome.org/Projects/Genie)编写的应用程序和库。一个`meson.build`骨架文件：

```meson
project('vala app', 'vala', 'c')

dependencies = [
    dependency('glib-2.0'),
    dependency('gobject-2.0'),
]

sources = files('app.vala')

executable('app_name', sources, dependencies: dependencies)
```

您必须始终指定`glib-2.0`和`gobject-2.0`库为依赖库，因为当前所有 Vala 应用程序都使用它们。[GLib](https://developer.gnome.org/glib/stable/)用于基本数据类型，[GObject](https://developer.gnome.org/gobject/stable/)用于运行时类型系统。

## 使用库

Meson 使用 `dependency()` 函数来查找相关的 VAPI、C 头文件和链接器标志。Vala 需要一个 VAPI 文件和一个或多个 C 头文件才能使用一个库。VAPI 文件有助于将 Vala 代码映射到库的 C 编程接口。正是[`pkg-config`](https://www.freedesktop.org/wiki/Software/pkg-config/)工具让这些安装文件在幕后无缝运行。如果库的`pkg-config`文件不存在，那么编译器对象的 `find_library()` 方法。稍后将给出示例。

注意 Vala 使用遵循 C 应用程序二进制接口 (C ABI) 的库。 然而，该库可以用 C、Vala、Rust、Go、C++ 或任何其他语言编写，只要这些语言能生成与 C ABI 兼容的二进制文件，并提供 C 头文件即可。

### 最简单的情况

第一个例子是对`meson.build`文件的简单添加，因为

-   该库有一个`pkg-config`文件，即`gtk+-3.0.pc`
-   VAPI 与 Vala 一起发布，因此与 Vala 编译器一起安装
-   VAPI 安装在 Vala 的标准搜索路径中
-   VAPI（`gtk+-3.0.vapi`）与`pkg-config`文件同名

一切都在后台无缝运行，只需增加一行即可：

```meson
project('vala app', 'vala', 'c')

dependencies = [
    dependency('glib-2.0'),
    dependency('gobject-2.0'),
    dependency('gtk+-3.0'),
]

sources = files('app.vala')

executable('app_name', sources, dependencies: dependencies)
```

GTK+ 是 GNOME、elementary操作系统和其他桌面环境使用的图形工具包。与该库的绑定，即 VAPI 文件，随 Vala 一起发布。

其他库的 VAPI 可能与库本身一起发布。此类库的 VAPI 文件将与其他开发文件一起安装。VAPI 安装在 Vala 的标准搜索路径中，因此使用`dependency()`函数也能无缝运行。

### 指定 GLib 版本

Meson 的 `dependency()`函数允许对库进行版本检查。这通常用于检查已安装的最低版本。在设置 GLib 的最低版本时，Meson 也会使用`--target-glib`选项将其传递给 Vala 编译器。

在使用 GTK+ 的用户接口定义文件和 Vala 的`[GtkTemplate]`、`[GtkChild]` 和`[GtkCallback]`属性时需要这样做。 这需要向 Vala 传递`--target-glib 2.38` 或更新的版本。在 Meson 中，只需使用：

```meson
project('vala app', 'vala', 'c')

dependencies = [
    dependency('glib-2.0', version: '>=2.38'),
    dependency('gobject-2.0'),
    dependency('gtk+-3.0'),
]

sources = files('app.vala')

executable('app_name', sources, dependencies: dependencies)
```

使用`[GtkTemplate]`还需要将 GTK+ 用户接口定义文件作为 GResources 内置于二进制文件中。为完整起见，下一个示例将对此进行说明：

```meson
project('vala app', 'vala', 'c')

dependencies = [
    dependency('glib-2.0', version: '>=2.38'),
    dependency('gobject-2.0'),
    dependency('gtk+-3.0'),
]

sources = files('app.vala')

sources += import( 'gnome' ).compile_resources(
    'project-resources',
    'src/resources/resources.gresource.xml',
    source_dir: 'src/resources',
)

executable('app_name', sources, dependencies: dependencies)
```

### 增加Vala的搜索路径

到目前为止，我们已经介绍了 VAPI 文件随 Vala 或库一起发布的情况。VAPI 也可以包含在项目的源文件中。惯例是将其放在项目的`vapi`目录中。

当一个库没有 VAPI 或您的项目需要链接到项目中使用 C ABI 的另一个组件时，就需要这样做。

Vala 编译器的`--vapidir`选项用于将项目目录添加到 VAPI 搜索路径中。在 Meson 中，则使用`add_project_arguments()`函数来完成：

```meson
project('vala app', 'vala', 'c')

vapi_dir = meson.current_source_dir() / 'vapi'

add_project_arguments(['--vapidir', vapi_dir], language: 'vala')

dependencies = [
    dependency('glib-2.0'),
    dependency('gobject-2.0'),
    dependency('foo'), # 'foo.vapi' will be resolved as './vapi/foo.vapi'
]

sources = files('app.vala')

executable('app_name', sources, dependencies: dependencies)
```

如果 VAPI 用于外部库，则确保 VAPI 名称与 pkg-config 文件名称一致。

[`vala-extra-vapis`资源库](https://gitlab.gnome.org/GNOME/vala-extra-vapis)是一个由社区维护的未分发 VAPI 资源库，开发人员使用该资源库共享新绑定和现有绑定改进的早期工作，因此 VAPI 可能会经常变化。建议将该资源库中的 VAPI 复制到您的项目源文件中。

在新绑定进入`vala-extra-vapis`软件库之前，对于开始编写新绑定很有效。

### 无 pkg-config 文件的库

如果一个库没有相应的 pkg-config 文件，则意味着`dependency()`不适合查找 C 和 Vala 接口文件。在这种情况下，有必要使用编译器对象的`find_library()`方法。

第一个示例使用了 Vala 的 POSIX 绑定。由于 POSIX 包含 Unix 系统上的标准 C 库，因此不需要 pkg-config 文件，只需要 VAPI 文件`posix.vapi`。该文件随 Vala 一起提供，并安装在 Vala 的标准搜索路径中。只需告诉 Meson 只查找 Vala 编译器的库即可：

```meson
project('vala app', 'vala', 'c')

dependencies = [
    dependency('glib-2.0'),
    dependency('gobject-2.0'),
    meson.get_compiler('vala').find_library('posix'),
]

sources = files('app.vala')

executable('app_name', sources, dependencies: dependencies)
```

下一个示例展示了如何与不需要额外 VAPI 的 C 库链接。标准数学函数已经绑定在`glib-2.0.vapi` 中，但 GNU C 库需要单独链接到数学库。在本例中，Meson 只为 C 编译器查找数学库：

```meson
project('vala app', 'vala', 'c')

dependencies = [
    dependency('glib-2.0'),
    dependency('gobject-2.0'),
    meson.get_compiler('c').find_library('m', required: false),
]

sources = files('app.vala')

executable('app_name', sources, dependencies: dependencies)
```

`required: false`项表示在使用其他其他未分离C语言数学库时，将继续构建。请参阅可移植地添加数学库 (-lm)。

最后一个示例展示了如何使用一个没有 pkg-config 文件的库，并且 VAPI 位于项目源文件的`vapi`目录中：

```meson
project('vala app', 'vala', 'c')

vapi_dir = meson.current_source_dir() / 'vapi'

add_project_arguments(['--vapidir', vapi_dir], language: 'vala')

dependencies = [
    dependency('glib-2.0'),
    dependency('gobject-2.0'),
    meson.get_compiler('c').find_library('foo'),
    meson.get_compiler('vala').find_library('foo', dirs: vapi_dir),
]

sources = files('app.vala')

executable('app_name', sources, dependencies: dependencies)
```

C 编译器对象的`find_library()`方法将尝试查找 C 头文件和链接库。

Vala 编译器对象的`find_library()`方法需要添加`dir`关键字，以包含项目 VAPI 目录。`add_project_arguments()` 不会自动添加该关键字。

### 使用 Vala 预处理器

向[Vala 预处理器](https://wiki.gnome.org/Projects/Vala/Manual/Preprocessor)传递参数需要指定语言为`Vala`。例如，以下语句设置了预处理器符号`USE_FUSE`：

```meson
add_project_arguments('-D', 'USE_FUSE', language: 'vala')
```

例如，要将 `FUSE_USE_VERSION` 设置为 26，请使用

```meson
add_project_arguments('-DFUSE_USE_VERSION=26', language: 'c')
```

## 构建库

### 更改 C 头文件和 VAPI 名称

Meson 的 `library()`目标会自动输出 C 头文件和 VAPI。可以通过设置`vala_header`和`vala_vapi`参数分别对它们进行重命名：

```meson
foo_lib = shared_library('foo', 'foo.vala',
                  vala_header: 'foo.h',
                  vala_vapi: 'foo-1.0.vapi',
                  dependencies: [glib_dep, gobject_dep],
                  install: true,
                  install_dir: [true, true, true])
```

在此示例中，`install_dir`数组的第二和第三个元素表示目的地，使用`true`表示使用默认目录（即`include`和`share/vala/vapi`）。

### GObject 自省和语言绑定

绑定允许另一种编程语言使用用 Vala 编写的库。由于 Vala 使用 GObject 类型系统作为其运行时类型系统，因此使用自省功能生成绑定非常容易。Vala 库的 Meson 构建可以生成 GObject 自省元数据。然后，这些元数据将被用于具有[特定语言工具的](https://wiki.gnome.org/Projects/Vala/LibraryWritingBindings)单独项目中，以生成绑定。

元数据的主要形式是 GObject 自省存储库（GIR）XML 文件。GIR 主要用于在编译时生成绑定的语言。在运行时生成绑定的语言大多使用从 GIR 生成的 typelib 文件。

Meson 可以在构建过程中生成 GIR。对于 Vala 库，必须为`library`设置`vala_gir`选项：

```meson
foo_lib = shared_library('foo', 'foo.vala',
                  vala_gir: 'Foo-1.0.gir',
                  dependencies: [glib_dep, gobject_dep],
                  install: true,
                  install_dir: [true, true, true, true])
```

`install_dir`中的`true`会告诉 Meson 使用默认目录（即 GIR 的`share/gir-1.0`）。`install_dir`数组中的第四个元素表示 GIR 文件的安装位置。

要生成 typelib 文件，可使用带有`g-ir-compiler`程序和依赖库的自定义目标：

```meson
g_ir_compiler = find_program('g-ir-compiler')
custom_target('foo typelib', command: [g_ir_compiler, '--output', '@OUTPUT@', '@INPUT@'],
              input: meson.current_build_dir() / 'Foo-1.0.gir',
              output: 'Foo-1.0.typelib',
              depends: foo_lib,
              install: true,
              install_dir: get_option('libdir') / 'girepository-1.0')
```