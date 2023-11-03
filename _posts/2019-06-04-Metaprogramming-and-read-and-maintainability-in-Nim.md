---
title: Nim中的元编程以及可读性和可维护性
author: kernel
date: 2019-06-04 20:34:00 +0800
categories: [Nim语言, 博客翻译]
tags: [Nim语言, 编程技巧]
---

在本网站的文章和讲座中，我都提到过关于Nim中元编程，那就是Nim中的元编程可以提高可读性和可维护性。元编程的反对者可能会对此嗤之以鼻，认为这恰恰相反。当然，元编程是一种功能强大的工具，但使用任何功能强大的工具都必须小心谨慎。一不小心，电锯就会把你的腿锯断，同样，宏写得不好也会把你的程序搞得一团糟。但这并不意味着我们不应该使用电锯或宏！也许我在Nim中使用模板和宏最多的地方并不是为了做一些奇怪而疯狂的事情，而只是为了重写我的代码，使其在不影响性能的前提下变得更可读。因为它们是在编译时解决的，而不是在程序运行时，所以只要我们确保输出是漂亮而高效的，我们就可以添加尽可能多的抽象和复杂性。

在这篇文章中，我想展示一个利用宏和模板重构代码以使其更具可读性的小例子。我将以我正在做的一个业余小项目为例，即我前天晚上写的一个小型栈计算器，我想把它扩展成HP-41C的继承者。程序非常简单，但我去掉了一些运算符，使它更简短一些。

```nim
import math, strutils
 
# We define an enum with our built in commands
type
  Commands = enum
    Plus = "+", Minus = "-", Multiply = "*", Divide = "/", Pop = "pop",
    StackSwap = "swap", StackRotate = "rot", Help = "help", Exit = "exit"
 
# Give them some documentation strings
const docstrings: array[Commands, string] = [
  "Adds two numbers",
  "Subtract two numbers",
  "Multiplies two numbers",
  "Divides two numbers",
  "Pops a number off the stack",
  "Swaps the two bottom elements on the stack",
  "Rotates the stack one level",
  "Lists all the commands with documentation",
  "Exits the program"
]
 
# Then create a simple "stack" implementation
var stack: seq[float]
 
proc push[T](stack: var seq[T], value: T) =
  stack.add value
 
proc pop[T](stack: var seq[T]): T =
  result = stack[^1]
  stack.setLen(stack.len - 1)
 
# Convenience template to execute an operation over two operands from the stack
template execute[T](stack: var seq[T], operation: untyped): untyped {.dirty.} =
  let
    a = stack[^1]
    b = stack[^2]
  stack.setLen(stack.len - 2)
  stack.push(operation)
 
# Program main loop, read input from stdin, try to parse it as a command and
# run it's corresponding operation, if that fails try to push it as a number.
# Print out our "stack" for every iteration of the loop
while true:
  for command in stdin.readLine.split(" "):
    try:
      case parseEnum[Commands](command):
      of Plus: stack.execute(a + b)
      of Minus: stack.execute(b - a)
      of Multiply: stack.execute(a * b)
      of Divide: stack.execute(b / a)
      of Pop: discard stack.pop
      of StackSwap:
        let
          a = stack[^1]
          b = stack[^2]
        stack[^1] = b
        stack[^2] = a
      of StackRotate:
        stack.insert(stack.pop, 0)
      of Help:
        echo "Commands:"
        for command in Commands:
          echo "\t", command, "\t", docstrings[command]
      of Exit: quit 0
    except:
      stack.push parseFloat(command)
    echo stack, " - ", command
```

## 栈计算器入门

现在，为了确保你知道这段代码的作用，我将快速解释栈计算器的工作原理。这种计算器并不是像"2 + 3 ="那样将运算符进行内含运算，然后得到一个结果（在本例中为5），而是使用一个堆栈。加号是一个需要两个参数的运算符，因此我们需要将两个数字推入堆栈，然后点击加号 "2 3 +"。这将从堆栈中弹出两个元素，将它们相加，然后将结果推回堆栈。这样，无需括号就能轻松求解更复杂的算式，如"（2 + 3）\*（8 / 2）"。只需运行 "2 3 +"，堆栈中就会出现数字5，然后运行 "8 2 /"，堆栈中就会出现数字 5 和 4，然后点击 "\*"将它们相乘，堆栈中就会出现20。一旦你习惯了这些计算器，就会发现它们不仅易于使用，而且实现起来也超级简单，正如你在上面所看到的。请注意，在运行示例时，你可以输入每个数字或命令，按回车键或空格分隔。

