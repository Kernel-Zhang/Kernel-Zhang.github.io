---
title: libgpiod在Nim中的使用
author: kernel
date: 2023-12-05 14:14:00 +0800
categories: [原创, Nim语言]
tags: [Nim语言, 语言交互, Nim入门]
---

由于Ubuntu 20系统自带的libgpiod库（版本V1.4.1）存在bug：在监听一组gpio状态后，再一次读取多个gpio的状态时会得到错误的结果。虽然在后面的更新中，该bug[已经修复](https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git/commit/?id=b56d6b6a452e47fee8c70514afb99ccd77ada677)，但为了不破坏系统软件包的版本，并且一劳永逸的避免其它的bug，我决定在Nim中使用最新的libgpiod库，并且静态链接到我的程序，摆脱对系统库的依赖。那就一步一步的开始吧！

## 下载最新的libgpiod库


```shell
git clone https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git
```

## 配置和编译

我的目标系统是arm64的linux，首先安装交叉编译工具：

```shell
sudo apt install gcc-aarch64-linux-gnu
```

然后开始配置libgpiod库。为了便于移植，我把库的安装路径配置为工程的out目录下：

```shell
./autogen.sh --enable-tools=yes --prefix=$(pwd)/out --host=aarch64-linux-gnu
```

配置完成后，就是make编译了：

```shell
make
```

最后链接的时候出错了：

```log
/usr/lib/gcc-cross/aarch64-linux-gnu/9/../../../../aarch64-linux-gnu/bin/ld: ./.libs/libtools-common.a(tools-common.o): in function `chip_paths':
/linux/app/libgpiod/tools/tools-common.c:456: undefined reference to `rpl_malloc'
/usr/lib/gcc-cross/aarch64-linux-gnu/9/../../../../aarch64-linux-gnu/bin/ld: ./.libs/libtools-common.a(tools-common.o): in function `resolver_init':
/linux/app/libgpiod/tools/tools-common.c:580: undefined reference to `rpl_malloc'
collect2: error: ld returned 1 exit status
make[2]: *** [Makefile:493: gpiodetect] Error 1
make[2]: Leaving directory '/linux/app/libgpiod/tools'
make[1]: *** [Makefile:465: all-recursive] Error 1
make[1]: Leaving directory '/linux/app/libgpiod'
make: *** [Makefile:395: all] Error 2

```

错误原因是找不到`rpl_malloc`函数，直接使用系统的`malloc`函数就好。

打开`config.h.in`文件，将最后一行的`#undef malloc`注释掉，然后重新`make`即可。

最后进行安装：

```shell
make install
```

然后检查一下最终生成的文件：

```
out/
├── bin
│   ├── gpiodetect
│   ├── gpioget
│   ├── gpioinfo
│   ├── gpiomon
│   ├── gpionotify
│   └── gpioset
├── include
│   └── gpiod.h
└── lib
    ├── libgpiod.a
    ├── libgpiod.la
    ├── libgpiod.so -> libgpiod.so.3.1.0
    ├── libgpiod.so.3 -> libgpiod.so.3.1.0
    ├── libgpiod.so.3.1.0
    └── pkgconfig
        └── libgpiod.pc

4 directories, 13 files

```

其中`libgpiod.a`和`gpiod.h`就是我们开发所需要的静态库和头文件，真是太好了！

## 库的使用

使用nimble新建一个Nim的工程，再在工程中新建一个`libgpiod`的目录，最后将`libgpiod.a`和`gpiod.h`放入其中。

将Futhark库的依赖添加到nimble项目文件中:

```nim
requires "futhark >= 0.12.0"
```

然后通过Futhark将libgpiod库自动化导入到Nim中，为了简便起见，我们只测试了libgpiod库的API版本号获取函数：

```nim
when defined(useFuthark) :
  import futhark, os

  importc:
    outputPath currentSourcePath.parentDir / "libgpiod.nim"
    path "../libgpiod"
    "gpiod.h"
else:
  include "libgpiod.nim"

{.passL: "-Llibgpiod -lgpiod".}

echo gpiodapiversion()

```

首次交叉编译我们需要生成对应的`libgpiod.nim`接口文件：

```shell
nimble build --os:linux --cpu:arm64 -d:useFuthark
```

为了加快编译，以后每次交叉编译程序都不必强制生成接口文件了：

```shell
nimble build --os:linux --cpu:arm64
```

一切大功告成，剩下的就是堆代码了！