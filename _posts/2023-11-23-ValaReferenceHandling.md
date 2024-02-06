---
title: Vala内存管理详解
author: kernel
date: 2023-11-23 10:34:00 +0800
categories: [Vala语言, 教程]
tags: [Vala语言, Vala官方教程, Vala入门]
---

Vala 的内存管理基于自动引用计数，而不是跟踪垃圾回收。

这种方法[有利有弊](https://en.wikipedia.org/wiki/Reference_counting#Advantages_and_disadvantages)。引用计数是确定性的，但在某些情况下会形成引用循环。在这种情况下，必须使用[弱引用](https://en.wikipedia.org/wiki/Weak_reference)才能打破这些循环。Vala 的关键字`weak`就是为弱引用设计的。

这就是自动参考计数的工作原理：

每次将引用类型对象赋值给变量（`referenced`）时，其内部引用计数就会加一（`ref`）；每次引用变量退出作用域时，该对象的内部引用计数就会减一（`unref`）。

如果对象的内部引用计数为 0，该对象就会被释放。释放对象时，其每个引用类型数据成员的引用计数都会减一（`unref`）。如果其中一个数据成员的引用计数为 0，则该数据成员将被释放，释放过程相同，依此类推。

但什么是引用循环，它为什么不好？让我们来看看一个简单的无数据[双链表](https://en.wikipedia.org/wiki/Doubly_linked_list)的实现：

```vala
class Node : Object {
    public Node prev;
    public Node next;

    public Node (Node? prev = null) {
        this.prev = prev;      // ref
        if (prev != null) {
            prev.next = this;  // ref
        }
    }
}

void main () {
    var n1 = new Node ();    // ref
    var n2 = new Node (n1);  // ref

    // print the reference count of both objects
    stdout.printf ("%u, %u\n", n1.ref_count, n2.ref_count);

}   // unref, unref

```

为便于您更好地理解，本例对发生引用和非引用的地方进行了注释。下图说明了`n1`和`n2`分配后的情况：

![refcycle1.png](/assets/img/2023-11-23/refcycle1.png)

图中的每个箭头（边）代表一个引用，每个节点都有一个数字，表示对象的引用计数（指向该对象的箭头数）。你可以很容易地发现引用循环（图中的往返箭头）。在代码块结束时，`n1`和`n2`被取消引用后，这两个节点的引用计数都将变为 1，但都不会变为 0 并被释放。

在上面的情况下，我们是幸运的，因为程序无论如何都会终止，操作系统无论如何都会释放进程的所有内存。但如果程序运行的时间更长，会发生什么情况呢？举个例子

```vala
void main () {
    while (true) {
        var n1 = new Node ();
        var n2 = new Node (n1);
        Thread.usleep (1000);
    }
}
```

打开任务管理器或进程监视器（如`gnome-system-monitor`）并启动程序。你可以看到它在吞噬你的内存。在它拖慢系统运行速度之前将其中断。（内存会立即被释放）。

类似的 C# 或 Java 程序不会有任何问题，因为跟踪垃圾回收器会定期丢弃所有从当前作用域无法直接或间接访问的对象。但在 Vala 中，您必须在引用循环的情况下采取对策。

我们可以将循环中的一个引用变为弱引用，从而打破循环：

```vala
    public weak Node prev;
    public Node next;
```

这意味着对这样的变量赋值不会增加对象的引用计数。现在的情况是这样的

![refcycle2.png](/assets/img/2023-11-23/refcycle2.png "refcycle2.png")

第一个对象的引用计数是 1，而不是之前的 2。当`n1`和`n2`在代码块结束退出作用域时，两个对象的引用计数都减少了 1。由于第一个对象的引用计数为 0，因此第一个对象被释放，第二个对象的引用计数为 1。然而，由于第一个对象持有对第二个对象的引用，因此第二个对象的引用计数再次减少，为 0，第二个对象也被释放。

再次运行程序，你会发现内存消耗保持稳定。

循环引用不一定是直接循环。这也是一个循环引用：

![refcycle3.png](/assets/img/2023-11-23/refcycle3.png "refcycle3.png")

## 非拥有引用

Vala 类的所有对象和基于 gobject 库的大多数对象都有引用计数。不过，Vala 也允许您使用默认不支持引用计数的非基于对象的 C 库类。这些类称为紧凑类（用`[Compact]`属性注释）。

非引用计数对象可能只有一个强引用（可将其视为“拥有”引用）。当这个引用超出范围时，对象就会被释放。所有其他引用必须是非拥有引用。当这些引用超出范围时，对象不会被释放。

![unowned-reference.png](/assets/img/2023-11-23/unowned-reference.png "unowned-reference.png")

因此，当你调用一个方法（很可能是非对象库中的方法）时，该方法会返回一个你感兴趣的对象的非拥有引用（这在方法的返回类型中用`unowned`关键字标记），这时你有两个选择：要么复制对象（如果它有复制方法），然后你可以用自己的单强引用到新复制的对象；要么将引用赋值给一个用`unowned`关键字声明的变量，这样 Vala 就会知道它不应该释放你这边的对象。

Vala 禁止将非拥有引用分配给强引用（即非非拥有引用）。但您可以使用`(owned)`将所有权转移给另一个引用：

```vala
[Compact]
class Foo { }

void main () {
    Foo a = new Foo ();
    unowned Foo b = a;
    Foo c = (owned) a;   // 'c' is now the new "owning" reference
}
```

![ownership-transfer.png](/assets/img/2023-11-23/ownership-transfer.png "ownership-transfer.png")

顺便说一下，Vala 字符串也没有引用计数，因为它们是基于 C 类型的`char*` 。不过，Vala 会在需要时自动复制它们。因此您实际上不必考虑它们。只有绑定的编写者才需要考虑字符串引用是否为非拥有，并在适当的 API 中进行标记。

目前，Vala 绑定 API 中的许多类型都被错误地标记为弱类型，即使它们实际上应该是非拥有的。这是因为在过去，这两种类型都只有`weak`关键字，所以不要被这一点所迷惑。目前，`weak`和`unowned`可以互换使用。不过，你应该只在打破引用循环时使用`weak`关键字，而在出现上述所有权问题时才使用`unowned`关键字。

## 内存管理实例

### 正常的引用计数类

```vala
public class Foo {
    public void method () { }
}

void main () {
    Foo foo = new Foo ();    // allocate, ref
    foo.method ();
    Foo bar = foo;           // ref
}  // unref, unref => free

```

一切都是自动管理的。

### 使用指针语法手动管理内存

如果您觉得必须完全控制内存，也可以选择手动管理内存。指针语法与 C/C++ 类似：

```vala
void main () {
    Foo* foo = new Foo ();   // allocate
    foo->method ();
    Foo* bar = foo;
    delete foo;              // free
}
```

### 无引用计数的紧凑型类

紧凑类是未在 Vala 类型系统中注册的类。它们通常是从非基于 gobject 的 C 库中绑定的类。不过，，如果您能接受，您也可以在 Vala 中定义自己的紧凑类，但这些紧凑类的功能非常有限（例如，没有继承、没有接口实现、没有私有字段等）。

创建和销毁紧凑类的速度比非紧凑类快 （大约是普通类的 2.5 倍，比 GObject 派生类快一个数量级），但其他操作没有什么不同。 尽管如此，现代硬件每秒可以在每个线程中创建和销毁数百万个GObject 派生类，因此在尝试优化之前，最好先确定这是性能瓶颈。

紧凑型类默认不支持引用计数。因此，，只有一个“拥有”引用会在对象退出作用域时导致对象被释放，所有其他引用都必须是非拥有的。

```vala
[Compact]
public class Foo {
    public void method ();
}

void main () {
    Foo foo = new Foo ();    // allocate
    foo.method ();
    unowned Foo bar = foo;
    Foo baz = (owned) foo;   /* ownership transfer: now 'baz' is the "owning"
                                reference for the object */
    unowned Foo bam = baz;
} // free ("owning" reference 'baz' went out of scope)

```

### 有引用计数的紧凑型类

您可以通过手动实现将引用计数添加到紧凑型类中。您必须使用`CCode`属性告诉 Vala 它应使用哪些函数进行引用和取消引用。

```vala
[Compact]
[CCode (ref_function = "foo_up", unref_function = "foo_down")]
public class Foo {

    public int ref_count = 1;

    public unowned Foo up () {
        GLib.AtomicInt.add (ref this.ref_count, 1);
        return this;
    }

    public void down () {
        if (GLib.AtomicInt.dec_and_test (ref this.ref_count)) {
            this.free ();
        }
    }

    private extern void free ();
    public void method () { }
}

void main () {
    Foo foo = new Foo ();    // allocate, ref
    foo.method ();
    Foo bar = foo;           // ref
} // unref, unref => free

```

正如您所看到的，一切都将重新自动管理。

### 具有复制功能的不可变紧凑型类

如果一个紧凑类不支持引用计数，但它是不可变的 （其内部状态不会改变），并且有一个复制函数来复制对象，那么如果它被分配给一个强引用，Vala 就会自动复制它。

```vala
[Compact]
[Immutable]
[CCode (copy_function = "foo_copy")]
public class Foo {
    public void method () { }

    public Foo copy () {
        return new Foo ();
    }
}

void main () {
    Foo foo = new Foo ();   // allocate
    foo.method ();
    Foo bar = foo;          // copy
} // free, free

```

如果出于微调的原因需要防止复制，仍可以使用非自有引用：

```vala
void main () {
    Foo foo = new Foo ();   // allocate
    foo.method ();
    unowned Foo bar = foo;
} // free

```