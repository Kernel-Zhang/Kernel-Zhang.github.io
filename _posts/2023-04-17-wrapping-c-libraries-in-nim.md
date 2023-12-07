---
title: 在Nim中封装C语言库
author: kernel
date: 2023-04-17 20:34:00 +0800
categories: [Nim语言, 博客翻译]
tags: [Nim语言, 语言交互]
---

正如我们在上一篇文章中发现的，Nim默认生成C代码，然后调用C编译器生成二进制文件。这似乎是个奇怪的选择，尤其是在LLVM时代。但实际上，编译或转译为另一种语言的情况并不少见。最初选择不使用LLVM，只是因为在创建Nim时，LLVM还不成熟。尽管通过C语言编译有一些相当大的好处。例如，你可以在任何可以运行C语言的地方运行Nim，这意味着你可以在几乎任何地方运行它。从最微小的微控制器，到移动应用程序，再到普通的桌面、服务器目标[<sup data-dl-uid="14" data-dl-original="true" data-dl-translated="true">1</sup>](https://peterme.net/wrapping-c-libraries-in-nim.html#fn1)。不过，本文重点讨论的另一个好处是，可以超级轻松地与C库或程序对接。这就是所谓的外来函数接口或FFI，意味着我们可以从一种语言中调用另外一种语言中的函数，在本例中，从C语言中调用Nim，或从Nim中调用C语言。当然，与C语言相比，Nim语言有许多额外的功能和系统，因此在使用C语言编写的库时，我们通常需要进行一些翻译工作，以便让接口对Nim用户更加自然[<sup data-dl-uid="16" data-dl-original="true" data-dl-translated="true">2</sup>](https://peterme.net/wrapping-c-libraries-in-nim.html#fn2)。尤其是涉及到内存管理和async时。在本文中，我将概述这在实践中是如何工作的，并展示在Nim中封装约束应用协议（CoAP）库的一些示例。

## 告诉Nim什么是可用的

在Nim中使用C库非常简单，只需告诉Nim过程的签名，然后告诉它这是C语言中存在的东西，就可以了。举一个非常简单的例子：

```nim
proc hello(x: string) {.importc.}
 
hello("world")
```

从Nim的角度来看，这样编译是没有问题的，并且会生成用Nim字符串对象调用过程`hello`的 C 代码。当然，如果这个过程不存在于C标准库的任何地方，或者不存在于我们通过开关导入的任何库中，那么C编译器就会产生`undefined reference to 'hello'`的错误[<sup data-dl-uid="42" data-dl-original="true" data-dl-translated="true">3</sup>](https://peterme.net/wrapping-c-libraries-in-nim.html#fn3)。你可以进一步告诉Nim这个过程来自哪个头文件，或者使用附加的实用程序来链接到哪个动态链接库。这将导致Nim在使用过程时自动导入或链接到这些程序。在封装C库时，你可能希望在Nim中使用“c”前缀的类型。例如，`cstring`、`cint`、`cuint`、`csize`、`clong` 和`cfloat`，它们都映射到C语言中没有前缀的相应类型。C和Nim之间的类型签名和对象必须完全匹配，但您可以使用Nim更好的类型系统来改进基本签名。例如，一个C过程应该接受一个`int`，其中允许传递的特殊数字是用`#define`定义的，而这个C过程将很乐意接受一个具有`cint`大小成员的枚举。这意味着，你现在只能传递有效的选项，而不能接受任何整数。同样，如果你的C过程接收（或返回）一个指向数组起点的指针，你可以在 Nim 中使用`ptr UncheckedArray[T]`。这样就可以在循环和索引查找中使用该值。一般来说，使用不同的类型也有助于提高整个库的类型安全性。

如果你只需要少量的过程，那么手工编写所有这些定义可能会很有效。但是，如果您与其他人共享代码，而其他人可能想使用不同的程序，那么他们就必须自己封装这些程序。另一个问题是，当你告诉Nim：C语言中存在什么时，Nim 会百分之百地相信你。因此，如果你不小心或故意给Nim提供错误的信息，最终要么会导致C语言编译器错误，要么甚至会导致奇怪的运行时行为和崩溃。例如，如果一个过程需要一个指向特定对象的指针，而你不小心在定义中写错了对象，Nim或C编译器都不会发现这一点。这意味着你最终会出现奇怪的未定义行为，从而导致崩溃和难以调试的问题。当然，如果你所封装的库发生了变化，你就必须仔细检查并煞费苦心地确保你封装的每一部分都是正确的。否则，你将回到起点。

## 工作外包

因此，与其手写所有这些定义，不如让计算机为我们完成这些转换。毕竟，电脑最擅长的就是高精度地完成重复性工作。

自Nim诞生之初，就有一款名为c2nim的工具，旨在将C文件（尤其是头文件）转换为Nim文件。然而，这个工具的功能非常欠缺，它不能处理导入，不能正确使用定义，而且它能解析的内容也有点有限。此外，官方文档还明确指出，翻译后的输出结果需要手工调整。这听起来似乎是个好主意，毕竟有时我们可能会做得更好。但实际上，这意味着如果C库有更新（或者你决定加入另一个定义），你就必须记得重复所有的调整。这与手动编写封装器的问题类似。

这是另一款工具是nimterop，它使用了一种更强大的树状结构解析算法，旨在实现自动生成。不过，它对C定义的理解仍不够透彻，虽然它是一个生成绑定的工具，但仍建议用户将其作为预编译步骤的一部分来运行。此外，它已经两年没有更新了，因此该项目是否仍在积极维护尚不确定。

## 走进Futhark

在尝试了手动方式和两个自动版本来封装相当大的[<sup data-dl-uid="63" data-dl-original="true" data-dl-translated="true">4</sup>](https://peterme.net/wrapping-c-libraries-in-nim.html#fn4) [Unbound](https://github.com/NLnetLabs/unbound)代码库之后，我感到厌倦了。我在c2nim和nimterop上各花了大约一周的时间来尝试让它工作，但最终还是放弃了。然后，我手工封装了我需要的部分。虽然工作很乏味，但至少我最终还是成功了。这个项目主要是在Nim中编写一个动态库，由Unbound加载。Unbound最初编写时并不支持动态库（这是我自己在项目的PR中添加的），因此API并不完全明确和稳定。Unbound也会定期更新，我不想在生产中使用过时的版本。这导致Unbound慢慢偏离了绑定，奇怪的错误开始出现。翻阅所有绑定文件时，微小的对齐错误等问题开始显现，显然手动绑定也不是办法。

但一定有更好的办法，这就是我编写[Futhark](https://github.com/pmunch/futhark)的原因。因为你知道什么能真正理解C代码吗？C编译器！当我在Zig中挖掘C交互工作原理的细节时，我萌生了这个想法。它不能编译成C语言，但仍能导入C头文件并使用C语言中的内容。Zig是一种LLVM语言，而Clang（LLVM 的 C 语言前端）有一个库，可以通过编程获取所有过程、结构、变量和其他信息。于是就有了Futhark背后的想法：使用这个库将C头文件解析为有用的信息，然后让Nim宏为我们生成所需的代码！除了使用上述工具普遍存在的困难之外，Futhark的建立是为了克服我对nimterop和c2nim的一些不满。首先，它应该能正常工作。其次，如果它不能工作，手动提供一个解决方法应该是轻而易举的。第三，它的使用应该尽可能透明，最好是用户根本不会想到翻译正在进行。最后但并非最不重要的一点是，绑定之后的使用应该非常简单，你可以按照C语言教程将语法移植到Nim中。这无疑是一项艰巨的任务，但除了一些小障碍外，我相信我已经成功实现了所有目标。

## 使用Futhark

既然我们已经找到了自己的工具，那么现在就是使用它的时候了。Futhark 的基础知识相当简单，毕竟这是设计目标之一。在本文中，我将展示我是如何将[libcoap](https://www.libcoap.net/)从最基本的在Nim中使用C库包装到构建更复杂的绑定的。

## 简单绑定

首先，我们来看看libcoap在C语言中是如何工作的，只需包含`coap.h`文件，然后在编译时使用`-lcoap-3`来动态链接`libcoap.so`。在使用Futhark的Nim中，过程与此类似：

```nim
import futhark
 
importc:
  path "/usr/include/coap3"
  "coap.h"
```

这需要很短的处理时间（在我的机器[<sup data-dl-uid="93" data-dl-original="true" data-dl-translated="true">5</sup>](https://peterme.net/wrapping-c-libraries-in-nim.html#fn5)上正好是3.5 秒），并在编译日志中打印出一些有用的提示。为简洁起见，这里将其缩短，但看起来有点像这样：

```log
Hint: Running: opir -I/usr/include/coap3 -I/usr/lib/clang/15.0.7/include /home/peter/.cache/nim/coaptest_d/futhark-includes.h [User]
Hint: Parsing Opir output [User]
Hint: Caching Opir output in /home/peter/.cache/nim/coaptest_d/opir_FB377077491B2805.json [User]
Hint: Generating Futhark output [User]
Hint: Renaming "addr" to "addrfield" in structcoapaddresst [User]
Hint: Renaming "addr" to "addrarg" [User]
Hint: Renaming "type" to "typefield" in structcoaptlsversiont [User]
Hint: Renaming "type" to "typearg" [User]
Hint: Renaming "block" to "blockarg" [User]
Hint: Renaming "method" to "methodarg" [User]
Hint: Caching Futhark output in /home/peter/.cache/nim/coaptest_d/futhark_6BEC84F620D201F5.nim [User]
Hint: Declaration of stderr already exists, not redeclaring [User]
Hint: Declaration of close already exists, not redeclaring [User]
Hint: Declaration of stdout already exists, not redeclaring [User]
```

这说明了几件事：首先，它调用了辅助工具opir[<sup data-dl-uid="185" data-dl-original="true" data-dl-translated="true">6</sup>](https://peterme.net/wrapping-c-libraries-in-nim.html#fn6)，给出了我们在系统中定义的路径和一个简略头文件。这会在我们的缓存目录中创建一个JSON文件（打印出来大约有38千行），Futhark会读取该文件。然后，Futhark会重命名一些会与内置名称冲突的内容，重命名的内容取决于上下文，我们可以看到结构中的字段会得到一个`field`后缀，过程中的参数会得到一个`arg`后缀。你还会注意到，所有名称都是小写，没有下划线。这只是为了内部目的，因为Nim对样式不敏感，所以你可以选择在代码中使用`coap_address_t`还是`coapAddressT`。你可能会问为什么结构体有一个`struct`前缀，这只是为了与C语言要求`struct`关键字的方式兼容，但大多数库都将结构体类型定义为相同的名称，而不包含`struct`部分。当然，Futhark也会翻译这种类型定义。在完成所有转换后，我们可以看到Futhark将其输出文件（对于此版本的库而言，该文件长达4.8 千行）存储到缓存目录中。这个文件如此之大的原因在接下来的三行中有所暗示，“Declaration of X already exists, not redeclaring”是在每个标识符周围添加的检查，以确保封装器不会试图覆盖任何Nim现有的类型，或者我们可能自己手动定义的过程和对象。这些检查会使输出结果难以阅读，因此Futhark也可以选择不添加这些检查，但这样更容易产生问题。由于设计目标的第一条就是“它应该能正常工作”，所以默认情况下一切都以这种方式进行保护。

## 基本用法

现在我们已经构建了封装器，可以编写一些CoAP代码了！在我们开始之前，先简单介绍一下CoAP究竟是什么。从本质上讲，它是HTTP的简化版，适用于受限设备（内存、CPU、程序存储空间等）。我不知道它有多常见，但我需要它来连接宜家的智能灯网关。让我们从尝试调用coap库中的内容开始，在importc代码块后添加这个代码：

```nim
var context = coapNewContext(nil)
```

瞧，我们完成了从Nim到C库的首次调用！令人兴奋的东西，但还不是很有用。这只是分配并实例化了一个新的`coap_context_t`，并返回了它的指针。这也是我们第一次出现内存泄漏，如果我们尝试编译代码，就会从 C 语言中看到一个相当讨厌的错误，说对`coap_new_context`的引用是未定义的。这是怎么回事？Futhark应该很简单的，对吧？这里的问题就像我前面提到的，当我们说C语言中存在某些东西时，Nim完全相信我们，所以它很高兴地为我们生成了对`coap_new_context`的调用。但是，由于我们从未告诉Nim链接动态链接库（还记得我们说过的`-lcoap-3`开关吗？基本上，我们对Nim编译器撒了谎，它很高兴地生成了不知道如何编译的C代码。解决这个问题的方法很简单，就像向C编译器传递`-lcoap-3`一样，我们需要向Nim编译器传递`--passL:"-lcoap-3"`。`passL`标志的意思很简单，就是 "把这个传递给C链接器"。Futhark需要链接开关而不是自动添加它的原因是，它不考虑你是否想静态或动态链接到C项目。

将这些标志添加到Nim配置文件中也是一种很好的做法，这样你就不必每次都记得输入这些标志了。最糟糕的事情莫过于你知道你的项目可以运行，但却不记得如何构建它。这样做的另一个好处是可以让你的编辑器了解它，所以如果你的编辑器有一个“编译并运行”系统，它也应该能够构建它。

说到编辑器工具，你可能已经注意到，当你尝试使用C库中的东西时，你的代码会像圣诞树一样亮起，错误百出。这只是因为你的编辑器不喜欢调用宏来获取包含语句（这就是`importc`块的全部内容）。我建议你阅读Futhark文档中的 “Shipping wrappers”部分，以获得更正确的方法，但一个简单的解决方法是将`importc`调用隐藏在`when defined(useFuthark)`开关后面，然后像这样手动包含缓存文件：

```nim
when defined(useFuthark):
  importc:
    path "/usr/include/coap3"
    "coap.h"
else:
  include "/home/peter/.cache/nim/coaptest_d/futhark_6BEC84F620D201F5.nim"
```

这意味着在编译时需要添加`-d:useFuthark`才能真正使用Futhark，但这也意味着编辑器可以轻松找到我们正在使用的实际定义，而不会抱怨。

不过，既然我们的代码已经编译成功，而且Nim和C编译器都同意“从哪里来”，那我们就继续编码吧。正如我提到的，我们毕竟出现了内存泄漏！记住，我们现在要处理的是一个C库，这意味着我们必须手动管理它所使用的内存。如果你只习惯于用Nim或其他垃圾回收或引用计数语言编写代码，这可能有点陌生。但它的意思是，当我们告诉C库创建一个资源时，我们还需要告诉C库在我们使用完该资源后将其销毁。因此，我们需要在代码末尾添加类似最后一行的内容：

```nim
var context = coapNewContext(nil)
coapFreeContext(context)
```

太好了，不再有内存泄露了！不过，手动管理内存可不是件好玩的事，毕竟 Nim 的一大特色就是自带内存管理系统，这样我们就不用再处理这些事情了。幸运的是，为我们的C对象添加析构函数非常简单。在本例中，`context` 将是一个指向`coap_context_t` 的指针。不过，如果你翻阅libcoap的所有头文件，你将找不到一个完整的定义。从本质上讲，我们不应该使用或操作这个结构，所以它的实现是隐藏的，实现者可以随时更改它。在这种情况下，Futhark 的做法是创建一个简单的`type coapcontextt = object`定义，我们可以用指针指向它。但是，我们不能为指针类型添加一个析构函数，那么我们该如何实现这一功能呢？有多种方法，但我们在此采用最简单的一种。

```nim
when defined(useFuthark):
  importc:
    path "/usr/include/coap3"
    "coap.h"
else:
  include "/home/peter/.cache/nim/coaptest_d/futhark_6BEC84F620D201F5.nim"
 
type Context = distinct ptr coapcontextt
 
converter toBase(c: Context): ptr coapcontextt = cast[ptr coapcontextt](c)
 
proc `=destroy`(x: var Context) =
  echo "Destroyed"
  coapFreeContext(x)
 
proc newContext(): Context =
  echo "Created"
  Context(coapNewContext(nil))
 
var context = newContext()
```

虽然我们不能为指针类型附加析构函数，但我们可以为distinct指针类型附加析构函数。如果运行上述代码，你会发现执行结果会打印出“创建”和“销毁”。有了转换回基本类型的小转换器，我们还能确保可以调用原始指针类型的C程序，只是要确保不要在Nim代码中存储这些指针，因为它们不会算作引用。

虽然这对这个库来说不是问题，但你需要记住一个事实，那就是Nim内存管理方案只适用于Nim代码。因此，如果你有一个析构函数，但将一个指针传入一个保持该指针的C函数，Nim不会看到该指针并释放数据，因为它检测到该指针在Nim 代码中已不再使用。通常情况下，C库会让你释放自己的资源，所以这并不常见，如果你真的需要，你可以使用`GC_ref`和`GC_unref`来手动增加或减少引用计数。

这种方法的唯一真正缺点是我们现在需要创建自己的构造函数方法，但由于我们是在封装C语言的东西，所以无论如何我们都很有可能希望这样做。例如，在本例中，`coapNewContext`需要一个指向要监听的地址的指针，以便在服务器中使用。在C语言中，我们需要将其作为`nil`指针传递，但在Nim中，我们可以轻松地将其修改为默认值：

```nim
proc newContext(listenAddr: ptr coapAddressT = nil): Context =
  Context(coapNewContext(listenAddr))
```

说到地址，`coap_address_t`结构的设置可能有点麻烦。基本上，你只需要初始化它们，然后用其他低级系统程序的数据填充它们。这正是将低级C语言逻辑隐藏在更符合人体工程学的Nim程序中的绝佳机会。这里我就不多说了，如果你想了解具体做法，可以查看我[发布的绑定](https://github.com/PMunch/libcoap/blob/master/src/coap.nim#L65-L90)程序。

## 使问题复杂化的异步

现在，我们已经解决了创建更符合人体工程学的绑定，以及从简单的C语言类型中创建垃圾类型的问题。但C和Nim之间还有许多不同之处。如果我对每一个细节都一一详述，这篇文章就会变得太长，但有一个话题我想更详细地探讨一下。这也有助于巩固一些更基本的概念。这个主题就是异步编程。Nim有一个async模块，允许使用更现代的async/await模式，但C语言没有类似的模块。正如我在[关于异步编程的文章](https://peterme.net/asynchronous-programming-in-nim.html)中所概述的，整个概念都是围绕协作式多任务处理构建的。我们告诉底层系统我们在等待某件事情，一旦该事情发生，系统就会将控制权交还给我们。当然，这也意味着要由其他系统负责向该目标推进。这通常用于等待操作系统处理的事情，如文件读取或网络活动。不过，在上一篇文章中，我只展示了系统如何使用异步休眠来模拟工作。但这是一个展示这种系统如何在更真实的场景中运行的绝佳机会。事实上，这种异步编程方式并不新鲜。举个例子，如果你想象一个典型的阻塞文件读取，概念是类似的，我们告诉操作系统我们要读取一个文件到内存中，它会从我们的程序中夺走控制权，并在文件读取完成后交还给我们。但如果我们想让程序的其他部分在我们等待时执行操作，我们可以使用非阻塞读取，然后偶尔轮询操作系统，看看文件加载是否完成。这正是Nim中的异步系统的工作原理。在等待读取资源时，该资源的文件句柄会在异步系统中注册，异步系统会以适当的时间间隔轮询操作系统，并决定继续执行哪个部分。在Nim 中，我们仍然可以使用这一底层系统，CoAP库以及许多其他C库都支持获取支持轮询的文件句柄。这意味着我们实际上可以将此类库集成到Nim异步系统中，并为C库提供真正的异步/等待支持。

如果要详细介绍如何做到这一点，又将需要一篇更长的文章，但有几件事可能会引起我们的特别兴趣。系统要求文件（或套接字）以非阻塞方式正确打开，它们的句柄要在异步系统中注册，并针对文件的状态注册回调。例如，在等待回复时，必须向异步系统注册CoAP套接字（该套接字已用所需的标志打开），并注册`read`回调。回调返回`true`或`false`，取决于回调后文件句柄是否仍应在异步系统中注册。

这非常简单，但要做到这一点并非易事，尤其是要不要保留注册的文件。我在CoAP绑定中使用的窍门是在会话中注册一个指向Nim处理过的内存的指针，以便以后调用。在存储这种指向托管内存的指针时必须小心谨慎。只有在这种情况下，指针才会起作用，因为实际数据所在的对象总是比CoAP会话更长远。这种将任意数据指针与回调一起传递的方法在C语言库中非常常见，因此在其他库中也可以使用类似的系统。

如果你仔细阅读CoAP库的代码，就会发现它实际上使用了一个间接的回调系统。回调只需调用CoAP库中名为`ioProcess`的进程，该进程会调用之前注册的主回调，然后检查等待消息的缓存，以确定是否应删除文件处理程序。这样做只是为了更好地适应库的架构。

在Nim中封装C库是一项繁琐的工作。有了Futhark，基本的工作都已完成，你可以专注于在C库之上构建更合适的Nim绑定。Futhark绑定可以确保从Nim到C的调用正确无误，让您可以在上面构建更好的系统，而不用再费力地编写C库。即使是像支持Webkit2的Gtk这样复杂的库，也可以直接从Nim中使用，只需Futhark完成自动绑定即可。

___

1.  [Nim完全可以在没有GC的情况下运行，因此它确实可以适用于任何C语言可以适用的地方。宏甚至可以在其他有限的系统上实现零成本抽象，非常方便。](https://peterme.net/wrapping-c-libraries-in-nim.html#fnref1)
    
2.  [与Python的Pythonic一词不同，我们在Nim中还没有真正确定一个类似的术语。我个人最喜欢的是“royale”，但我担心这个词过于晦涩。](https://peterme.net/wrapping-c-libraries-in-nim.html#fnref2)
    
3.  [这很有可能，因为我想不出你在C代码库中导入了什么库来接收Nim字符串对象。](https://peterme.net/wrapping-c-libraries-in-nim.html#fnref3)
    
4.  [约8千行头文件，无注释。](https://peterme.net/wrapping-c-libraries-in-nim.html#fnref4)
    
5.  [虽然3.5秒的时间并不长，但从长远来看，一次又一次地重建包装器会让人感到厌烦。幸运的是，Futhark 会缓存其输出，因此只要`导入`块中没有任何变化，缓存就会简单地包含在你的文件中。](https://peterme.net/wrapping-c-libraries-in-nim.html#fnref5)
    
6.  [Futhark以古老的符文字母命名，因为它能读取我们过去的文字（实际的Futhark是符文铭文，项目Futhark是C代码）。辅助工具Øpir以最著名的符文大师命名，因为他也能读取符文，就像我们的辅助工具能读取C代码一样。不过二进制被称为`opir`是为了避免特殊字符的问题。Ø的发音类似于 "thunder "或 "pur "中的 "U"。](https://peterme.net/wrapping-c-libraries-in-nim.html#fnref6)