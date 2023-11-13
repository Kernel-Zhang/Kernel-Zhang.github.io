---
title: C++编译器如何实现异常处理
author: kernel
date: 2023-11-13 13:34:00 +0800
categories: [C++, 编译原理]
tags: [C++, 编译原理]
---

深入探讨VC++如何实现异常处理。源代码包括VC++的异常处理库。

## 简介

与传统语言相比，C++ 的革命性特点之一是它对异常处理的支持。传统的错误处理技术往往不够完善，而且容易出错，而 C++ 提供了一个很好的替代方案。正常代码和错误处理代码之间的明确分离使程序非常整洁和易于维护。本文将讨论编译器如何实现异常处理。本文假定读者普遍了解异常处理机制及其语法。我为 VC++实现了异常处理库，并随本文一起发布。要使用我的异常处理程序替换 VC++提供的异常处理程序，请调用以下函数：

```cpp
install_my_handler()；
```

在这之后，程序中出现的任何异常--从抛出异常到堆栈展开、调用捕获块再到恢复执行--都将由我的异常处理库来处理。

与 C++ 中的其他功能一样，C++ 标准并没有对如何实现异常处理做出任何规定。这意味着每个厂商都可以自由地使用他认为合适的实现方式。我将介绍 VC++是如何实现这一功能的，但对于那些使用其他编译器或操作系统的人来说，这也应该是一个很好的学习材料[^1]。VC++在Windows操作系统提供的结构化异常处理（SEH）基础上构建了异常处理支持[^2]。

## 结构化异常处理 - 概述

在本次讨论中，我将把异常视为那些明确抛出的异常或由于除以零或空指针访问等条件而发生的异常。异常发生时，会产生中断并将控制权转移到操作系统。反过来，操作系统会调用异常处理程序，该程序会检查从当前函数开始的函数调用序列，并执行堆栈展开和控制权转移的工作。我们可以编写自己的异常处理程序，并向操作系统注册，以便在发生异常时调用。

Windows 定义了一种特殊的注册结构，称为`EXCEPTION_REGISTRATION`：

```cpp
struct EXCEPTION_REGISTRATION
{
   EXCEPTION_REGISTRATION *prev;
   DWORD handler;
};
```

要注册自己的异常处理程序，请创建该结构并将其地址存储在FS寄存器指向的段的偏移量0处，如下伪汇编语言指令所示：

```asm
mov FS:[0], exc_regp
```

prev 字段表示`EXCEPTION_REGISTRATION`结构的链接列表。当我们注册`EXCEPTION_REGISTRATION`结构时，我们会在 prev 字段中存储之前注册的结构的地址。

那么异常回调函数是怎样的呢？Windows 要求在 EXCPT.h 中定义的异常处理程序的签名如下：

```cpp
EXCEPTION_DISPOSITION (*handler)(
    _EXCEPTION_RECORD *ExcRecord,
    void * EstablisherFrame, 
    _CONTEXT *ContextRecord,
    void * DispatcherContext);
```

你可以暂时忽略所有参数和返回类型。下面的程序向操作系统注册了异常处理程序，并通过尝试除以零产生了一个异常。异常处理程序捕获了这个异常，但并没有做什么。它只是打印一条信息并退出。

```cpp
#include <iostream>
#include <windows.h>

using std::cout;
using std::endl;


struct EXCEPTION_REGISTRATION
{
   EXCEPTION_REGISTRATION *prev;
   DWORD handler;
};


EXCEPTION_DISPOSITION myHandler(
    _EXCEPTION_RECORD *ExcRecord,
    void * EstablisherFrame, 
    _CONTEXT *ContextRecord,
    void * DispatcherContext)
{
	cout << "In the exception handler" << endl;
	cout << "Just a demo. exiting..." << endl;
	exit(0);
	return ExceptionContinueExecution; //will not reach here
}

int g_div = 0;

void bar()
{
	//initialize EXCEPTION_REGISTRATION structure
	EXCEPTION_REGISTRATION reg, *preg = &reg;
	reg.handler = (DWORD)myHandler;
	
	//get the current head of the exception handling chain	
	DWORD prev;
	_asm
	{
		mov EAX, FS:[0]
		mov prev, EAX
	}
	reg.prev = (EXCEPTION_REGISTRATION*) prev;
	
	//register it!
	_asm
	{
		mov EAX, preg
		mov FS:[0], EAX
	}

	//generate the exception
	int j = 10 / g_div;  //Exception. Divide by 0. 
}

int main()
{
	bar();
	return 0;
}

/*-------output-------------------
In the exception handler
Just a demo. exiting...
---------------------------------*/
```

