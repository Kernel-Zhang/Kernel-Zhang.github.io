---
title: Nim官方教程第三部分
author: kernel
date: 2023-11-08 12:34:00 +0800
categories: [Nim语言, 官方教程]
tags: [Nim语言, Nim官方教程, Nim入门]
---

| 条目        | 内容说明          |
| ----------- | ----------------- |
| 作者：      | Arne Döring |
| 版本：      | 2.0.0             |
| 翻译：      | Kernel Zhang      |

## [导言](https://nim-lang.org/docs/tut3.html#introduction)

> "权力越大，责任越大"-- 蜘蛛侠叔叔

本文档是关于Nim宏系统的教程。宏是一种在编译时执行的函数，它将Nim语法树转换成另一棵树。

可在宏中实现的功能示例

-   `myAssert(a == b)`转换为`if a != b: quit($a " != " $b)`
-   `myDebugEcho(a)`转换为`echo "a：", a`
-   将`diff(a*pow(x,3) + b*pow(x,2) + c*x + d, x)`转换为`3*a*pow(x,2) + 2*b*x + c`

### [宏参数](https://nim-lang.org/docs/tut3.html#introduction-macro-arguments)

宏参数类型有两个面。一个用于重载解析，另一个用于宏主体。例如，如果在表达式`foo(x)`中调用宏`foo( arg: int)`，那么x的类型必须与int兼容，但在宏的主体中，arg的类型是NimNode，而不是int！为什么要这样做，稍后我们看到具体例子后就会明白。

向宏传递参数有两种方式，参数可以是类型化（typed）的，也可以是非类型化（untyped）的。

### [非类型化参数](https://nim-lang.org/docs/tut3.html#introduction-untyped-arguments)

未经语义检查的宏参数会先传递给宏。这意味着传给宏的语法树对Nim来说还没有意义，唯一的限制是它必须是可解析的。通常，宏也不会检查参数，但会在转换结果中使用参数。编译器始终会检查宏扩展的结果，因此除了奇怪的错误信息外，不会发生任何不好的情况。

无类型参数的缺点是不能很好地使用Nim的重载方案。

非类型化参数的好处是语法树相当可预测，与类型化参数相比也不那么复杂。

### [类型化参数](https://nim-lang.org/docs/tut3.html#introduction-typed-arguments)

对于类型化参数，语义检查器会在参数上运行并进行转换，然后再将参数传递给宏。在这里，标识符节点被解析为符号，隐式类型转换作为调用在树中可见，模板被展开，最重要的可能是节点具有类型信息。类型化参数可以在参数列表中显示类型。但所有其他类型，如int、float或MyObjectType也是类型化参数，它们以语法树的形式传递给宏。

### [静态参数](https://nim-lang.org/docs/tut3.html#introduction-static-arguments)

静态参数是将值作为值而不是语法树节点传递给宏的一种方式。例如，对于表达式`foo(x)`中的宏`foo(arg: static[int])`，x需要是一个整数常量，但在宏体中arg就像一个int类型的普通参数。

```nim
import std/macros

macro myMacro(arg: static[int]): untyped =
  echo arg # just an int (7), not `NimNode`

myMacro(1 + 2 * 3)
```

### [将代码块作为参数](https://nim-lang.org/docs/tut3.html#introduction-code-blocks-as-arguments)

将单独的代码块作进行缩进，可以为调用表达式的最后一个参数进行传递，并加上缩进。例如，下面的代码示例是调用echo的有效方法（但不是推荐方法）：

```nim
echo "Hello ":
  let a = "Wor"
  let b = "ld!"
  a & b
```

对于宏来说，这种调用方式非常有用；可以用这种符号将任意复杂的语法树传递给宏。

### [语法树](https://nim-lang.org/docs/tut3.html#introduction-the-syntax-tree)