## 重构代码以使用宏

既然我们已经掌握了实际工作内容，那就开始重构吧。正如你所看到的，我已经擅自添加了一个简单的执行模板。它只是简单地接收一条带有“a”和“b”符号的语句，并将结果推入堆栈。这样做是为了避免手动进行所有变量的弹出、推送和存储。不过，下一步就比较有趣了。请注意，如果我们想添加一个新的运算符，需要在三个不同的地方进行三次修改。首先，将它添加到枚举中，并添加用于运行它的字符串命令。然后在数组的正确位置编写docstring。最后在主循环的分支选择中实现新功能。值得庆幸的是，Nim的类型系统能让我们避免很多可能出错的地方。大小写开关是严格的，这意味着如果我们没有实现所有的大小写，或者实现了太多的大小写，编译器就会抱怨。在枚举类型上定义的数组将确保我们不会在其中放入过多或过少的元素。但这还远远不够，我们最终可能会在错误的位置写入我们的数组，这将会覆盖所有其他数组。而且在文件中跳转到三个不同的地方也不是最佳选择。因此，让我们尝试用元编程来解决这个问题。

因此，我们要为操作创建一个简单的定义语法。它应包括操作名称、调用字符串、文档字符串，当然还有操作本身。我最终的选择是这样的：

```
Plus = "+"; "Adds two numbers":
  stack.execute(a + b)
Minus = "-"; "Subtract two numbers":
  stack.execute(b - a)
```

它结构紧凑、易于理解和维护。现在，我们只需要让它运行起来！Nim 将此输入视为此语法树（为简洁起见，此处仅表示加号）

```
StmtList
  Asgn
    Ident "Plus"
    StrLit "+"
  Call
    StrLit "Adds two numbers"
    StmtList
      Call
        DotExpr
          Ident "stack"
          Ident "execute"
        Infix
          Ident "+"
          Ident "a"
          Ident "b"
```

在确定输入之后，我们需要确定要生成哪些代码。类型定和docstring都在顶层，因此可以直接生成。但是 分支选择在主循环中，所以我们把它放在模板中，然后在循环中调用即可。

## 创建枚举

在一个标准的枚举类型定义上运行`dumpTree`向我们展示了如何生成一个枚举类型：

```
StmtList
  TypeSection
    TypeDef
      Ident "Commands"
      Empty
      EnumTy
        Empty
        EnumFieldDef
          Ident "Plus"
          StrLit "+"
        EnumFieldDef
          Ident "Minus"
          StrLit "-"
```

我们感兴趣的是`EnumFieldDef`节点。通过使用第一个`Empty`节点创建`EnumTy`节点，然后遍历输入中的`Asgn`节点，并将`Ident`和`StrLit`复制到`EnumFieldDef`节点中，再将`EnumFieldDef`节点添加到`EnumTy`节点中，我们就可以建立起这棵树的内部结构。然后只需将其放入外壳，我们的枚举就完成了。

虽然我们仍然只是输出结果枚举，但我们可以看到，我们现在创建了我们想要的结果：

```nim
import macros
 
macro defineCommands(definitions: untyped): untyped =
  result = newstmtlist()
  echo "This is the input to our macro"
  echo definitions.treeRepr
  var enumDef = nnkTypeSection.newTree(
      nnkTypeDef.newTree(
        newIdentNode("Commands"),
        newEmptyNode(),
        nnkEnumTy.newTree(newEmptyNode())))
  for i in countup(0, definitions.len - 1, 2):
    let
      enumInfo = definitions[i]
      commandInfo = definitions[i+1].treeRepr
    enumDef[0][2].add nnkEnumFieldDef.newtree(enumInfo[0], enumInfo[1])
  echo "This is the enum we create"
  echo enumDef.treeRepr
 
static: echo "This is what we want"
dumpTree:
  type Commands = enum
    Plus = "+", Minus = "-"
 
defineCommands:
  Plus = "+"; "Adds two numbers":
    stack.execute(a + b)
  Minus = "-"; "Subtract two numbers":
    stack.execute(b - a)
```

## 创建文档字符串数组

接下来是文档字符串数组。它的结构有更多的静态部分。请注意，上面大部分内容都是在循环中生成的，而这里只需要将`StrLit`节点复制到shell中：

