---
title: Nim官方教程第二部分
author: kernel
date: 2023-11-07 12:34:00 +0800
categories: [Nim语言, 官方教程]
tags: [Nim语言, Nim官方教程, Nim入门]
---

| 条目        | 内容说明          |
| ----------- | ----------------- |
| 作者：      | 安德烈亚斯-鲁普夫 |
| 版本：      | 2.0.0             |
| 翻译：      | Kernel Zhang      |

## [导言](https://nim-lang.org/docs/tut2.html#introduction)

> “重复让荒谬变得合理。”-- Norman Wildberger

本文档是Nim编程语言高级结构的教程。请注意，本文档已有些过时，因为[手册](https://nim-lang.org/docs/manual.html)中包含更多高级语言功能的示例。

## [Pragmas](https://nim-lang.org/docs/tut2.html#pragmas)

Pragmas是Nim在不引入大量新关键字的情况下，为编译器提供额外信息、命令的方法。Pragmas用特殊的`{.`和`.}`大括号括起来。本教程不涉及语法。有关可用语法的说明，请参阅[手册](https://nim-lang.org/docs/manual.html#pragmas)或[用户指南](https://nim-lang.org/docs/nimc.html#additional-features)。

## [面向对象程序设计](https://nim-lang.org/docs/tut2.html#object-oriented-programming)

虽然Nim对面向对象编程（OOP）的支持是最低限度的，但可以使用强大的OOP技术。OOP被视为设计程序的一种方法，而不是*唯一的*方法。通常情况下，程序化方法能带来更简单、更高效的代码。特别是，相比继承，更倾向于组合往往是更好的设计。（译者：最新的语言似乎都在抛弃OOP，而采用组合）

### [继承](https://nim-lang.org/docs/tut2.html#object-oriented-programming-inheritance)

Nim中的继承完全是可选的。要启用带有运行时类型信息的继承，对象需要从RootObj继承。 这可以直接实现，也可以通过继承一个从RootObj继承的对象来间接实现。通常，具有继承性的类型也会被标记为ref类型，尽管这并不是严格执行的。要在运行时检查对象是否属于某种类型，可以使用of操作符。

```nim
type
  Person = ref object of RootObj
    name*: string  # the * means that `name` is accessible from other modules
    age: int       # no * means that the field is hidden from other modules
  
  Student = ref object of Person # Student inherits from Person
    id: int                      # with an id field

var
  student: Student
  person: Person
assert(student of Student) # is true
# object construction:
student = Student(name: "Anton", age: 5, id: 2)
echo student[]
```

继承是通过`object of`语法完成的。目前不支持多重继承。如果一个对象类型没有合适的祖先，可以使用RootObj作为其祖先，但这只是一种约定俗成的做法。没有祖先的对象隐含为最终对象。除了system.RootObj 之外，您还可以使用`inheritable`语法来引入新的对象根（例如，GTK 封装器中就使用了这种方法）。

只要使用继承，就应该使用Ref对象。严格来说，这并不是必须的，但对于非Ref对象，诸如`let person.Person = Student(id: 123)`这样的赋值会截断子类的字段。

**注**：就简单代码重用而言，组合（has-a 关系）通常比继承（is-a 关系）更可取。由于对象在Nim中是值类型，因此组合与继承一样有效。

### [相互递归类型](https://nim-lang.org/docs/tut2.html#object-oriented-programming-mutually-recursive-types)

对象、元组和引用可以构成相互依赖的复杂数据结构；它们是*相互递归*的。在Nim中，这些类型只能在单个类型部分中声明。（如果在任何类型中进行声明都支持，会大大降低编译速度）。

例如：

```nim
type
  Node = ref object  # a reference to an object with the following field:
    le, ri: Node     # left and right subtrees
    sym: ref Sym     # leaves contain a reference to a Sym
  
  Sym = object       # a symbol
    name: string     # the symbol's name
    line: int        # the line the symbol was declared in
    code: Node       # the symbol's abstract syntax tree
```

### [类型转换](https://nim-lang.org/docs/tut2.html#object-oriented-programming-type-conversions)

Nim区分强制类型转换和值类型转换。强制类型转换是使用`cast`操作符完成的，它强制编译器将位模式解释为另一种类型。

值类型转换是将一种类型转换为另一种类型的更礼貌的方法：它们保留的是抽象值，而不一定是位模式。如果无法进行值类型转换，编译器会输出错误或引发异常。

值类型转换的语法是`destination_type(expression_to_convert)`（与普通调用相同）：

```nim
proc getID(x: Person): int =
  Student(x).id
```

如果x不是学生，则会引发`InvalidObjectConversionDefect`异常。

### [对象变体](https://nim-lang.org/docs/tut2.html#object-oriented-programming-object-variants)

在某些需要简单变体类型的情况下，对象层次结构往往是多余的。

举个例子：

```nim
# 下面举例说明如何在Nim中模拟抽象语法树
type
  NodeKind = enum  # the different node types
    nkInt,          # a leaf with an integer value
    nkFloat,        # a leaf with a float value
    nkString,       # a leaf with a string value
    nkAdd,          # an addition
    nkSub,          # a subtraction
    nkIf            # an if statement
  Node = ref object
    case kind: NodeKind  # the `kind` field is the discriminator
    of nkInt: intVal: int
    of nkFloat: floatVal: float
    of nkString: strVal: string
    of nkAdd, nkSub:
      leftOp, rightOp: Node
    of nkIf:
      condition, thenPart, elsePart: Node

var n = Node(kind: nkFloat, floatVal: 1.0)
# the following statement raises an `FieldDefect` exception, because
# n.kind's value does not fit:
n.strVal = ""
```

从示例中可以看出，对象层次结构的优点是不需要在不同对象类型之间进行转换。然而，访问无效的对象字段会引发运行时异常。

### [方法调用语法](https://nim-lang.org/docs/tut2.html#object-oriented-programming-method-call-syntax)

调用过程有一种语法糖：可以用`obj.methodName(args)`来取代`methodName(obj, args)`。如果没有其余参数，可以省略括号：`obj.len`（而不是`len(obj)`）。

这种方法调用语法并不局限于对象，它可以用于任何类型：

```nim
import std/strutils

echo "abc".len # is the same as echo len("abc")
echo "abc".toUpperAscii()
echo({'a', 'b', 'c'}.card)
stdout.writeLine("Hallo") # the same as writeLine(stdout, "Hallo")
```

另一种看待方法调用语法的方式是，它提供了缺失的后缀符号。

因此，“纯面向对象”代码很容易编写：

```nim
import std/[strutils, sequtils]

stdout.writeLine("Give a list of numbers (separated by spaces): ")
stdout.write(stdin.readLine.splitWhitespace.map(parseInt).max.`$`)
stdout.writeLine(" is the maximum!")
```

### [属性](https://nim-lang.org/docs/tut2.html#object-oriented-programming-properties)

如上例所示，Nim不需要专门获取属性的get语法：使用普通的过程就能达到同样的目的。但设置值则不同，需要使用特殊的setter语法：

```nim
type
  Socket* = ref object of RootObj
    h: int # cannot be accessed from the outside of the module due to missing star

proc `host=`*(s: var Socket, value: int) {.inline.} =
  ## setter of host address
  s.h = value

proc host*(s: Socket): int {.inline.} =
  ## getter of host address
  s.h

var s: Socket
new s
s.host = 34  # same as `host=`(s, 34)
```

（该示例还显示了内联程序。）

可以重载`[]`数组访问操作符，以提供数组属性：

```nim
type
  Vector* = object
    x, y, z: float

proc `[]=`* (v: var Vector, i: int, value: float) =
  # setter
  case i
  of 0: v.x = value
  of 1: v.y = value
  of 2: v.z = value
  else: assert(false)

proc `[]`* (v: Vector, i: int): float =
  # getter
  case i
  of 0: result = v.x
  of 1: result = v.y
  of 2: result = v.z
  else: assert(false)
```

这个例子很傻，因为用元组来模拟向量更好，因为元组已经提供了`v[]`访问。

### [动态分发](https://nim-lang.org/docs/tut2.html#object-oriented-programming-dynamic-dispatch)

过程总是使用静态调度。对于动态派发，用`method`代替`proc`关键字：

```nim
type
  Expression = ref object of RootObj ## abstract base class for an expression
  Literal = ref object of Expression
    x: int
  PlusExpr = ref object of Expression
    a, b: Expression

# watch out: 'eval' relies on dynamic binding
method eval(e: Expression): int {.base.} =
  # override this base method
  quit "to override!"

method eval(e: Literal): int = e.x
method eval(e: PlusExpr): int = eval(e.a) + eval(e.b)

proc newLit(x: int): Literal = Literal(x: x)
proc newPlus(a, b: Expression): PlusExpr = PlusExpr(a: a, b: b)

echo eval(newPlus(newPlus(newLit(1), newLit(2)), newLit(4)))
```

请注意，在示例中，构造函数newLit和newPlus是procs，因为它们使用静态绑定更合理，但eval是方法，因为它需要动态绑定。

**注意**：从Nim 0.20开始，要使用多参数动态派发，必须在编译时明确传递`--multimethods:on`。

在多参数动态派发中，所有具有对象类型的参数都用于调度：

```nim
type
  Thing = ref object of RootObj
  Unit = ref object of Thing
    x: int

method collide(a, b: Thing) {.inline.} =
  quit "to override!"

method collide(a: Thing, b: Unit) {.inline.} =
  echo "1"

method collide(a: Unit, b: Thing) {.inline.} =
  echo "2"

var a, b: Unit
new a
new b
collide(a, b) # output: 2
```

正如示例所示，多方法的调用不能含糊不清：“collide 2”优先于“collide 1”，因为解析是从左向右进行的。因此，“Unit, Thing”比“Thing, Unit”更受青睐。

**性能说明**：Nim不生成虚拟方法表，而是生成调度树。这避免了方法调用中昂贵的间接分支，并实现了内联。不过，其他优化（如编译时求值或消除死代码）对方法不起作用。

## [异常](https://nim-lang.org/docs/tut2.html#exceptions)

在Nim中，异常是对象。按照惯例，异常类型的后缀是`Error`。[system](https://nim-lang.org/docs/system.html)模块定义了一个异常层次结构，你可能需要遵守。异常源于system.Exception，后者提供了通用接口。

异常必须在堆上分配，因为它们的生命周期是未知的。编译器会阻止你引发在栈上创建的异常。所有引发的异常至少应在msg字段中说明引发的原因。

按照惯例，异常应在*特殊*情况下出现，而不应作为控制流的替代方法。

### [Raise语法](https://nim-lang.org/docs/tut2.html#exceptions-raise-statement)

使用raise语句可以引发异常：

```nim
var
  e: ref OSError
new(e)
e.msg = "the request to the OS failed"
raise e
```

如果`raise`关键字后面没有表达式，则会*重新引发*上一个异常。为了避免重复上面常见的代码模式，可以使用系统模块中的newException模板：

```nim
raise newException(OSError, "the request to the OS failed")
```

### [Try语法](https://nim-lang.org/docs/tut2.html#exceptions-try-statement)

try语句处理异常：

```nim
from std/strutils import parseInt

# read the first two lines of a text file that should contain numbers
# and tries to add them
var
  f: File
if open(f, "numbers.txt"):
  try:
    let a = readLine(f)
    let b = readLine(f)
    echo "sum: ", parseInt(a) + parseInt(b)
  except OverflowDefect:
    echo "overflow!"
  except ValueError:
    echo "could not convert string to integer"
  except IOError:
    echo "IO error!"
  except CatchableError:
    echo "Unknown exception!"
    # reraise the unknown exception:
    raise
  finally:
    close(f)
```

首先会执行try后面的语句，如果出现异常则会跳出try语句，然后执行相应的except部分。

最后的`CatchableError`涵盖了未明确列出的异常，它类似于if语句中的else部分。（译者：原文为空异常，应该是笔误）

如果有`finally`部分，它总是在异常处理程序之后执行。

异常在`except`部分中*消耗*。如果异常没有得到处理，它就会在调用栈中传播。这意味着程序的其余部分（不在最终子句中）通常不会被执行（如果出现异常）。

如果需要访问异常分支中的实际异常对象或消息，可以使用[system](https://nim-lang.org/docs/system.html)模块中的[getCurrentException()](https://nim-lang.org/docs/system.html#getCurrentException)和[getCurrentExceptionMsg()](https://nim-lang.org/docs/system.html#getCurrentExceptionMsg)过程。

示例：

```nim
try:
  doSomethingHere()
except CatchableError:
  let
    e = getCurrentException()
    msg = getCurrentExceptionMsg()
  echo "Got exception ", repr(e), " with message ", msg
```

### [异常过程的注释](https://nim-lang.org/docs/tut2.html#exceptions-annotating-procs-with-raised-exceptions)

通过使用可选的`{.raises.}`语义，您可以指定一个过程引发一组特定的异常，或者完全不引发异常。如果使用了`{.raises.}`语义，编译器将验证这一点是否属实。例如，如果你指定某个过程引发`IOError`，而在某个时候它（或它调用的某个过程）却引发了其他的异常，编译器就会阻止该proc的编译。使用示例

```nim
proc complexProc() {.raises: [IOError, ArithmeticDefect].} =
  ...

proc simpleProc() {.raises: [].} =
  ...
```

一旦有了这样的代码，如果引发的异常列表发生变化，编译器就会停止编译，并给出错误信息，指明出错语义和引发的异常未被捕获的程序行，以及引发未捕获异常的文件和行，这可能有助于找到发生变化的违规代码。

如果您想在现有代码中添加`{.raises.}`语义，编译器也可以帮您。你可以在proc中添加`{.effects.}`语义的语句，编译器将输出该部分所有侦测出的效果（异常跟踪是Nim效果系统的一部分）。另一种更迂回的方法是使用Nim的doc命令，它可以生成整个模块的文档，并用异常列表装饰所有可执行过程，从而找出某个执行过程引发的异常列表。你可以[在手册中](https://nim-lang.org/docs/manual.html#effect-system)阅读更多关于Nim效果系统和相关实用程序的内容。

## [泛型](https://nim-lang.org/docs/tut2.html#generics)

泛型是Nim使用类型参数对过程、迭代器或类型进行参数化的方法。泛型参数写在方括号内，例如`Foo[T]`。它们对高效的类型安全容器最有用：

```nim
type
  BinaryTree*[T] = ref object # BinaryTree is a generic type with
                              # generic param `T`
    le, ri: BinaryTree[T]     # left and right subtrees; may be nil
    data: T                   # the data stored in a node

proc newNode*[T](data: T): BinaryTree[T] =
  # constructor for a node
  new(result)
  result.data = data

proc add*[T](root: var BinaryTree[T], n: BinaryTree[T]) =
  # insert a node into the tree
  if root == nil:
    root = n
  else:
    var it = root
    while it != nil:
      # compare the data items; uses the generic `cmp` proc
      # that works for any type that has a `==` and `<` operator
      var c = cmp(it.data, n.data)
      if c < 0:
        if it.le == nil:
          it.le = n
          return
        it = it.le
      else:
        if it.ri == nil:
          it.ri = n
          return
        it = it.ri

proc add*[T](root: var BinaryTree[T], data: T) =
  # convenience proc:
  add(root, newNode(data))

iterator preorder*[T](root: BinaryTree[T]): T =
  # Preorder traversal of a binary tree.
  # This uses an explicit stack (which is more efficient than
  # a recursive iterator factory).
  var stack: seq[BinaryTree[T]] = @[root]
  while stack.len > 0:
    var n = stack.pop()
    while n != nil:
      yield n.data
      add(stack, n.ri)  # push right subtree onto the stack
      n = n.le          # and follow the left pointer

var
  root: BinaryTree[string] # instantiate a BinaryTree with `string`
add(root, newNode("hello")) # instantiates `newNode` and `add`
add(root, "world")          # instantiates the second `add` proc
for str in preorder(root):
  stdout.writeLine(str)
```

该示例显示了一棵通用二叉树。根据上下文的不同，括号既可以用来引入类型参数，也可以用来实例化一个泛型过程、迭代器或类型。如示例所示，泛型可以重载：使用add的最佳匹配。序列的内置add过程并未隐藏，而是在`preorder`迭代器中使用。

在使用泛型和方法调用语法时，有一种特殊的`[:T]`语法：

```nim
proc foo[T](i: T) =
  discard

var i: int

# i.foo[int]() # Error: expression 'foo(i)' has no type (or is ambiguous)

i.foo[:int]() # Success
```

## [模板](https://nim-lang.org/docs/tut2.html#templates)

模板是一种简单的替换机制，可在Nim的抽象语法树上运行。模板在编译器的语义传递中进行处理。它们与语言的其他部分结合得很好，没有C语言预处理器宏的缺陷。

要*调用*模板，就像调用存储过程一样。

例如

```nim
template `!=` (a, b: untyped): untyped =
  # this definition exists in the System module
  not (a == b)

assert(5 != 6) # the compiler rewrites that to: assert(not (5 == 6))
```

`!=`， `>`，` >=`，`in`， `notin`，`isnot`操作符实际上是模板：这样做的好处是，如果重载`==`操作符，`!=`操作符就会自动可用，并执行正确的操作。（IEEE浮点数除外——NaN会破坏基本的布尔逻辑。）

`a > b`被转换为`b < a`。`a in b`被转换为`contains(b, a)`。`notin`和`isnot`的含义显而易见。

模板对惰性求值尤其有用。请看一个简单的日志记录程序：

```nim
const
  debug = true

proc log(msg: string) {.inline.} =
  if debug: stdout.writeLine(msg)

var
  x = 4
log("x has the value: " & $x)
```

这段代码有一个缺点：如果有一天调试设置为false，则仍会执行相当昂贵的$和&操作！（过程的参数计算是立即的。）

将`log`过程变成模板就可以解决这个问题：

```nim
const
  debug = true

template log(msg: string) =
  if debug: stdout.writeLine(msg)

var
  x = 4
log("x has the value: " & $x)
```

参数的类型可以是普通类型或元类型（untyped、typed或type）。type表示只能将类型符号作为参数，untyped表示在表达式传递给模板之前不进行符号查找和类型解析。

如果模板没有明确的返回类型，则使用void，以便与过程和方法保持一致。

要向模板传递语句块，最后一个参数应使用untyped：

```ni
template withFile(f: untyped, filename: string, mode: FileMode,
                  body: untyped) =
  let fn = filename
  var f: File
  if open(f, fn, mode):
    try:
      body
    finally:
      close(f)
  else:
    quit("cannot open: " & fn)

withFile(txt, "ttempl3.txt", fmWrite):
  txt.writeLine("line 1")
  txt.writeLine("line 2")
```

在示例中，两条writeLine语句与body参数绑定。withFile模板包含模板代码，有助于避免一个常见错误：忘记关闭文件。请注意`let fn = filename`语句如何确保filename只被评估一次。

### [应用示例：赋能过程](https://nim-lang.org/docs/tut2.html#templates-examplecolon-lifting-procs)

```nim
import std/math

template liftScalarProc(fname) =
  ## Lift a proc taking one scalar parameter and returning a
  ## scalar value (eg `proc sssss[T](x: T): float`),
  ## to provide templated procs that can handle a single
  ## parameter of seq[T] or nested seq[seq[]] or the same type
  ##
  ##   ```Nim
  ##   liftScalarProc(abs)
  ##   # now abs(@[@[1,-2], @[-2,-3]]) == @[@[1,2], @[2,3]]
  ##   ```
  proc fname[T](x: openarray[T]): auto =
    var temp: T
    type outType = typeof(fname(temp))
    result = newSeq[outType](x.len)
    for i in 0..<x.len:
      result[i] = fname(x[i])

liftScalarProc(sqrt)   # make sqrt() work for sequences
echo sqrt(@[4.0, 16.0, 25.0, 36.0])   # => @[2.0, 4.0, 5.0, 6.0]
```

## [编译为JavaScript](https://nim-lang.org/docs/tut2.html#compilation-to-javascript)

Nim代码可以编译为JavaScript。但是，为了编写与JavaScript兼容的代码，您应该记住以下几点：

-   addr和ptr在JavaScript中的语义略有不同。如果不确定它们在JavaScript中的转换，建议避免使用。
-   在JavaScript中，`cast[T](x)`被转换为`(x)`，但在有符号和无符号int之间的转换除外，在这种情况下，它的行为与C语言中的静态转换相同。
-   cstring在JavaScript中的意思是JavaScript字符串。只有在语义合适的情况下使用cstring才是一种好的做法。例如，不要将cstring用作二进制数据缓冲区。

## [第三部分](https://nim-lang.org/docs/tut2.html#part-3)

下一部分完全是关于通过宏进行元编程：[第三部分](../nim-tutorial-part-3)