---
title: 将Nim二进制大小从160KB降至150Bytes
author: kernel
date: 2015-05-04 20:34:00 +0800
categories: [Nim语言, 博客翻译]
tags: [Nim语言, 编译优化]
---

[最近](https://forum.nim-lang.org/t/1171)，[Nim](https://nim-lang.org/)编程语言二进制文件的大小似乎[成了一个](https://www.schipplock.software/p/static-linking-with-nim.html) [热门话题](https://forum.nim-lang.org/t/963)。Nim 的口号是“ **表现力强、高效、优雅** ” ，因此让我们在这篇文章中探讨**高效**的部分，探索几种在Linux上缩小简单的Nim`Hello World`二进制文件大小的方法。在此过程中，我们将：

-   将常规程序编译成6KB二进制文件
-   不考虑C标准库，构建952字节的二进制文件
-   使用自定义链接器脚本和ELF头文件构建150字节的二进制文件[（比Rust小1个字节）](https://mainisusuallyafunction.blogspot.de/2015/01/151-byte-static-linux-binary-in-rust.html)

本文章的完整源代码可在[其资源库](https://github.com/def-/nim-binary-size)中找到。所有测试均在Linux x86-64上进行，使用GCC 5.1和Clang 3.6.0。

## [使用C标准库](https://hookrace.net/blog/nim-binary-size/#using-the-c-standard-library)

```nim
echo "Hello!"
```

默认情况下，Nim在大多数平台上使用GCC作为后端C编译器，并根据glibc进行动态链接。我们可以尝试优化速度和大小，并在编译后去除不必要的符号：

| 命令（GCC后端） | 二进制大小 |
| --- | --- |
| `nim c hello` | 160 KB |
| `nim -d:release c hello` | 61 KB |
| `nim -d:release --opt:size c hello` | 25 KB |
| `nim -d:release --opt:size c hello && strip -s hello` | 19 KB |

这很好，任何Nim程序都可以这样做，以减少二进制文件的大小。

现在，让我们试着摆脱glibc，至少是暂时摆脱（我们稍后再讨论更持久的解决方案）。我们现在静态链接的是 musl libc，而不是 glibc：

```
$ nim --gcc.exe:/usr/local/musl/bin/musl-gcc \
  --gcc.linkerexe:/usr/local/musl/bin/musl-gcc \
  -d:release --opt:size --passL:-static c hello
$ strip -s hello
30 KB
```

更新：nim的参数顺序很重要，`--passL:-static`必须在设置gcc exe之后传递，这样它才不会被覆盖。

因此，这是一个静态链接的二进制文件，只有30KB，部署时无需依赖任何glibc版本（或任何其他库）！

如果设置`--cc:clang`来使用 Clang 而不是 GCC 呢？

| 命令（Clang后端） | 二进制大小 |
| --- | --- |
| `nim --cc:clang c hello` | 168 KB |
| `nim --cc:clang -d:release c hello` | 33 KB |
| `nim --cc:clang -d:release --opt:size c hello` | 29 KB |
| `nim --cc:clang -d:release --opt:size c hello && strip -s hello` | 23 KB |

速度优化后的二进制文件要小得多，但大小优化后的二进制文件则不然。当然，Clang和GCC的具体行为取决于它们的版本，因此预计在您的系统上看到的数字（至少）会略有不同。

目前看来，GCC后端是一个更好的选择，所以让我们尝试用它来进一步剥离二进制文件：

第一步，我们禁用垃圾回收器，反正这个程序也不需要它：

```
$ nim --gc:none -d:release --opt:size c hello
$ strip -s hello
11 KB
```

接下来，我们使用`--os:standalone`（这意味着`--gc:none`）移除所有漂亮的动态内存、错误处理和其他依赖于操作系统的好东西。我们必须提供一个包含这两个程序的`panicoverride.nim`，反正我们也不在乎这两个程序。有了 6KB的二进制文件，谁还需要错误处理呢：

```nim
proc rawoutput(s: string) = discard
proc panic(s: string) = discard
```

```
$ nim --os:standalone -d:release c hello
$ strip -s hello
6.1 KB
```

## [忽略C标准库](https://hookrace.net/blog/nim-binary-size/#disregarding-the-c-standard-library)

现在，我们必须开始发散思维了：如果我们想要一个什么都不做的程序，甚至不打印`Hello！`，我们可以直接使用一个空文件。现在我们不再依赖C标准库了，可以尝试使用`-passL:-nostdlib`（passL只是在链接步骤中将该参数传递给GCC）来完全排除它：

```
$ nim --os:standalone -d:release --passL:-nostdlib c hello
CC: hello
CC: stdlib_system
[Linking]
ld: warning: cannot find entry symbol _start; defaulting to 0000000000400160
$ strip -s hello
1.4 KB
```

哇，真小！让我们运行我们的程序，什么也不做，尽情享受吧：

```
$ ./hello
Segmentation fault (core dumped)
```
哎哟进展不太顺利。再看看链接器的输出，就会发现问题所在：我们不能只指望运行我们的代码，二进制文件会在某个随机的、错误的位置开始执行。相反，我们现在必须接管C标准库的工作，并提供我们自己的`_start`函数：

```nim
import syscall

proc main {.exportc: "_start".} =
  discard syscall(EXIT, 0)
```

我们还必须明确地退出程序，为此我们使用了我的[syscall库](https://github.com/def-/nim-syscall)，它在Nim中为Linux内核提供了原始系统调用。让我们来封装我们需要的系统调用，并用它们编写合适的Nim代码：

```nim
import syscall

const STDOUT = 1

proc write(fd: cint, buf: cstring, len: csize): clong
          {.inline, discardable.} =
  syscall(WRITE, fd, buf, len)

proc exit(n: clong): clong {.inline, discardable.} =
  syscall(EXIT, n)

proc main {.exportc: "_start".} =
  write STDOUT, "Hello!\n", 7
  exit 0
```

现在我们可以像这样成功编译了：

```
$ nim --os:standalone -d:release --passL:-nostdlib --noMain c hello
$ strip -s hello
1.5 KB
$ ./hello
Hello!
```

本节的最后一个技巧是告诉GCC优化未使用的函数。这些函数是Nim模块的初始化函数，比如我们的`hello`模块或标准库中的`系统`模块，但无论如何，它们在这里都是空的。也许Nim编译器可以自行将它们删除，但通常情况下，你并不在乎节省几个字节，而是有更重要的事情要做。今天我们要做的是，首先在编译步骤中告诉GCC将函数和数据项放入不同的部分（`-ffunction-sections`& `-fdata-sections`）。在链接步骤中，我们让Nim告诉GCC将`--gc-sections`传递给链接器`ld`，后者会删除没有被引用的部分：

```
$ nim --os:standalone -d:release --passL:-nostdlib --noMain \
  --passC:-ffunction-sections --passC:-fdata-sections \
  --passL:-Wl,--gc-sections c hello
$ strip -s hello
952 B
```

太好了！我们的二进制大小从最初的 160 KB 降到了 952 字节。还能更小吗？当然可以，但不能使用默认工具。

## [自定义链接以实现150字节](https://hookrace.net/blog/nim-binary-size/#custom-linking-to-achieve-150-bytes)

这与[Rust](https://mainisusuallyafunction.blogspot.de/2015/01/151-byte-static-linux-binary-in-rust.html)博文中 151 字节的 Linux 静态二进制文件使用的方法完全相同，只不过使用GCC的Nim能多压缩1个字节。同时，Clang需要比Rust版本多1个字节。

我们继续执行刚才缩减到952字节的程序。但我们并不是让Nim来完成所有工作（现在已经有很多选项了），而是简单地从Nim创建一个对象文件（`--app:staticlib`），然后从这里开始手动操作。一个[自定义链接器脚本](https://github.com/def-/nim-binary-size/blob/master/script.ld)和一个[自定义ELF头文件](https://github.com/def-/nim-binary-size/blob/master/elf.s)完成了主要工作。但实际执行的代码仍完全由我们的 Nim 代码提供：

```
$ nim --app:staticlib --os:standalone -d:release --noMain \
  --passC:-ffunction-sections --passC:-fdata-sections \
  --passL:-Wl,--gc-sections c hello
$ ld --gc-sections -e _start -T script.ld -o payload hello.o
$ objcopy -j combined -O binary payload payload.bin
$ ENTRY=$(nm -f posix payload | grep '^_start ' | awk '{print $3}')
$ nasm -f bin -o hello -D entry=0x$ENTRY elf.s
$ chmod +x hello
$ wc -c < hello
158
$ ./hello
Hello!
```

158字节！还有一个最后的小窍门，可以减少8个字节。我们将字符串放在[ELF头](https://github.com/def-/nim-binary-size/blob/master/elf.s)的填充中，然后手动访问内存：

```nim
proc main {.exportc: "_start".} =
  write STDOUT, cast[cstring](0x00400008), 7
  exit 0
```

```
$ wc -c < hello
150
$ ./hello
Hello!
```

150 字节！这就是我们要得到的最终大小。如果你还是觉得不够，想手动编写二进制文件，你可以使用更多的方法来减小到45字节，如《[Whirlwind Tutorial on Creating Really Teensy ELF Executables for Linux](https://www.muppetlabs.com/%7Ebreadbox/software/tiny/teensy.html)》（关于为Linux创建真正的Teensy ELF可执行文件的出色教程）中所述。

## [总结](https://hookrace.net/blog/nim-binary-size/#summary)

Nim非常适合编写小型二进制文件。现在你还知道了如何在不使用C标准库的情况下编写Nim。从头开始用Nim编写自己的运行时可能会很有趣。你可以查看[软件仓库](https://github.com/def-/nim-binary-size/)，获得自己的成果：

```
$ ./run.sh
== Using the C Standard Library ==
hello_unoptimized    163827
hello_release         62131
hello_optsize         25248
hello_optsize_strip   18552
hello_gcnone          10344
hello_standalone       6208

== Disregarding the C Standard Library ==
hello2                 1776
hello3                  952

== Custom Linking ==
hello3_custom           158
hello4_custom           150

$ objdump -rd nimcache/hello4.o
...
0000000000000000 <_start>:
 0: b8 01 00 00 00          mov    $0x1,%eax
 5: ba 07 00 00 00          mov    $0x7,%edx
 a: be 08 00 40 00          mov    $0x400008,%esi
 f: 48 89 c7                mov    %rax,%rdi
12: 0f 05                   syscall 
14: 31 ff                   xor    %edi,%edi
16: b8 3c 00 00 00          mov    $0x3c,%eax
1b: 0f 05                   syscall 
1d: c3                      retq 
...
```

可以在[Hacker News](https://news.ycombinator.com/item?id=9485526)和[Reddit](https://www.reddit.com/r/programming/comments/34tb7r/nim_binary_size_from_160_kb_to_150_bytes/) 上讨论。

### [扩展](https://hookrace.net/blog/nim-binary-size/#addendum)

我现在也对32位x86进行了这样的处理，结果是使用GCC会生成119字节的二进制文件，使用Clang会生成118字节的二进制文件，更多信息请参见[代码仓库](https://github.com/def-/nim-binary-size#x86)。

通过对Nim编译器的[简单修补](https://github.com/Araq/Nim/pull/2657)，`{.noReturn.}`pragma现在实际上删除了`EXIT`系统调用后无用的最后`retq`调用。因此，现在x86-64的最终二进制文件大小为149字节，x86为116字节。