请注意，Windows 严格执行一条规则：`EXCEPTION_REGISTRATION` 结构应位于栈上，且其内存地址应低于其上一个节点。如果发现情况并非如此，Windows 将终止进程。

## 函数和栈

栈是一个连续的内存区域，用于存储函数的本地对象。更具体地说，每个函数都有一个相关的栈框架，用于存放函数的所有本地对象以及函数中的表达式产生的任何临时对象。请注意，这是一幅典型的图片。实际上，编译器可能会将全部或部分对象存储在内部寄存器中，以加快访问速度。栈是处理器层面支持的概念。处理器提供内部寄存器和特殊指令来操作它。

下图显示了当函数 foo 调用函数 bar，bar 调用函数 widget 时典型的堆栈情况。请注意，在这种情况下，栈是向下增长的。这意味着下一个要推入栈的项目的内存地址将低于前一个项目的内存地址。

![Image 1](/assets/img/2023-11-13/figure2.gif)

编译器使用 EBP 寄存器来识别当前的活动栈帧。如图所示，EBP 寄存器指向 widget 的栈帧。函数相对于栈帧指针访问本地对象。编译器会在编译时将所有本地对象名称解析为从帧指针开始的某个固定偏移量。例如，widget 通常会以帧指针下的某个固定字节数（如 EBP-24）访问其局部变量。

图中还显示了 ESP 寄存器，即指向堆栈最后一项的栈指针，或者在当前情况下，ESP 指向 widget 帧的末尾。下一帧将在该位置后创建。

处理器支持入栈和出栈两种操作。考虑一下：

```asm
pop EAX
```

表示从 ESP 指向的位置读取 4 个字节，并将 ESP 增加 4（记住，在我们的情况下，栈是向下增长的）（在 32 位处理器中）。类似地

```asm
push EBP
```

表示将 ESP 递减 4，然后将 EBP 寄存器的内容写入 ESP 指向的位置。

编译器在编译一个函数时，会在函数的开头添加一些代码，称为帧头（prologue），用于创建和初始化函数的栈帧。同样，编译器也会在函数末尾添加名为帧尾（epilogue）的代码，以弹出退出函数的栈帧。

编译器通常会为帧头生成以下序列：

```asm
Push EBP      ; save current frame pointer on stack
Mov EBP, ESP  ; Activate the new frame
Sub ESP, 10   ; Subtract. Set ESP at the end of the frame
```

第一条语句将当前帧指针 EBP 保存在栈中。第二条语句将 EBP 寄存器设置到存储调用者 EBP 的位置，从而激活被调用者的帧。第三条语句将 ESP 寄存器的值减去函数将创建的所有本地对象和临时对象的总大小，从而在当前帧的末尾设置 ESP 寄存器。编译器在编译时就知道函数的所有本地对象的类型和大小，因此它实际上知道帧的大小。

帧尾与帧头相反，它必须从栈中删除当前帧：

```
Mov ESP, EBP    
Pop EBP         ; activate caller's frame
Ret             ; return to the caller
```

它将 ESP 设置在保存调用者帧指针的位置（即被调用者帧指针指向的位置），然后将其从 EBP 中弹出，从而激活调用者的栈帧，然后执行返回。

当处理器遇到 return 指令时，会执行以下操作：从栈中出栈返回地址，并将控制权转移到该地址。返回地址是在调用者执行调用指令时放入栈的。调用指令首先如栈应返回控制权的下一条指令的地址，然后跳转到被调用者的起始位置。图 3 显示了运行时栈的详细情况。如图所示，函数参数也是函数栈帧的一部分。调用者将被调用者的参数推入栈。当函数返回时，调用者会将参数的大小加到 ESP 中，从而从栈中移除被调用者的参数：

``` asm
Add ESP, args_size
```

另外，被调用者也可以通过在返回指令中指定参数的总大小来移除参数。下面的指令在返回给调用者之前从栈中删除了 24 个字节，假定参数的总大小为 24：

```asm
Ret 24
```

根据被调用者的调用习惯，每次只能使用其中一种方案。请注意，进程中的每个线程都有自己的相关栈。

![Image 2](/assets/img/2023-11-13/figure3.gif)

## C++和异常

记得我在第一节中谈到过`EXCEPTION_REGISTRATION`结构。它用于向操作系统注册异常回调，在异常发生时调用。

VC++通过在末尾添加两个字段扩展了该函数的语义：

```cpp
struct EXCEPTION_REGISTRATION
{
   EXCEPTION_REGISTRATION *prev;
   DWORD handler;
   int   id;
   DWORD ebp;
};
```

除少数例外情况外，VC++ 会为每个函数创建`EXCEPTION_REGISTRATION`结构作为其局部变量[^3]。该结构的最后一个字段与帧指针 EBP 指向的位置重叠。函数的帧头在栈帧上创建该结构，并向操作系统注册。帧尾恢复调用者的`EXCEPTION_REGISTRATION`。我将在接下来的章节中讨论 id 字段的重要性。