```
ConstSection
  ConstDef
    Ident "docstrings"
    BracketExpr
      Ident "array"
      Ident "Commands"
      Ident "string"
    Bracket
      StrLit "Adds two numbers"
      StrLit "Subtract two numbers"
```

像处理枚举一样，创建一棵空树，然后在循环中填充，最后得到如下结果：

```nim
import macros
 
macro defineCommands(definitions: untyped): untyped =
  result = newstmtlist()
  echo "This is the input to our macro"
  echo definitions.treeRepr
  var
    enumDef = nnkTypeSection.newTree(
      nnkTypeDef.newTree(
        newIdentNode("Commands"),
        newEmptyNode(),
        nnkEnumTy.newTree(newEmptyNode())))
    docstrings =  nnkConstSection.newTree(
      nnkConstDef.newTree(
        newIdentNode("docstrings"),
        nnkBracketExpr.newTree(
          newIdentNode("array"),
          newIdentNode("Commands"),
          newIdentNode("string")
        ),
        nnkBracket.newTree()))
  for i in countup(0, definitions.len - 1, 2):
    let
      enumInfo = definitions[i]
      commandInfo = definitions[i+1]
    enumDef[0][2].add nnkEnumFieldDef.newtree(enumInfo[0], enumInfo[1])
    docstrings[0][2].add commandInfo[0]
  echo "This is the enum we create"
  echo enumDef.treeRepr
  echo "This is the docstring array we create"
  echo docstrings.treeRepr
 
static: echo "This is what we want"
dumpTree:
  const docstrings: array[Commands, string] = [
    "Adds two numbers",
    "Subtract two numbers"
  ]
 
defineCommands:
  Plus = "+"; "Adds two numbers":
    stack.execute(a + b)
  Minus = "-"; "Subtract two numbers":
    stack.execute(b - a)
```

## 创建我们的案例切换模板

同样，我们现在可以创建“case switch”，首先创建一个case语句的空树，然后在循环中添加`OfBranch`节点。

```
CaseStmt
  Call
    BracketExpr
      Ident "parseEnum"
      Ident "Commands"
    Ident "command"
  OfBranch
    Ident "Plus"
    StmtList
      <nodes of the actual code comes here>
```

最后我们得出了这样的结果：

```nim
import macros
 
macro defineCommands(definitions: untyped): untyped =
  result = newstmtlist()
  echo "This is the input to our macro"
  echo definitions.treeRepr
  var
    enumDef = nnkTypeSection.newTree(
      nnkTypeDef.newTree(
        newIdentNode("Commands"),
        newEmptyNode(),
        nnkEnumTy.newTree(newEmptyNode())))
    docstrings =  nnkConstSection.newTree(
      nnkConstDef.newTree(
        newIdentNode("docstrings"),
        nnkBracketExpr.newTree(
          newIdentNode("array"),
          newIdentNode("Commands"),
          newIdentNode("string")
        ),
        nnkBracket.newTree()))
    caseSwitch =  nnkCaseStmt.newTree(
      nnkCall.newTree(
        nnkBracketExpr.newTree(
          newIdentNode("parseEnum"),
          newIdentNode("Commands")
        ),
        newIdentNode("command")))
  for i in countup(0, definitions.len - 1, 2):
    let
      enumInfo = definitions[i]
      commandInfo = definitions[i+1]
    enumDef[0][2].add nnkEnumFieldDef.newtree(enumInfo[0], enumInfo[1])
    docstrings[0][2].add commandInfo[0]
    caseSwitch.add nnkOfBranch.newTree(
      enumInfo[0],
      commandInfo[1])
  echo "This is the enum we create"
  echo enumDef.treeRepr
  echo "This is the docstring array we create"
  echo docstrings.treeRepr
  echo "This is the case statement we create"
  echo caseSwitch.treeRepr
 
static: echo "This is what we want"
dumpTree:
  case parseEnum[Commands](command):
  of Plus: stack.execute(a + b)
 
defineCommands:
  Plus = "+"; "Adds two numbers":
    stack.execute(a + b)
  Minus = "-"; "Subtract two numbers":
    stack.execute(b - a)
```

## 将所有内容串联起来并生成正确的代码

到目前为止，我们只是专注于创建预期输出。但我们还需要考虑一些额外的因素。如果我们想多次调用宏，比如从不同的模块调用宏，该怎么办？还有我们说过的模板怎么办？因此，让我们稍微修改一下宏定义，增加一些参数，用来命名我们的代码所创建的内容。我们添加参数来命名枚举、文档数组和要创建的模板名称。然后，我们通过调用`quote`将所有内容连接在一起，并添加我们的部分和模板。然后再添加所有定义和主循环，最后我们的程序看起来就像这样了：

