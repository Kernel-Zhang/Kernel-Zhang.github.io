---
title: Vala教程
author: kernel
date: 2023-11-20 12:34:00 +0800
categories: [Vala语言, 教程]
tags: [Vala语言, Vala官方教程, Vala入门]
---

## 导言

免责声明：Vala 是一个正在进行中的项目，其功能可能会发生变化。我会尽量更新本教程，但我并不保证完美。此外，我也不能保证我所建议的技术一定是实践中最好的技术，但我会努力跟上这类事情的发展。

### Vala是什么？

Vala 是一种新的编程语言，它允许使用现代编程技术编写在 GNOME 运行时库（尤其是 GLib 和 GObject）上运行的应用程序。长期以来，该平台提供了一个非常完整的编程环境，具有动态类型系统和辅助内存管理等功能。在 Vala 之前，为该平台编程的唯一方法是使用机器本机 C API（该 API 暴露了许多通常不需要的细节）、使用带有虚拟机的高级语言（如 Python 或 Mono C# 语言），或者通过封装库使用 C++。

Vala 不同于所有这些技术，因为它输出的 C 代码可以在 GNOME 平台之外编译运行，无需额外的库支持。这有几个后果，但最重要的是：

-   用 Vala 编写的程序应与直接用 C 语言编写的程序具有大致相同的性能，同时编写和维护程序也更容易、更快。
-   Vala 应用程序可以做任何 C 语言所不能做的事情。虽然 Vala 引入了许多 C 语言所不具备的语言特性，但这些特性都可以映射到 C 语言结构中，尽管直接编写这些结构通常比较困难或耗时过多。

因此，虽然 Vala 是一门现代语言，具备您所期望的所有功能，但它的力量来自于现有平台，在某些方面必须遵守现有平台制定的规则。

### 本教程适合哪些人？

本教程不会深入介绍基本编程实践。它只会简要解释面向对象编程的原理，而重点是介绍 Vala 如何应用这些概念。因此，如果您已经掌握了多种编程语言，本教程将对您有所帮助，但并不要求您对任何一种编程语言有深入的了解。

Vala 与 C# 共享许多语法，但我将尽量避免从与 C# 或 Java 的异同角度来描述这些特性，目的是使教程更易于理解。

虽然理解 Vala 本身并不需要了解 C 语言，但重要的是要认识到 Vala 程序是以 C 语言执行的，并经常与 C 语言库交互。了解 C 语言肯定会更容易深入理解 Vala。

### 惯例

代码将使用单行距文本，所有命令都放在shell代码块中。除此之外，一切都应该是显而易见的。我倾向于非常明确地编写代码，包括一些实际上隐含的信息。我会尽量解释哪些地方可以省略，但这并不意味着我鼓励你这样做。

到时候我会加入对 Vala 文档的引用，但现在还不可用。

## 第一个程序

可悲的是，这些都是可以预料的，但仍然是：

```vala
class Demo.HelloWorld : GLib.Object {
    public static int main(string[] args) {
        stdout.printf("Hello, World\n");
        return 0;
    }
}
```

当然，这是一个 Vala Hello World 程序。我希望你能很好地识别其中的某些部分，但为了全面起见，我还是要一步步来。

```vala
class Demo.HelloWorld : GLib.Object {
```

这一行表示类定义的开始。Vala 中类的概念与其他语言非常相似。类基本上是一种对象类型，可以创建实例，所有实例都具有相同的属性。类类型的实现由 gobject 库负责，但其中的细节对一般使用并不重要。

值得注意的是，该类被明确描述为 GLib.Object 的子类。这是因为 Vala 允许使用其他类型的类，但在大多数情况下，您需要的是这种类型的类。事实上，只有当你的类是 GLib.Object 的子类时，才允许使用 Vala 的某些语言特性。

该行的其他部分显示了名称间距和完全限定名称，不过这些将在后面解释。

```vala
public static int main(string[] args) {
```

这是方法定义的开头。方法是与某种类型的对象相关的函数，可以在该类型的对象上执行。静态方法的意思是，调用该方法时无需拥有该类型的特定实例。这个方法被称为 `main`，并且具有这样的签名，这意味着 Vala 会将其识别为程序的入口点。

`main`方法不一定要在类中定义。但是，如果它被定义在类中，就必须是静态的。是公有还是私有并不重要。返回类型可以是 int 或 void。如果返回类型为 void，程序将隐式终止，退出代码为 0。包含命令行参数的字符串数组参数是可选的。

```vala
stdout.printf("Hello, World\n");
```

`stdout`是`GLib`命名空间中的一个对象，Vala 可确保您在需要时访问该对象。这一行指示 Vala 以 hello 字符串作为参数，执行`stdout`对象的`printf`方法。在 Vala 中，调用对象的方法或访问对象的数据时总是使用这种语法。`\n` 是换行的转义序列。

```vala
return 0;
```

return 是向调用者返回一个值，并终止`main`方法的执行，同时终止程序的执行。`main`方法返回的值就是程序的退出代码。

最后几行简单地结束了方法和类的定义。

### 编译和运行

假设您已安装了 Vala，那么编译和执行该程序只需

```shell
valac hello.vala
./hello
```

`valac`是Vala编译器，可将 Vala 代码编译成二进制文件。生成的二进制文件与源文件的名称相同，可以在机器上直接执行。你也许能猜到输出结果。

