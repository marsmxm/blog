---
title: "依赖类型与定理证明"
date: 2019-04-14T00:56:13+08:00
draft: true
author: "Mu Xian Ming"

categories: [P]
tags: [dependent type, idris, pie]
---

## 什么是依赖类型

程序语言中的类型描述的是程序的行为，可以被用来证明程序不会在运行时出现某些错误行为。在一般的静态类型语言里，类型和值有明确的区分，类型信息可以出现在哪也有严格的限制。而在支持依赖类型（dependent type）的语言里，类型和值之间的界限变得模糊，类型可以像值一样被计算：可以把类型作为参数传递给函数，函数也可以把类型像值一样返回。同时值也可以出现在类型信息里，依赖类型的“依赖”两个字指的就是是类型可以依赖于值。因此类型和值的关系不再是单向的，两者变得可以互相描述了。这样的好处一是类型系统变得更加强大，可以检测并阻止更多的错误类型，让程序更可靠；二是有了依赖类型之后，我们甚至可以让计算机像运行传统的计算类程序一样来运行数学证明。比如著名的[四色定理](https://en.wikipedia.org/wiki/Four_color_theorem)的证明就是在 1976 年由计算机的定理证明程序来辅助推导得出的。

为了对依赖类型有一个更直观的理解，举一个具体一些的例子。假如 Java 中加入了对依赖类型的支持，那么我们以 Java 的数组类型为例，可以让它包含更多的信息，例如数组的长度：

```java
String[3] threeNames = {"张三", "李四", "成吉思汗"}; // 虚构的语法
```

这么做的好处是，所有围绕于数组类型的函数或类现在都可以在类型信息中包含更具体的对行为的描述。比如合并两个数组的函数`concat`的签名可以写成：

```java
String[n + m] concat(String[n] start, String[m] end) {...} // 虚构的语法
```

通过类型就可以知道`concat`返回的数组的长度是两个参数数组长度的和。这样不仅程序变得更易读，所有可以通过数组长度反映出来的程序错误都能在运行前被检测出来（后面会有实例说明）。

当然，为了更直观准确的阐释依赖类型，需要一个实在的能运行起来的程序语言，而本文所用到的语言叫做 [Pie](https://github.com/the-little-typer/pie)。它是为了 Friedman 和 Christiansen 的[《The Little Typer》](http://thelittletyper.com/)这本书被开发出来的包含依赖类型所有核心功能的一个小巧简洁的语言。

### 准备工作

Pie 是一个用 [Racket](https://racket-lang.org/) 开发的语言，所以它的开发环境也是基于 DrRacket。具体的安装步骤很简单，可以参照[官方网站](http://thelittletyper.com)上的指示。这里再说一个让开发环境更易用的设置。安装成功后打开 DrRacket 的 Preferences 界面（Mac 上是 ⌘+,），将 Background expansion 设置成 "with gold highlighting"：

![racket-pref](/dt/pie-pref.png)

这样可以更容易地定位到问题代码：

![err1](/dt/err1.png)

### Pie 语言简介

Pie 的语法类似于 Lisp 的各种方言（如 Scheme，Racket 和 Clojure），这就意味着程序中括号的数量会比较可观 :P。在 Pie 语言里，每个变量或函数在定义前都需要声明类型：

```pie
(claim age
  Nat)
(define age
  18)
```

这里是先声明变量`age`的类型为自然数（Nat），再把它的值定义为 18。Pie 的几个基础类型分别是，

- 自然数，`Nat`。表示所有大于等于 0 的整数。在程序里直接用`0`，`1`，`2`等数字来表示类型为 Nat 的值。
- 原子，`Atom`。用单引号接上一个或多个字母或连字符来表示。相当于 Lisp 中的 symbol 类型，或者也可以粗略的等同于大部分语言里的字符串。`'abc`，`'an-atom`和`'Atom`都是原子类型的值。
- 函数，`(-> Type-1 Type-2 ... Type-n)`。这里`Type-n`是函数的返回值类型，其他的`Type-1 Type-2 ...`是参数类型。