当 VC++编译一个函数时，它会为函数生成两组数据：

1. 异常回调函数。  
2. 包含函数重要信息的数据结构，如捕获块、捕获块地址、捕获的异常类型等。我将把这个结构称为`funcinfo`，并在下一节详细介绍。

图 4展示了运行时异常处理的全貌。Widget 的异常回调位于 `FS:[0]`（由 widget 的帧头设置）指向的异常链的头部。异常处理程序将 widget 的`funcinfo`结构的地址传递给`__CxxFrameHandler`函数，该函数会检查该数据结构，查看函数中是否有任何捕获块对捕获当前异常感兴趣。如果找不到，它就把`ExceptionContinueSearch`值返回给操作系统。操作系统从异常处理列表中获取下一个节点，并调用其异常处理程序（即当前函数调用方的处理程序）。

![Image 3](/assets/img/2023-11-13/figure4.gif)

这种情况一直持续到异常处理程序找到有兴趣捕获异常的捕获块为止，在这种情况下，异常处理程序不会返回操作系统。但在调用捕获块（从`funcinfo`结构中知道捕获块的地址，见图 4）之前，它必须执行堆栈展开操作：清理该函数框架下面的函数栈帧。清理栈帧涉及到一些复杂的小问题：异常处理程序必须找到异常发生时在栈帧上存活的函数的所有本地对象，并调用它们的析构函数。我将在后面的章节中详细讨论。

异常处理程序将清理帧的任务委托给与该帧关联的异常处理程序。它从 `FS:[0]` 指向的异常处理列表的开头开始，在每个节点上调用异常处理程序，告诉它栈正在展开。作为回应，处理程序会调用帧上所有本地对象的析构函数并返回。这个过程一直持续到与自己对应的节点为止。

由于 catch 代码块是函数的一部分，因此它使用所属函数的栈帧。因此，异常处理程序需要在调用 catch 块之前激活它的栈帧。其次，每个 catch 代码块只接受一个参数，其类型就是它要捕获的异常类型。异常处理程序应将异常对象或其引用复制到 catch 块的帧中。它知道从`funcinfo`结构中将异常复制到哪里。编译器会慷慨地为它生成这些信息。

复制异常并激活帧后，异常处理程序会调用 catch 块。在 try-catch 块之后，catch 块会返回函数中应转移控制的地址。请注意，此时虽然栈已展开，帧也已清理，但它们并没有被移除，仍然占据着栈上的物理空间。这是因为异常处理程序仍在执行，而且和其他正常函数一样，它也使用栈来存放本地对象，其帧存在于异常产生的上一个函数帧的下方。当 catch 块返回时，它需要销毁异常。在此之后，异常处理程序会通过将 ESP 设置在函数帧（它必须将控制权转移到函数帧）的末尾来移除所有帧，包括它自己的帧，并在 try-catch 块结束时转移控制权。它如何知道函数帧的结束位置？它无从得知。这就是为什么编译器会（通过函数的帧头）将其保存在函数的栈帧中，以便异常处理程序找到它。参见图 4。它位于堆栈帧指针 EBP 的下方 16 个字节处。

捕获块本身可能会抛出一个新的异常或重新抛出同一个异常。异常处理程序必须注意这种情况，并采取适当的措施。如果 catch 块抛出新异常，异常处理程序必须销毁旧异常。如果捕获块指定重新抛出异常，那么异常处理程序必须传播旧异常。

这里有一点需要注意：由于每个线程都有自己的栈，这意味着每个线程都有自己的 `EXCEPTION_REGISTRATION` 结构列表，该列表由 `FS:[0]` 指向。

## C++和异常-2

图 5显示了`funcinfo`结构的布局。请注意，这些名称可能与 VC++编译器使用的实际名称不同，我只显示了相关字段。下一节将讨论展开表的结构。

![Image 4](/assets/img/2023-11-13/figure5.gif)

当异常处理程序需要在函数中查找捕获块时，它首先要确定的是，在函数中异常产生的地方是否有外层 try 块。如果没有找到任何 try 代码块，就会返回。否则，它会搜索与外层 try 代码块相关的 catch 代码块列表。

首先，让我们看看它是如何查找 try 代码块的。在编译时，编译器会为每个 try 代码块分配一个 start id 和 end id。异常处理程序也可以通过 `funcinfo` 结构访问这些 id。见图 5。编译器会为函数中的每个 try 块生成 tryblock 数据结构。

