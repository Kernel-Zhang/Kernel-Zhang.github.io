---
title: Nim官方教程第一部分
author: kernel
date: 2023-11-06 12:34:00 +0800
categories: [Nim语言, 官方教程]
tags: [Nim语言, Nim官方教程, Nim入门]
---

| 条目        | 内容说明          |
| ----------- | ----------------- |
| 作者：      | 安德烈亚斯-鲁普夫 |
| 版本：      | 2.0.0             |
| 翻译：      | Kernel Zhang      |

## [导言](https://nim-lang.org/docs/tut1.html#introduction)

> "人毕竟是视觉动物--我希望看到美好的事物。"。

本文档是编程语言Nim的教程。

本教程假设您熟悉变量、类型或语句等基本编程概念。

这里还有其他一些学习Nim的资源：

-   [Nim基础教程](https://narimiran.github.io/nim-basics/)：对上述概念的基本介绍（[已翻译](../Nim-basic/)）
-   [5分钟学会Nim](https://learnxinyminutes.com/docs/nim/)：5分钟快速入门Nim
-   [Nim手册](https://nim-lang.org/docs/manual.html)： 里面有更多的高级语言功能示例

本教程中的所有代码示例以及Nim文档中的其他示例都遵循[Nim风格指南](https://nim-lang.org/docs/nep1.html)。

## [第一个程序](https://nim-lang.org/docs/tut1.html#the-first-program)

我们从一个修改过的“hello world”程序开始参观：

```nim
# 这是一条注释
echo "What's your name? "
var name: string = readLine(stdin)
echo "Hi, ", name, "!"
```

将代码保存到文件“greetings.nim”中，现在编译并运行它：

```shell
nim compile --run greetings.nim
```

使用`--run`[参数](https://nim-lang.org/docs/nimc.html#compiler-usage-commandminusline-switches)，Nim会在编译后自动执行文件。您可以在文件名后添加命令行参数，为程序提供命令行参数：

```shell
nim compile --run greetings.nim arg1 arg2
```

常用命令和参数都有缩写，因此也可以使用：
```shell
nim c -r greetings.nim
```

这是**调试版本**。要编译发布版本，请使用：

```
nim c -d:release greetings.nim
```

默认情况下，Nim编译器会生成大量运行时检查，以方便您进行调试。使用“-d:release”参数，某些[检查将被关闭，优化将被开启](https://nim-lang.org/docs/nimc.html#compiler-usage-compileminustime-symbols)。

对于基准测试或生产代码，请使用“-d:release”参数。要与C语言等不安全语言进行性能比较，请使用“-d:danger”参数，以便获得有意义的可比结果。否则，Nim可能会因为C语言**不具备**的检查而受到影响。

虽然程序的作用显而易见，但我还是要解释一下语法：程序启动时，未缩进的语句将被执行。缩进是Nim对语句进行分组的方式。缩进仅使用**空格**，*不允许使用制表符*。

字符串文字用双引号括起来。var语句声明了一个名为name的字符串类型的新变量，其值为过程（procedure，相当于C语言中的函数，为了做区别后面我都会翻译为过程）[readLine](https://nim-lang.org/docs/syncio.html#readLine,File)返回的值。由于编译器知道[readLine](https://nim-lang.org/docs/syncio.html#readLine,File)返回的是字符串，因此可以在声明中省略类型（这称为局部类型推断）。因此，可以简写成这样：

```nim
var name = readLine(stdin)
```

请注意，这基本上是Nim中**唯一存在**的类型推论形式：它是简洁性和可读性之间的一个很好的折中。

“hello world”程序包含几个编译器已经知道的标识符：echo、[readLine](https://nim-lang.org/docs/syncio.html#readLine,File) 等。这些内置标识符在[system](https://nim-lang.org/docs/system.html)模块中声明，任何其他模块都会隐式导入该模块，所以我们无需显式的使用`import system`导入。

## [基本语法](https://nim-lang.org/docs/tut1.html#lexical-elements)

让我们更详细地了解一下Nim的基本语法：与其他编程语言一样，Nim由字面量、标识符、关键字、注释、运算符和其他标点符号组成。

### [字符串和字符字面量](https://nim-lang.org/docs/tut1.html#lexical-elements-string-and-character-literals)

字符串用双引号括起来，字符用单引号括起来。特殊字符用`\`转义：`\n`表示换行符，`\t`表示制表符等。也有原始字符串字面量：

```nim
r"C:\program files\nim"
```

在原始文字中，反斜杠不是转义字符。

第三种也是最后一种字符串字面量的写法是*长字符串字面量*。它们用三个引号来书写：`"""...""""`；它们可以跨越多行，而且`\`也不是转义字符。例如，它们在嵌入HTML代码模板时非常有用。

### [注释](https://nim-lang.org/docs/tut1.html#lexical-elements-comments)

注释从字符串或字符字面以外的任何地方开始，以散列字符`#`开始。文档注释以`##`开头：

```nim
# A comment.

var myVariable: int ## a documentation comment
```

文档注释是标记，它们只允许出现在输入文件的特定位置，因为它们属于语法树！这项功能可以简化文档生成器。

多行注释以`#[`开始，以`]#`结束。 多行注释也可以嵌套。

```nim
#[
You can have any Nim code text commented
out inside this with no indentation restrictions.
      yes("May I ask a pointless question?")
  #[
     Note: these can be nested!!
  ]#
]#
```

### [数字](https://nim-lang.org/docs/tut1.html#lexical-elements-numbers)

数字字面的写法与大多数其他语言相同。为了提高可读性，允许使用下划线：1\_000\_000（一百万）。包含点（或 "e "或 "E"）的数字是浮点字面量：1.0e9（十亿）。十六进制字面量的前缀是0x，二进制字面量的前缀是 0b，八进制字面量的前缀是0o。前导零不会产生八进制（意思是`0123`会当成10进制处理，和`123`是相同的）。

## [var语句](https://nim-lang.org/docs/tut1.html#the-var-statement)

var语句声明一个新的局部变量或全局变量：
```nim
var x, y: int # declares x and y to have the type `int`
```

可以在var关键字后使用缩进来列出整段变量：
```nim
var
  x, y: int
  # a comment can occur here too
  a, b, c: string
```

## [常数](https://nim-lang.org/docs/tut1.html#constants)

常量是与一个值绑定的符号。常量的值不能改变。编译器必须能在编译时计算常量声明中的表达式：
```nim
const x = "abc" # the constant x contains the string "abc"
```


可以在const关键字后使用缩进来列出整段常量：
```nim
const
  x = 1
  # a comment can occur here too
  y = 2
  z = y + 5 # computations are possible
```

## [let语句](https://nim-lang.org/docs/tut1.html#the-let-statement)

let语句的作用与var语句类似，但声明的符号是*单赋值*变量：初始化后，其值不会改变：
```nim
let x = "abc" # introduces a new variable `x` and binds a value to it
x = "xyz"     # Illegal: assignment to `x`
```

let和const的区别在于：let引入的变量不能重新赋值，而const表示 "执行编译时计算并将其放入数据部分"：
```nim
const input = readLine(stdin) # Error: constant expression expected
```

```nim
let input = readLine(stdin)   # works
```

## [赋值语句](https://nim-lang.org/docs/tut1.html#the-assignment-statement)

赋值语句将一个新值赋给一个变量或更一般的存储位置：
```nim
var x = "abc" # introduces a new variable `x` and assigns a value to it
x = "xyz"     # assigns a new value to `x`
```

\=是*赋值运算符*。赋值操作符可以重载。您可以用一条赋值语句声明多个变量，所有变量都将具有**相同**的值：

```nim
var x, y = 3  # assigns 3 to the variables `x` and `y`
echo "x ", x  # outputs "x 3"
echo "y ", y  # outputs "y 3"
x = 42        # changes `x` to 42 without changing `y`
echo "x ", x  # outputs "x 42"
echo "y ", y  # outputs "y 3"
```

## [控制流语句](https://nim-lang.org/docs/tut1.html#control-flow-statements)

最开始的`hello`程序由3个顺序执行的语句组成。除非是最原始的程序，我们还需要分支和循环。

### [if语句](https://nim-lang.org/docs/tut1.html#control-flow-statements-if-statement)

if语句是分支控制流的一种方法：
```nim
let name = readLine(stdin)
if name == "":
  echo "Poor soul, you lost your name?"
elif name == "name":
  echo "Very funny, your name is name."
else:
  echo "Hi, ", name, "!"
```

`elif`部分可以有零个或多个，`else`部分是可选的。关键字`elif`是`else if`的缩写，用于避免过度缩进。（`""`是空字符串，不包含任何字符）。

### [case语句](https://nim-lang.org/docs/tut1.html#control-flow-statements-case-statement)

另一种分支方式由case语句提供。一个case语句允许多个分支：
```nim
let name = readLine(stdin)
case name
of "":
  echo "Poor soul, you lost your name?"
of "name":
  echo "Very funny, your name is name."
of "Dave", "Frank":
  echo "Cool name!"
else:
  echo "Hi, ", name, "!"
```

可以看出，对于一个分支，也允许使用以逗号分隔的值列表。

case语句可以处理整数、其他序数类型（后面会介绍什么是序数）和字符串。对于整数或其他序数类型，也可以使用值范围：

```nim
# this statement will be explained later:
from std/strutils import parseInt

echo "A number please: "
let n = parseInt(readLine(stdin))
case n
of 0..2, 4..7: echo "The number is in the set: {0, 1, 2, 4, 5, 6, 7}"
of 3, 8: echo "The number is 3 or 8"
```

然而，上述代码**无法编译**：原因是必须涵盖n可能包含的所有值，但代码只处理0...8的值。由于要列出所有其他可能的整数并不现实（不过由于使用了范围符号，还是有可能的），我们可以通过告诉编译器对于所有其他值都不做任何处理来解决这个问题：
```nim
case n
of 0..2, 4..7: echo "The number is in the set: {0, 1, 2, 4, 5, 6, 7}"
of 3, 8: echo "The number is 3 or 8"
else: discard
```

空的[discard语句](https://nim-lang.org/docs/tut1.html#procedures-discard-statement)是*什么都不做的*语句。编译器知道，带有`else`部分的`case`语句不会失败，因此错误也就消失了。请注意，不可能涵盖所有可能的字符串值：这就是字符串作为`case`条件总是需要`else`分支的原因。

一般情况下，case 语句用于子范围类型或枚举类型，因为编译器会检查是否涵盖了任何可能的值。

### [while语句](https://nim-lang.org/docs/tut1.html#control-flow-statements-while-statement)

while 语句是一种简单的循环结构：

```nim
echo "What's your name? "
var name = readLine(stdin)
while name == "":
  echo "Please tell me your name: "
  name = readLine(stdin) # no `var`, because we do not declare a new variable here
```

只要用户不输入任何内容（只按“RETURN”键），示例就会使用`while`循环不断询问用户的姓名。

### [for语句](https://nim-lang.org/docs/tut1.html#control-flow-statements-for-statement)

for语句是一种对*迭代器*提供的任何元素进行循环的结构。本示例使用内置的[countup](https://nim-lang.org/docs/system.html#countup.i,T,T,Positive)迭代器：

```nim
echo "Counting to ten: "
for i in countup(1, 10):
  echo i
# --> Outputs 1 2 3 4 5 6 7 8 9 10 on different lines
```

for循环隐式声明了变量i，其类型为int，因为[countup](https://nim-lang.org/docs/system.html#countup.i,T,T,Positive)返回的就是int类型。每个值都进行了回显处理。这段代码也是如此：

```nim
echo "Counting to 10: "
var i = 1
while i <= 10:
  echo i
  inc i # increment i by 1
# --> Outputs 1 2 3 4 5 6 7 8 9 10 on different lines
```

由于程序中经常出现“向上计数”的情况，因此Nim也有一个[..](https://nim-lang.org/docs/system.html#...i,T,T)迭代器来做同样的事情：

```nim
for i in 1 .. 10:
  ...
```

倒计时也很容易实现（但不太常用）：

```nim
echo "Counting down from 10 to 1: "
for i in countdown(10, 1):
  echo i
# --> Outputs 10 9 8 7 6 5 4 3 2 1 on different lines
```

零指数计数法有两个快捷方式`..<`和`.. ^1`（[后向索引运算符](https://nim-lang.org/docs/system.html#^.t%2Cint)）来简化计数，使其小于较高索引：

```nim
for i in 0 ..< 10:
    # the same as 0 .. 9
```

或：
```nim
var s = "some string"
for i in 0 ..< s.len:
  ...
```

或：
```nim
var s = "some string"
for idx, c in s[0 .. ^1]:
  ... # ^1 is the last element, ^2 would be one before it, and so on
```

其他对集合（如数组和序列）有用的迭代器有：

-   `items`和`mitems`，分别提供不可变和可变元素，以及
-   `pairs`和`mpairs`，分别提供元素和索引号（不可变和可变）。
    
```nim
for index, item in ["a","b"].pairs:
  echo item, " at index ", index
# => a at index 0
# => b at index 1
```

### [作用域和block语句](https://nim-lang.org/docs/tut1.html#control-flow-statements-scopes-and-the-block-statement)

控制流语句有一个尚未涉及的特性：它们会打开一个新的作用域。这意味着在下面的示例中，x在循环之外是不可访问的：

```nim
while false:
  var x = "hi"
echo x # does not work
```

while (for) 语句引入了一个隐式代码块。标识符只在其声明的代码块中可见。块语句可用于显式打开一个新块：

```nim
block myblock:
  var x = "hi"
echo x # does not work either
```

块的*标签*（示例中为myblock）是可选的。

### [break语句](https://nim-lang.org/docs/tut1.html#control-flow-statements-break-statement)

使用break语句可使程序块过早退出。break语句可以离开while、for或代码块语句。除非给出了代码块的标签，否则它会离开最内层的结构体：

```nim
block myblock:
  echo "entering block"
  while true:
    echo "looping"
    break # leaves the loop, but not the block
  echo "still in block"
echo "outside the block"

block myblock2:
  echo "entering block"
  while true:
    echo "looping"
    break myblock2 # leaves the block (and the loop)
  echo "still in block" # it won't be printed
echo "outside the block"
```

### [continue语句](https://nim-lang.org/docs/tut1.html#control-flow-statements-continue-statement)

与许多其他编程语言一样，continue语句会立即开始下一次迭代：

```nim
for i in 1 .. 5:
  if i <= 3: continue
  echo i # will only print 4 and 5
```

### [when语句](https://nim-lang.org/docs/tut1.html#control-flow-statements-when-statement)

例如：

```nim
when system.hostOS == "windows":
  echo "running on Windows!"
elif system.hostOS == "linux":
  echo "running on Linux!"
elif system.hostOS == "macosx":
  echo "running on Mac OS X!"
else:
  echo "unknown operating system"
```

when语句与if语句几乎相同，但有以下区别：

-   每个条件都必须是常量表达式，因为编译器会对其进行计算。
-   分支中的语句不会打开新的作用域。
-   编译器会检查语义，并*只*为属于第一个求值为真的条件的语句生成代码。

when语句可用于编写特定平台代码，类似于C编程语言中的`#ifdef`结构。

## [语句和缩进](https://nim-lang.org/docs/tut1.html#statements-and-indentation)

在介绍了基本的控制流语句后，我们再来看看Nim的缩进规则。

在Nim中，*简单语句*和*复杂语句*是有区别的。*简单语句*不能包含其他语句：赋值、过程调用或返回语句都属于简单语句。*复杂语句*如if、when、for、while可以包含其他语句。为避免歧义，复杂语句必须始终缩进，而单个简单语句则不需要：

```nim
# no indentation needed for single-assignment statement:
if x: x = false

# indentation needed for nested if statement:
if x:
  if y:
    y = false
  else:
    y = true

# indentation needed, because two statements follow the condition:
if x:
  x = false
  y = false
```

*表达式*是语句的一部分，通常会产生一个值。if语句中的条件就是表达式的一个例子。表达式可以在某些地方缩进，以提高可读性：

```nim
if thisIsaLongCondition() and
    thisIsAnotherLongCondition(1,
       2, 3, 4):
  x = true
```

根据经验，表达式中的缩进可以出现在运算符、开放括号和逗号之后。

通过括号和分号`(;)`，您可以使用只允许使用表达式的语句：

```nim
# computes fac(4) at compile time:
const fac4 = (var x = 1; for i in 1..4: x *= i; x)
```

## [过程](https://nim-lang.org/docs/tut1.html#procedures)

要定义示例中的[echo](https://nim-lang.org/docs/system.html#echo,varargs[typed,])和[readLine](https://nim-lang.org/docs/syncio.html#readLine,File)等新命令，需要使用*过程*的概念。在其他语言中，它们可能被称为*方法*或*函数*，但 Nim[区分了这些概念](https://nim-lang.org/docs/tut1.html#procedures-funcs-and-methods)。在Nim中，新程序是用`proc`关键字定义的：

```nim
proc yes(question: string): bool =
  echo question, " (y/n)"
  while true:
    case readLine(stdin)
    of "y", "Y", "yes", "Yes": return true
    of "n", "N", "no", "No": return false
    else: echo "Please be clear: yes or no"

if yes("Should I delete all your important files?"):
  echo "I'm sorry Dave, I'm afraid I can't do that."
else:
  echo "I think you know what the problem is just as well as I do."
```

这个示例显示了一个名为yes的过程，该过程向用户提问，如果用户回答 “是”（或类似答案），则返回`true`；如果用户回答 “否”（或类似答案），则返回`false`。返回语句会立即退出过程（因此也退出`while`循环）。`(question: string): bool`语法说明过程需要一个名为`question`的字符串类型参数，并返回一个`bool`类型的值。`bool`类型是内置的：`bool`的只有两种有效值`true`和`false`。`if`或`while` 语句中的条件必须是`bool`类型。

一些术语：在示例中，`question`被称为（形式）*参数*，`"Should I..."`被称为传递给该参数的*实参*。

### [result变量](https://nim-lang.org/docs/tut1.html#procedures-result-variable)

一个带返回值的过程会声明一个隐式`result`变量来表示返回值。`return`是`return result`的简称（译者：并不是返回值为空的意思）。如果过程的没有`return`语句，那么在过程结束时总是自动返回`result`值。

```nim
proc sumTillNegative(x: varargs[int]): int =
  for i in x:
    if i < 0:
      return
    result = result + i

echo sumTillNegative() # echoes 0
echo sumTillNegative(3, 4, 5) # echoes 12
echo sumTillNegative(3, 4 , -1 , 6) # echoes 7
```

函数开始时已经隐式声明了`result`变量，因此再次用`var result`声明结果变量时，同名的普通变量会隐藏结果变量。结果变量也已经用该类型的默认值进行了初始化。请注意，引用数据类型在过程开始时为nil，因此可能需要手动初始化。

过程如果没有返回语句，也没有使用特殊的结果变量，则返回最后一个表达式的值。例如，过程
```nim
proc helloWorld(): string =
  "Hello, World!"
```
返回字符串 "Hello, World!"。

### [参数](https://nim-lang.org/docs/tut1.html#procedures-parameters)

参数在过程主体中是不可变的。默认情况下，它们的值是不能改变的，因为这允许编译器以最有效的方式实现参数传递。如果需要在过程中使用可变变量，则必须在过程正文中使用var声明。对参数名进行隐藏处理是可行的，实际上也是一种习惯做法：

```nim
proc printSeq(s: seq, nprinted: int = -1) =
  var nprinted = if nprinted == -1: s.len else: min(nprinted, s.len)
  for i in 0 ..< nprinted:
    echo s[i]
```

如果过程需要为调用者修改参数，可以使用var参数：

```nim
proc divmod(a, b: int; res, remainder: var int) =
  res = a div b        # integer division
  remainder = a mod b  # integer modulo operation

var
  x, y: int
divmod(8, 5, x, y) # modifies x and y
echo x
echo y
```

在示例中，res和remainder是`var`参数。过程可以修改`var`参数，并且调用者可以看到参数的变化。请注意，上述示例最好使用元组作为返回值，而不是使用`var`参数。

### [discard语句](https://nim-lang.org/docs/tut1.html#procedures-discard-statement)

如果要调用一个只为其副作用返回值而忽略其返回值的过程，**必须**使用丢弃语句。Nim不允许隐式忽略返回值：

```nim
discard yes("May I ask a pointless question?")
```

如果被调用的proc/iterator已用discardable语义声明，则返回值可以隐式忽略：

```nim
proc p(x, y: int): int {.discardable.} =
  return x + y

p(3, 4) # now valid
```

### [具名参数](https://nim-lang.org/docs/tut1.html#procedures-named-arguments)

通常情况下，一个过程有许多参数，而参数出现的顺序并不明确。对于构造复杂数据类型的过程来说尤其如此。因此，可以对过程的参数进行命名，以便明确实参属于哪个形参：

```nim
proc createWindow(x, y, width, height: int; title: string;
                  show: bool): Window =

...

var w = createWindow(show = true, title = "My Application",
                     x = 0, y = 0, height = 600, width = 800)
```

既然我们使用命名参数来调用createWindow，参数顺序就不再重要了。将具名参数与有序参数混合使用也是可行的，但可读性不高：

```nim
var w = createWindow(0, 0, title = "My Application",
                     height = 600, width = 800, true)
```

编译器会检查每个参数是否正好接收一个实参。

### [默认值](https://nim-lang.org/docs/tut1.html#procedures-default-values)

为了使createWindow过程更易于使用，它应该提供默认值；如果调用者没有指定参数，这些值将被用作参数：

```nim
proc createWindow(x = 0, y = 0, width = 500, height = 700,
                  title = "unknown",
                  show = true): Window =
   

var w = createWindow(title = "My Application", height = 600, width = 800)
```

现在，调用createWindow时只需设置与默认值不同的值。

请注意，类型推断适用于具有默认值的参数；例如，不需要写`title: string = "unknown"`。

### [过程重载](https://nim-lang.org/docs/tut1.html#procedures-overloaded-procedures)

Nim提供了与C++类似的重载过程的能力：

```nim
proc toString(x: int): string =
  result =
    if x < 0: "negative"
    elif x > 0: "positive"
    else: "zero"

proc toString(x: bool): string =
  result =
    if x: "yep"
    else: "nope"

assert toString(13) == "positive" # calls the toString(x: int) proc
assert toString(true) == "yep"    # calls the toString(x: bool) proc
```

编译器会为`toString`调用选择最合适的过程（在Nim中`toString`过程一般为`$`运算符）。这里不讨论重载解析算法的具体工作原理，详情请参见Nim手册。不明确的调用将作为错误报告。

### [运算符](https://nim-lang.org/docs/tut1.html#procedures-operators)

Nim标准库大量使用重载，原因之一是每个运算符（如`+`）都是一个重载的过程。解析器允许您使用*中缀*运算符（`a + b`）或*前缀*运算符（`+ a`）。中缀运算符总是接收两个参数，前缀运算符总是接收一个参数。（后缀运算符是不可能的，因为这会产生歧义：`a @ @ b`是指`(a)@ (@b)`还是`(a@) @ (b)`？它总是表示`(a) @ (@b)`，因为Nim中没有后缀运算符）。

除了一些内置关键字运算符（如and、or、not）外，运算符总是由这些字符组成：`+ - * \ / < > = @ $ ~ & % ! ? ^ . |`

允许用户定义操作符。没有什么能阻止你定义自己的`@!?+~`操作符，但这样做可能会降低可读性。

运算符的优先级由其第一个字符决定。详情请参见[手册](https://nim-lang.org/docs/manual.html#syntax-precedence)。

要定义新的运算符，请用“\`”（键盘上，数字键“1”前面的字符）括住运算符：

```nim
proc `$` (x: myDataType): string = ...
# now the $ operator also works with myDataType, overloading resolution
# ensures that $ works for built-in types just like before
```

“\`”符号也可以用来调用操作符，就像调用其他过程一样：

```nim
if `==`( `+`(3, 4), 7): echo "true"
```

### [前置声明](https://nim-lang.org/docs/tut1.html#procedures-forward-declarations)

每个变量、过程等在使用前都需要声明。（这样做的原因是，在像Nim这样广泛支持元编程的语言中，要避免这种需要并非易事）。尤其对于相互递归的过程，则必须这样做：

```nim
# forward declaration:
proc even(n: int): bool
```

```nim
proc odd(n: int): bool =
  assert(n >= 0) # makes sure we don't run into negative recursion
  if n == 0: false
  else:
    n == 1 or even(n-1)

proc even(n: int): bool =
  assert(n >= 0) # makes sure we don't run into negative recursion
  if n == 1: false
  else:
    n == 0 or odd(n-1)
```

奇数依赖于偶数，反之亦然。因此，编译器需要在完全定义偶数之前将其引入。这种前置声明的语法很简单：省略`=`号和过程的主体即可。assert只是添加边界条件，稍后将在[模块](https://nim-lang.org/docs/tut1.html#modules)部分介绍。

以后的语言版本将弱化对正向声明的要求。

该示例还表明，proc的主体可以由一个表达式组成，然后隐式返回表达式的值。

### [函数和方法](https://nim-lang.org/docs/tut1.html#procedures-funcs-and-methods)

正如介绍中提到的，Nim区分了过程、函数和方法，分别由proc、func 和method关键字定义。在某些方面，Nim的定义比其他语言更保守。

函数更接近于纯数学函数的概念，如果你曾经学习过函数式编程，可能会对它比较熟悉。从本质上讲，函数是带有额外限制的过程：不能访问全局状态（const除外），不能产生副作用。func关键字基本上是带有`{.noSideEffects.}`标记的过程别名。不过，函数仍可更改其可变参数，即标记为var的参数以及任何ref对象。

与过程不同，方法是动态派发的。这听起来有点复杂，但它是一个与继承和面向对象编程密切相关的概念。如果重载一个过程（两个名称相同但类型不同或参数集不同的过程被称为重载），那么使用哪个过程将在编译时确定。另一方面，方法依赖于继承自RootObj的对象。[本教程的第二部分](../nim-tutorial-part-2/)将更深入地介绍这一点。

## [迭代器](https://nim-lang.org/docs/tut1.html#iterators)

让我们回到简单的计数例子：

```nim
echo "Counting to ten: "
for i in countup(1, 10):
  echo i
```

[countup](https://nim-lang.org/docs/system.html#countup.i,T,T,Positive)过程能否写成这样？让我们试试看：

```nim
proc countup(a, b: int): int =
  var res = a
  while res <= b:
    return res
    inc(res)
```

然而，这样做是行不通的。问题在于所需要的过程不仅要返回迭代值，而且要在迭代值返回后**能够继续**。这种返回和继续被称为`yield`语句。现在唯一要做的就是用`iterator`替换`proc`关键字，这就是我们的第一个迭代器：

```nim
iterator countup(a, b: int): int =
  var res = a
  while res <= b:
    yield res
    inc(res)
```

迭代器看起来与过程非常相似，但有几个重要的区别：

-   迭代器只能在for循环中调用。
-   迭代器不能包含return语句（proc不能包含yield语句）。
-   迭代器没有隐式结果变量。
-   迭代器不支持递归。
-   迭代器不能前置声明，因为编译器必须能够内联迭代器。（这一限制将在未来版本的编译器中消失）。

不过，你也可以使用闭包迭代器来获得一组不同的限制。详情请参阅[一等迭代器](https://nim-lang.org/docs/manual.html#iterators-and-the-for-statement-firstminusclass-iterators)。迭代器可以与过程具有相同的名称和参数，因为从本质上讲，迭代器有自己的命名空间。因此，通常会将迭代器封装在同名的过程中，这些过程会累加迭代器的结果，并以序列的形式返回，如[strutils模块](https://nim-lang.org/docs/strutils.html)中的split。

## [基本类型](https://nim-lang.org/docs/tut1.html#basic-types)

本节将详细介绍基本内置类型及其可用操作。

### [布尔类型](https://nim-lang.org/docs/tut1.html#basic-types-booleans)

Nim的布尔类型称为bool，由两个预定义值true和false组成。while、if、elif 和when语句中的条件必须是 bool类型。

运算符not、 and、 or、 xor、<、 <=、\>、 \>=、 !\=、 \==是为bool类型定义的。and和or运算符执行短路求值。例如

```nim
while p != nil and p.name != "xyz":
  # p.name is not evaluated if p == nil
  p = p.next
```

### [字符类型](https://nim-lang.org/docs/tut1.html#basic-types-characters)

*字符类型*称为char。它的大小总是一个字节，因此不能表示大多数UTF-8字符，但可以表示组成多字节 UTF-8字符的一个字节。这样做的原因是为了提高效率：在绝大多数使用情况下，生成的程序仍能正确处理 UTF-8，因为UTF-8是专门为此而设计的。字符字面量用单引号括起来。

字符可以用\==、<、 <=、\>、\> \= 操作符进行比较。操作符$可以将字符转换为字符串。字符不能与整数混合；要获取字符的序号值，请使用`ord`过程。从整数到字符的转换使用`chr`过程。

### [字符串类型](https://nim-lang.org/docs/tut1.html#basic-types-strings)

字符串变量是**可变的**，因此对字符串进行追加是可能的，而且相当高效。Nim中的字符串都是以0结尾，并有一个长度字段。字符串的长度可以通过内置的`len`过程获取；长度从不计算终止零。访问终止零是一个错误，它的存在只是为了让Nim字符串能在不进行复制的情况下转换为cstring。

字符串的赋值运算符会复制字符串。您可以使用&操作符连接字符串，使用add操作符追加字符串。

字符串比较使用的是它们的词典顺序。支持所有比较运算符。按照惯例，所有字符串都采用UTF-8编码，但并不强制执行。例如，从二进制文件读取字符串时，字符串只是字节序列。索引操作`s[i]`表示s的第i个*字符*，而不是第i个`unichar`。

字符串变量的初始化值为空字符串`""`。

### [整型](https://nim-lang.org/docs/tut1.html#basic-types-integers)

Nim内置以下整数类型：`int int8 int16 int32 int64 uint uint8 uint16 uint32 uint64`。

默认整数类型为int。整数字面可以使用*类型后缀*来指定非默认整数类型：

```nim
let
  x = 0     # x is of type `int`
  y = 0'i8  # y is of type `int8`
  z = 0'i32 # z is of type `int32`
  u = 0'u   # u is of type `uint`
```

整数通常用于计算内存中的对象，因此int的大小与指针相同。

常用运算符`+ - * div mod < <= == != > >=`是为整数定义的。`and or xor not`运算符也是为整数定义的，并提供*位操作*。向左位移使用`shl`操作符，向右位移使用`shr`操作符。位移运算符始终将其参数视为*无符号*参数。对于算术位移，可以使用普通的乘法或除法。

无符号运算都是回绕运算，不会导致上溢或下溢错误。

在使用不同整数类型的表达式中，会进行无损自动类型转换。但是，如果类型转换会导致信息丢失，则会引发RangeDefect（如果在编译时无法检测到错误）。

### [浮点类型](https://nim-lang.org/docs/tut1.html#basic-types-floats)

Nim内置这些浮点类型：float float32 float64。

默认浮点类型为浮点。在当前实现中，float始终是64位。

浮点运算字面量可以有一个*类型后缀*，用于指定非默认的浮点运算类型：

```nim
var
  x = 0.0      # x is of type `float`
  y = 0.0'f32  # y is of type `float32`
  z = 0.0'f64  # z is of type `float64`
```

常用运算符`+ - * / < <= == != > >=`是为浮点数定义的，遵循IEEE-754标准。

在表达式中使用不同类型的浮点类型时，会进行自动类型转换：将较小的类型转换为较大的类型。整数类型**不会**自动转换为浮点类型，反之亦然。请使用[toInt](https://nim-lang.org/docs/system.html#toInt,float)和[toFloat](https://nim-lang.org/docs/system.html#toFloat,int)程序进行转换。

### [类型转换](https://nim-lang.org/docs/tut1.html#basic-types-type-conversion)

数值类型之间的转换是通过将类型作为函数来执行的：

```nim
var
  x: int32 = 1.int32   # same as calling int32(1)
  y: int8  = int8('a') # 'a' == 97'i8
  z: float = 2.5       # int(2.5) rounds down to 2
  sum: int = int(x) + int(y) + int(z) # sum == 100
```

## [内部类型表示法](https://nim-lang.org/docs/tut1.html#internal-type-representation)

如前所述，内置的[`$`](https://nim-lang.org/docs/dollars.html)（stringify）操作符可以将任何基本类型转化为字符串，然后使用echo程序将字符串打印到控制台。不过，高级类型和你自己的自定义类型，在你为它们定义`$`操作符之前，是无法使用`$`操作符的。有时，你只想调试复杂类型的当前值，而无需编写`$`操作符。 这时可以使用[`repr`](https://nim-lang.org/docs/system.html#repr,T)过程，它适用于任何类型，甚至是带有循环的复杂数据图。下面的示例表明，即使是基本类型，`$`和`repr`输出之间也有区别：

```nim
var
  myBool = true
  myCharacter = 'n'
  myString = "nim"
  myInteger = 42
  myFloat = 3.14
echo myBool, ":", repr(myBool)
# --> true:true
echo myCharacter, ":", repr(myCharacter)
# --> n:'n'
echo myString, ":", repr(myString)
# --> nim:0x10fa8c050"nim"
echo myInteger, ":", repr(myInteger)
# --> 42:42
echo myFloat, ":", repr(myFloat)
# --> 3.14:3.14
```

## [高级类型](https://nim-lang.org/docs/tut1.html#advanced-types)

在Nim中，可以在类型语句中定义新类型：

```nim
type
  biggestInt = int64      # biggest integer type that is available
  biggestFloat = float64  # biggest float type that is available
```

枚举和对象类型只能在类型语句中定义。

### [枚举类型](https://nim-lang.org/docs/tut1.html#advanced-types-enumerations)

枚举类型的变量只能分配一个这个类型所支持的一个指定值，这些支持值是一组有序的符号。每个符号都在内部映射为一个整数值。第一个符号在运行时用0表示，第二个用1表示，依此类推。例如：

```nim
type
  Direction = enum
    north, east, south, west

var x = south     # x是一个Direction类型的枚举，并且它的值为south
echo x            # 显式变量x
```

所有比较运算符都可以与枚举类型一起使用。

为了避免歧义，可以将枚举值限定表示为“Direction.south”。

“$”运算符可以将任何枚举值转换为其名称，而ord过程则可以将其转换为底层的整数值。

为了更好地与其他编程语言进行交互，可以为枚举类型的符号分配明确的序号值。不过，序数值必须按**升序排列**。


### [序数类型](https://nim-lang.org/docs/tut1.html#advanced-types-ordinal-types)


枚举类型、整型、字符型和布尔型（以及其子范围）被称为序数类型。序数类型有许多特殊操作：

| 操作 | 注释 |
| --- | --- |
| ord(x) | 返回用于表示x 值的整数值 |
| inc(x) | 使x增 1 |
| inc(x, n) | 使x增加n；n为整数 |
| dec(x) | 将x减一 |
| dec(x, n) | 将x递减n；n为整数 |
| succ(x) | 返回x的后继 |
| succ(x, n) | 返回x的第n 个后继 |
| pred(x) | 返回x的前代 |
| pred(x, n) | 返回x的第n 个前代 |

[inc](https://nim-lang.org/docs/system.html#inc,T,int)、[dec](https://nim-lang.org/docs/system.html#dec,T,int)、[succ](https://nim-lang.org/docs/system.html#succ,T,int)和[pred](https://nim-lang.org/docs/system.html#pred,T,int)操作可能因引发RangeDefect或OverflowDefect异常而失败。（如果代码在编译时开启了适当的运行时检查）。

### [子范围](https://nim-lang.org/docs/tut1.html#advanced-types-subranges)

子范围类型是一个整数或枚举类型（基本类型）的数值范围。例如：

```nim
type
  MySubrange = range[0..5]
```

MySubrange是int的子范围，只能容纳 0 至 5 的值。向MySubrange类型的变量赋值任何其他值都会导致编译时或运行时错误。允许从基本类型向子范围类型赋值（反之亦然）。

系统模块将重要的[Natural](https://nim-lang.org/docs/system.html#Natural)类型定义为`range[0..high(int)]`（[high](https://nim-lang.org/docs/system.html#high,typedesc[T])返回最大值）。其他编程语言可能会建议使用无符号整数来表示自然数。这通常是**不明智的**：你会因为数字是无符号类型就采用了不希望的无符号运算（回绕运算）。Nim的Natural类型有助于避免这种常见的编程错误。

### [集合](https://nim-lang.org/docs/tut1.html#advanced-types-sets)

集合类型模拟了集合的数学概念。集合的基类型只能是一定大小的序数类型，即

-   int8-int16
-   uint8/byte-uint16
-   字符串
-   枚举
-   序数子范围类型，即`range[-10..10]`

或同等类型。在使用有符号整数字面构建集合时，集合的基本类型被定义为范围`0 .. DefaultSetElements-1`，其中DefaultSetElements目前总是2^8。集合基本类型的最大范围长度是MaxSetElements，目前总是2^16。具有更大范围长度的类型会被强制归入范围`0 .. MaxSetElements-1`。

原因是集合是作为高性能位矢量实现的。如果试图使用更大的类型声明集合，将会导致错误：

```nim
  var s: set[int64] # Error: set is too large; use `std/sets` for ordinal types
                    # with more than 2^16 elements
```

**注意**：Nim还提供了[哈希集合](https://nim-lang.org/docs/sets.html)（你需要用`import std/sets`来导入），它没有这样的限制。

集合可以通过集合构造函数构造：{}是空集。空集与任何具体的集合类型都是类型兼容的。构造函数还可以用来包含元素（以及元素的范围）：

```nim
type
  CharSet = set[char]
var
  x: CharSet
x = {'a'..'z', '0'..'9'} # This constructs a set that contains the
                         # letters from 'a' to 'z' and the digits
                         # from '0' to '9'
```

模块[`std/setutils`](https://nim-lang.org/docs/setutils.html)提供了一种从可迭代类型初始化集合的方法：

```
import std/setutils

let uniqueChars = myString.toSet
```

集合支持这些操作：

| 操作 | 意义 |
| --- | --- |
| A + B | 两个集合的并集 |
| A \* B | 两个集合的交集 |
| A \- B | 两个集合的差（A 不含 B 的元素） |
| A \== B | 集合相等 |
| A <= B | 子集关系（A 是 B 的子集或等于 B） |
| A < B | 严格子集关系（A 是 B 的真子集） |
| e in A 中 | 集合成员关系（A 包含元素 e） |
| e notin A中 | A 不包含元素 e |
| contains(A， e) | A 包含元素 e |
| card(A) | A 的基数（A 中元素的个数） |
| incl(A, elem) | 与A \= A + {elem}相同 |
| excl(A, elem) | 与A \= A \- {elem}相同 |

#### [位字段](https://nim-lang.org/docs/tut1.html#sets-bit-fields)

集合通常用于定义过程的*标志类型*。这比定义整数常量更简洁（且类型安全），因为整数常量必须一起进行位或操作。

枚举、集合和转换可以一起使用，例如

```nim
type
  MyFlag* {.size: sizeof(cint).} = enum
    A
    B
    C
    D
  MyFlags = set[MyFlag]

proc toNum(f: MyFlags): int = cast[cint](f)
proc toFlags(v: int): MyFlags = cast[MyFlags](v)

assert toNum({}) == 0
assert toNum({A}) == 1
assert toNum({D}) == 8
assert toNum({A, C}) == 5
assert toFlags(0) == {}
assert toFlags(7) == {A, B, C}
```

注意该集合如何将枚举值转化为2的幂次。

如果在C中使用枚举和集合，请使用截然不同的cint。

为实现与C语言的互操作性，请参阅[bitsize pragma](https://nim-lang.org/docs/manual.html#implementation-specific-pragmas-bitsize-pragma)。

### [数组](https://nim-lang.org/docs/tut1.html#advanced-types-arrays)

数组是一个简单的**固定长**度容器。数组中的每个元素都具有**相同**的类型。数组的索引类型可以是任何序数类型。

数组可以使用`[]`来构造：

```nim
type
  IntArray = array[0..5, int] # an array that is indexed with 0..5
var
  x: IntArray
x = [1, 2, 3, 4, 5, 6]
for i in low(x) .. high(x):
  echo x[i]
```

`x[i]`符号用于访问x中的第i个元素。数组访问总是经过边界检查（编译时或运行时）。这些检查可以通过语法或使用`--bound_checks:off`命令行参数来禁用。

数组和其他Nim类型一样，都是值类型。赋值操作符会复制整个数组的内容。

内置的[len](https://nim-lang.org/docs/system.html#len,TOpenArray)过程返回数组的长度。[low(a)](https://nim-lang.org/docs/system.html#low,openArray[T])返回数组a的最低有效索引，[high(a)](https://nim-lang.org/docs/system.html#high,openArray[T])返回数组a的最高有效索引。

```nim
type
  Direction = enum
    north, east, south, west
  BlinkLights = enum
    off, on, slowBlink, mediumBlink, fastBlink
  LevelSetting = array[north..west, BlinkLights]
var
  level: LevelSetting
level[north] = on
level[south] = slowBlink
level[east] = fastBlink
echo level        # --> [on, fastBlink, slowBlink, off]
echo low(level)   # --> north
echo len(level)   # --> 4
echo high(level)  # --> west
```

在其他语言中，嵌套数组（多维）的语法只需添加更多的括号，因为通常每个维度都被限制为与其他维度相同的索引类型。在Nim中，您可以拥有不同索引类型的不同维度，因此嵌套语法略有不同。在前面的示例中，层级被定义为由另一个枚举索引的枚举数组，在此基础上，我们可以添加以下几行来添加一个`LightTower`类型，该类型被细分为高度层级，通过其整数索引进行访问：

```nim
type
  LightTower = array[1..10, LevelSetting]
var
  tower: LightTower
tower[1][north] = slowBlink
tower[1][east] = mediumBlink
echo len(tower)     # --> 10
echo len(tower[1])  # --> 4
echo tower          # --> [slowBlink, mediumBlink, more output..
# The following lines don't compile due to type mismatch errors
#tower[north][east] = on
#tower[0][1] = on
```

注意内置的len过程如何只返回数组的第一维长度。 为了更好地说明LightTower的嵌套性质，另一种定义方法是省略前面对LevelSetting类型的定义，而直接将其写成第一个维度的类型：

```nim
type
  LightTower = array[1..10, array[north..west, BlinkLights]]
```

让数组从0开始是很常见的，因此有一种快捷语法可以指定从0到指定索引减1的范围：

```nim
type
  IntArray = array[0..5, int] # an array that is indexed with 0..5
  QuickArray = array[6, int]  # an array that is indexed with 0..5
var
  x: IntArray
  y: QuickArray
x = [1, 2, 3, 4, 5, 6]
y = x
for i in low(x) .. high(x):
  echo x[i], y[i]
```

### [序列](https://nim-lang.org/docs/tut1.html#advanced-types-sequences)

序列与数组类似，但长度是动态的，可能在运行时发生变化（如字符串）。由于序列是可调整大小的，因此总是在堆上分配并进行垃圾回收。

序列的索引总是从位置0开始的int。[len](https://nim-lang.org/docs/system.html#len,seq[T])、[low](https://nim-lang.org/docs/system.html#low,openArray[T])和[high](https://nim-lang.org/docs/system.html#high,openArray[T])操作也适用于序列。符号`x[i]`可用来访问x的第i个元素。

序列可以通过数组构造函数`[]`结合数组序列化操作符`@`来构造。为序列分配空间的另一种方法是调用内置的[newSeq](https://nim-lang.org/docs/system.html#newSeq)过程。

序列可以传递给一个openarray类型的参数。

示例：

```nim
var
  x: seq[int] # a reference to a sequence of integers
x = @[1, 2, 3, 4, 5, 6] # the @ turns the array into a sequence allocated on the heap
```

序列变量用`@[]`进行初始化。

与序列一起使用时，for语句可以使用一个或两个变量。使用单变量形式时，变量将保存序列提供的值。for语句循环遍历[system](https://nim-lang.org/docs/system.html)模块中[items()](https://nim-lang.org/docs/iterators.html#items.i,seq[T])迭代器的结果。 但如果使用双变量形式，第一个变量将保存索引位置，第二个变量将保存值。这里的for语句循环遍历[system](https://nim-lang.org/docs/system.html)模块中[pairs()](https://nim-lang.org/docs/iterators.html#pairs.i,seq[T])iterator的结果。示例：

```nim
for value in @[3, 4, 5]:
  echo value
# --> 3
# --> 4
# --> 5

for i, value in @[3, 4, 5]:
  echo "index: ", $i, ", value:", $value
# --> index: 0, value:3
# --> index: 1, value:4
# --> index: 2, value:5
```

### [开放数组](https://nim-lang.org/docs/tut1.html#advanced-types-open-arrays)

**注意**：`openArray`（开放数组）只能用于参数。

固定大小的数组往往不够灵活，程序应能处理不同大小的数组。openarray类型可以做到这一点。开放数组的索引总是从位置0开始的int。[len](https://nim-lang.org/docs/system.html#len,TOpenArray)、[low](https://nim-lang.org/docs/system.html#low,openArray[T])和[high](https://nim-lang.org/docs/system.html#high,openArray[T])操作也适用于开放数组。任何具有兼容基本类型的数组都可以传递给 openarray参数，索引类型并不重要。

```nim
var
  fruits:   seq[string]       # reference to a sequence of strings that is initialized with '@[]'
  capitals: array[3, string]  # array of strings with a fixed size

capitals = ["New York", "London", "Berlin"]   # array 'capitals' allows assignment of only three elements
fruits.add("Banana")          # sequence 'fruits' is dynamically expandable during runtime
fruits.add("Mango")

proc openArraySize(oa: openArray[string]): int =
  oa.len

assert openArraySize(fruits) == 2     # procedure accepts a sequence as parameter
assert openArraySize(capitals) == 3   # but also an array type
```

openarray类型不能嵌套：不支持多维openarray，因为很少需要这样做，而且无法高效地完成。

### [变长参数](https://nim-lang.org/docs/tut1.html#advanced-types-varargs)

varargs参数与openarray参数类似。不过，它也是实现向过程传递可变数量参数的一种方法。编译器会自动将参数列表转换为数组：

```nim
proc myWriteln(f: File, a: varargs[string]) =
  for s in items(a):
    write(f, s)
  write(f, "\n")

myWriteln(stdout, "abc", "def", "xyz")
# is transformed by the compiler to:
myWriteln(stdout, ["abc", "def", "xyz"])
```

只有当varargs参数是过程头部的最后一个参数时，才会进行这种转换。在这种情况下也可以进行类型转换：

```nim
proc myWriteln(f: File, a: varargs[string, `$`]) =
  for s in items(a):
    write(f, s)
  write(f, "\n")

myWriteln(stdout, 123, "abc", 4.0)
# is transformed by the compiler to:
myWriteln(stdout, [$123, $"abc", $4.0])
```

在本例中，[$](https://nim-lang.org/docs/dollars.html)应用于传递给参数a的任何参数。

### [切片](https://nim-lang.org/docs/tut1.html#advanced-types-slices)

切片的语法与子范围类型相似，但使用环境不同。切片只是一个Slice类型的对象，它包含两个边界（a和b）。切片本身用处不大，但其他集合类型定义的操作符可以接受Slice对象来定义范围。

```
var
  a = "Nim is a programming language"
  b = "Slices are useless."

echo a[7 .. 12] # --> 'a prog'
b[11 .. ^2] = "useful"
echo b # --> 'Slices are useful.'
```

在前面的例子中，切片用于修改字符串的一部分。切片的边界可以容纳其类型所支持的任何值，但使用片段对象的进程才是决定接受哪些值的关键。

要理解指定字符串、数组、序列等索引的不同方法，必须记住Nim使用基于零的索引。

因此，字符串b的长度为19，指定索引的两种不同方法是：

```
"Slices are useless."
 |          |     |
 0         11    17   using indices
^19        ^8    ^2   using ^ syntax
```

其中` b[0 .. ^1]`等同于`b[0 .. b.len-1]`和`b[0.. < b.len]`，可以看出`^1`提供了一种指定b.len-1的速记方法。参见[向后索引操作符](https://nim-lang.org/docs/system.html#^.t%2Cint)。

在上例中，由于字符串以句号结束，因此要获取字符串中"useless"的部分，并将其替换为"useful"的部分。

`b[11 .. ^2]`是"useless"部分，而`b[11 .. ^2] = "useful"`将"useless"部分替换为"useful"，得到结果 "Slices are useful."。

注1：另一种写法是`b[^8 .. ^2] = "useful"`或`b[11 .. b.len-2] = "useful"`或`b[11 ..< b.len-1] = "useful"`。

注2：由于^模板返回的是BackwardsIndex类型的[distinct int](https://nim-lang.org/docs/manual.html#types-distinct-type)，我们可以将lastIndex常量定义为`const lastIndex = ^1`，然后将其用作`b[0 .. lastIndex]`。

### [对象](https://nim-lang.org/docs/tut1.html#advanced-types-objects)

对象类型是一种默认类型，它可以将不同的值封装在一个具有名称的结构中。对象是一种值类型，这意味着当一个对象被赋值给一个新变量时，它的所有组成部分也会被复制。

每个对象类型Foo都有一个构造函数`Foo(field: value, ...)`，可以对所有字段进行初始化。未指定的字段将获得默认值。

```nim
type
  Person = object
    name: string
    age: int

var person1 = Person(name: "Peter", age: 30)

echo person1.name # "Peter"
echo person1.age  # 30

var person2 = person1 # copy of person 1

person2.age += 14

echo person1.age # 30
echo person2.age # 44


# the order may be changed
let person3 = Person(age: 12, name: "Quentin")

# not every member needs to be specified
let person4 = Person(age: 3)
# unspecified members will be initialized with their default
# values. In this case it is the empty string.
doAssert person4.name == ""
```

从定义模块外部可见的对象字段必须用\* 标记。

```nim
type
  Person* = object # the type is visible from other modules
    name*: string  # the field of this type is visible from other modules
    age*: int
```

### [元组](https://nim-lang.org/docs/tut1.html#advanced-types-tuples)

元组（tuple）与目前所见的对象非常相似。它们是值类型，赋值操作符会复制每个组件。与对象类型不同的是，元组类型是结构类型，也就是说，如果不同的元组类型以相同的顺序指定了相同类型和相同名称的字段，那么它们就是*等价*的。

构造函数`()`可用于构造元组。构造函数中字段的顺序必须与元组定义中的顺序一致。但与对象不同的是，这里不能使用元组类型的名称。

与对象类型一样，`t.field`符号用于访问元组的字段。另一个对象无法使用的符号是`t[i]`，用于访问第i 个字段。这里的i必须是一个常数整数。

```nim
type
  # type representing a person:
  # A person consists of a name and an age.
  Person = tuple
    name: string
    age: int
  
  # Alternative syntax for an equivalent type.
  PersonX = tuple[name: string, age: int]
  
  # anonymous field syntax
  PersonY = (string, int)

var
  person: Person
  personX: PersonX
  personY: PersonY

person = (name: "Peter", age: 30)
# Person and PersonX are equivalent
personX = person

# Create a tuple with anonymous fields:
personY = ("Peter", 30)

# A tuple with anonymous fields is compatible with a tuple that has
# field names.
person = personY
personY = person

# Usually used for short tuple initialization syntax
person = ("Peter", 30)

echo person.name # "Peter"
echo person.age  # 30

echo person[0] # "Peter"
echo person[1] # 30

# You don't need to declare tuples in a separate type section.
var building: tuple[street: string, number: int]
building = ("Rue del Percebe", 13)
echo building.street

# The following line does not compile, they are different tuples!
#person = building
# --> Error: type mismatch: got (tuple[street: string, number: int])
#     but expected 'Person'
```

尽管使用元组不需要声明类型，但用不同的字段名创建的元组将被视为不同的对象，尽管它们具有相同的字段类型。

元组可以在变量赋值时*拆包*。这可以方便地将元组的字段直接赋值给单独命名的变量。[os模块](https://nim-lang.org/docs/os.html)中的[splitFile](https://nim-lang.org/docs/os.html#splitFile,string)过程就是一个例子，它可以同时返回路径的目录、名称和扩展名。要使元组解包有效，必须在要解包的值周围使用括号，否则就会给所有单个变量分配相同的值！例如：

```nim
import std/os

let
  path = "usr/local/nimc.html"
  (dir, name, ext) = splitFile(path)
  baddir, badname, badext = splitFile(path)
echo dir      # outputs "usr/local"
echo name     # outputs "nimc"
echo ext      # outputs ".html"
# All the following output the same line:
# "(dir: usr/local, name: nimc, ext: .html)"
echo baddir
echo badname
echo badext
```

for 循环中也支持元组解包：

```nim
let a = [(10, 'a'), (20, 'b'), (30, 'c')]

for (x, c) in a:
  echo x
# This will output: 10; 20; 30

# Accessing the index is also possible:
for i, (x, c) in a:
  echo i, c
# This will output: 0a; 1b; 2c
```

元组的字段总是公有的，它们不需要显式地标记为输出，这与对象类型中的字段不同。

### [引用和指针类型](https://nim-lang.org/docs/tut1.html#advanced-types-reference-and-pointer-types)

引用（类似于其他编程语言中的指针）是一种引入多对一关系的方法。这意味着不同的引用可以指向并修改内存中的同一位置。

Nim区分跟踪引用和非跟踪引用。非跟踪引用也称为*指针*。跟踪引用指向垃圾堆中的对象，无跟踪引用指向手动分配的对象或内存中其他位置的对象。因此，非跟踪引用是*不安全的*。不过，对于某些低级操作（如访问硬件），非跟踪引用是必要的。

使用**ref**关键字声明跟踪引用；使用**ptr**关键字声明非跟踪引用。

空`[]`下标符号可用于*解引用*，即检索引用指向的对象。`.`（访问元组、对象字段操作符）和`[]`（数组、字符串、序列索引操作符）操作符对引用类型执行隐式解引用操作：

```nim
type
  Node = ref object
    le, ri: Node
    data: int

var n = Node(data: 9)
echo n.data
# no need to write n[].data; in fact n[].data is highly discouraged!
```

要分配一个新的跟踪对象，可以使用内置过程new：
```nim
var n: Node
new(n)
```

要处理未跟踪的内存，可以使用程序alloc、dealloc和realloc。[system](https://nim-lang.org/docs/system.html)模块文档包含更多详细信息。

如果引用没有指向任何内容，则其值为nil。

### [过程类型](https://nim-lang.org/docs/tut1.html#advanced-types-procedural-type)

过程类型是指向过程的指针（有点抽象）。nil是过程类型变量的允许值。Nim使用过程类型来实现函数式编程技术。

例如：

```nim
proc greet(name: string): string =
  "Hello, " & name & "!"

proc bye(name: string): string =
  "Goodbye, " & name & "."

proc communicate(greeting: proc (x: string): string, name: string) =
  echo greeting(name)

communicate(greet, "John")
communicate(bye, "Mary")
```

过程类型的一个微妙问题是，过程的调用约定会影响类型的兼容性：过程类型只有在具有相同的调用约定时才是兼容的。[手册](https://nim-lang.org/docs/manual.html#types-procedural-type)中列出了不同的调用约定。

### [区分类型](https://nim-lang.org/docs/tut1.html#advanced-types-distinct-type)

区别类型允许创建一种新类型，这种类型 "并不意味着它与其基础类型之间存在子类型关系"。您必须**明确**定义独特类型的所有行为。为了帮助实现这一点，独特类型及其基础类型都可以从一种类型转换为另一种类型。[手册](https://nim-lang.org/docs/manual.html#types-distinct-type)中提供了相关示例。

## [模块](https://nim-lang.org/docs/tut1.html#modules)

Nim使用模块将程序分割成多个部分，每个模块都有自己的源文件。模块可实现信息隐藏和单独编译。一个模块可以通过import语句访问另一个模块的符号。只有标有星号`*`的顶级符号才会被导出：

```nim
# Module A
var
  x*, y: int

proc `*` *(a, b: seq[int]): seq[int] =
  # 为新序列分配空间
  newSeq(result, len(a))
  # 对两个整型序列的元素进行相乘
  for i in 0 ..< len(a): result[i] = a[i] * b[i]

when isMainModule:
  # 对整型序列的新运算符“*”进行测试
  assert(@[1, 2, 3] * @[1, 2, 3] == @[1, 4, 9])
```

上述模块导出了变量“x”和“\*”运算符，但没有导出y。

模块的顶层语句在程序开始时执行。例如，这可用于初始化复杂的数据结构。

每个模块都有一个特殊的神奇常量isMainModule，如果模块作为主文件编译，则该常量为true。如上例所示，这对于在模块中嵌入测试非常有用。

模块的符号可以用module.symbol语法来限。如果符号有歧义，则必须加以限定。如果一个符号被定义在两个（或多个）不同的模块中，并且这两个模块都被第三个模块导入，那么这个符号就是模棱两可的：

```nim
# Module A
var x*: string
```

```nim
# Module B
var x*: int
```

```nim
# Module C
import A, B
write(stdout, x) # error: x is ambiguous
write(stdout, A.x) # okay: qualifier used

var x = 4
write(stdout, x) # not ambiguous: uses the module C's x
```

但这条规则不适用于过程或迭代器。下面的例子适用于重载规则：

```nim
# Module A
proc x*(a: int): string = $a
```

```nim
# Module B
proc x*(a: string): string = $a
```

```nim
# Module C
import A, B
write(stdout, x(3))   # no error: A.x is called
write(stdout, x(""))  # no error: B.x is called

proc x*(a: int): string = discard
write(stdout, x(3))   # ambiguous: which `x` is to call?
```

### [排除符号](https://nim-lang.org/docs/tut1.html#modules-excluding-symbols)

一般的`import`语句会导入模块所有可导入的符号。可以通过使用`except`限定符排除不需要导入的符号。

```nim
import mymodule except y
```

### [From语句](https://nim-lang.org/docs/tut1.html#modules-from-statement)

我们已经看过简单的import语句，它只导入所有可导入的符号。另一种只导入列出符号的导入语句是`from import`语句：

```nim
from mymodule import x, y, z
```

from语句还可以强制对模块符号进行命名空间限定，从而使模块符号可用，但必须经过限定才能使用。

```nim
from mymodule import x, y, z

x()           # use x without any qualification
```

```nim
from mymodule import nil

mymodule.x()  # must qualify x with the module name as prefix

x()           # using x here without qualification is a compile error
```

由于模块名通常较长，因此也可以定义一个较短的别名来限定模块符号。

```nim
from mymodule as m import nil

m.x()         # m is aliasing mymodule
```

### [包含语句](https://nim-lang.org/docs/tut1.html#modules-include-statement)

`include`语句与导入模块有本质区别：它只是包含一个文件的内容。include语句在将一个大模块分割成多个文件时非常有用：

```nim
include fileA, fileB, fileC
```

## [第二部分](https://nim-lang.org/docs/tut1.html#part-2)

基础知识学完了，现在让我们看看Nim除了提供基础的漂亮语法外，还能提供什么：[第二部分](../nim-tutorial-part-2)