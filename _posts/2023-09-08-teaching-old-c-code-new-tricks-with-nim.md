---
title: 通过Nim赋能旧的C代码新功能
author: kernel
date: 2023-09-08 20:34:00 +0800
categories: [Nim语言, 博客翻译]
tags: [Nim语言, 编译技巧, 语言交互]
---

最近，我在用Nim封装一个C库时遇到了一个有趣的问题。这个库名叫[MAPM](https://github.com/LuaDist/mapm)，是一个比较古老但相当完整的库，用于处理任意精度的数学运算。不幸的是，这个库没有什么错误处理功能。如果出错，它几乎总是写入`stderr`并返回数字 0。几乎所有出错的情况都是因为函数的错误输入，例如尝试除以0或尝试得到不可能角度的三角函数结果。然而，当`malloc/realloc`无法分配更多数据时，它会写入`stderr`，然后调用`exit(100)`。这听起来很糟糕，但正如[作者](https://github.com/LuaDist/mapm/blob/master/DOCS/commentary.txt)所指出的，替代方案也不是很好，而且有办法解决这个问题。我真希望作者能像许多C标准库函数一样使用错误标志，这样处理这些错误就容易多了。

那么我们该怎么办呢？我可以在封装器中为所有输入添加范围检查，这虽然可行，但对性能不是很好。当然，我也可以像Nim编译器本身那样，在用户使用`-d:danger`进行编译时禁用范围检查。但这仍然不是一个很好的解决方案。此外，MAPM 本身也会进行所有这些检查，因此我们需要对所有内容进行两次检查！起初，我在想是否有可能从程序自己的`stderr` 中读取数据，或者在调用 MAPM 函数之前用一个数据流代替`stderr`，然后再把它换回来。但这样做似乎麻烦很多，收益却很小。

## 解决方案：老式C语言技巧

幸运的是，程序库通过一个名为`M_apm_log_error_msg` 的内部函数来执行所有这些错误处理。这个函数需要两个参数，一个是决定是否是致命错误并调用`exit(100)`，另一个是要显示的信息。事实证明，与`gcc` 配套的 GNU 链接器`ld` 有一个名为`--wrap`的选项，并在文档中作了如下说明：

```
--wrap symbol
        为symbol使用封装函数。任何对symbol的未定义引用都将解析为
        __wrap_symbol。对__real_symbol的任何未定义引用都将解析为symbol。这个
        可用于为系统函数提供封装。包装函数应
        称为 __wrap_symbol.如果要调用系统函数，则应调用
        __real_symbol。下面是一个微不足道的例子：
 
        void *
        __wrap_malloc (int c)
        {
          printf ("malloc called with %ld\n", c);
          return __real_malloc (c);
        }
 
        如果使用 --wrap malloc将其他代码与此文件链接，那么所有对 malloc 的调用都将调用
        将调用 __wrap_malloc函数。在 __wrap_malloc中对 __real_malloc 的调用将
        调用真正的malloc函数。 

```

因此，只需通过`--wrap M_apm_log_error_msg`，库中对该函数的所有调用都将转换为对`__wrap_Mapm_log_error_msg`的调用，其签名与原始签名相同。这意味着如果我们在以C语言可调用的方式提供该函数时将其传递给链接器，那么库将直接回调给我们，而不是调用MAPM的实现。如果我们想在封装器中调用原始函数，我们只需调用`__real_M_apm_log_error_msg` 即可。但在我们的例子中，我们只想替换整个函数。

## 在Nim中运行

有了关于`--wrap`的新知识，让我们来研究一下`M_apm_log_error_msg`函数的实际作用，看看能否将它转换成 Nim 中有用的东西：

```c
void M_apm_log_error_msg(int fatal, char *message) {
    if (fatal) {
        fprintf(stderr, "MAPM Error: %s\n", message);
        exit(100);
    } else {
        fprintf(stderr, "MAPM Warning: %s\n", message);
    }
}
```

我们可以看到，它有两种模式，一种是简单地向`stderr`写一条信息，另一种是终止程序。错误的情况很容易推理，只需将其转换为`Defect`并抛出即可（尽管我们会发现实现起来并不容易）。而简单地将信息写入`stderr`则比较困难。正如介绍中提到的，以这种方式出错的函数会写入这些信息，但仍返回数字 0。首先，我们不能保证只调用该函数一次，`m_apm_set_string`就是一个很好的例子，它可以将数字的文本表示解析为MAPM数字：

```c
if (((int)ch & 0xFF) >= 100) {
    M_apm_log_error_msg(M_APM_RETURN,
    "\'m_apm_set_string\', Non-digit char found in parse");
 
    M_apm_log_error_msg(M_APM_RETURN, "Text =");
    M_apm_log_error_msg(M_APM_RETURN, s_in);
 
    M_set_to_zero(ctmp);
    return;
}
```

我们可以看到，`M_apm_log_error_msg`被多次调用，以显示不止一行的错误信息。

第二个问题是，如果在任何一条日志信息后抛出异常，都会扰乱控制流。假设我们找到了一种方法，只在第三条日志信息后抛出异常，那么我们就不会调用`M_set_to_zero`。当然，在这种情况下，由于异常，我们无论如何都不会使用我们的结果变量，但MAPM可以在返回之前进行其他清理工作，这可能更为重要。

因此，总的来说，我们需要收集信息，只有在调用返回后才能将其转化为异常。我最终的做法是简单地创建一个全局变量来保存消息，然后创建一个模板来检查该变量是否为空并抛出异常。大致如下：

```nim
type MapmError = object of CatchableError
 
var messages = ""
 
{.passL:"-Wl,--wrap=M_apm_log_error_msg".}
proc mapmErrorHandler(fatal: cint, message: cstring) {.exportc: "__wrap_M_apm_log_error_msg.} =
  if fatal == 0:
      messages.add message & "\n"
 
template errChk(): untyped =
  if messages.len != 0:
        let ex = newException(MapmError, messages.strip)
        messages = "" # 这样，如果我们捕捉到异常并继续使用MAPM，缓冲区中就不会已经有东西了
        raise ex
```

正如我们在这里看到的，我们需要传递`-Wl,--wrap=`，而不仅仅是`--wrap` ，这仅仅是因为我们使用 GCC 进行编译，而`-Wl`标志告诉 GCC 这应该传递给`ld`。除此之外，这一切都很简单：将消息添加到消息缓冲区，在每次调用MAPM函数后，我们使用`errChk`检查缓冲区中是否有任何内容，如果有，则用收集到的消息引发异常。由于异常是由Nim代码引发的，因此运行良好，甚至Valgrind也对我们在C函数中使用托管`messages`变量表示满意。

## 中断C的流程

你可能已经注意到，在上面的示例中，我很关键地遗漏了`fatal == 1`的路径。正如我们看到的`fatal == 0`，我们需要确保原始控制流仍然不变。毕竟，`fatal == 1`错误意味着`malloc`或`realloc`失败，内存不足。如果在此之后继续执行程序，而程序员原本预计程序会退出，这不是一个好主意，只会导致错误。因此，我们需要立即退出函数，并确保在退出函数时不再分配内存。你可能会说："难道我们不能在`mapmErrorHandler` 中抛出异常吗？"这似乎是明智之举。但问题是，由于C语言没有异常，Nim需要自己处理异常。这就意味着，在每次调用Nim函数（该函数可以抛出任何东西）后，都要注入一小段逻辑，以检查是否出现异常，如果出现异常，则对其进行实际处理。但在C语言中并不存在这种逻辑，而且由于我们是在一个函数中，所以没有办法强制C语言返回到Nim的链上（除非用跳转技术黑掉一些东西，但我们还是不要说这个了）。我最终使用的是emit、内存复制和异常抛出组合。看起来有点像这样：

```nim
type
  MapmError = object of CatchableError
  MapmDefect = object of Defect
 
var
  messages = ""
  defect = newException(MapmDefect, newString(100)) # 为异常预分配空间
 
{.passL:"-Wl,--wrap=M_apm_log_error_msg".}
proc mapmErrorHandler(fatal: cint, message: cstring) {.exportc: "__wrap_M_apm_log_error_msg.} =
  if fatal == 0:
      messages.add message & "\n"
    else:
        copyMem(defect.msg[0].addr, message, message.len) # 复制消息到预分配buffer
        raise defect # 抛出异常
        {.emit: "nimTestErrorFlag();".} # 实际检查是否出现异常并采取相应措施
```

由于`raise`语句除了将我们的异常注册为抛出之外，实际上并没有做更多的事情，因此我们需要手动将`nimTestErrorFlag`插入到我们的代码中，以确保异常被立即检查和处理。当然，对于`Defect`来说，这并不十分重要，因为Nim默认情况下不会让你捕获它们。现在，如果我们运行这个程序并触发某个致命错误（我作弊了，只是反转了`fatal == 0`检查），我们可以看到程序正确地抛出了 Defect，并写出了发生错误的堆栈跟踪。

## 结束语和注意事项

总而言之，这个系统运行得很好，它允许一个在Nim出现之前编写的C库引发异常，并将`stderr`消息转换为适当的异常消息。不过，我在这里忽略了一些需要提及的细节。注意事项一是C编译器可以自由优化调用，因此我们显然无法保证`--wrap`会获取我们函数的所有使用情况。我用MAPM测试过这一点，似乎没有问题，但如果启用了更激进的编译器标志，这一点就需要注意了。其次，我提到在我们的defect情形中，我们得到了一个很好的堆栈跟踪。我们注意预分配异常和字符串缓冲区，但不能保证Nim不会为堆栈跟踪分配数据。当然，如果请求的`malloc`非常大，Nim也有可能为此分配一个较小的缓冲区。但在这种情况下，你的里程数可能会有所不同。