在上一节中，我谈到 VC++ 扩展了`EXCEPTION_REGISTRATION`结构以包含 id 字段。回想一下，该结构存在于函数的栈帧中。参见图 4。发生异常时，异常处理程序会从堆栈帧中读取该 id，并检查 tryblock 结构，查看 id 是否等于或介于开始 id 和结束 id 之间。如果是，则异常源于此 try 块。否则，它会查看 tryblocktable 中的下一个 tryblock 结构。

谁在堆栈中写入 id 值？编译器会在函数的不同位置添加语句，更新 id 值以反映当前的运行时状态。例如，编译器会在进入 try 块时在函数中添加一条语句，将 try 块的起始 id 写入栈帧。

一旦异常处理程序找到 try 代码块，它就可以遍历与 try 代码块关联的 catchblock 表，查看是否有任何 catchblock 有兴趣捕获异常。请注意，在嵌套 try 块的情况下，源于内部 try 块的异常也源于外部 try 块。异常处理程序应首先查找内部 try 代码块的捕获代码块。如果找不到，则查找外层 try 块的捕获块。当在 tryblock 表中放置结构时，VC++ 会将内部 try 块结构放在外部 try 块之前。

异常处理程序如何（通过 catchblock 结构）确定某个 catch 块是否有兴趣捕获当前异常？它是通过比较异常的类型和捕获块参数的类型来确定的。考虑一下

```cpp
void foo()
{
   try {
      throw E();
   }
   catch(H) {
      //.
   }
}
```

如果 H 和 E 的类型完全相同，catch 块就会捕获异常。异常处理程序必须在运行时比较这两种类型。通常，像 C 语言这样的语言不提供在运行时确定对象类型的工具。C++ 提供了运行时类型识别机制（RTTI），并有在运行时比较类型的标准方法。它在标准头`<typeinfo>`中定义了一个类`type_info`，在运行时表示类型。catchblock 结构的第二个字段（见图 5）是指向`type_info`结构的指针，该结构在运行时代表 catchblock 参数的类型。因此，异常处理程序只需将捕获块参数的`type_info`（可通过 catchblock 结构获取）与异常的`type_info`进行比较（调用运算符`==`），即可确定捕获块是否有兴趣捕获当前异常。

异常处理程序从`funcinfo`结构中知道了 catch 块参数的类型，但它如何知道异常的 `type_info`？当编译器遇到以下语句时：

```cppp
throw E();
```

这样的语句时，编译器会为抛出的异常生成`excpt_info`结构体。见图 6。请注意，这些名称可能与 VC++编译器使用的实际名称不同，我只显示了相关字段。如图所示，异常的`type_info`可通过`excpt_info`结构获取。在某些时候，异常处理程序需要销毁异常（在调用 catch 块之后）。它可能还需要复制异常（在调用 catch 块之前）。为了帮助异常处理程序完成这些任务，编译器会通过`excpt_info`结构向异常处理程序提供异常的析构函数、复制构造函数和大小。

![Image 5](/assets/img/2023-11-13/figure6.gif)

如果 catch 代码块的参数类型是基类，而异常是其派生类，异常处理程序仍应调用该 catch 代码块。然而，在这种情况下比较两者的 typeinfo 会得出错误结果，因为它们不是同一类型。类`type_info`也没有提供任何成员函数或运算符来说明一个类是否是另一个类的基类。然而，异常处理程序必须调用这个 catch 块。为此，编译器为处理程序生成了更多信息。如果异常是一个派生类，那么`etypeinfo_table`（可通过`excpt_info`结构获取）就包含了层次结构中所有类的`etype_info`（扩展类型信息，我的名字）指针。因此，异常处理程序会将 catch 块参数的`type_info`与`excpt_info`结构中的所有 `type_info` 进行比较。如果发现任何匹配，就会调用 catch 代码块。

在结束本节之前，我还想说最后一点：异常处理程序是如何知道异常和 `excpt_info` 结构的？我将在下面的讨论中尝试回答这个问题。

VC++会将抛出语句翻译成类似的内容：

```cpp
//throw E(); //compiler generates excpt_info structure for E.
E e = E();  //create exception on the stack
_CxxThrowException(&e, E_EXCPT_INFO_ADDR);
```