如果您从 C 语言编译器中收到一些警告，请跳转到[Valac](https://wiki.gnome.org/Projects/Vala/Tutorial#valac)了解原因和解决方案。

## 基础知识

### 源文件和编译

Vala 代码写在扩展名为`.vala`的文件中。Vala 并不像 Java 等语言那样强制执行大量的结构——没有包或类文件等相同的概念。取而代之的是，结构由每个文件内的文本定义，通过命名空间等结构描述代码的逻辑位置。当您要编译 Vala 代码时，您只需给编译器一个所需的文件列表，Vala 就会计算出这些文件是如何组合在一起的。

这样做的结果是，你可以在一个文件中放入任意多的类或函数，甚至可以将不同命名空间的部分内容组合在一起。这并不一定是个好主意。你可能需要遵循某些约定。Vala 项目本身就是一个很好的例子。

同一软件包的所有源文件都作为命令行参数与编译器标志一起提供给 Vala 编译器 valac。这与 Java 源代码的编译方式类似。例如

```shell
valac compiler.vala --pkg libvala
```

将生成一个二进制文件，名称为`compiler`，并与`libvala`软件包链接。事实上，`valac`编译器就是这样生成的！

如果您希望二进制文件有一个不同的名称，或者您向编译器传递了多个源文件，您可以使用 -o 开关明确指定二进制文件的名称：

```shell
valac source1.vala source2.vala -o myprogram
./myprogram
```

如果给`valac`加上 -C 开关，它就不会将程序编译成二进制文件。相反，它会将每个 Vala 源文件的中间 C 代码输出到相应的 C 源文件中，本例中为`source1.c`和`source2.c`。您还会注意到，这个类是在运行系统中动态注册的。这是 GNOME 平台强大功能的一个很好的例子，但正如我之前所说，使用 Vala 不需要了解太多这方面的知识。

如果您想为您的项目添加 C 语言头文件，可以使用 -H 开关：

```shell
valac hello.vala -C -H hello.h
```

### 语法概述

Vala 的语法在很大程度上是基于 C# 的混合语法。因此，对于熟悉任何 C# 类语言的程序员来说，大部分内容都不会陌生。

使用大括号定义范围。对象或引用只在 { 和 } 之间有效。这些也是用于定义类、方法、代码块等的分隔符，因此它们自动拥有自己的范围。Vala 对变量的声明位置并不严格。

标识符由其类型和名称定义，例如 `int c` 表示名为`c` 的整数。对于引用类型，标识符只是定义一个新的引用，最初并不指向任何东西。

标识符名称可以是字母（`[a-z]`、`[A-Z]`）、下划线和数字的任意组合。不过，要定义或引用一个以数字开头或关键字的标识符名称，必须在其前缀加上"@"字符。该字符不被视为名称的一部分。例如，您可以用 @foreach 来命名一个方法`foreach`，尽管这是一个保留的 Vala 关键字。如果'@'字符能被明确解释为标识符名称，则可以省略，如 "foo.foreach() "中的 "foo.foreach()"。

引用类型使用 new 操作符和构造方法的名称（通常只是类型的名称）进行实例化，例如 Object o = new Object() 创建一个新对象，并将`o`作为该对象的引用。

### 注释

Vala 允许以不同方式对代码进行注释。

```vala
// Comment continues until end of line

/* Comment lasts between delimiters */

/**
 * Documentation comment
 */
```

这些注释的处理方式与大多数其他语言相同，因此无需过多解释。文档注释实际上对 Vala 来说并不特别，但像[Valadoc](https://wiki.gnome.org/Projects/Valadoc)这样的文档生成工具可以识别它们。

### 数据类型

一般来说，Vala 有两种数据类型：_引用类型_和_值类型_。这些名称描述了这些类型的实例在系统中的传递方式--值类型在分配给新标识符时会被复制，而引用类型则不会被复制，相反，新标识符只是对同一对象的新引用。

常量的定义是在类型前加上 const。常量的命名规则是 ALL\_UPPER\_CASE。

#### 值类型

与大多数其他语言一样，Vala 支持一组简单类型。

-   字节：`char`、`uchar`；由于历史原因，它们的名称都是`char`。
    
-   字符：`unicar`；32 位统一字符编码字符
    
-   整型：`int`、`uint`
    
-   长整型：`long`、`ulong`
    
-   短整型：`short`、`ushort`
    
-   保证大小整型：`int8`、`int16`、`int32`、`int64` 以及它们的无符号同类整型 `uint8`、`uint16`、`uint32`、`uint64`。后缀数字表示长度（位）。
    
-   浮点数：`float`, `double`
    
-   布尔值：`bool`；可能的值是 `true` 和 `false`
    
-   复合类型：`struct`
    
-   枚举：`enum`；用整数值表示，而不是像 Java 的枚举那样用类表示
    

下面是一些例子。

```vala
/* atomic types */
unichar c = 'u';
float percentile = 0.75f;
const double MU_BOHR = 927.400915E-26;
bool the_box_has_crashed = false;

/* defining a struct */
struct Vector {
    public double x;
    public double y;
    public double z;
}

/* defining an enum */
enum WindowType {
    TOPLEVEL,
    POPUP
}
```

除了保证大小的整数类型外，大多数类型在不同平台上可能有不同的大小。sizeof 运算符返回给定类型的变量所占字节的大小：

```vala
ulong nbytes = sizeof(int32);    // nbytes will be 4 (= 32 bits)
```

您可以使用`.MIN`和`.MAX` 确定数值类型的最小值和最大值，例如 `int.MIN` 和 `int.MAX`。

#### 字符串

字符串的数据类型是`string`。Vala 字符串采用 UTF-8 编码，不可更改。

```vala
string text = "A string literal";
```

Vala 提供一种称为*逐字字符串*的功能。在这些字符串中，转义序列（如 `\n`）不会被解释，换行符将被保留，引号也无需屏蔽。它们用三重双引号括起来。换行后可能出现的缩进也是字符串的一部分。

```vala
string verbatim = """This is a so-called "verbatim string".
Verbatim strings don't process escape sequences, such as \n, \t, \\, etc.
They may contain quotes and may span multiple lines.""";
```

前缀为"@"的字符串是字符串模板。它们可以评估嵌入变量和前缀为"$"的表达式：

```vala
int a = 6, b = 7;
string s = @"$a * $b = $(a * b)";  // => "6 * 7 = 42"
```

相等运算符 `==` 和 `!=` 比较两个字符串的内容，这与 Java 的行为相反，Java 在这种情况下会检查引用相等。

您可以使用 `[start:end]` 对字符串进行切片。负值表示相对于字符串末尾的位置：

```vala
string greeting = "hello, world";
string s1 = greeting[7:12];        // => "world"
string s2 = greeting[-4:-2];       // => "or
```

请注意，与大多数其他编程语言一样，Vala 中的索引以 0 开始。从 Vala 0.11 开始，您可以使用 `[index]` 访问字符串的单字节：

```vala
uint8 b = greeting[7];             // => 0x77
```

但是，由于 Vala 字符串是不可变的，因此不能为该位置分配新的字节值。

例如，许多基本类型都有合理的字符串解析和转换方法：

```vala
bool b = bool.parse("false");           // => false
int i = int.parse("-52");               // => -52
double d = double.parse("6.67428E-11"); // => 6.67428E-11
string s1 = true.to_string();           // => "true"
string s2 = 21.to_string();             // => "21"
```

向控制台写入字符串和从控制台读取字符串的两个有用方法（也是您首次探索 Vala 的方法）是`stdout.printf()`和`stdin.read_line()`：

你已经在Hello World示例中了解了`stdout.printf()`。实际上，它可以接受任意多个不同类型的参数，而第一个参数是一个格式字符串，遵循与[C 格式字符串](https://en.wikipedia.org/wiki/Printf)相同的规则。如果必须输出错误信息，可以使用`stderr.printf()` 代替`stdout.printf()`。

此外，`in`操作还可用于确定一个字符串是否包含另一个字符串，例如

```vala
if ("ere" in "Able was I ere I saw Elba.") ...
```

如需了解更多信息，请参阅[字符串类的完整概述](https://www.valadoc.org/glib-2.0/string.html)。

此外，还提供了一个演示字符串用法的[示例程序](https://wiki.gnome.org/Projects/Vala/StringSample)。

#### 数组

数组的声明方式是给出一个类型名称，后面跟上`[]`，然后使用 new 操作符创建数组，例如，`int[] a = new int[10]` 可以创建一个整数数组。这样一个数组的长度可以通过`length`成员变量获得，例如 `int count = a.length`。请注意，如果写 `Object[] a = new Object[10]`，则不会创建任何对象，只会创建一个数组来存储这些对象。

```vala
int[] a = new int[10];
int[] b = { 2, 4, 6, 8 };
```

您可以使用 `[start:end]` 对数组进行切片：

```vala
int[] c = b[1:3];     // => { 4, 6 }
```

对数组进行切片将导致对请求数据的引用，而不是拷贝。然而，将切片赋值给一个自有变量（如上所述）会导致拷贝。如果想避免拷贝，必须将切片赋值给一个非自有数组，或者直接将其传递给一个参数（默认情况下，参数是非自有的）：

```vala
unowned int[] c = b[1:3];     // => { 4, 6 }
```

多维数组用`[,]`或`[,,]`等定义。

```vala
int[,] c = new int[3,4];
int[,] d = { {2, 4, 6, 8},
            {3, 5, 7, 9},
            {1, 3, 5, 7} };
d[2,3] = 42;
```

这种数组由一个连续的内存块表示。目前还不支持锯齿状多维数组（`[][]`，也称为 "堆叠数组 "或 "数组的数组"），其中每一行的长度可能不同。

要查找多维数组中每个维度的长度，`length`成员会变成一个数组，存储每个维度的长度。

```vala
int[,] arr = new int[4,5];
int r = arr.length[0];
int c = arr.length[1];
```

请注意，您无法从多维数组获取单维数组，甚至无法对多维数组进行切片：

```vala
int[,] arr = { {1,2},
                {3,4} };
int[] b = arr[0];  // won't work
int[] c = arr[0,];  // won't work
int[] d = arr[:,0];  // won't work
int[] e = arr[0:1,0];  // won't work
int[,] f = arr[0:1,0:1];  // won't work
```

您可以使用 += 操作符动态追加数组元素。不过，这只适用于本地定义的数组或私有数组。如果需要，数组会自动重新分配。在内部，出于提高运行效率的考虑，这种重新分配会以 2 的幂级数增长的大小进行。不过，.length 保存的是元素的实际数量，而不是内部大小。

```vala
int[] e = {};
e += 12;
e += 5;
e += 37;
```

您可以通过调用`resize()`来调整数组的大小。它将保留原来的内容（尽可能合适）。

```vala
int[] a = new int[5];
a.resize(12);
```

通过调用`move(src, dest, length)`可以移动数组中的元素。原来的位置将被填充为 0。

```vala
uint8[] chars = "hello world".data;
chars.move (6, 0, 5);
print ((string) chars); // "world "
```

如果在标识符后加上方括号，并注明大小，就会得到一个固定大小的数组。固定大小的数组会在堆栈中分配（如果用作局部变量）或内联分配（如果用作字段），而且以后不能重新分配。

```vala
int f[10];     // no 'new ...'
```

Vala 不会在运行时对数组访问进行任何边界检查。如果需要更高的安全性，则应使用更复杂的数据结构，如`ArrayList`。您将在后面有关集合的章节中了解更多相关信息。

#### 引用类型

引用类型是所有声明为类的类型，无论它们是否是 GLib对象的后代。Vala 将确保当你通过引用传递对象时，系统将跟踪当前存活的引用数量，以便为你管理内存。不指向任何地方的引用值为空。有关类及其功能的更多信息，请参阅面向对象编程部分。

```vala
/* defining a class */
class Track : GLib.Object {             /* subclassing 'GLib.Object' */
    public double mass;                 /* a public field */
    public double name { get; set; }    /* a public property */
    private bool terminated = false;    /* a private field */
    public void terminate() {           /* a public method */
        terminated = true;
    }
}
```

#### 静态类型转换

在 Vala 中，您可以将变量从一种类型转换为另一种类型。对于静态类型转换，变量是通过所需的类型名称和括号进行转换的。静态类型转换不进行任何运行时类型安全检查。它适用于所有 Vala 类型。例如

```vala
int i = 10;
float j = (float) i;
```

Vala 还支持另一种称为*动态转换*的转换机制，该机制可执行运行时类型检查，在面向对象编程一节中进行了介绍。

#### 类型推断

Vala 有一种称为“类型推断”的机制，根据这种机制，局部变量可以使用 `var` 来定义，而不用给出类型，只要能够明确是什么类型即可。类型是从赋值的右侧推断出来的。这有助于减少代码中不必要的冗余，同时又不会牺牲静态类型：

```vala
var p = new Person();     // same as: Person p = new Person();
var s = "hello";          // same as: string s = "hello";
var l = new List<int>();  // same as: List<int> l = new List<int>();
var i = 10;               // same as: int i = 10;
```

这只对局部变量有效。类型推断对带有泛型参数的类型尤其有用（稍后将详细介绍）。比较

```vala
MyFoo<string, MyBar<string, int>> foo = new MyFoo<string, MyBar<string, int>>();
```

对

```vala
var foo = new MyFoo<string, MyBar<string, int>>();
```

#### 从其他类型定义新类型

定义新类型只需从您需要的类型中派生出来即可。下面是一个例子：

```vala
/* Define a new type from a container like GLib.List with elements type GLib.Value */
public class ValueList : GLib.List<GLib.Value> {
        [CCode (has_construct_function = false)]
        protected ValueList ();
        public static GLib.Type get_type ();
}
```

### 操作符

```
=
```

赋值。左操作数必须是标识符，右操作数必须是值或引用。


```
+, -, /, *, %
```

基本运算，适用于左右操作数。+ 运算符还可以连接字符串。


```
+=, -=, /=, *=, %=
```

在左操作数和右操作数之间进行算术运算，其中左操作数必须是标识符，运算结果将赋值给该标识符。

```
++, --
```

隐式赋值的递增和递减操作。这些操作只需要一个参数，参数必须是简单数据类型的标识符。数值将被更改并赋值给标识符。这些运算符可以放在前缀或后缀位置，前者的语句值将是新计算出的值，后者的语句值将返回原来的值。

```
|, ^, &, ~, |=, &=, ^=
```

比特运算：或、异或、与、非。第二组运算包括赋值，与算术运算类似。这些操作可以应用于任何简单值类型。（由于 `~` 是一元运算符，因此没有赋值运算符。等价操作是 `a = ~a`）。

```
<<, >>
```

位移操作，根据右操作数将左操作数位移若干位。

```
<<=, >>=
```

位移操作，根据右操作数将左操作数位移若干位。左操作数必须是一个标识符，结果将分配给该标识符。

```
==
```

相等测试。根据左操作数和右操作数是否相等，计算出一个 bool 值。对于值类型，这意味着它们的值相等；对于引用类型，这意味着对象是相同的实例。字符串类型是一个例外，它是通过值进行等价测试的。

```
<, >, >=, <=, !=
```

不等式测试。根据左操作数和右操作数是否以所述方式不同，求值为一个 bool 值。这些运算符对简单值数据类型和字符串类型有效。对于字符串，这些运算符会比较词序。

```
!, &&, ||
```

逻辑运算：非、与、或。这些运算可用于布尔值——前者只有一个参数，后者有两个参数。


```
? :
```

三元条件运算符。根据条件为真或假，评估条件并返回左侧或右侧子表达式的值：`condition ? value if true : value if false`。

```
??
```

可空运算符：`a ?? b` 相当于 `a != null ? a : b`。例如，该操作符可在引用为空时提供默认值：

```vala
stdout.printf("Hello, %s!\n", name ?? "unknown person");
```

```
in
```

检查右操作数是否包含左操作数。该操作符适用于数组、字符串、集合或任何其他具有相应`contains()`方法的类型。对于字符串，它会执行子串搜索。

在 Vala 中，操作符不能重载。在 lambda 声明和其他特定任务中，有一些额外的操作符是有效的——这些操作符将在其适用的上下文中进行解释。

### 控制结构

```vala
while (a > b) { a--; }
```

会重复递减`a`，每次迭代前都会检查`a`是否大于`b`。

```vala
do { a--; } while (a > b);
```

会重复递减`a`，每次迭代后都会检查`a`是否大于`b`。

```vala
for (int a = 0; a < 10; a++) { stdout.printf("%d\n", a); }
```

会将`a`初始化为 0，然后重复打印`a`，直到`a`不再小于 10，每次迭代后`a`都会递增。

```vala
foreach (int a in int_array) { stdout.printf("%d\n", a); }
```

将打印出数组或其他可迭代集合中的每个整数。关于 "可迭代 "的含义，稍后会有说明。

前面四种类型的循环都可以用关键字 break 和 continue 来控制。break指令将使循环立即终止，而continue指令将直接跳转到迭代的测试部分。

```vala
if (a > 0) { stdout.printf("a is greater than 0\n"); }
else if (a < 0) { stdout.printf("a is less than 0\n"); }
else { stdout.printf("a is equal to 0\n"); }
```

根据一组条件执行一段特定代码。第一个匹配条件决定执行哪段代码，如果`a`大于 0，则不测试它是否小于 0。

```vala
switch (a) {
case 1:
    stdout.printf("one\n");
    break;
case 2:
case 3:
    stdout.printf("two or three\n");
    break;
default:
    stdout.printf("unknown\n");
    break;
}
```

`switch`语句根据传递给它的值运行一段或零段代码。在 Vala 中，匹配case后，不会做任何跳转。为了确保这一点，每个非空情况都必须以 `break`、`return` 或 `throw` 语句结束。在字符串中也可以使用 switch 语句。

C 语言程序员须知：条件必须始终求值为布尔值。这意味着如果要检查变量是否为 null 或 0，必须明确地执行： `if (object != null) { }` 或 `if (number != 0) { }`。

### 语言要素

#### 方法

在 Vala 中，无论函数是否在类中定义，它们都被称为方法。从现在起，我们将使用方法一词。

```vala
int method_name(int arg1, Object arg2) {
    return 1;
}
```

这段代码定义了一个名为`method_name` 的方法，它接受两个参数，一个是整数，另一个是对象（第一个参数按值传递，第二个参数作为引用传递，如前所述）。该方法将返回一个整数，在本例中为 1。

所有 Vala 方法都是 C 函数，因此可以接受任意数量的参数并返回一个值（如果方法声明为`void`，则不返回任何值）。它们可以通过在调用代码的固定位置放置数据来来获得更多的返回值。本教程高级部分的 “参数方向”章节将详细介绍如何做到这一点。

Vala 中方法的命名约定是全小写，下划线作为单词分隔符。这对于习惯于使用驼峰方法名的 C# 或 Java 程序员来说可能有些陌生。但使用这种风格，您将与其他 Vala 和 C/GObject 库保持一致。

在同一作用域内不能有多个名称相同但签名不同的方法（“方法重载”）：

```vala
void draw(string text) { }
void draw(Shape shape) { }  // not possible
```

这是因为使用 Vala 生成的库也要供 C 程序员使用。在 Vala 中，您可以这样做：

```vala
void draw_text(string text) { }
void draw_shape(Shape shape) { }
```

选择稍有不同的名称可以避免名称冲突。在支持方法重载的语言中，方法重载通常用于提供方便的方法，这些方法的参数更少，但可以链到最通用的方法：

```vala
void f(int x, string s, double z) { }
void f(int x, string s) { f(x, s, 0.5); }  // not possible
void f(int x) { f(x, "hello"); }           // not possible
```

在这种情况下，您可以使用 Vala 为方法参数提供的默认参数功能，以便只用一个方法就能实现类似的行为。您可以为方法的最后一个参数定义默认值，这样就不必在方法调用中明确传递这些参数：

```vala
void f(int x, string s = "hello", double z = 0.5) { }
```

这种方法的一些可能调用方式如下：

```vala
f(2);
f(2, "hi");
f(2, "hi", 0.75);
```

甚至还可以使用真正的可变长度参数列表（`varargs`）来定义方法，如`stdout.printf()`，但不一定推荐使用。稍后您将学习如何做到这一点。

Vala 会对方法参数和返回值进行基本的无效性检查。如果允许方法参数或返回值为空，则应在类型符号后添加 `?` 修饰符。这些额外的信息有助于 Vala 编译器执行静态检查，并为方法的先决条件添加运行时断言，这可能有助于避免相关错误（如取消引用空引用）。

```vala
string? method_name(string? text, Foo? foo, Bar bar) {
    // ...
}
```

在这个示例中，`text`、`foo` 和返回值可以为空，但 `bar` 不能为空。

#### 委托

```vala
delegate void DelegateType(int a);
```

委托代表方法，允许代码块像对象一样传递。上面的示例定义了一种名为`DelegateType`的新类型，它代表参数值为`int`但不返回值的方法。任何符合该签名的方法都可以赋值给该类型的变量或作为该类型的方法参数传递。

```vala
delegate void DelegateType(int a);

void f1(int a) {
    stdout.printf("%d\n", a);
}

void f2(DelegateType d, int a) {
    d(a);       // Calling a delegate
}

void main() {
    f2(f1, 5);  // Passing a method as delegate argument to another method
}
```

这段代码将执行方法`f2`，并将方法`f1`的引用和数字 5 传递给它。

委托也可以在本地创建。成员方法也可以分配给一个委托，例如：

```vala
class Foo {

    public void f1(int a) {
        stdout.printf("a = %d\n", a);
    }

    delegate void DelegateType(int a);

    public static int main(string[] args) {
        Foo foo = new Foo();
        DelegateType d1 = foo.f1;
        d1(10);
        return 0;
    }
}
```

更多示例请参考[委托手册](https://wiki.gnome.org/Projects/Vala/Manual/Delegates)。

#### 匿名方法和闭包

```vala
(a) => { stdout.printf("%d\n", a); }
```

在 Vala 中，可以使用 => 操作符定义匿名方法（也称为lambda 表达式、函数文字或闭包）。参数列表位于运算符的左侧，方法主体位于运算符的右侧。

像上面这样的匿名方法本身并没有什么意义。只有将匿名方法直接赋值给一个委托类型的变量，或将其作为方法参数传递给另一个方法时，匿名方法才会有用。

请注意，参数和返回类型都没有明确给出。相反，这些类型是根据与之配合使用的委托的签名推断出来的。

将匿名方法赋值给委托变量：

```vala
delegate void PrintIntFunc(int a);

void main() {
    PrintIntFunc p1 = (a) => { stdout.printf("%d\n", a); };
    p1(10);

    // Curly braces are optional if the body contains only one statement:
    PrintIntFunc p2 = (a) => stdout.printf("%d\n", a);
    p2(20);
}
```

将匿名方法传递给另一个方法：

```vala
delegate int Comparator(int a, int b);

void my_sorting_algorithm(int[] data, Comparator compare) {
    // ... 'compare' is called somewhere in here ...
}

void main() {
    int[] data = { 3, 9, 2, 7, 5 };
    // An anonymous method is passed as the second argument:
    my_sorting_algorithm(data, (a, b) => {
        if (a < b) return -1;
        if (a > b) return 1;
        return 0;
    });
}
```

匿名方法是真正的[闭包](https://en.wikipedia.org/wiki/Closure_(computer_science))。这意味着你可以在 lambda 表达式中访问外层方法的局部变量：

```vala
delegate int IntOperation(int i);

IntOperation curried_add(int a) {
    return (b) => a + b;  // 'a' is an outer variable
}

void main() {
    stdout.printf("2 + 4 = %d\n", curried_add(2)(4));
}
```

在本例中，`curried_add`（参见[Currying](https://en.wikipedia.org/wiki/Currying)）返回了一个新创建的方法，该方法保留了`a`的值。

#### 命名空间

```vala
namespace NameSpaceName {
    // ...
}
```

本语句中括号之间的所有内容都属于名称空间`NameSpaceName`，必须以此为引用。该命名空间之外的任何代码都必须对命名空间名称内的任何内容使用限定名称，或者在文件中使用适当的 using 声明，以便导入该命名空间：

```vala
using NameSpaceName;

// ...

```

例如，如果名称空间`Gtk`是使用 `using Gtk;` 导入的，则可以简单地写成`Window`而不是`Gtk.Window`。只有在模棱两可的情况下，例如在`GLib.Object`和`Gtk.Object`之间，才有必要使用完全限定名称。

GLib命名空间是默认导入的。想象一下，在每个 Vala 文件的开头都有一行不可见的 `using GLib;`。

所有未放入独立命名空间的内容都将进入匿名全局命名空间。如果由于歧义而必须明确引用全局命名空间，可以使用 `global::` 前缀。

命名空间可以嵌套，可以是一个声明嵌套在另一个声明中，也可以是`NameSpace1.NameSpace2`的形式。

其他几种类型的定义也可以通过遵循相同的命名约定将自己声明为名称空间的一部分，例如：`class NameSpace1.Test { ... }`。请注意，这样做时，定义的最终命名空间将是声明嵌套的命名空间加上定义中声明的命名空间。

#### 结构体

```vala
struct StructName {
    public int a;
}
```

定义了一种结构类型，即复合值类型。Vala 结构体可以有有限的方法，也可以有私有成员，这意味着需要显式公共访问修饰符。

```vala
struct Color {
    public double red;
    public double green;
    public double blue;
}
```

这就是初始化结构的方法：

```vala
// without type inference
Color c1 = Color();  // or Color c1 = {};
Color c2 = { 0.5, 0.5, 1.0 };
Color c3 = Color() {
    red = 0.5,
    green = 0.5,
    blue = 1.0
};

// with type inference
var c4 = Color();
var c5 = Color() {
    red = 0.5,
    green = 0.5,
    blue = 1.0
};
```

结构体在堆栈或内联分配，并在赋值时复制。

要定义结构体数组，请参阅[常见问题](https://wiki.gnome.org/Projects/Vala/FAQ#How_do_I_create_an_array_of_structs.3F)。

#### 类

```vala
class ClassName : SuperClassName, InterfaceName {
}
```

定义了一个类，即引用类型。与结构体不同，类的实例是堆分配的。与类相关的语法还有很多，将在面向对象编程一节中进行更全面的讨论。

#### 接口

```vala
interface InterfaceName : SuperInterfaceName {
}
```

定义了一个接口，即不可实例化的类型。要创建接口的实例，必须先在非抽象类中实现接口的抽象方法。Vala 接口比 Java 或 C# 接口更强大。事实上，它们可以作为[混合体](https://en.wikipedia.org/wiki/Mixin)使用。有关接口的详细信息，请参阅面向对象编程一节。

### 代码属性

代码属性向 Vala 编译器提供代码在目标平台上运行的详细信息。其语法为 `[AttributeName]` 或 `[AttributeName(param1=value1,param2=value2,...)]`。

它们主要用于vapi文件中的绑定，`[CCode(...)]` 是其中最突出的属性。另一个例子是用于通过[D-Bus](https://www.freedesktop.org/wiki/Software/dbus) 输出远程接口的 `[DBus(...)]` 属性。

## 面向对象程序设计

虽然 Vala 并不强迫你使用对象，但有些功能是其他方式无法实现的。因此，在大多数情况下，您肯定希望以面向对象的方式进行编程。与当前大多数语言一样，为了定义自己的对象类型，您需要编写一个类定义。

类定义说明了其类型的每个对象拥有哪些数据、可以包含哪些其他对象类型的引用，以及可以执行哪些方法。定义还可以包括另一个类的名称，新类应是该类的子类。类的实例也是所有超类的实例，因为它继承了所有超类的方法和数据，尽管它自己可能无法访问全部的内容。类还可以实现任意数量的接口，这些接口是类必须实现的方法集——类的实例也是其类或超类的每个接口的实例。

Vala 中的类也可以有“静态”成员。这种修饰符允许将数据或方法定义为属于整个类，而不是属于它的某个特定实例。此类成员无需拥有该类的实例即可访问。

### 基础知识

简单类的定义如下

```vala
public class TestClass : GLib.Object {

    /* Fields */
    public int first_data = 0;
    private int second_data;

    /* Constructor */
    public TestClass() {
        this.second_data = 5;
    }

    /* Method */
    public int method_1() {
        stdout.printf("private data: %d", this.second_data);
        return this.second_data;
    }
}
```

这段代码将定义一个包含三个成员的新类型（已在`gobject`库的类型系统中自动注册）。其中有两个数据成员，即顶部定义的整数，以及一个名为`method_1` 的方法，该方法返回一个整数。类声明指出，该类是`GLib.Object` 的子类，因此其实例也是`Object`对象的实例，并包含该类型的所有成员。该类是`Object`的子类这一事实也意味着 Vala 的一些特殊功能可用于轻松访问`Object`的某些功能。

这个类被描述为公共类（默认情况下，类是内部的）。这意味着它可以被该文件之外的代码直接引用——如果你是一个使用 glib/gobject 的 C 语言程序员，你会发现这相当于在头文件中定义了类的接口，其他代码可以将其包含在内。

这些成员也都被描述为公共或私有。成员`first_data`是公共的，因此该类的任何用户都可以直接看到它，并且可以在包含实例的情况下进行修改。第二个数据成员是私有的，因此只有属于该类的代码才能引用。Vala 支持四种不同的访问修饰符：

- `public`： 访问无限制
- `private`： 访问仅限于类/结构体定义内部。如果未指定访问修饰符，则默认使用此方法
- `protected`： 访问仅限于类的定义和从该类继承的任何类
- `internal`： 访问仅限于同一软件包中定义的类

构造函数用于初始化类的新实例。构造函数的名称与类的名称相同，可以接受 0 个或多个参数，并且没有返回类型。

该类的最后一部分是方法定义。该方法名为 `method_1`，它将返回一个整数。由于该方法不是静态方法，它只能在该类的实例上执行，因此可以访问该实例的成员。它可以通过 `this` 引用来实现这一点，该引用始终指向方法被调用的实例。除非有歧义，否则可以省略 `this` 标识符。

您可以如下使用这个新类：

```vala
TestClass t = new TestClass();
t.first_data = 5;
t.method_1();
```

### 构造

Vala 支持两种略有不同的构造方案：一种是 Java/C# 风格的构造方案，我们现在主要讨论这种方案；另一种是 GObject 风格的构造方案，我们将在本章末尾的一节中介绍这种方案。

Vala 不支持构造函数重载，原因与不允许方法重载相同，这意味着一个类不能有多个同名的构造函数。不过，这不是问题，因为 Vala 支持命名的构造函数。如果您想提供多个构造函数，可以给它们添加不同的名称：

```vala
public class Button : Object {

    public Button() {
    }

    public Button.with_label(string label) {
    }

    public Button.from_stock(string stock_id) {
    }
}
```

实例化与此类似：

```vala
new Button();
new Button.with_label("Click me");
new Button.from_stock(Gtk.STOCK_OK);
```

您可以通过 this() 或 this_.name\_extension_() 链接构造函数：

```vala
public class Point : Object {
    public double x;
    public double y;

    public Point(double x, double y) {
        this.x = x;
        this.y = y;
    }

    public Point.rectangular(double x, double y) {
        this(x, y);
    }

    public Point.polar(double radius, double angle) {
        this.rectangular(radius * Math.cos(angle), radius * Math.sin(angle));
    }
}

void main() {
    var p1 = new Point.rectangular(5.7, 1.2);
    var p2 = new Point.polar(5.7, 1.2);
}
```

### 析构

虽然 Vala 会为您管理内存，但如果您选择使用指针进行手动内存管理（稍后详述）或必须释放其他资源，则可能需要添加自己的析构函数。语法与 C# 和 C++ 相同：

```vala
class Demo : Object {
    ~Demo() {
        stdout.printf("in destructor");
    }
}
```

由于 Vala 的内存管理基于引用计数，而不是跟踪垃圾回收，因此析构函数是确定的，可用于实现资源管理的[RAII](https://en.wikipedia.org/wiki/RAII)模式（关闭流、数据库连接......）。

### 信号

信号是 GLib 中Object类提供的一个系统，Vala 使Object的所有子类都能轻松访问信号。对于 C# 程序员来说，信号就像一个事件，而对于 Java 程序员来说，信号则是实现事件监听器的另一种方式。简而言之，信号只是在大致相同的时间执行任意数量的外部相同方法（即具有相同签名的方法）的一种方式。实际的执行方法在gobject内部，对 Vala 程序来说并不重要。

信号被定义为类的一个成员，看起来类似于没有主体的方法。信号处理程序可以通过 `connect()` 方法添加到信号中。为了深入浅出，下面的示例还介绍了 lambda 表达式，这是一种在 Vala 中编写信号处理代码的非常有用的方法：

```vala
public class Test : GLib.Object {

    public signal void sig_1(int a);

    public static int main(string[] args) {
        Test t1 = new Test();

        t1.sig_1.connect((t, a) => {
            stdout.printf("%d\n", a);
        });

        t1.sig_1(5);

        return 0;
    }
}
```

这段代码使用熟悉的语法引入了一个名为“Test”的新类。该类的第一个成员是一个信号，名为 `sig_1`，它被定义为传递一个整数。在该程序的主方法中，我们首先创建了一个 Test 实例——这是一个必要条件，因为信号总是属于类的实例。然后，我们为实例的 `sig_1` 信号分配一个处理程序，并将其定义为一个 lambda 表达式。该定义指出，该方法将接收两个参数，我们称之为 `t`和 `a`，但没有提供参数类型。我们可以如此简洁，因为 Vala 已经知道信号的定义，因此可以理解需要哪些类型。

处理程序之所以有两个参数，是因为每当发出一个信号时，该信号所指向的对象就会作为第一个参数传递给处理程序。第二个参数是信号说要提供的参数。

最后，我们迫不及待，决定发出一个信号。我们的做法是调用信号，就好像它是我们类中的一个方法，然后让 gobject 负责将消息转发给所有附加的处理程序。使用 Vala 的信号并不需要了解其中的机制。

注：目前，`public`访问修改器是唯一可能的选项——所有信号都可以连接到任何代码，也可以由任何代码发出。

信号可以用任意标志组合进行注释：

```vala
[Signal (action=true, detailed=true, run=true, no_recurse=true, no_hooks=true)]
    public signal void sig_1 ();
```

更多例子请参考[信号](https://wiki.gnome.org/Projects/Vala/SignalsAndCallbacks)。

### 属性

向类的用户隐藏实现细节[（信息隐藏原则](https://en.wikipedia.org/wiki/Information_hiding)）是面向对象编程的良好做法，这样以后就可以在不破坏公共应用程序接口的情况下更改内部结构。一种做法是将字段设为私有，并提供获取和设置其值的访问方法（getters 和 setters）。

如果你是一名 Java 程序员，你可能会想到这样的事情：

```vala
class Person : Object {
    private int age = 32;

    public int get_age() {
        return this.age;
    }

    public void set_age(int age) {
        this.age = age;
    }
}
```

这种方法可行，但Vala可以做得更好。问题是这些方法操作起来很麻烦。假设您想把一个人的年龄增加一岁：

```vala
var alice = new Person();
alice.set_age(alice.get_age() + 1);
```

这就是属性发挥作用的地方：

```vala
class Person : Object {
    private int _age = 32;  // underscore prefix to avoid name clash with property

    /* Property */
    public int age {
        get { return _age; }
        set { _age = value; }
    }
}
```

C# 程序员对这种语法应该不陌生。一个属性有一个 get 和一个 set 块，用于获取和设置其值。

现在，你可以像访问公共字段一样访问该属性。但在幕后，get 和 set 块中的代码会被执行。

```vala
var alice = new Person();
alice.age = alice.age + 1;  // or even shorter:
alice.age++;
```

如果只按上面所示的标准实现，那么属性的写法会更简短：

```vala
class Person : Object {
    /* Property with standard getter and setter and default value */
    public int age { get; set; default = 32; }
}
```

有了属性，你就可以在不改变公共 API 的情况下改变类的内部工作方式。例如

```vala
static int current_year = 2525;

class Person : Object {
    private int year_of_birth = 2493;

    public int age {
        get { return current_year - year_of_birth; }
        set { year_of_birth = current_year - value; }
    }
}
```

这一次，年龄是根据出生年份即时计算出来的。请注意，在 get 和 set 块中，您可以做的不仅仅是简单的变量访问或赋值。您还可以访问数据库、记录日志、更新缓存等。

如果想让类的用户对属性只读，则应将设置器设置为私有：

```vala
public int age { get; private set; default = 32; }
```

或者，您也可以省略设置块：

```vala
class Person : Object {
    private int _age = 32;

    public int age {
        get { return _age; }
    }
}
```

属性不仅可以有名称，还可以有简短描述（称为`nick`）和详细描述（称为`blurb`）。您可以使用特殊属性对其进行注释：

```vala
    [Description(nick = "age in years", blurb = "This is the person's age in years")]
    public int age { get; set; default = 32; }
```

属性及其附加说明可在运行时查询。一些程序（如[Glade](https://glade.gnome.org/)图形用户界面设计器）可以使用这些信息。通过这种方式，Glade 可以为 GTK+ 部件的属性提供人类可读的描述。

每个派生自 `GLib.Object` 的类实例都有一个名为 `notify` 的信号。每次对象的某个属性发生变化时，都会发出该信号。因此，如果您对一般的更改通知感兴趣，可以连接到此信号：

```vala
obj.notify.connect((s, p) => {
    stdout.printf("Property '%s' has changed!\n", p.name);
});
```

`s` 是信号源（本例中为 `obj`），`p` 是已更改属性的`ParamSpec`类型的属性信息。如果您只对单个属性的更改通知感兴趣，可以使用此语法：

```vala
alice.notify["age"].connect((s, p) => {
    stdout.printf("age has changed\n");
});
```

请注意，在这种情况下，您必须使用属性名称的字符串表示法，下划线由破折号代替：在这种表示法中，`my_property_name` 变成了 `"my-property-name"`，这是 GObject 属性命名约定。

可以在属性声明前使用 `CCode` 属性标签禁用更改通知：

```vala
public class MyObject : Object {
    [CCode(notify = false)]
    // notify signal is NOT emitted upon changes in the property
    public int without_notification { get; set; }
    // notify signal is emitted upon changes in the property
    public int with_notification { get; set; }
}
```

还有一种属性叫做构造属性，将在后面有关 gobject 风格构造的章节中介绍。

注意：如果属性是结构体类型，要使用 Object.get() 获取属性值，必须像下面的示例一样声明变量：

```vala
struct Color
{
    public uint32 argb;

    public Color() { argb = 0x12345678; }
}

class Shape: GLib.Object
{
    public Color c { get; set; default = Color(); }
}

int main()
{
    Color? c = null;
    Shape s = new Shape();
    s.get("c", out c);
}
```

这样，`c` 就是一个引用，而不是堆栈中的 `Color` 实例。你传入 s.get() 的是 `Color **`，而不是 `Color *`。

### 继承

在 Vala 中，一个类可能派生自一个或零个其他类。在实践中，虽然没有像 Java 等语言中那样的隐式继承，但一般都会采用继承。

在定义一个从另一个继承的类时，你会在类之间创建一种关系，即子类的实例也是超类的实例。这意味着对超类实例的操作也适用于子类实例。因此，只要需要超类的实例，就可以用子类的实例来代替。

在编写类的定义时，可以精确控制谁可以访问对象中的哪些方法和数据。下面的示例演示了这些选项的范围：

```vala
class SuperClass : GLib.Object {

    private int data;

    public SuperClass(int data) {
        this.data = data;
    }

    protected void protected_method() {
    }

    public static void public_static_method() {
    }
}

class SubClass : SuperClass {

    public SubClass() {
        base(10);
    }
}
```

`data`是`SuperClass` 的实例数据成员。`SuperClass` 的每个实例中都有一个这种类型的成员，它被声明为私有，因此只有作为`SuperClass` 一部分的代码才能访问。

`protected_method`是`SuperClass` 的实例方法。您只能在`SuperClass`或其子类的实例中执行该方法，而且只能在属于`SuperClass`或其子类的代码中执行——后一条规则是 protected 修饰符的结果。

`public_static_method_`有两个修饰符。`static` 修饰符表示调用此方法时，无需拥有`SuperClass`或其子类的实例。因此，此方法在执行时将无法访问`this`引用。`public` 修饰符的意思是，无论该方法与`SuperClass`或其子类的关系如何，都可以从任何代码中调用。

根据这些定义，`SubClass`的实例将包含`SuperClass` 的所有三个成员，但只能访问非私有成员。外部代码只能访问公共方法。

有了基类，子类的构造函数就可以连锁到基类的构造函数。

### 抽象类

方法还有另一种修饰符，叫做抽象（`abstract`）。使用该修饰符，可以描述一个在类中并未实际实现的方法。相反，它必须由子类实现后才能调用。这样，您就可以定义可在一个类型的所有实例上调用的操作，同时确保所有更具体的类型都能提供自己版本的功能。

包含抽象方法的类也必须声明为抽象类。这样做的结果是阻止该类型的任何实例化。

```vala
public abstract class Animal : Object {

    public void eat() {
        stdout.printf("*chomp chomp*\n");
    }

    public abstract void say_hello();
}

public class Tiger : Animal {

    public override void say_hello() {
        stdout.printf("*roar*\n");
    }
}

public class Duck : Animal {

    public override void say_hello() {
        stdout.printf("*quack*\n");
    }
}
```

抽象方法的实现必须标记为`override`。属性也可以是抽象的。

#### 虚拟方法

虚拟方法允许为抽象类定义默认实现，并允许派生类覆盖其行为，这与隐藏方法不同。

```vala
public abstract class Caller : GLib.Object {
   public abstract string name { get; protected set; }
   public abstract void update (string new_name);
   public virtual bool reset ()
   {
      name = "No Name";
      return true;
   }
}

public class ContactCV : Caller
{
   public override string name { get; protected set; }
   public override void update (string new_name)
   {
     name = "ContactCV - " + new_name;
   }
   public override bool reset ()
   {
      name = "ContactCV-Name";
      stdout.printf ("CotactCV.reset () implementation!\n");
      return true;
   }
}

public class Contact : Caller {
   public override string name { get; protected set; }
   public override void update (string new_name)
   {
     name = "Contact - " + new_name;
   }

   public static void main ()
   {
      var c = new Contact ();
      c.update ("John Strauss");
      stdout.printf(@"Name: $(c.name)\n");
      c.reset ();
      stdout.printf(@"Reset Name: $(c.name)\n");

      var cv = new ContactCV ();
      cv.update ("Xochitl Calva");
      stdout.printf(@"Name: $(cv.name)\n");
      cv.reset ();
      stdout.printf(@"Reset Name: $(cv.name)\n");
      stdout.printf("END\n");
   }
}
```

如上例所示，`Caller` 是一个抽象类，定义了一个抽象属性和一个方法，但增加了一个可被派生类覆盖的虚拟方法。`Contact` 类实现了 `Caller` 的抽象方法和属性，同时使用了 `reset()` 的默认实现，避免了定义新的实现。`ContactCV` 类实现了 `Caller` 的所有抽象定义，但重载了 `reset()`，以便定义自己的实现。

### 接口

Vala 中的一个类可以实现任意数量的接口。每个接口都是一种类型，很像类，但不能实例化。通过“实现”一个或多个接口，类可以声明其实例也是接口的实例，因此可以在任何需要该接口实例的情况下使用。

实现接口的程序与继承具有抽象方法的类的程序相同——如果类要有用，就必须为所有已描述但尚未实现的方法提供实现。

简单的接口定义如下：

```vala
public interface ITest : GLib.Object {
    public abstract int data_1 { get; set; }
    public abstract void method_1();
}
```

这段代码描述了一个接口 "ITest"，它要求 GLib.Object 作为实现类的父类，并包含两个成员。"data\_1" 是一个属性，如上所述，但它被声明为抽象属性。因此，Vala 将不实现该属性，而是要求实现该接口的类具有一个名为 "data\_1 "的属性，该属性具有 get 和 set 访问器——要求该属性必须是抽象的，因为接口可能没有任何数据成员。第二个成员 "method\_1 "是一个方法。这里声明该方法必须由实现该接口的类来实现。

该接口最简单的完整实现方式是：

```vala
public class Test1 : GLib.Object, ITest {
    public int data_1 { get; set; }
    public void method_1() {
    }
}
```

使用方法如下：

```vala
var t = new Test1();
t.method_1();

ITest i = t;
i.method_1();
```

#### 确定先决条件

Vala 中的接口不能从其他接口继承，但它们可以声明其他接口为先决条件，其工作方式大致相同。例如，我们可以规定，任何实现 List 接口的类必须同时实现 `Collection` 和 `Traversable` 接口。其语法与在类中描述接口实现的语法完全相同：

```vala
public interface List : Collection, Traversable {
}
```

因此，对于希望实现“List”的类，Vala 强制执行以下声明样式，其中必须描述所有已实现的接口：

```vala
public class ListClass : GLib.Object, Collection, List {
}
```

Vala 接口也可以有一个类作为先决条件。如果在先决条件列表中给出了类名，那么接口只能在派生自该先决条件类的类中实现。这通常用于确保接口的实例也是 GLib.Object 的子类，因此接口可用作属性类型等。

接口不能继承其他接口这一事实主要只是技术上的区别——实际上，Vala 的系统在这方面与其他语言的工作原理相同，只是多了一个先决类的功能。

#### 在方法中定义默认实现

Vala 接口与 Java/C# 接口还有一个重要区别：Vala 接口可能有非抽象方法。

Vala 实际上允许在接口中实现方法，那么具有默认实现的方法必须声明为虚拟方法。基于这一事实，Vala 接口可以作为[混合接口（mixins](https://en.wikipedia.org/wiki/Mixins)）。这是多重继承的一种受限形式。

```vala
public interface Callable : GLib.Object {
   public abstract bool answering { get; protected set; }
   public abstract void answer ();
   public virtual bool hang ()
   {
      answering = false;
      return true;
   }
}
```

接口 Callable 定义了一个名为应答的抽象属性，任何实现该接口的类都可以监控呼叫的状态，关于应答呼叫的细节由实现者自行决定，但 hang 定义了一个默认实现，即在挂断呼叫时将应答设置为 false。

```vala
public class Phone : GLib.Object, Callable {
   public bool answering { get; protected set; }
   public void answer ()
   {
     /* answer code implementation */
   }

   public static void main ()
   {
      var f = new Phone ();
      if (f.hang ())
         stdout.printf("Hand done.\n");
      else
         stdout.printf("Hand Error!\n");
      stdout.printf("END\n");
   }
}
```

在编译和运行时，你会发现 Phone 类实际上并没有实现 Callable.hang() 方法，但它能够使用该方法，那么结果就是一条 Hang done 的消息。

```vala
public class TechPhone : GLib.Object, Callable
{
   public bool answering { get; protected set; }
   public void answer ()
   {
     /* answer code implementation */
   }
   public bool hang ()
   {
      answering = false;
      stdout.printf ("TechPhone.hang () implementation!");
      return false;
   }
}
```

在这种情况下，`TechPhone` 是 `Callable` 的另一种实现，那么当在 `TechPhone` 实例上调用 `hang()` 方法时，它将始终返回 `false` 并打印出信息 `TechPhone.hang () implementation!` ，完全隐藏 `Callable.hang()` 的默认实现。

#### 属性

接口可定义必须由类来实现的属性。实现者的类必须定义与属性的 `get` 和 `set` 具有相同签名和访问权限的属性。

与任何 GObject 属性一样，您可以在实现者类中为属性的 `set` 和 `get` 定义一个主体，如果没有使用主体，则默认值为 set 和 get。如果需要，您必须定义一个`private`字段来存储属性值，以便在类外部或内部使用。

`Callable` 接口定义了一个`answering`属性。在这种情况下，该接口定义了一个带有`protected set`的`answering`属性，允许使用 `Callable` 实例的任何对象只读该属性，但允许类实现者向其写入值，就像 `TechPhone` 类在实现 `hang()` 方法时所做的那样。

#### 混合体和多重继承

如上所述，Vala 虽然以 C 和 GObject 为后盾，但可以通过在接口中添加虚拟方法来提供有限的多重继承机制。有可能在接口实现类中添加一些定义默认方法实现的方法，并允许派生类覆盖这些方法。

如果在接口中定义了一个虚拟方法，并在类中实现了该方法，那么就不能覆盖接口的方法，否则派生类就无法访问接口的默认方法。请看下面的代码：

```vala
public interface Callable : GLib.Object {
   public abstract bool answering { get; protected set; }
   public abstract void answer ();
   public abstract bool hang ();
   public static bool default_hang (Callable call)
   {
      stdout.printf ("At Callable.hang()\n");
      call.answering = false;
      return true;
   }
}

public abstract class Caller : GLib.Object, Callable
{
   public bool answering { get; protected set; }
   public void answer ()
   {
     stdout.printf ("At Caller.answer()\n");
     answering = true;
     hang ();
   }
   public virtual bool hang () { return Callable.default_hang (this); }
}

public class TechPhone : Caller {
        public string number { get; set; }
}

public class Phone : Caller {
   public override bool hang () {
        stdout.printf ("At Phone.hang()\n");
        return false;
   }

   public static void main ()
   {
      var f = (Callable) new Phone ();
      f.answer ();
      if (f.hang ())
         stdout.printf("Hand done.\n");
      else
         stdout.printf("Hand Error!\n");

      var t = (Callable) new TechPhone ();
      t.answer ();
      if (t.hang ())
         stdout.printf("Tech Hand done.\n");
      else
         stdout.printf("Tech Hand Error!\n");
      stdout.printf("END\n");
   }
}
```

在本例中，我们定义了一个 `Callable` 接口，它有一个名为`abstract bool hang ()` `default_hang`的静态方法，的抽象 bool hang () 的默认实现，可以是，也可以是虚拟方法。然后，Caller 是为 TechPhone 和 Phone 类实现 Callable 的基类，而 Caller 的 hang () 方法只是简单地调用 Callable 的默认实现。TechPhone 什么也不做，只是将 Caller 作为基类，使用默认方法实现；但 Phone 重载了 Caller.hang()，这使得它使用自己的实现，即使它被转换为 Callable 对象，也可以始终调用它。

##### 显式方法实现

显式接口方法实现允许实现两个具有同名方法（而非属性）的接口。例如

```vala
interface Foo {
 public abstract int m();
}

interface Bar {
 public abstract string m();
}

class Cls: Foo, Bar {
 public int Foo.m() {
  return 10;
 }

 public string Bar.m() {
  return "bar";
 }
}

void main () {
 var cls = new Cls ();
 message ("%d %s", ((Foo) cls).m(), ((Bar) cls).m());
}
```

输出：“10 bar”。

### 多态性

多态性（Polymorphism）描述的是同一个对象可以被当作一种以上不同类型的事物来使用的方式。这里已经介绍的几种技术说明了如何在 Vala 中实现这一点：一个类的实例可以作为一个超类的实例或任何实现的接口的实例使用，而无需知道它的实际类型。

这种能力的一个逻辑扩展是，在以完全相同的方式处理子类型时，允许子类型的行为与其父类型不同。要解释这个概念并不容易，所以我将从一个示例开始，说明如果不直接瞄准这个目标会发生什么：

```vala
class SuperClass : GLib.Object {
    public void method_1() {
        stdout.printf("SuperClass.method_1()\n");
    }
}

class SubClass : SuperClass {
    public void method_1() {
        stdout.printf("SubClass.method_1()\n");
    }
}
```

这两个类都实现了一个名为`method_1`的方法，因此“子类”包含两个名为`method_1`的方法，因为它从“超类”继承了一个方法。如下代码所示，每个方法都可以被调用：

```vala
SubClass o1 = new SubClass();
o1.method_1();
SuperClass o2 = o1;
o2.method_1();
```

这实际上会导致调用两个不同的方法。第二行认为`o1`是一个“子类”，因此将调用该类的方法。第四行认为`o2`是“超类”，因此将调用该类的方法。

这个例子暴露出的问题是，任何持有“超类”引用的代码都会调用该类中实际描述的方法，即使实际对象属于子类。改变这种行为的方法就是使用虚拟方法。下面是上一个示例的重写版本：

```vala
class SuperClass : GLib.Object {
    public virtual void method_1() {
        stdout.printf("SuperClass.method_1()\n");
    }
}

class SubClass : SuperClass {
    public override void method_1() {
        stdout.printf("SubClass.method_1()\n");
    }
}
```

当以与之前相同的方式使用这段代码时，“子类”的`method_1`将被调用两次。这是因为我们已经告诉系统，`method_1`是一个虚拟方法，也就是说，如果它在子类中被重载，那么无论调用者是否知道，新版本的方法都会在该子类的实例上执行。

这种区别对于某些语言（如 C++）的程序员来说可能并不陌生，但它实际上与 Java 风格的语言恰恰相反，在 Java 风格语言中，必须采取措施防止方法被虚拟化。

您现在可能也已经认识到，当方法被声明为`abstract`时，它也必须是虚拟的。否则，就无法在实现该方法的实例中执行该方法。因此，在子类中实现抽象方法时，可以选择将实现声明为`override`，从而传递该方法的虚拟性，并允许子类根据自己的需要也这样做。

也可以通过子类可以改变接口方法的实现方式来实现接口方法。在这种情况下，最初的实现过程是将方法的实现声明为`virtual`，然后子类可以根据需要进行覆盖。

在编写类时，通常会希望使用从一个类继承而来的类中定义的功能。如果方法名称在类的继承树中被多次使用，情况就会变得复杂。为此，Vala 提供了 `base` 关键字。最常见的情况是，你重载了一个虚拟方法以提供额外的功能，但仍需要调用父类的方法。下面的示例就是这种情况：

```vala
public override void method_name() {
    base.method_name();
    extra_task();
}
```

Vala 还允许虚拟属性：

```vala
class SuperClass : GLib.Object {
    public virtual string prop_1 {
        get {
            return "SuperClass.prop_1";
        }
    }
}

class SubClass : SuperClass {
    public override string prop_1 {
        get {
            return "SubClass.prop_1";
        }
    }
}
```

### 方法隐藏

通过使用 new 修饰符，可以用一个同名的新方法隐藏一个继承方法。新方法可能具有不同的签名。方法隐藏不能与方法覆盖混为一谈，因为方法隐藏不会表现出多态性。

```vala
class Foo : Object {
    public void my_method() { }
}

class Bar : Foo {
    public new void my_method() { }
}
```

您仍然可以通过转换到基类或接口来调用原始方法：

```vala
void main() {
    var bar = new Bar();
    bar.my_method();
    (bar as Foo).my_method();
}
```

### 运行时间类型信息

由于 Vala 类在运行时注册，且每个实例都携带类型信息，因此您可以使用 `is` 操作符动态检查对象的类型：

```vala
bool b = object is SomeTypeName;
```

您可以使用 `get_type()` 方法获取`Object`实例的类型信息：

```vala
Type type = object.get_type();
stdout.printf("%s\n", type.name());
```

通过 `typeof()` 操作符，您可以直接获取一个类型的类型信息。根据这些类型信息，您可以使用 `Object.new()` 创建新实例：

```vala
Type type = typeof(Foo);
Foo foo = (Foo) Object.new(type);
```

哪个构造函数会被调用？是`construct {}`块，将在有关 gobject 风格构造的章节中介绍。

### 动态类型转换

对于动态类型转换，变量通过后缀表达式`as DesiredTypeName`实现。Vala 会在运行时进行类型检查，以确保这种类型转换是合理的——如果是非法类型转换，将返回`null`。不过，这要求源类型和目标类型都是引用类类型。

例如：

```vala
as DesiredTypeName
```

如果由于某种原因，`widget` 实例的类不是 Button 类或其子类之一，或者没有实现 Button 接口，则 `b` 将为空。这种转换相当于：

```vala
Button b = (widget is Button) ? (Button) widget : null;
```

### 泛型

Vala 包含一个运行时泛型系统，通过该系统，一个类的特定实例可以使用在构造时选择的特定类型或类型集进行限制。这种限制一般用于要求存储在对象中的数据必须是特定类型的，例如，为了实现特定类型的对象列表。在这个例子中，Vala 将确保只有所需要的类型对象才能被添加到列表中，并且在检索时，所有对象都将被转换为该类型。

在 Vala 中，泛型是在程序运行时处理的。当你定义了一个可以受类型限制的类时，仍然只有一个类，每个实例都是单独定制的。这与 C++ 不同，C++ 会为每种类型限制创建一个新类，而 Vala 则与 Java 使用的系统类似。这带来了各种后果，其中最重要的是：静态成员由整个类型共享，而不管每个实例受到什么限制；给定一个类和一个子类，由子类完善的泛型可以用作由类完善的泛型。

下面的代码演示了如何使用泛型系统来定义一个最小封装类：

```vala
public class Wrapper<G>: GLib.Object {
    public G data { get; set; }
}
```

这个“封装”类必须受类型限制才能实例化——在这种情况下，类型将被标识为“G”，因此该类的实例将存储一个“G”类型的对象。

要实例化这个类，必须选择一个类型，例如内置的字符串类型（在 Vala 中，泛型中使用什么类型没有限制）。使用该类可创建一个简述：

```vala
void main () {
    var w = new Wrapper<string>();
    w.data = "test";
    stdout.printf(w.data);
}
```

如您所见，当从封装器中检索数据时，数据被分配给一个没有明确类型的标识符。之所以能做到这一点，是因为 Vala 知道每个封装器实例中的对象类型，因此可以代劳。

Vala 不会从您的泛型定义中创建多个类，这意味着您可以按如下方式编写代码：  
（这就是所谓的 "类型擦除"（Type Erasure），在 Java 中也是如此，以避免生成的 C 代码出现大量展开）。

```vala
class TestClass : GLib.Object {
}

void accept_object_wrapper(Wrapper<Glib.Object> w) {
}

...
var test_wrapper = new Wrapper<TestClass>();
accept_object_wrapper(test_wrapper);
...
```

由于所有测试类实例也都是对象，因此`accept_object_wrapper`方法将愉快地接受传递给它的对象，并将其封装对象视为 GLib.Object 实例。

### GObject式构造

如前所述，Vala 支持另一种构造方案，这种方案与前述方案略有不同，但更接近 GObject 的构造方式。您更喜欢哪一种，取决于您是从 GObject 迁移，还是从 Java 或 C# 迁移。gobject 风格的构造方案引入了一些新的语法元素：构造属性、特殊的 `Object(...)` 调用和`construct`块。让我们来看看它们是如何工作的：

```vala
public class Person : Object {

    /* Construction properties */
    public string name { get; construct; }
    public int age { get; construct set; }

    public Person(string name) {
        Object(name: name);
    }

    public Person.with_age(string name, int years) {
        Object(name: name, age: years);
    }

    construct {
        // do anything else
        stdout.printf("Welcome %s\n", this.name);
    }
}
```

在 gobject 风格的构造方案中，每个构造方法只包含一个 `Object(...)` 调用，用于设置所谓的构造属性。`Object(...)`调用以`property: value`的形式接收数量可变的命名参数。这些属性必须声明为`construct`或`set`属性。它们将被设置为给定的值，然后将调用从`GLib.Object`到我们类的所有`construct {}`块。

`construct`块保证在创建该类的实例时被调用，即使该实例是作为子类型创建的。它既没有任何参数，也没有返回值。在该代码块中，您可以根据需要调用其他方法并设置成员变量。

构造属性的定义与 `get` 和 `set` 属性一样，因此可以在赋值时运行任意代码。如果需要根据单个构造属性进行初始化，可以为该属性编写一个自定义`construct`块，该构造块将在赋值时立即执行，然后再执行其他构造代码。

如果声明的构造属性没有`set`，那么它就是所谓的仅构造属性，这意味着它只能在构造时赋值，之后就不能再赋值了。在上面的示例中，`name`就是这样一个只能构造的属性。

以下是各种类型属性的摘要，以及基于 gobject 的库文档中通常使用的术语：

```vala
    public int a { get; private set; }    // Read
    public int b { private get; set; }    // Write
    public int c { get; set; }            // Read / Write
    public int d { get; set construct; }  // Read / Write / Construct
    public int e { get; construct; }      // Read / Write-Construct-Only
```

在某些情况下，您可能还想执行一些操作——不是在创建类的实例时，而是在 GObject 运行时创建类本身时。在 GObject 术语中，我们指的是在有关类的 `class_init` 函数中运行的代码片段。在 Java 中，这被称为_态初始化块。在 Vala 中，这看起来像：

```vala
    /* This snippet of code is run when the class
     * is registered with the type system */
    static construct {
      ...
    }
```

## 高级功能

### 断言和合约编程

通过断言，程序员可以在运行时检查假设。语法是 `assert(condition)`。如果断言失败，程序将以适当的错误信息结束。GLib 标准命名空间中还有一些断言方法，例如：

```
assert_not_reached()

return_if_fail(bool expr)

return_if_reached()

warn_if_fail(bool expr)

warn_if_reached()
```

您可能很想使用断言来检查方法参数是否为空。然而，这并不是必须的，因为 Vala 会隐式地对所有未标记为`?`可空的参数进行检查。

```vala
void method_name(Foo foo, Bar bar) {
    /* Not necessary, Vala does that for you:
    return_if_fail(foo != null);
    return_if_fail(bar != null);
    */
}
```

Vala 支持基本的[合约编程](https://en.wikipedia.org/wiki/Contract_programming)功能。一个方法可能有前置条件（`requires`）和后置条件（`ensures`），必须分别在方法开始或结束时满足：

```vala
double method_name(int x, double d)
        requires (x > 0 && x < 10)
        requires (d >= 0.0 && d <= 1.0)
        ensures (result >= 0.0 && result <= 10.0)
{
    return d * x;
}
```

`result` 是一个特殊变量，代表返回值。

### 错误处理

GLib 有一个管理运行时异常的系统，称为 GError。Vala 将其转换为现代编程语言所熟悉的形式，但其实现方式与 Java 或 C# 中的不尽相同。重要的是要考虑何时使用这种类型的错误处理——GError 专门设计用于处理可恢复的运行时错误，即在实时系统上运行程序之前不知道的因素，而且不会对执行造成致命影响。您不应该使用 GError 来处理可以预见的问题，例如报告一个无效值被传递给了一个方法。例如，如果一个方法需要一个大于 0 的数字作为参数，那么它就应该使用契约编程技术（如上一节中描述的前置条件或断言）在出现负值时失效。

Vala 错误是所谓的“检查异常”，这意味着错误必须在某个时刻得到处理。但是，如果不捕获错误，Vala 编译器只会发出警告，而不会停止编译进程。

使用异常（或用 Vala 术语说的`errors`）需要一系列的步骤：

1.  声明方法可能引发错误：

    ```vala
    void my_method() throws IOError {
        // ...
    }
    ```

2.  在适当的时候抛出错误：

    ```vala
    if (something_went_wrong) {
        throw new IOError.FILE_NOT_FOUND("Requested file could not be found.");
    }
    ```

3.  从调用代码中捕捉错误：

    ```vala
    try {
        my_method();
    } catch (IOError e) {
        stdout.printf("Error: %s\n", e.message);
    }
    ```

4.  使用`is`操作符比较错误代码：

    ```vala
    IOChannel channel;
    try {
        channel = new IOChannel.file("/tmp/my_lock", "w");
    } catch (FileError e) {
        if(e is FileError.EXIST) {
            throw e;
        }
        GLib.error("", e.message);
    }
    ```

所有这些都与其他语言大致相同，但允许出现的错误类型定义却相当独特。错误有三个组成部分，即“域”、“代码”和消息。我们已经看到，信息只是创建错误时提供的一段文本。错误域描述了问题的类型，相当于 Java 或类似语言中“异常”的子类。在上述示例中，我们设想了一个名为“IOError”的错误域。第三部分，错误代码是对所遇到问题的具体种类的细化描述。每个错误域都有一个或多个错误代码——在示例中就有一个名为 `FILE_NOT_FOUND` 的代码。

定义错误类型信息的方法与 glib 的实现有关。为了使这里的示例有效，需要定义如下内容：

```vala
errordomain IOError {
    FILE_NOT_FOUND
}
```

在捕获错误时，您需要给出希望捕获错误的错误域，如果该域中的错误被抛出，处理程序中的代码就会运行，并将错误赋值给所提供的名称。您可以根据需要从错误对象中提取错误代码和信息。如果要捕获来自多个域的错误，只需提供额外的`catch`块即可。还有一个可选的代码块可以放在 `try` 和任何 `catch` 代码块之后，称为 `finally`。无论是否抛出错误或执行了任何捕获块，即使错误实际上没有得到处理，并将再次抛出，这段代码都将在该部分结束时运行。例如，这样就可以释放 try 代码块中保留的任何资源，而不管是否出现错误。这些功能的一个完整示例：

```vala
public errordomain ErrorType1 {
    CODE_1A
}

public errordomain ErrorType2 {
    CODE_2A,
    CODE_2B
}

public class Test : GLib.Object {
    public static void thrower() throws ErrorType1, ErrorType2 {
        throw new ErrorType1.CODE_1A("Error");
    }

    public static void catcher() throws ErrorType2 {
        try {
            thrower();
        } catch (ErrorType1 e) {
            // Deal with ErrorType1
        } finally {
            // Tidy up
        }
    }

    public static int main(string[] args) {
        try {
            catcher();
        } catch (ErrorType2 e) {
            // Deal with ErrorType2
            if (e is ErrorType2.CODE_2B) {
                // Deal with this code
            }
        }
        return 0;
    }
}
```

本例有两个错误域，“thrower”方法都可以抛出这两个错误域。捕获器只能抛出第二种类型的错误，因此如果“thrower”抛出第一种类型的错误，捕获器就必须处理第一种类型的错误。最后，“main”方法将处理来自“catcher”的任何错误。

### 参数方向

Vala 中的方法可传递零个或多个参数。方法被调用时的默认行为如下：

-   任何值类型的参数都会在方法执行时复制到方法的本地位置。
-   任何引用类型的参数都不会被复制，而只是将其引用传递给方法。

这种行为可以通过修改器 "ref "和 "out "来改变。

- 调用方`out`
您可以向方法传递一个未初始化的变量，并期望在方法返回后对其进行初始化。

- 被调方`out`
参数被视为未初始化，您必须将其初始化

- 调用方`ref`
你传递给方法的变量必须初始化，方法可能会更改或不更改它

- 调方的`ref`
该参数被视为已初始化，您可以更改或不更改它

```vala
void method_1(int a, out int b, ref int c) { ... }
void method_2(Object o, out Object p, ref Object q) { ... }
```

这些方法的调用方法如下：

```vala
int a = 1;
int b;
int c = 3;
method_1(a, out b, ref c);

Object o = new Object();
Object p;
Object q = new Object();
method_2(o, out p, ref q);
```

每个变量的处理方法如下：

-   `a`是值类型。该值将被复制到方法本地的一个新的内存位置，因此调用者看不到它的变化。
-   `b`也是值类型，但作为输出参数传递。在这种情况下，值不会被复制，而是将指向数据的指针传递给方法，因此方法参数的任何变化对调用代码来说都是可见的。
-   `c`的处理方式与`b`相同，唯一的变化是方法的调用意图。
-   `o`是引用类型。方法传递给调用者的是同一个对象的引用。因此，方法可以更改该对象，但如果重新分配给参数，调用者将无法看到这一更改。
-   `p`类型相同，但作为输出参数传递。这意味着方法将收到指向对象引用的指针。因此，它可能会用另一个对象的引用来替换该引用，当方法返回时，调用者将拥有另一个对象的引用。使用这种类型的参数时，如果不为参数赋值新的引用，它将被设置为空。
    
-   `q`也属于同一类型。这种情况的处理方法与`p`相同，但有一个重要区别，即方法可以选择不更改引用，也可以访问被引用的对象。Vala 将确保在这种情况下，`q`实际上是指任何对象，而不是设置为空。

下面举例说明如何实现 `method_1()`：

```vala
void method_1(int a, out int b, ref int c) {
    b = a + c;
    c = 3;
}
```

在设置输出参数`b`的值时，Vala 会确保`b`不是空值。因此，如果您对该值不感兴趣，可以放心地将空值作为 `method_1()` 的第二个参数。

### 容器

[Gee](https://wiki.gnome.org/Projects/Libgee)是一个用 Vala 编写的集合类库。这些类对于使用 Java 的基础类（Foundation Classes）等库的用户来说并不陌生。Gee 由一组接口和以不同方式实现这些接口的各种类型组成。

如果您想在自己的应用程序中使用 Gee，请在系统中单独安装该库。Gee 可从[https://live.gnome.org/Projects/Libgee](https://live.gnome.org/Projects/Libgee) 获取。要使用该库，必须使用 `--pkg gee-0.8` 编译程序。

容器的基本类型有

-   列表：有序的项目集合，可通过数字索引访问。
-   集合：不同的无序集合。
-   映射：无序的项目集合，可通过任意类型的索引访问。

库中的所有列表和集合都实现了`Collection`接口，所有映射都实现了`Map`接口。列表也实现了`List`接口，集合则实现了`Set`接口。这些通用接口不仅意味着所有相似类型的集合可以互换使用，还意味着可以使用相同的接口编写新的集合，从而与现有代码一起使用。

`Iterable`接口也是每个容器类型的共同特征。这意味着该类别中的任何对象都可以使用标准方法集进行遍历，或直接在 Vala 中使用 `foreach` 语法进行遍历。

所有类和接口都使用泛型系统。这意味着它们在实例化时必须包含特定的类型或类型集。该系统将确保只有指定的类型才能被放入容器中，并确保在检索对象时，它们能以正确的类型返回。

[完整的 Gee API 文档](https://valadoc.org/gee-0.8/index.htm)、[Gee 示例](https://wiki.gnome.org/Projects/Vala/GeeSamples)

一些重要的 Gee 类包括：

#### ArrayList

实现：`Iterable<G>`, `Collection<G>`, `List<G>`。

由G类型构成的有序列表是由可以动态调整大小的数组实现的 。这种类型在访问数据时速度非常快，但在插入除末尾以外的其他项目时，或在内部数组已满时插入项目时，速度可能会很慢。

#### HashMap

实现：`Iterable<Entry<K,V>>`, `Map<K,V>`

从 K 类型元素到 V 类型元素的 1:1 映射。映射是通过计算每个键的哈希值实现的——可以通过提供指向哈希函数的指针和以特定方式测试键的相等性来定制。

例如，您可以选择将自定义哈希函数和等价函数传递给构造函数：

```vala
var map = new Gee.HashMap<Foo, Object>(foo_hash, foo_equal);
```

对于字符串和整数，哈希函数和等价函数会自动检测，而对象则默认通过引用来区分。只有当你想覆盖默认行为时，才需要提供自定义的哈希和等价函数。

#### HashSet

实现：`Iterable<G>`, `Collection<G>`, `Set<G>`。

G 类型元素的集合。通过计算每个密钥的哈希值来检测重复——可以通过提供指向哈希函数的指针来定制哈希值，并以特定方式测试密钥的相等性。

#### 只读视图

您可以通过`read_only_view`属性获取容器的只读视图，例如 `my_map.read_only_view`，这将返回一个包装器，该包装器与其包含的容器具有相同的接口，但不允许对包含的容器进行任何形式的修改或访问。

### 具有特殊语义的方法

Vala 可识别某些具有特定名称和签名的方法，并为其提供语法支持。例如，如果一个类型有一个`contains()`方法，该类型的对象就可以使用 `in` 操作符。下表列出了这些特殊方法。表中的`T`和`Tn`只是类型占位符，应替换为实际类型。

- 索引器

    | 索引器 | 操作 |
    | --- | --- |
    | `T2 get(T1 index)` | 索引访问：`obj[index]` |
    | `void set(T1 index, T2 item)` | 索引赋值：`obj[index] = item` |

- 具有多个索引的索引器

    | 索引器 | 操作 |
    | --- | --- |
    | `T3 get(T1 index1, T2 index2)` | 索引访问：`obj[index1, index2]` |
    | `void set(T1 index1, T2 index2, T3 item)` | 索引赋值：`obj[index1, index2] = item` |
    | 更多索引参数以此类推 | …… |

- 其它

    | 索引器 | 操作 |
    | --- | --- |
    | `T slice(long start, long end)` | 切片：`obj[start:end]` |
    | `bool contains(T needle)` | `in`操作: `bool b = needle in obj` |
    | `string to_string()` | 支持字符串模板：`@"$obj"` |
    | `Iterator iterator()` | 通过`foreach`迭代 |
    | `T2 get(T1 index)`<br> `T1 size { get; }` | 通过`foreach`迭代 |


迭代器类型可以取任何名称，并且必须实现这两个协议中的一个：

| 迭代器 | 协议 |
| --- | --- |
| `bool next()`<br> `T get()` | 标准迭代器协议：迭代直到`.next()`返回`false`。当前元素通过`.get()` 获取。 |
| `T? next_value()` | 替代迭代器协议：如果迭代器对象有一个`.next_value()`函数返回一个可为空的类型，那么我们就通过调用这个函数来进行迭代，直到它返回空为止。 |

下面的例子实现了其中一些方法：

```vala
public class EvenNumbers {
    public int get(int index) {
        return index * 2;
    }

    public bool contains(int i) {
        return i % 2 == 0;
    }

    public string to_string() {
        return "[This object enumerates even numbers]";
    }

    public Iterator iterator() {
        return new Iterator(this);
    }

    public class Iterator {
        private int index;
        private EvenNumbers even;

        public Iterator(EvenNumbers even) {
            this.even = even;
        }

        public bool next() {
            return true;
        }

        public int get() {
            this.index++;
            return this.even[this.index - 1];
        }
    }
}

void main() {
    var even = new EvenNumbers();
    stdout.printf("%d\n", even[5]);   // get()
    if (4 in even) {                  // contains()
        stdout.printf(@"$even\n");    // to_string()
    }
    foreach (int i in even) {         // iterator()
        stdout.printf("%d\n", i);
        if (i == 20) break;
    }
}

```

### 多线程

#### Vala中的线程

用 Vala 编写的程序可能有多个执行线程，允许程序同时做多件事。  
具体如何管理不在 Vala 的范围之内——线程可能共享一个处理器内核，也可能不共享，这取决于环境。

一个非常简单的例子：

```vala
void thread_func() {
    stdout.printf("child_thread is running.\n");
}

void main() {
    if (!Thread.supported()) {
        error("Cannot run without thread support.\n");
    }
    var thread = new Thread<void> ("child_thread", thread_func);
    stdout.printf("main_thread is running");
}
```

请注意`main`方法开始时对线程支持的测试。
最初，UNIX 没有线程，这意味着一些传统的 UNIX API 在线程程序中会出现问题。  
使用该测试可以检查当前环境是否支持线程。  
在大多数情况下，这个测试可以省略。

现在看看下面的语句：

```vala
var thread = new Thread<void> ("new_thread", thread_func);    // Note
```

我们创建一个新线程，它将立即启动。  
第一个参数是线程名称，第二个参数是新线程的内容。  
泛型参数是线程返回值的类型。

程序仍然不会按照我们的预期运行，因为我们没有任何形式的事件循环，Vala 程序会在其主或父线程（为运行“main”而创建的线程）结束时终止。为了控制这种行为，可以进行线程协作。  
这可以通过使用事件循环和异步队列来实现，但在这篇线程介绍中，我们将只展示线程的基本功能。

如果主线程/父线程已经结束，子线程就会被杀死。  
根据这一事实，我们应该调用 `Thread` 模块中的 `join` 方法，让主线程等待子线程结束。

```vala
    // ......
    var thread = new Thread<void> ("child_thread", thread_func);
    stdout.printf("main_thread is running");
    thread.join();   // Note
```

由于`join`方法，主线程必须等待子线程结束。

此外，一个线程还可以告诉系统它目前没有执行的必要，从而建议运行另一个线程，这可以通过静态方法`_Thread.yield()` 来实现。如果将该语句放在上述`main`方法的末尾，运行时系统就会暂停主线程一段时间，并检查是否有其他线程可以运行——一旦发现新创建的线程处于可运行状态，系统就会运行该线程，直到其结束——程序就会按照它看起来应该做的那样运行。不过，并不能保证这种情况总会发生。系统可以决定线程的运行时间，因此可能不会允许新线程在主线程重新调度和结束前完成。

所有这些示例都有一个潜在的问题，即新创建的线程不知道它应该在什么环境中运行。在 C 语言中，你会向线程创建方法提供一些数据，而在 Vala 中，你通常会传递一个实例方法，而不是静态方法。

更多示例请参考[线程示例](https://wiki.gnome.org/Projects/Vala/ThreadingSamples)。

#### 资源控制

只要有一个以上的执行线程同时运行，就有可能同时访问数据。这可能会导致竞争条件，其结果取决于系统何时决定切换线程。

为了控制这种情况，可以使用`lock`关键字来确保某些代码块不会被其他需要访问相同数据的线程打断。为了说明这一点，我们最好举个例：

```vala
public class Test : GLib.Object {

    private int a { get; set; }

    public void action_1() {
        lock (a) {
            int tmp = a;
            tmp++;
            a = tmp;
        }
    }

    public void action_2() {
        lock (a) {
            int tmp = a;
            tmp--;
            a = tmp;
        }
    }
}
```

该类定义了两个方法，这两个方法都需要改变`a`的值。如果这里没有锁语句，这些方法中的指令就有可能交织在一起，从而导致对`a`的更改实际上是随机的。由于这里有锁语句，Vala 将保证，如果一个线程锁定了`a`，另一个需要相同锁的线程将不得不等待轮到它。

在 Vala 中，只能锁定正在执行代码的对象的成员。这似乎是一个很大的限制，但事实上，这种技术的标准用途应该是涉及单独负责控制资源的类，因此所有锁定都将在类的内部进行。同样，在上例中，对`a`的所有访问都封装在类中。

### 主循环

GLib 包含一个运行事件循环的系统，在 `MainLoop` 相关的类中。该系统的目的是让您可以编写一个等待事件并对其做出响应的程序，而不必不断地检查条件。这是 GTK+ 使用的模型，因此程序可以等待用户交互，而无需任何当前运行的代码。

下面的程序创建并启动了一个 `MainLoop`，然后为其附加了一个事件源。在本例中，事件源是一个简单的计时器，它将在 2000ms 后执行给定的方法。该方法实际上只是停止主循环，在这种情况下会退出程序。

```vala
void main() {

    var loop = new MainLoop();
    var time = new TimeoutSource(2000);

    time.set_callback(() => {
        stdout.printf("Time!\n");
        loop.quit();
        return false;
    });

    time.attach(loop.get_context());

    loop.run();
}
```

使用 GTK+ 时，主循环将自动创建，并在调用 "Gtk.main() "方法时启动。这标志着程序已准备好运行并开始接受来自用户或其他地方的事件。GTK+ 中的代码等同于上面的简短示例，因此您可以用大致相同的方式添加事件源，当然您需要使用 GTK+ 方法来控制主循环。

```vala
void main(string[] args) {

    Gtk.init(ref args);
    var time = new TimeoutSource(2000);

    time.set_callback(() => {
        stdout.printf("Time!\n");
        Gtk.main_quit();
        return false;
    });

    time.attach(null);

    Gtk.main();
}
```

图形用户界面程序中的一个常见要求是尽快执行某些代码，但必须在不打扰用户的情况下执行。为此，需要使用 `IdleSource` 实例。这些实例会向程序主循环发送事件，但要求只有在没有更重要的事情时才处理这些事件。

有关事件循环的更多信息，请参阅 GLib 和 GTK+ 文档。

### 异步方法

异步方法是可以在程序员控制下暂停和恢复执行的方法。异步方法常用于应用程序的主线程中，在这种情况下，方法需要等待外部的慢速任务完成，但又不能阻止其他处理过程的进行。（例如，一个慢速操作不能冻结整个图形用户界面。）当方法需要等待时，它会将 CPU 的控制权交还给调用者（即`yields`），但会安排在数据准备就绪时被调用回来继续执行。异步方法可能等待的外部慢任务包括：等待来自远程服务器的数据，或等待另一个线程中的计算完成，或等待从磁盘驱动器加载数据。

异步方法通常在运行 GLib 主循环时使用，因为空闲回调用于处理一些内部回调。但在某些情况下，异步方法也可以在不运行 GLib 主循环的情况下使用，例如，如果异步方法总是`yield`，且从未使用过 `Idle.add()`。（修正：查看具体条件）。

异步方法设计用于在单个线程内交错处理许多不同的长期操作。它们本身不会将负载分散到不同的线程中。不过，异步方法可用于控制后台线程并等待其完成，或为后台线程处理操作排队。

Vala 中的异步方法使用 GIO 库处理回调，因此必须使用 `--pkg=gio-2.0` 选项构建。

异步方法是用 `async` 关键字定义的。例如

```vala
  async void display_jpeg(string fnam) {
     // Load JPEG in a background thread and display it when loaded
     [...]
  }
```

或：

```vala
  async int fetch_webpage(string url, out string text) throws IOError {
     // Fetch a webpage asynchronously and when ready return the
     // HTTP status code and put the page contents in 'text'
     [...]
     text = result;
     return status;
  }
```

该方法可以像其他方法一样接受参数并返回值。它可以随时使用 yield 语句将 CPU 的控制权交还给调用者。

异步方法可以这两种形式中的任何一种调用：

```vala
  display_jpeg.begin("test.jpg");
  display_jpeg.begin("test.jpg", (obj, res) => {
      display_jpeg.end(res);
  });
```

这两种形式都会使用给定的参数启动异步方法。第二个表单还会注册一个 AsyncReadyCallback，在方法结束时执行。回调以源对象 `obj` 和 GAyncResult 实例 `res` 为参数。回调中应调用 .end() 方法，以接收异步方法的返回值（如果有的话）。如果异步方法会抛出异常，.end() 调用就是异常出现的地方，必须捕获异常。如果方法有 out 参数，则应在 .begin() 调用中省略这些参数，将其添加到 .end() 调用中。

例如：

```vala
  fetch_webpage.begin("https://www.example.com/", (obj, res) => {
      try {
          string text;
          var status = fetch_webpage.end(res, out text);
          // Result of call is in 'text' and 'status' ...
      } catch (IOError e) {
          // Problem ...
      }
  });
```

当异步方法开始运行时，它会控制 CPU，直到执行到第一个 yield 语句，然后返回给调用者。当方法恢复运行时，它会紧接着该 yield 语句继续执行。有几种常见的 yield 使用方法：

该形式放弃了控制，但安排 GLib 主循环在没有更多事件需要处理时恢复该方法：

```vala
  Idle.add(fetch_webpage.callback);
  yield;
```

该形式放弃了控制权，并存储了回调细节，供其他代码用来恢复方法的执行：

```vala
  SourceFunc callback = fetch_webpage.callback;
  [... store 'callback' somewhere ...]
  yield;
```

现在，其他地方的代码必须调用存储的 SourceFunc 才能恢复该方法。这可以通过调度 GLib 主循环来实现：

```vala
  Idle.add((owned) callback);
```

或者，如果调用者在主线程中运行，也可以直接调用：

```vala
  callback();
```

如果使用上述直接调用，则恢复的异步方法会立即控制 CPU，并运行到下一次`yield`时才返回执行 `callback()` 的代码。如果必须从后台线程进行回调，例如在完成某些后台处理后恢复异步方法，则 `Idle.add()` 方法非常有用。（为了避免出现关于复制委托的警告，有必要使用（`owned`）转换）。

使用 `yield` 的第三种常见方式是调用另一个异步方法，例如调用一个异步方法：

```vala
  yield display_jpeg(fnam);
```

或：

```vala
  var status = yield fetch_webpage(url, out text)
```

在这两种情况下，调用方法都会放弃对 CPU 的控制，在被调用方法完成之前不会恢复运行。`yield` 语句会自动向被调用方法注册一个回调，以确保调用者正确恢复。自动回调也会收集被调用方法的返回值。

当执`yield`服语句时，CPU 的控制权首先传递给被调用的方法，该方法运行到第一次`yield`为止，然后返回到调用方法，该方法完成`yield`语句，然后将控制权交还给自己的调用者。

#### 实例

请参阅[Async 方法](https://wiki.gnome.org/Projects/Vala/AsyncSamples)示例，了解使用 async 的不同方式。

### 弱参考

Vala 的内存管理基于自动引用计数。每次将对象分配给变量时，其内部引用计数都会增加 1；每次引用对象的变量退出作用域时，其内部引用计数都会减少 1。

不过，数据结构也有可能形成引用循环。例如，在树形数据结构中，子节点持有对父节点的引用，反之亦然；在双链表中，每个元素都持有对前代元素的引用，而前代元素又持有对后代元素的引用。

在这种情况下，尽管对象应该被释放，但它们可以通过相互引用来保持自身的活力。要打破这种引用循环，可以对其中一个引用使用弱修饰符：

```vala
class Node {
    public weak Node prev;
    public Node next;
}
```

本页将详细解释这一主题：[Vala内存管理](https://wiki.gnome.org/Projects/Vala/ReferenceHandling)。

### 所有权

#### 无主引用

在 Vala 中创建对象时，通常会返回一个对象的引用。具体来说，这意味着除了传递一个指向内存中对象的指针外，对象本身也记录了这个指针的存在。同样，无论何时创建指向对象的另一个引用，也会被记录下来。由于对象知道自己有多少个引用，因此可以在需要时自动删除。这就是[Vala 内存管理](https://wiki.gnome.org/Projects/Vala/ReferenceHandling)的基础。

#### 方法所有权

反之，无主引用不会记录在其引用的对象中。这样就可以在逻辑上应该删除对象时将其删除，而无需考虑可能仍存在对它的引用这一事实。实现这一点的通常方法是定义一个方法来返回一个非自有引用，例如：

```vala
class Test {
    private Object o;

    public unowned Object get_unowned_ref() {
        this.o = new Object();
        return this.o;
    }
}
```

调用此方法时，为了收集返回对象的引用，必定会收到一个弱引用：

```vala
unowned Object o = get_unowned_ref();
```

之所以要举出这个看似过于复杂的例子，是因为所有权的概念。

-   如果对象`o`没有存储在类中，那么当方法 `get_unowned_ref`返回时，`o`就会成为非所有对象（即没有对它的引用）。在这种情况下，对象将被删除，而方法将永远不会返回有效的引用。
-   如果返回值没有被定义为非所有，则所有权将传递给调用代码。然而，调用代码期待的是一个非所有引用，因此无法接收所有权。

如果调用代码写成：

```vala
Object o = get_unowned_ref();
```

Vala 会尝试获取无主引用指向的实例的引用或副本。

#### 属性所有权

与普通方法不同的是，属性的返回值始终是非自有的。这意味着你不能返回在 get 方法中创建的新对象。这也意味着，你不能使用方法调用中的自有返回值。一个令人恼火的事实是，属性值为拥有该属性的对象所有。获取该属性值的调用不应窃取或复制（通过复制或增加引用计数）对象方的值。

因此，以下示例将导致编译错误：

```vala
public Object property {
    get {
        return new Object();   // WRONG: property returns an unowned reference,
                               // the newly created object will be deleted when
                               // the getter scope ends the caller of the
                               // getter ends up receiving an invalid reference
                               // to a deleted object.
    }
}
```

也不能这样做：

```vala
public string property {
    get {
        return getter_method();   // WRONG: for the same reason above.
    }
}

public string getter_method() {
    return "some text"; // "some text" is duplicated and returned at this point.
}
```

另一方面，这样做却很好：

```vala
public string property {
    get {
        return getter_method();   // GOOD: getter_method returns an unowned value
    }
}

public unowned string getter_method() {
    return "some text";
    // Don't be alarmed that the text is not assigned to any strong variable.
    // Literal strings in Vala are always owned by the program module itself,
    // and exist as long as the module is in memory
}
```

`unowned`修改器可用于使自动属性的存储空间成为无主存储空间。这意味着：

```vala
public unowned Object property { get; private set; }
```

等同于：

```vala
private unowned Object _property;

public Object property {
    get { return _property; }
}
```

关键字 `owned` 可用于明确要求属性返回一个属性值的自有引用，从而使属性值在对象中重现。在添加 `owned` 关键字前请三思。它是一个属性还是一个简单的 `get_xxx` 方法？你的设计也可能存在问题。总之，以下代码是正确的代码段：

```vala
public owned Object property { owned get { return new Object(); } }
```

无主引用的作用类似于稍后介绍的指针。不过，它们的使用要比指针简单得多，因为它们可以很容易地转换为普通引用。不过，一般来说，除非你知道自己在做什么，否则不应在程序中广泛使用它们。

#### 所有权转让

关键字 owned 用于转移所有权。

-   作为参数类型的前缀，它意味着对象的所有权已转移到此代码上下文中。
-   作为一个类型转换操作符，它可以用来避免重复非引用计数类，这在 Vala 中通常是不可能的。例如

```vala
Foo foo = (owned) bar;
```

这意味着`bar`将被设置为空，而`foo`将继承`bar`引用对象的引用/所有权。

### 可变长度参数列表

Vala 支持 C 风格的方法可变长度参数列表（"varargs"）。它们在方法签名中用省略号（"..."）声明。带有 varargs 的方法至少需要一个固定参数：

```vala
void method_with_varargs(int x, ...) {
    var l = va_list();
    string s = l.arg();
    int i = l.arg();
    stdout.printf("%s: %d\n", s, i);
}
```

在本例中，`x` 是满足要求的固定参数。通过 `va_list()`，可以获得 varargs 列表。然后，可以在这个列表上依次调用泛型方法a`rg<T>()`来获取一个又一个参数，其中 T 是参数应被解释为的类型。如果从上下文中可以明显看出类型（就像我们的示例一样），则会自动推断出类型，此时只需调用 `arg()`，而无需泛型参数。

本例解析任意数量的字符串和双精度浮点参数对：

```vala
void method_with_varargs(int fixed, ...) {
    var l = va_list();
    while (true) {
        string? key = l.arg();
        if (key == null) {
            break;  // end of the list
        }
        double val = l.arg();
        stdout.printf("%s: %g\n", key, val);
    }
}

void main() {
    method_with_varargs(42, "foo", 0.75, "bar", 0.25, "baz", 0.32);
}
```

它会检查`null`作为界限，以识别 varargs 列表的结束。Vala 总是隐式地将`null`作为 varargs 方法调用的最后一个参数。

Varargs 有一个值得注意的严重缺点：它们不是类型安全的。编译器无法判断你是否向方法传递了正确类型的参数。这就是为什么只有在有充分理由的情况下才考虑使用 varargs，例如：为使用 Vala 库的 C 程序员提供方便的函数，绑定 C 函数。通常，数组参数是更好的选择。

使用 varargs 的一个常见模式是将交替的`string - value`对作为参数，通常是指gobject `property - value`。在这种情况下，可以用`property: value`代替，例如：

```vala
actor.animate (AnimationMode.EASE_OUT_BOUNCE, 3000, x: 100.0, y: 200.0, rotation_angle_z: 500.0, opacity: 0);
```

等同于：

```vala
actor.animate (AnimationMode.EASE_OUT_BOUNCE, 3000, "x", 100.0, "y", 200.0, "rotation-angle-z", 500.0, "opacity", 0);
```

### 指针

指针是 Vala 允许手动管理内存的一种方式。通常，当您创建一个类型的实例时，您会收到一个指向它的引用，当没有更多的引用时，Vala 会负责销毁该实例。通过请求一个指向实例的指针，当实例不再需要时，你就可以负责销毁它，从而更好地控制内存的使用。

大多数时候并不一定需要这种功能，因为现代计算机的速度通常足以处理引用计数，而且内存也足够大，小的低效并不重要。以下情况可能需要手动管理内存：

-   当你特别想优化程序的一部分，而[非自有引用](https://wiki.gnome.org/Projects/Vala/Tutorial#Unowned_References)又不足时。
    
-   当您使用的外部库没有为内存管理实施引用计数时（可能指的是不基于 gobject 的外部库），您可以使用。

以创建一个类型的实例，并接收指向该实例的指针：

```vala
Object* o = new Object();
```

为了访问该实例的成员：

```vala
o->method_1();
o->data_1;
```

为了释放指向的内存：

```vala
delete o;
```

Vala 还支持 C 语言中的寻址(`&`) 和间接(`*`) 操作符：

```vala
int i = 42;
int* i_ptr = &i;    // address-of
int j = *i_ptr;     // indirection
```

引用类型的行为有些不同，在赋值时可以省略 address-of 和间接操作符：

```vala
Foo f = new Foo();
Foo* f_ptr = f;    // address-of
Foo g = f_ptr;     // indirection

unowned Foo f_weak = f;  // equivalent to the second line

```

使用引用类型指针等同于使用非自有引用。

### 非对象类

定义为非`GLib.Object`派生的类被视为特例。它们直接从 GLib 的类型系统派生，因此重量更轻。在最新的 Vala 编译器中，我们还可以使用这些类实现接口、信号和属性。

使用这些非`Object`类的一个明显例子就是 GLib 绑定。因为 GLib 的级别比 GObject 低，所以绑定中定义的大多数类都是此类。另外，如前所述，非对象类的轻量级使它们在许多实际情况下都非常有用（例如 Vala 编译器本身）。不过，非对象类的详细用法不在本教程的讨论范围之内。请注意，这些类与结构体有着本质区别。

### D-Bus集成

[D-Bus](https://freedesktop.org/wiki/Software/dbus)与语言紧密结合，通过Vala，使用从未如此简单。

要将自定义类导出为 D-Bus 服务，只需用`DBus`代码属性对其进行注释，并在本地`D-Bus`会话中注册该类的实例即可。

```vala
[DBus(name = "org.example.DemoService")]
public class DemoService : Object {
    /* Private field, not exported via D-Bus */
    int counter;

    /* Public field, not exported via D-Bus */
    public int status;

    /* Public property, exported via D-Bus */
    public int something { get; set; }

    /* Public signal, exported via D-Bus
     * Can be emitted on the server side and can be connected to on the client side.
     */
    public signal void sig1();

    /* Public method, exported via D-Bus */
    public void some_method() {
        counter++;
        stdout.printf("heureka! counter = %d\n", counter);
        sig1();  // emit signal
    }

    /* Public method, exported via D-Bus and showing the sender who is
       is calling the method (not exported in the D-Bus interface) */
    public void some_method_sender(string message, GLib.BusName sender) {
        counter++;
        stdout.printf("heureka! counter = %d, '%s' message from sender %s\n",
                      counter, message, sender);
    }
}
```

注册服务实例并启动主循环：

```vala
void on_bus_aquired (DBusConnection conn) {
    try {
        // start service and register it as dbus object
        var service = new DemoService();
        conn.register_object ("/org/example/demo", service);
    } catch (IOError e) {
        stderr.printf ("Could not register service: %s\n", e.message);
    }
}

void main () {
    // try to register service name in session bus
    Bus.own_name (BusType.SESSION, "org.example.DemoService", /* name to register */
                  BusNameOwnerFlags.NONE, /* flags */
                  on_bus_aquired, /* callback function on registration succeeded */
                  () => {}, /* callback on name register succeeded */
                  () => stderr.printf ("Could not acquire name\n"));
                                                     /* callback on name lost */

    // start main loop
    new MainLoop ().run ();
}
```

您必须使用`gio-2.0`软件包编译此示例：

```vala
valac --pkg gio-2.0 dbus-demo-service.vala
./dbus-demo-service
```

所有成员的名称都会自动从 Vala 的`lower_case_with_underscores`命名方式转换为 D-Bus的`CamelCase`驼峰命名方式。本例导出的 D-Bus 接口将有一个属性`Something`、一个信号`Sig1`和一个方法`SomeMethod`。您可以打开一个新的终端窗口，通过命令行调用该方法：

```shell
  dbus-send --type=method_call                   \
            --dest=org.example.DemoService       \
            /org/example/demo                    \
            org.example.DemoService.SomeMethod
```

或：

```vala
  dbus-send --type=method_call                   \
            --dest=org.example.DemoService       \
            /org/example/demo                    \
            org.example.DemoService.SomeMethodSender \
            string:'hello world'
```

您还可以使用图形化 D-Bus 调试器（如[D-Feet](https://wiki.gnome.org/Apps/DFeet)）来浏览 D-Bus 接口和调用方法。

一些综合示例：[DBus 客户端示例](https://wiki.gnome.org/Projects/Vala/DBusClientSamples)和[DBus 服务器示例](https://wiki.gnome.org/Projects/Vala/DBusServerSample)

### 配置

通过使用 `valac` 的 `--profile` 选项，生成的 C 代码可以使用不同的最小运行时间。Vala 支持两种不同的配置文件：

-   gobject（默认）
    
-   libc

配置文件决定了哪些 Vala 语言特性可用于匹配最小运行时环境。`gobject` 配置文件可以生成需要 GLib 的 GType 运行时类型系统的代码，因此运行时环境通常需要 libgobject 及其少量依赖项。

`libc` 配置文件移除了对 GLib 的依赖，并禁用了运行时类型系统。如果使用的 Vala 语言功能需要运行时类型系统，该配置文件会在编译时生成替代代码或错误。这对于编写针对微控制器的代码或生成系统实用程序或超小型容器映像的二进制文件非常有用。运行时环境通常需要一小部分 ISO C 标准库。`posix` 配置文件目前是 `libc` 的别名，因为兼容 POSIX 的操作系统包含 C 标准库，但该配置文件生成的代码可以用于非 POSIX 平台，在这些平台上，最小的 C 标准库可用于运行时动态链接或静态链接到二进制文件中。

要选择不同的配置文件，请使用 valac 的 --profile 开关：

```shell
valac --profile=libc somecode.vala
```

当然，二进制文件仍然需要使用 `valac` 的 `--pkg` 选项所针对的库的运行时依赖项。

## 实验特性

Vala 的某些功能是试验性的。这意味着它们尚未经过全面测试，在未来版本中可能会有所更改。

### 链式关系表达式

该功能允许您编写复杂的关系表达式，如：

```vala
if (1 < a && a < 5) {}

if (0 < a && a < b && b < c && c < d && d < 255) {
    // do something
}
```

更自然的方式：

```vala
if (1 < a < 5) {}

if (0 < a < b < c < d < 255) {
    // do something
}
```

### 正则表达式

[正则表达式](https://www.regular-expressions.info/)是一种强大的字符串模式匹配技术。Vala 试验性地支持正则表达式字面 (`/regex/`)。示例：

```vala
string email = "tux@kernel.org";
if (/^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,4}$/i.match(email)) {
    stdout.printf("Valid email address\n");
}
```

尾部的`i`使表达式不区分大小写。您可以在`Regex` 类型的变量中存储正则表达式：

```vala
Regex regex = /foo/;
```

正则表达式替换示例：

```vala
var r = /(foo|bar|cow)/;
var o = r.replace ("this foo is great", -1, 0, "thing");
print ("%s\n", o);
```

可以使用以下尾部字符：

-   `i`, 模式中的字母既匹配大写字母也匹配小写字母
    
-   `m`，"行首 "和 "行尾 "结构分别匹配字符串中换行符之后或之前的字符串，以及字符串的最开始和结尾。
    
-   `s`，点元组匹配所有字符，包括换行符。如果不使用，则不包括换行符。
    
-   `x`，模式中的空白数据字符将被完全忽略，除非是转义字符或字符类内的字符。
    

### 严格的非空模式

如果使用 --enable-experimental-non-null 来编译代码，Vala 编译器将以严格的非空类型检查模式运行，默认情况下会将每个类型都视为不可空，除非用问号标记明确声明为可空：

```vala
Object o1 = new Object();     // not nullable
Object? o2 = new Object();    // nullable
```

编译器将执行静态编译时分析，以确保不会将可空引用赋值给不可空引用，例如，这是不可能的：

```vala
o1 = o2;
```

`o2`可能为空，而`o1`已被声明为不可为空，因此这种赋值是被禁止的。不过，如果确定`o2`不是null，可以使用显式非 null 转置来覆盖这种行为：

```vala
o1 = (!) o2;
```

严格的非空模式有助于避免不必要的空引用错误。如果能正确标记绑定中所有返回类型的无效性，这一功能就能充分发挥潜力，但目前情况并非总是如此。

## 库

在系统层面，Vala 库与 C 库完全相同，因此使用的工具也相同。为了简化流程，使 Vala 编译器能够理解流程，还需要额外的 Vala 特定信息。

因此，“Vala库”也是系统的一部分：

-   系统库（如`libgee.so`）
    
-   `pkg-config`条目（例如`gee-1.0.pc`）
    

这两个文件都安装在标准位置。还有Vala专用文件：

-   一个 VAPI 文件（如`gee-1.0.vapi`）
    
-   可选的依赖文件（如`gee-1.0.deps`）
    

本节稍后将解释这些文件。需要注意的是，Vala 特定文件中的库名与`pkg-config`文件中的库名相同。

### 使用库

如果使用`valac`编译器，在 Vala 中使用库在很大程度上是自动化的。Vala 特定的库文件组成一个包。您可以通过以下方式告诉编译器您的程序需要一个软件包：

```shell
valac --pkg gee-1.0 test.vala
```

该命令意味着您的程序可以使用`gee-1.0.vapi`文件中的任何定义，也可以使用`gee-1.0`依赖的任何软件包中的任何定义。如果有依赖包的话，这些依赖包将列在`gee-1.0.deps`文件中。在本例中，`valac`被设置为以二进制方式编译，因此会结合`pkg-config`中的信息来链接正确的库。这就是为什么`pkg-config`名称也被用于 Vala 软件包名称的原因。

包通常与命名空间一起使用，但两者在技术上并无关联。这意味着，即使您的应用程序是参照软件包构建的，您仍必须在每个文件中酌情包含所需的 using 语句，否则就必须使用所有符号的完全限定名称。

也可以将本地库（未安装的库）视为软件包。作为比较，Vala 本身使用内部版本的 Gee。当`valac`生成时，它会为这个内部库创建一个 VAPI 文件，其使用方法大致如下：

```shell
valac --vapidir ../gee --pkg gee ...
```

有关如何生成该库的详细信息，请参阅下一节或示例。

### 创建库

#### 使用Autotools

可以使用 Autotools 创建用 Vala 编写的库。库是通过使用 Vala 编译器生成的 C 代码创建的，与其他库一样进行链接和安装。然后，您需要说明哪些 C 文件必须用于创建库，以及哪些 C 文件必须是可分发的，以便他人在没有 Vala 的情况下使用标准 Autotools 命令编译压缩包：**configure**、**make**和**make install**。

##### 示例

本示例取自 GXml 最近的新增内容。GXmlDom 是一个以 GObject 为基础的 libxml2 替代库；由 Vala 编写，最初用于使用 WAF 构建。

**valac**可用于从 Vala 源生成 C 代码和头文件。目前还可以从 Vala 源文件中生成 GObjectIntrospection 和 VAPI 文件。

**gxml.vala.stamp**用作我们库的代码源。

添加 `--pkg` 开关很重要，这样才能使 valac 成功，并设置 C 库编译和链接所需的所有 CFLAGS 和 LIBS。

```sh
NULL =


AM_CPPFLAGS = \
        -DPACKAGE_LOCALE_DIR=\""$(prefix)/$(DATADIRNAME)/locale"\" \
        -DPACKAGE_SRC_DIR=\""$(srcdir)"\" \
        -DPACKAGE_DATA_DIR=\""$(datadir)"\"

BUILT_SOURCES = gxml.vala.stamp
CLEANFILES = gxml.vala.stamp

AM_CFLAGS =\
         -Wall\
         -g \
        $(GLIB_CFLAGS) \
        $(LIBXML_CFLAGS) \
        $(GIO_CFLAGS) \
        $(GEE_CFLAGS) \
        $(VALA_CFLAGS) \
        $(NULL)

lib_LTLIBRARIES = libgxml.la

VALAFLAGS = \
    $(top_srcdir)/vapi/config.vapi \
    --vapidir=$(top_srcdir)/vapi \
    --pkg libxml-2.0 \
    --pkg gee-1.0 \
    --pkg gobject-2.0 \
    --pkg gio-2.0 \
    $(NULL)

libgxml_la_VALASOURCES = \
        Attr.vala \
        BackedNode.vala \
        CDATASection.vala \
        CharacterData.vala \
        Comment.vala \
        Document.vala \
        DocumentFragment.vala \
        DocumentType.vala \
        DomError.vala \
        Element.vala \
        Entity.vala \
        EntityReference.vala \
        Implementation.vala \
        NamespaceAttr.vala \
        NodeList.vala \
        NodeType.vala \
        Notation.vala \
        ProcessingInstruction.vala \
        Text.vala \
        XNode.vala \
        $(NULL)

libgxml_la_SOURCES = \
        gxml.vala.stamp \
        $(libgxml_la_VALASOURCES:.vala=.c) \
        $(NULL)

# Generate C code and headers, including GObject Introspection GIR files and VAPI file
gxml-1.0.vapi gxml.vala.stamp GXml-1.0.gir: $(libgxml_la_VALASOURCES)
        $(VALA_COMPILER) $(VALAFLAGS) -C -H $(top_builddir)/gxml/gxml-dom.h --gir=GXmlDom-1.0.gir  --library gxmldom-1.0 $^
        @touch $@


# Library configuration
libgxml_la_LDFLAGS =

libgxml_la_LIBADD = \
        $(GLIB_LIBS) \
        $(LIBXML_LIBS) \
        $(GIO_LIBS) \
        $(GEE_LIBS) \
        $(VALA_LIBS) \
        $(NULL)

include_HEADERS = \
        gxml.h \
        $(NULL)


pkgconfigdir = $(libdir)/pkgconfig
pkgconfig_DATA = libgxml-1.0.pc

gxmlincludedir=$(includedir)/libgxml-1.0/gxml
gxmlinclude_HEADERS= gxml-dom.h

# GObject Introspection

if ENABLE_GI_SYSTEM_INSTALL
girdir = $(INTROSPECTION_GIRDIR)
typelibsdir = $(INTROSPECTION_TYPELIBDIR)
else
girdir = $(datadir)/gir-1.0
typelibsdir = $(libdir)/girepository-1.0
endif

# GIR files are generated automatically by Valac so is not necessary to scan source code to generate it
INTROSPECTION_GIRS =
INTROSPECTION_GIRS += GXmlDom-1.0.gir
INTROSPECTION_COMPILER_ARGS = \
    --includedir=. \
    --includedir=$(top_builddir)/gxml

GXmlDom-1.0.typelib: $(INTROSPECTION_GIRS)
        $(INTROSPECTION_COMPILER) $(INTROSPECTION_COMPILER_ARGS)  $< -o $@

gir_DATA = $(INTROSPECTION_GIRS)
typelibs_DATA = GXmlDom-1.0.typelib

vapidir = $(VALA_VAPIDIR)
vapi_DATA=gxmldom-1.0.vapi

CLEANFILES += $(INTROSPECTION_GIRS) $(typelibs_DATA) gxml-1.0.vapi

EXTRA_DIST = \
        libgxml-1.0.pc.in \
        $(libgxml_la_VALASOURCES) \
        $(typelibs_DATA) \
        $(INTROSPECTION_GIRS) \
        gxml.vala.stamp
```

#### 使用命令行进行编译和链接

Vala 还不能直接创建动态或静态库。要创建一个库，请使用 -c（仅编译）开关，并使用您最喜欢的链接器（如 libtool 或 ar）链接对象文件。

```shell
valac -c ...(source files)
ar cx ...(object files)
```

或使用gcc编译中间 C 代码

```shell
valac -C ...(source files)
gcc -o my-best-library.so --shared -fPIC ...(compiled C code files)...
```

##### 示例

下面的示例说明了如何用 Vala 编写一个简单的库，以及如何在本地编译和测试该库，而无需先进行安装。

将以下代码保存到文件`test.vala`。这是实际的库代码，包含我们要从主程序中调用的函数。

```vala
public class MyLib : Object {

    public void hello() {
        stdout.printf("Hello World, MyLib\n");
    }

    public int sum(int x, int y) {
        return x + y;
    }
}
```

使用下一条命令生成`test.c`、`test.h`和`test.vapi`文件。这些文件是要编译的库的 C 语言版本，以及代表库公共接口的 VAPI 文件。

```shell
valac -C -H test.h --library test test.vala --basedir ./
```

现在编译该库：

```shell
gcc --shared -fPIC -o libtest.so $(pkg-config --cflags --libs gobject-2.0) test.c
```

将以下代码保存到名为`hello.vala`的文件中。这段代码将使用我们创建的库。

```vala
void main() {
    var test = new MyLib();
    test.hello();
    int x = 4, y = 5;
    stdout.printf("The sum of %d and %d is %d\n", x, y, test.sum(x, y));
}
```

现在编译应用程序代码，告诉编译器我们要与刚刚创建的库进行链接。

```shell
valac -X -I. -X -L. -X -ltest -o hello hello.vala test.vapi --basedir ./
```

现在我们可以运行程序了。该命令表示将在当前目录下找到所需的任何库。

```shell
LD_LIBRARY_PATH=$PWD ./hello
```

程序的输出结果应该是：

```
Hello World, MyLib
The sum of 4 and 5 is 9
```

您也可以使用 `--gir` 选项为您的库创建 GObjectIntrospection GIR 文件：

```shell
valac -C test.vala --library test --gir Test-1.0.gir
```

GIR 文件是对应用程序接口的 XML 描述。

### 用 VAPI 文件绑定程序库

VAPI 文件是对外部 Vala 库公共接口的描述。用 Vala 编写库时，Vala 编译器会创建该文件，基本上是所有 Vala 源文件中所有公共定义的合并。对于用 C 语言编写的库，VAPI 文件会变得更加复杂，尤其是当库的命名约定不遵循 GLib 约定时。在这种情况下，VAPI 文件将包含许多注释，描述标准化 Vala 接口如何与 C 语言版本相混淆。

创建过程一般分为三个步骤、

-   运行`vala-gen-introspect`从 C 库中提取元数据。
-   添加额外的元数据，使接口标准化或进行其他各种更改。
-   使用`vapigen` 从上述源文件生成 VAPI 文件。

有关如何生成绑定的具体说明，请参阅[《Vala 绑定教程》](https://wiki.gnome.org/Projects/Vala/Bindings)。

## 工具

Vala 发行版包含多个程序，可帮助您构建和使用 Vala 应用程序。有关每个工具的详细信息，请参阅手册。

### valac

`valac` 是 Vala 编译器。它的主要功能是将 Vala 代码转换为可编译的 C 代码。

使用 Vala 时，您通常可以忽略 C 编译器的警告，只需注意 `Valac` 的警告即可。  
Vala 拥有比 C 编译器更好的信息，因此它知道某些东西是有效的，而 C 编译器却无从得知。

遗憾的是，我们不能在所有地方都添加转换，因为在某些情况下，我们无法生成有效的转换。  
（更重要的是，我们无法知道这些情况是什么）。

例如，编译[Hello World](https://wiki.gnome.org/Projects/Vala/Tutorial#A_First_Program)时会出现一些警告，因为 valac 默认生成的代码与旧版本的 GLib 兼容。

想象一下，一台使用旧版 glib 的机器想要运行你的 vala 程序！

valac 可以生成目标 GLib 版本的 C 代码：

```shell
valac --target-glib auto hello.vala     # It will use the latest version of GLib which may not be compatible
```

建议的方法是通过向 C 编译器传递选项来禁用这些警告：

```shell
valac -X -w hello.vala     # Generated code is compatible, `-X` will pass `-w` to C compiler to disable all warnings.
```

您可以在 bash/zsh/fish shell 中设置一个别名。

valac 还能在简单情况下自动执行整个构建和链接项目：

```shell
valac -o appname --pkg gee-1.0 file_name_1.vala file_name_2.vala
```

`-o`开关要求创建对象文件，而不仅仅是输出C源文件。`--pkg`选项表示本次链接需要`gee-1.0`软件包中的信息。你不需要指定链接哪些库的细节，软件包内部就有这些信息。最后，会给出一个源文件列表。如果需要更复杂的编译过程，可以使用 -C 开关生成 C 文件而不是二进制文件，然后手动或通过脚本继续编译。

### vapigen

vapigen 是一种绑定工具。它可根据库的元数据和任何所需的额外信息创建 VAPI 文件。另请参阅[《Vala 绑定教程》](https://wiki.gnome.org/Projects/Vala/Bindings)。

### vala-gen-introspect

vala-gen-introspect 是一个用于提取基于 GObject 的库的元信息的工具。现在，首选方法是使用 GObjectIntrospection，因为 vapigen 可以直接使用 GIR 文件。另请参阅[《Vala 绑定教程》](https://wiki.gnome.org/Projects/Vala/Bindings)。

## 技术

### 调试

为演示起见，我们将故意引用空引用，从而创建一个错误程序，这将导致分段故障：

```vala
class Foo : Object {
    public int field;
}

void main() {
    Foo? foo = null;
    stdout.printf("%d\n", foo.field);
}
```

```shell
valac debug-demo.vala
./debug-demo
Segmentation fault
```

那么我们该如何调试这个程序呢？命令行选项 -g 告诉 Vala 编译器在编译后的二进制文件中包含 Vala 源代码行信息，--save-temps 保留临时 C 源文件：

```shell
valac -g --save-temps debug-demo.vala
```

Vala 程序可使用 GNU 调试器 gdb 或其图形前端（如[Nemiver](https://projects.gnome.org/nemiver/)）进行调试。

```shell
nemiver debug-demo
```

gdb调试示例：

```shell
gdb debug-demo
(gdb) run
Starting program: /home/valacoder/debug-demo

Program received signal SIGSEGV, Segmentation fault.
0x0804881f in _main () at debug-demo.vala:7
7           stdout.printf("%d\n", foo.field);
(gdb)
```

### 使用 GLib

GLib 包含大量实用程序，包括大多数标准 libc 函数的封装程序等。这些工具适用于所有 Vala 平台，甚至那些不兼容 POSIX 的平台。有关 GLib 提供的所有功能的完整描述，请参阅[《GLib 参考手册》](https://developer.gnome.org/glib/)。该参考资料与 GLib 的 C API 有关，但主要是如何将其翻译成 Vala 非常简单。

GLib 函数可通过以下命名约定在 Vala 中使用：
| C API | Vala | 示例 |
| --- | --- | --- |
| g_topic_foobar() | GLib.Topic.foobar() | GLib.Path.get_basename() |

GLib 类型也可以类似地使用：

```vala
GLib.Foo foo = new GLib.Foo();
foo.bar();
```

C 和 Vala 的 API 并不完全相同，但这些命名规则意味着您可以在 Vala 附带的 GLib VAPI 文件中找到所需的函数，并从中找到参数。不用翻看更多 Vala 文档，这样已经足够了。

#### 文件处理

有关灵活的文件 I/O 和文件处理，请参阅[GIO 示例](https://wiki.gnome.org/Projects/Vala/GIOSamples)。

您还可以使用 FileUtils.get_contents 将文件载入字符串。

```vala
string content;
FileUtils.get_contents("file.vala", out content);
```