为了构建Nim语法树，我们需要知道Nim源代码是如何以语法树的形式进行表示的，以及如何构建这样的语法树，这样Nim编译器才能理解它。[宏](https://nim-lang.org/docs/macros.html)模块中记录了Nim语法树的节点。但使用`macros.treeRepr`是探索Nim语法树的一种更具交互性的方法，它能将语法树转换为多行字符串，并打印到控制台上。dumpTree是一个预定义宏，它只是以树的形式打印参数，除此之外什么也不做。下面是一个树形表示的示例：

```nim
dumpTree:
  var mt: MyType = MyType(a:123.456, b:"abcdef")

# output:
#   StmtList
#     VarSection
#       IdentDefs
#         Ident "mt"
#         Ident "MyType"
#         ObjConstr
#           Ident "MyType"
#           ExprColonExpr
#             Ident "a"
#             FloatLit 123.456
#           ExprColonExpr
#             Ident "b"
#             StrLit "abcdef"
```

### [自定义语义检查](https://nim-lang.org/docs/tut3.html#introduction-custom-semantic-checking)

宏处理参数的第一件事就是检查参数的形式是否正确。并不是每一种错误的输入都需要在这里被捕获，但任何可能在宏计算过程中导致崩溃的输入都应该被捕获并创建一个漂亮的错误信息。`macros.expectKind`和`macros.expectLen`是一个很好的开始。如果需要更复杂的检查，可以使用`macros.error`过程创建任意的错误信息。

```nim
macro myAssert(arg: untyped): untyped =
  arg.expectKind nnkInfix
```

### [生成代码](https://nim-lang.org/docs/tut3.html#introduction-generating-code)

生成代码有两种方法。一种是使用包含大量调用`newTree`和`newLit`创建语法树，另一种是使用`quote do:`表达式。第一种方法为语法树的生成提供了最好的底层控制，但第二种方法的冗长程度要低得多。如果您选择通过调用newTree和newLit来创建语法树，宏`macros.dumpAstGen`可以帮助您控制语法树的冗长程度。

`quote do:`允许您按字面意思写入要生成的代码。反引号用于将NimNode符号中的代码插入到生成的表达式中。

```nim
import std/macros
macro a(i) = quote do:
  let `i` = 0

a b
doAssert b == 0
```

在需要使用反引号的地方，可以定义自定义前缀运算符。

```nim
import std/macros
macro a(i) = quote("@") do:
  assert @i == 0

let b = 0
a b
```

注入的符号在解析为一个符号时需要加引号。

```nim
import std/macros
macro a(i) = quote("@") do:
  let `@i` = 0

a b
doAssert b == 0
```

确保只向生成的语法树注入NimNode类型的符号。您可以使用newLit将任意值转换为NimNode类型的表达式树，以便安全地将它们注入树中。

```nim
import std/macros

type
  MyType = object
    a: float
    b: string

macro myMacro(arg: untyped): untyped =
  var mt: MyType = MyType(a:123.456, b:"abcdef")
  
  # 
  
  let mtLit = newLit(mt)
  
  result = quote do:
    echo `arg`
    echo `mtLit`

myMacro("Hallo")
```

调用myMacro将生成以下代码：

```nim
echo "Hallo"
echo MyType(a: 123.456'f64, b: "abcdef")
```

### [构建第一个宏](https://nim-lang.org/docs/tut3.html#introduction-building-your-first-macro)

为了给编写宏提供一个起点，我们现在将演示如何实现前面提到的myAssert宏。首先要做的是建立一个使用宏的简单示例，然后直接打印参数。这样就可以了解正确的参数应该是什么样的。

```nim
import std/macros

macro myAssert(arg: untyped): untyped =
  echo arg.treeRepr

let a = 1
let b = 2

myAssert(a != b)
```

```
Infix
  Ident "!="
  Ident "a"
  Ident "b"
```

从输出结果可以看出，参数是一个中缀运算符（节点类型为"Infix"），两个操作数分别位于索引 1 和 2。有了这些信息，就可以编写实际的宏了。

```nim
import std/macros

macro myAssert(arg: untyped): untyped =
  # all node kind identifiers are prefixed with "nnk"
  arg.expectKind nnkInfix
  arg.expectLen 3
  # operator as string literal
  let op  = newLit(" " & arg[0].repr & " ")
  let lhs = arg[1]
  let rhs = arg[2]
  
  result = quote do:
    if not `arg`:
      raise newException(AssertionDefect,$`lhs` & `op` & $`rhs`)

let a = 1
let b = 2

myAssert(a != b)
myAssert(a == b)
```

这就是将生成的代码。要调试宏实际生成的内容，可以在宏的最后一行使用语句`echo result.repr`。这也是用于获取输出结果的语句。

```nim
if not (a != b):
  raise newException(AssertionDefect, $a & " != " & $b)
```

### [权利越大责任越大](https://nim-lang.org/docs/tut3.html#introduction-with-power-comes-responsibility)

宏功能非常强大。一个好的建议是尽量少用宏，但在必要时尽量多用。宏可以改变表达式的语义，使不知道宏具体作用的人无法理解代码。因此，如果不需要使用宏，而同样的逻辑可以使用模板或泛型来实现，那么最好不要使用宏。而当使用宏来实现某些功能时，宏最好有一个写得很好的文档。对于那些声称只编写完全不言自明的代码的人：当涉及到宏时，仅有实现还不足以替代文档。

### [局限性](https://nim-lang.org/docs/tut3.html#introduction-limitations)

由于宏是在NimVM的编译器中进行计算的，因此宏也受到NimVM的所有限制。宏必须以纯Nim代码实现。宏可以在shell上启动外部进程，但不能调用C函数，编译器内置的函数除外。

## [更多实例](https://nim-lang.org/docs/tut3.html#more-examples)

本教程只能介绍宏系统的基础知识。您可以从其他宏中得到启发，了解使用宏的可能性。

### [Strformat](https://nim-lang.org/docs/tut3.html#more-examples-strformat)

在Nim标准库中，strformat库提供了一个在编译时解析字符串字面的宏。一般来说，不建议使用类似的宏来解析字符串。解析后的AST不能包含类型信息，而且在虚拟机上执行解析通常速度也不快。在AST节点上工作几乎总是推荐的方式。不过，strformat仍然是一个很好的宏实际用例，它比assert宏稍微复杂一些。

[Strformat](https://github.com/nim-lang/Nim/blob/devel/lib/pure/strformat.nim)

### [Ast模式匹配](https://nim-lang.org/docs/tut3.html#more-examples-ast-pattern-matching)

Ast模式匹配是一个帮助编写复杂宏的宏库。这可以看作是一个很好的例子，说明了如何用新的语义重新利用Nim语法树。

[Ast Pattern Matching](https://github.com/nim-lang/ast-pattern-matching)

### [OpenGL沙盒](https://nim-lang.org/docs/tut3.html#more-examples-opengl-sandbox)

该项目拥有一个完全使用宏编写的Nim到GLSL的编译器。它通过递归扫描所有使用过的函数符号来编译它们，以便在GPU上执行跨库函数。

[OpenGL Sandbox](https://github.com/krux02/opengl-sandbox)