`_CxxThrowException`将控制权传递给操作系统（通过软件中断，参见函数`RaiseException`），并将其两个参数传递给操作系统。操作系统在准备调用异常回调时，会将这两个参数打包到`_EXCEPTION_RECORD`结构中。它从 `FS:[0]` 指向的`EXCEPTION_REGISTRATION`列表的首部开始，调用该节点上的异常处理程序。指向`EXCEPTION_REGISTRATION` 的指针也是异常处理程序的第二个参数。请注意，在 VC++ 中，每个函数都会在栈帧上创建自己的`EXCEPTION_REGISTRATION`，并对其进行注册。将第二个参数传递给异常处理程序，可以使其获得重要信息，如 `EXCEPTION_REGISTRATION` 的 id 字段（对于找到捕获块非常重要）。它还能让异常处理程序知道函数的栈帧（对清理堆栈帧很有用）以及`EXCEPTION_REGISTRATION`节点在异常列表中的位置（对展开堆栈很有用）。第一个参数是指向`_EXCEPTION_RECORD` 结构的指针，通过该结构可以获得异常指针及其`excpt_info`结构。EXCPT.H 中定义的异常处理程序的签名是：

```cpp
EXCEPTION_DISPOSITION (*handler)(
    _EXCEPTION_RECORD *ExcRecord,
    void * EstablisherFrame, 
    _CONTEXT *ContextRecord,
    void * DispatcherContext);
```

可以忽略最后两个参数。返回类型是一个枚举（参见 EXCPT.H）。正如我之前所讨论的，如果异常处理程序找不到捕获块，它就会将`ExceptionContinueSearch`值返回给系统。在本讨论中，其他值并不重要。在 WINNT.H 中，`_EXCEPTION_RECORD`结构的定义如下

```cpp
struct _EXCEPTION_RECORD
{
    DWORD ExceptionCode;
    DWORD ExceptionFlags; 
    _EXCEPTION_RECORD *ExcRecord;
    PVOID   ExceptionAddress; 
    DWORD NumberParameters;
    DWORD ExceptionInformation[15]; 
} EXCEPTION_RECORD;
```

`ExceptionInformation`数组中条目数量和种类取决于`ExceptionCode`字段。如果`ExceptionCode`代表 C++（异常代码为 0xe06d7363）异常（如果异常是由于抛出引起的），则`ExceptionInformation`数组包含指向异常的指针和`excpt_info`结构。对于其他类型的异常，数组中几乎没有任何条目。其他类型的异常包括除以零、违反访问权限等，它们的值可以在 WINNT.H.ExceptionInformation 数组中找到。

异常处理程序会查看`_EXCEPTION_RECORD`结构中的`ExceptionFlags`字段，以确定要采取的措施。如果其值为`EH_UNWINDING`（定义在 Except.inc），则向异常处理程序表明堆栈正在展开，它应该清理堆栈帧并返回。清理工作包括查找异常发生时栈帧上的所有本地对象，并调用它们的析构函数。下一节将对此进行讨论。否则，异常处理程序必须搜索函数中的捕获块，如果找到则调用它。

## 栈帧清理

C++ 标准规定，在释放栈时，应调用异常发生时所有本地对象的析构函数。考虑一下

```cpp
int g_i = 0;
void foo()
{
   T o1, o2;
   {
       T o3;
   }
   10/g_i; //exception occurs here
   T o4;
   //...
}
```

当异常发生时，本地对象 o1 和 o2 存在于 foo 的框架中，而 o3 的生命周期已经结束。O4 从未创建。异常处理程序应了解这一事实，并调用 o1 和 o2 的析构函数。

正如我之前所写，编译器会在不同的特殊点为函数添加代码，以便在执行过程中记录函数的当前运行时状态。编译器会为函数中的这些特殊区域分配 id。例如，try 块入口点就是一个特殊区域。如前所述，编译器会在进入 try 代码块时在函数中添加语句，将 try 代码块的起始 id 写入函数框架。

函数中的另一个特殊区域是创建或销毁本地对象的地方。换句话说，编译器会为每个局部对象分配一个唯一的 id。当编译器遇到对象定义，如

```cpp
void foo()
{
   T t1;
   //.
}
```

它在定义后（对象创建后）添加了语句，以便在帧上写入其 id 值：

```cpp
void foo()
{
   T t1;
   _id = t1_id; //statement added by the compiler
   //.
}
```

编译器创建了一个隐藏的局部变量（在上述代码中指定为 `_id`），该变量与`EXCEPTION_REGISTRATION`结构的`id`字段重叠。同样，编译器在调用对象的析构函数之前添加了语句，以写入前一个区域的 id。

当异常处理程序需要清理帧时，它会从帧（ `EXCEPTION_REGISTRATION`结构的 `id` 字段或帧指针 EBP 下面的 4 个字节）中读取 id 值。该 id 表示函数中的代码在执行到当前 id 所对应的点时没有出现任何异常。在这一点之上的所有对象都已创建。需要调用该点以上所有或部分对象的析构函数。请注意，如果这些对象是子代码块的一部分，其中一些可能已经被销毁。不应调用这些对象的析构函数。