```nim
import math, strutils
import macros
 
macro defineCommands(enumName, docarrayName, runnerName,
  definitions: untyped): untyped =
  var
    enumDef = nnkTypeSection.newTree(
      nnkTypeDef.newTree(
        enumName,
        newEmptyNode(),
        nnkEnumTy.newTree(newEmptyNode())))
    docstrings =  nnkConstSection.newTree(
      nnkConstDef.newTree(
        docarrayName,
        nnkBracketExpr.newTree(
          newIdentNode("array"),
          enumName,
          newIdentNode("string")
        ),
        nnkBracket.newTree()))
    templateArgument = newIdentNode("command")
    caseSwitch =  nnkCaseStmt.newTree(
      nnkCall.newTree(
        nnkBracketExpr.newTree(
          newIdentNode("parseEnum"),
          enumName
        ),
        templateArgument))
  for i in countup(0, definitions.len - 1, 2):
    let
      enumInfo = definitions[i]
      commandInfo = definitions[i+1]
    enumDef[0][2].add nnkEnumFieldDef.newtree(enumInfo[0], enumInfo[1])
    docstrings[0][2].add commandInfo[0]
    caseSwitch.add nnkOfBranch.newTree(
      enumInfo[0],
      commandInfo[1])
  result = quote do:
    `enumDef`
    `docstrings`
    template `runnerName`(`templateArgument`: untyped): untyped =
      `caseSwitch`
  echo result.repr
 
# First create a simple "stack" implementation
var stack: seq[float]
 
proc push[T](stack: var seq[T], value: T) =
  stack.add value
 
proc pop[T](stack: var seq[T]): T =
  result = stack[^1]
  stack.setLen(stack.len - 1)
 
# Convenience template to execute an operation over two operands from the stack
template execute[T](stack: var seq[T], operation: untyped): untyped {.dirty.} =
  let
    a = stack[^1]
    b = stack[^2]
  stack.setLen(stack.len - 2)
  stack.push(operation)
 
# Then define all our commands using our macro
defineCommands(Commands, docstrings, runCommand):
  Plus = "+"; "Adds two numbers":
    stack.execute(a + b)
  Minus = "-"; "Subtract two numbers":
    stack.execute(b - a)
  Multiply = "*"; "Multiplies two numbers":
    stack.execute(a * b)
  Divide = "/"; "Divides two numbers":
    stack.execute(b / a)
  Pop = "pop"; "Pops a number off the stack":
    discard stack.pop
  StackSwap = "swap"; "Swaps the two bottom elements on the stack":
    let
      a = stack[^1]
      b = stack[^2]
    stack[^1] = b
    stack[^2] = a
  StackRotate = "rot"; "Rotates the stack one level":
    stack.insert(stack.pop, 0)
  Help = "help"; "Lists all the commands with documentation":
    echo "Commands:"
    for command in Commands:
      echo "\t", command, "\t", docstrings[command]
  Exit = "exit"; "Exits the program":
    quit 0
 
# Program main loop, read input from stdin, run our template to parse the
# command and run the corresponding operation. if that fails try to push it as
# a number. Print out our "stack" for every iteration of the loop
while true:
  for command in stdin.readLine.split(" "):
    try:
      runCommand(command)
    except:
      stack.push parseFloat(command)
    echo stack, " - ", command
```

## 结束语

从最终结果中我们可以看到，我们的操作定义现在被集中到了一处。我们在同一个地方创建了枚举、docstring数组和case开关。由于所有这些都在AST层面上被简单地重写为有效的Nim代码，因此它仍然会评估为与我们开始时相同的代码，这意味着它仍然具有原始版本的所有类型安全性和速度。由于我们的宏还可以接受参数，因此我们可以轻松创建多个不同类别的命令，并将它们放在不同的模块中，或在主循环中以不同的方式处理它们。现在你可能会说，宏本身只是需要维护的更多代码，这没错。但在我看来，维护一段做简单转换的代码和另一段做高级程序逻辑的代码要比维护原来的意大利面更容易。特别是如果我们考虑到，在我们将这些代码分割成不同模块后，只需更改宏，就能轻松改变每个模块的格式。因此，我希望这能帮助说明，像元编程这样的强大工具是如何提高代码的可读性和可维护性的，而不是成为这两方面的障碍。