编译器会为函数生成另一个数据结构，即`unwindtable`(自己取名)，它是一个`unwind` 结构数组。该表可通过 funcinfo 结构获取。参见图 5。函数中的每个特殊区域都有一个 unwind 结构。与对象相对应的 unwind 结构值得关注（请记住，每个对象定义都表示特殊区域，并有与之相关的 id）。它包含销毁对象的信息。当编译器遇到对象定义时，它会生成一个简短的例程，了解该对象在帧上的地址（或其从帧指针的偏移量）并销毁该对象。unwind 结构的一个字段包含该例程的地址：

```cpp
typedef  void (*CLEANUP_FUNC)();
struct unwind
{
    int prev;
    CLEANUP_FUNC  cf;
};
```

try 块的 unwind 结构的第二个字段值为 0。prev 字段表示 unwintable 也是一个 unwind 结构的链接列表。当异常处理程序需要清理帧时，它会从帧中读取当前 ID，并将其作为索引进入展开表。它会读取该索引处的展开结构，并调用该结构第二个字段指定的清理函数。这将销毁与该 ID 对应的对象。然后，处理程序根据 prev 字段指定的索引，从展开表中读取前一个展开结构。这一过程一直持续到列表结束（prev 为-1）。图 7 显示了图中函数的展开表。

![Image 6](/assets/img/2023-11-13/figure7.gif)

考虑 new 操作符的情况：

```cpp
T* p = new T();
```

系统首先为 T 分配内存，然后调用构造函数。如果构造函数抛出异常，则系统必须释放为该对象分配的内存。为了实现这一点，VC++ 还为具有非平凡构造函数的类型的每个 new 操作符分配了 id。展开表中有相应的条目，清理例程会释放已分配的空间。在调用构造函数之前，它会将分配的 id 保存在`EXCEPTION_REGISTRATION`结构中。在构造函数成功返回后，它会恢复先前特殊区域的 id。

此外，当构造函数抛出异常时，对象可能已经部分构造完成。如果它有成员子对象或基类子对象，而其中一些在异常发生时已经构造完成，则必须调用这些对象的析构函数。为执行这些任务，编译器会为构造函数生成与普通函数相同的数据集。

请注意，异常处理程序在释放栈时会调用用户定义的析构函数。析构函数有可能抛出异常。C++ 标准规定，在释放栈时，析构函数不得抛出异常。如果抛出异常，系统将调用 `std::terminate`。

## 实现

本节将讨论上文未讨论的三个主题：

1. 安装异常处理程序。  
2. Catch 块重抛异常或抛出新异常。  
3. 每线程异常处理支持。

请查看源代码发行版中的 Readme.txt 文件，了解构建说明[1](https://www.codeproject.com/Articles/2126/How-a-C-compiler-implements-exception-handling#1)。它还包含一个演示项目。

第一项任务是安装异常处理库，或者换句话说，替换 VC++ 提供的库。从上面的讨论可以看出，VC++ 提供的`__CxxFrameHandler`函数是所有异常的入口点。对于每个函数，编译器都会生成异常处理例程，如果异常发生在该函数中，就会调用该例程。该例程将 funcinfo 指针传递给`__CxxFrameHandler`函数。

`install_my_handler()`函数在`__CxxFrameHandler`的开头插入代码，跳转到`my_exc_handler()`。但是，`__CxxFrameHandler`位于只读代码页中。任何写入它的尝试都会导致访问违规。因此，第一步是使用 Windows API 提供的`VirtualProtectEx`函数将该页面的访问权限改为读写权限。写入内存后，我们恢复页面的旧保护。该函数将 `jmp_instr` 结构的内容写入`__CxxFrameHandler` 的开头。

```cpp
//install_my_handler.cpp

#include <windows.h>
#include "install_my_handler.h"

//C++'s default exception handler
extern "C" 
EXCEPTION_DISPOSITION  __CxxFrameHandler(
     struct _EXCEPTION_RECORD *ExceptionRecord,
     void * EstablisherFrame,
     struct _CONTEXT *ContextRecord,
     void * DispatcherContext
     );

namespace
{
    char cpp_handler_instructions[5];
    bool saved_handler_instructions = false;
}

namespace my_handler
{
    //Exception handler that replaces C++'s default handler.
    EXCEPTION_DISPOSITION my_exc_handler(
         struct _EXCEPTION_RECORD *ExceptionRecord,
         void * EstablisherFrame,
         struct _CONTEXT *ContextRecord,
         void * DispatcherContext
         ) throw();

#pragma pack(1)
    struct jmp_instr
    {
        unsigned char jmp;
        DWORD         offset;
    };
#pragma pack()
    
    bool WriteMemory(void * loc, void * buffer, int size)
    {
        HANDLE hProcess = GetCurrentProcess();
        
        //change the protection of pages containing range of memory 
        //[loc, loc+size] to READ WRITE
        DWORD old_protection;
        
        BOOL ret;
        ret = VirtualProtectEx(hProcess, loc, size, 
                         PAGE_READWRITE, &old_protection);
        if(ret == FALSE)
            return false;

        ret = WriteProcessMemory(hProcess, loc, buffer, size, NULL);
       
        //restore old protection
        DWORD o2;
        VirtualProtectEx(hProcess, loc, size, old_protection, &o2);

		return (ret == TRUE);
    }

    bool ReadMemory(void *loc, void *buffer, DWORD size)
    {
        HANDLE hProcess = GetCurrentProcess();
        DWORD bytes_read = 0;
        BOOL ret;
        ret = ReadProcessMemory(hProcess, loc, buffer, size, &bytes_read);
        return (ret == TRUE && bytes_read == size);
    }

    bool install_my_handler()
    {
        void * my_hdlr = my_exc_handler;
        void * cpp_hdlr = __CxxFrameHandler;

        jmp_instr jmp_my_hdlr; 
        jmp_my_hdlr.jmp = 0xE9;
        //We actually calculate the offset from __CxxFrameHandler+5
        //as the jmp instruction is 5 byte length.
        jmp_my_hdlr.offset = reinterpret_cast<char*>(my_hdlr) - 
                    (reinterpret_cast<char*>(cpp_hdlr) + 5);
        
        if(!saved_handler_instructions)
        {
            if(!ReadMemory(cpp_hdlr, cpp_handler_instructions,
                        sizeof(cpp_handler_instructions)))
                return false;
            saved_handler_instructions = true;
        }

        return WriteMemory(cpp_hdlr, &jmp_my_hdlr, sizeof(jmp_my_hdlr));
    }

    bool restore_cpp_handler()
    {
        if(!saved_handler_instructions)
            return false;
        else
        {
            void *loc = __CxxFrameHandler;
            return WriteMemory(loc, cpp_handler_instructions, 
                           sizeof(cpp_handler_instructions));
        }
    }
}
```

在`jmp_instr`结构定义处的`#pragma pack(1)`指令告诉编译器布局结构成员时，成员之间不加任何填充。如果不使用该指令，该结构的大小为 8 字节。定义此指令后，结构体的大小为 5 字节。

回到异常处理，当异常处理程序调用 catch 块时，catch 块可能会重新抛出异常或抛出一个全新的异常。如果 catch 代码块抛出一个新的异常，那么异常处理程序必须先销毁之前的异常，然后再继续处理。如果 catch 块决定重新抛出异常，异常处理程序就必须传播当前异常。此时，异常处理程序必须解决两个问题：异常处理程序如何知道异常来自捕获块，以及如何跟踪旧异常？我解决这个问题的方法是，在处理程序调用捕获块之前，将当前异常存储在`exception_storage`对象中，并注册一个特殊用途的异常处理程序 `catch_block_protector`。异常存储对象可通过调用`get_exception_storage()`函数获得：

```cpp
exception_storage* p = get_exception_storage();
p->set(pexc, pexc_info);
register catch_block_protector
call catch block
//....
```

如果异常从 catch 块中（再次）抛出，控制权将转到 `catch_block_protector`。现在，该函数可以从 `exception_storage` 对象中提取先前的异常，并在 catch block 抛出新异常时将其销毁。如果捕获块执行了重抛操作（可以通过检查`ExceptionInformation`数组的前两个条目来确定，这两个条目都为 0。请参阅下面的代码），那么处理程序需要将当前异常复制到`ExceptionInformation`数组中，以传播该异常。下面的代码段显示了`catch_block_protector()`函数。

```cpp
//-------------------------------------------------------------------
// If this handler is calles, exception was (re)thrown from catch 
// block. The  exception  handler  (my_handler)  registers this 
// handler before calling the catch block. Its job is to determine
// if the  catch block  threw  new  exception or did a rethrow. If 
// catch block threw a  new  exception, then it should destroy the 
// previous exception object that was passed to the catch block. If 
// the catch block did a rethrow, then this handler should retrieve
// the original exception and save in ExceptionRecord for the 
// exception handlers to use it.
//-------------------------------------------------------------------
EXCEPTION_DISPOSITION  catch_block_protector(
	 _EXCEPTION_RECORD *ExceptionRecord,
	 void * EstablisherFrame,
	 struct _CONTEXT *ContextRecord,
	 void * DispatcherContext
	 ) throw()
{
    EXCEPTION_REGISTRATION *pFrame;
    pFrame = reinterpret_cast<EXCEPTION_REGISTRATION*>
    
    (EstablisherFrame);if(!(ExceptionRecord->ExceptionFlags & (  
          _EXCEPTION_UNWINDING | _EXCEPTION_EXIT_UNWIND)))
    {
        void *pcur_exc = 0, *pprev_exc = 0;
        const excpt_info *pexc_info = 0, *pprev_excinfo = 0;
        exception_storage *p = 
        get_exception_storage();  pprev_exc=
        p->get_exception();  pprev_excinfo=
        p->get_exception_info();p->set(0, 0);
        bool cpp_exc = ExceptionRecord->ExceptionCode == MS_CPP_EXC;
        get_exception(ExceptionRecord, &pcur_exc);
        get_excpt_info(ExceptionRecord, &pexc_info);
        if(cpp_exc && 0 == pcur_exc && 0 ==   pexc_info)
        //rethrow
            {ExceptionRecord->ExceptionInformation[1] = 
                reinterpret_cast<DWORD>
            (pprev_exc);ExceptionRecord->ExceptionInformation[2] = 
                reinterpret_cast<DWORD>(pprev_excinfo);
        }
        else
        {
            exception_helper::destroy(pprev_exc, pprev_excinfo);
        }
    }
    return ExceptionContinueSearch;
}
```

考虑 `get_exception_storage()` 函数的一种可能实现：

```cpp
exception_storage* get_exception_storage()
{
    static exception_storage es;
    return &es;
}
```

这将是一个完美的实现，除非是在多线程世界中。考虑到不止一个线程获取此对象并试图将异常对象存储在其中。这将是一场灾难。每个线程都有自己的栈和异常处理链。我们需要的是一个线程专用的异常存储对象。每个线程都有自己的对象，该对象在线程开始运行时创建，并在线程结束时销毁。Windows 为此提供了线程本地存储。线程本地存储使每个对象都能拥有自己的私有对象副本，该副本可通过全局密钥访问。为此，Windows 提供了`TLSGetValue()`和`TLSSetValue()`函数。

Excptstorage.cpp 文件定义了`get_exception_storage()`函数。该文件以 DLL 的形式构建。这是因为它能让我们随时了解线程的创建或销毁情况。每次创建或销毁线程时，Windows 都会调用每个 DLL（已加载到本进程的地址空间）的`DllMain()`函数。该函数在创建的线程中调用。这样我们就有机会初始化线程特定的数据，在我们的例子中就是`exception_storage`对象。

```cpp
//excptstorage.cpp

#include "excptstorage.h"
#include <windows.h>

namespace
{
    DWORD dwstorage;
}

namespace my_handler
{
    __declspec(dllexport) exception_storage* get_exception_storage() throw()
    {
        void *p = TlsGetValue(dwstorage);
        return reinterpret_cast<exception_storage*>(p);
    }
}


BOOL APIENTRY DllMain( HANDLE hModule, 
                       DWORD  ul_reason_for_call, 
                       LPVOID lpReserved
					 )
{
    using my_handler::exception_storage;
    exception_storage *p;
    switch(ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        //For the first main thread, this case also contains implicit 
        //DLL_THREAD_ATTACH, hence there is no DLL_THREAD_ATTACH for 
        //the first main thread.
        dwstorage = TlsAlloc();
        if(-1 == dwstorage)
            return FALSE;
        p = new exception_storage();
        TlsSetValue(dwstorage, p);
        break;
    case DLL_THREAD_ATTACH:
        p = new exception_storage();
        TlsSetValue(dwstorage, p);
        break;
    case DLL_THREAD_DETACH:
        p = my_handler::get_exception_storage();
        delete p;
        break;
    case DLL_PROCESS_DETACH:
        p = my_handler::get_exception_storage();
        delete p;
        break;
    }
    return TRUE;
}
```

## 结论

如上所述，在操作系统的支持下，C++ 编译器和运行时异常库合作执行异常处理。

## 注释和参考文献

[^1]: 截至本文讨论时，Visual Studio 7.0 已发布。我主要使用在奔腾处理器上运行的 Windows 2000 上的 VC++ 6.0 对异常处理库进行了编译和测试。我还用 VC++5.0 和 VC++7.0测试版进行了测试。6.0 和 7.0 之间的差异很小。6.0 首先将异常（或其引用）复制到 catch 块的帧上，然后在调用 catch 块之前执行栈展开。7.0 库则首先执行堆栈展开。在这方面，我的库的行为与 6.0 库类似。

[^2]: 请参阅 Matt Pietrek 在 MSDN 上撰写的关于[结构化异常处理](https://www.microsoft.com/msj/defaultframe.asp?page=/msj/0197/exception/exception.htm&nav=/msj/0197/newnav.htm)的精彩文章

[^3]: 如果函数没有 try 代码块，也没有定义任何具有非三维析构函数的对象，编译器可能不会为该函数生成任何与异常相关的数据。
