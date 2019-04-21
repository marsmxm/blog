---
title: "依赖类型与定理证明"
date: 2019-04-14T00:56:13+08:00
draft: true
author: "Mu Xian Ming"

categories: [P]
tags: [dependent type, idris, pie, coq]
---

## 什么是依赖类型

程序语言中的类型描述的是程序的行为，可以保证程序在运行时不会出现某一类的错误行为。一般的静态类型语言对类型和值有明确的区分，对于类型信息可以出现在哪也有严格的限制。而在支持依赖类型（dependent type）的语言里，类型和值之间的界限变得模糊，类型可以像值一样被计算（即所谓的 first-class 类型）：可以把类型作为参数传递给函数，函数也可以把类型像值一样返回。同时值也可以出现在类型信息里，依赖类型的“依赖”两个字指的就是类型可以依赖于值。因此类型和值的关系不再是单向的，两者变得可以互相描述了。这样的好处一是类型系统变得更加强大，可以检测并阻止更多的错误类型，让程序更可靠；二是有了依赖类型之后，我们甚至可以让计算机像运行传统的计算类程序一样来运行数学证明。比如著名的[四色定理](https://en.wikipedia.org/wiki/Four_color_theorem)的证明就是在 1976 年由计算机的定理证明程序来辅助推导得出的。

为了对依赖类型有一个更直观的理解，举一个假想的例子。假如 Java 中加入了对依赖类型的支持，那么我们以 Java 的数组类型为例，可以让它包含更多的信息，例如数组的长度：

```java
String[3] threeNames = {"张三", "赵四", "成吉思汗"}; // 虚构的语法
```

这么做的好处是，所有围绕于数组类型的函数或类现在都可以在类型信息中包含更具体的对行为的描述。比如合并两个数组的函数`concat`的签名可以写成：

```java
String[n + m] concat(String[n] start, String[m] end) {...} // 虚构的语法
```

通过类型就可以知道`concat`返回的数组的长度是两个参数数组长度的和。这样不仅程序变得更易读，所有可以借由数组长度反映出来的程序错误都能在运行前被检测出来（后面会有实例说明）。再举一个 first-class 类型的例子，下面是用 Idris 写的一个程序片段，Idris 是一个支持依赖类型、和 Hashkell 非常接近的语言。

```idris
StringOrInt : Bool -> Type
StringOrInt x = case x of
  True => Int
  False => String

getStringOrInt : (x : Bool) -> StringOrInt x
getStringOrInt x = case x of
  True => 42
  False => "Fourty two"

valToString : (x : Bool) -> StringOrInt x -> String
valToString x val = case x of
  True => cast val
  False => val
```

这段程序里的`StringOrInt`、`getStringOrInt`和`valToString`是三个函数，第 1、6、11 行分别声明的这三个函数的类型。`StringOrInt`接受一个布尔类型的参数，返回值的类型是`Type`（也就是**类型的类型**，因为类型也是一种值），当参数是`True`，返回`Int`类型，参数为`False`时，返回`String`类型。而在`getStringOrInt`的类型声明中可以看到它的返回值类型是`StringOrInt x`，也就是说返回值类型依赖于参数`x`的值：当`x`的值为`True`时，返回值类型是`StringOrInt True`的值，也就是`Int`；当`x`是`False`是，返回值类型就变成了`String`。第 6 行的类型声明体现了依赖类型的两个特性：

1. 类型中可以包括变量或者复杂的表达式，如`StringOrInt x`。
2. 声明的类型可以处于一个动态的状态，最后具体是什么取决于所依赖的值。

最后一个函数`valToString`和`getStringOrInt`的类型类似，区别只在于它的第二个参数类型依赖于参数`x`的值。如果`x`是`True`，参数`val`的类型就是`Int`，又因为返回值类型是`String`，所以需要用`cast`函数做一个类型转换；若`x`等于`False`，参数`val`的类型和返回值类型一致，就可以直接返回。

前边两个例子是为了给予一个直观的感受，为了更详细准确地阐释依赖类型以及基于依赖类型的定理证明，本文将要用到一个叫做 [Pie](https://github.com/the-little-typer/pie) 的语言。它是为了 Friedman 和 Christiansen 的[《The Little Typer》](http://thelittletyper.com/)这本书而被开发出来的包含依赖类型所有核心功能的一个小巧简洁（完整的[参考手册](https://docs.racket-lang.org/pie/index.html)只有不到 3000 字）的语言。

### 准备工作

Pie 是一个用 [Racket](https://racket-lang.org/) 开发的语言，所以它的开发环境也是基于 DrRacket。具体的安装步骤很简单，可以参照[官方网站](http://thelittletyper.com)上的指示。这里再说几个让开发环境更易用的设置。一是安装成功后可以在 View 菜单设置一下 layout 和 toolbar：

![conf-layout](/dt/conf1.png)
![conf-toolbar](/dt/conf2.png)

下图是设置之后的界面布局。左右两个编辑区域分别是定义和交互区，工具栏在最右侧垂直排列。在定义区输入变量、函数定义等程序的主体部分后，点击工具栏的绿色箭头（或者按快捷键 ⌘-R 或 Ctrl-R）来运行，这时右侧的交互区就变成一个基于当前程序的 REPL 环境。

![racket-pref](/dt/whole.png)

再有就是可以打开 DrRacket 的 Preferences 界面（⌘-, 或 Ctrl-,），将 Background expansion 设置成 "with gold highlighting"：

![racket-pref](/dt/conf3.png)

这样就可以更容易地定位到程序中有问题的部分：

![err1](/dt/err1.png)

### Pie 语言简介

Pie 的语法类似于 Lisp 的各种方言（Scheme，Racket 和 Clojure 等），也就意味着程序中括号的数量会比较可观。在 Pie 语言里，每个变量或函数在定义前都必须声明类型：

```pie
(claim age
  Nat)
(define age
  18)
```

这里是先声明变量`age`的类型为自然数`Nat`，再把它的值定义为`18`。Pie 的几个基础类型分别是，

- 自然数`Nat`。表示所有大于等于 0 的整数。
- 原子`Atom`。相当于 Lisp 中的 symbol 类型，或者也可以粗略的近似于大部分语言里的字符串。`Atom`的值由单撇号和一个或多个字母或连字符组成，例如`'abc`，`'---`和`'Atom`。
- 函数`(-> Type-1 Type-2 ... Type-n)`。这里最后的`Type-n`是函数的返回值类型，其他的`Type-1 Type-2 ...`是参数类型。

还有一些复合类型会在后面用到的时候再作详细解释。Pie 的每个类型都有对应的 constructor 和 eliminator。前者用来构造该类型的值；后者的用处是**运用该类型的值所包含的信息来得到所需要的新值**，如果把某种类型的值看作一个罐头的话，那它对应的 eliminator 就像一个起子，而且不同构造的罐头需要用到不一样的起子。

#### 函数

函数的 constructor 拥有如下结构：

```pie
(lambda (x1 x2 ...)
  e1 e2 ...)
```

`x1 x2 ...`是函数的参数，`e1 e2 ...`是作为函数体的一个或多个表达式。另外`lambda`关键字也可以用希腊字母`λ`（在 DrRacket 中可以通过快捷键 ⌘-\ 输入）来代替，让程序看起来更简洁。下面是一个完整的函数定义示例：

```pie
(claim echo
  (-> Atom
    Atom))
(define echo
  (λ (any-atom)
    any-atom))
```

这里先是声名这个函数的类型：接受一个`Atom`类型的值作为参数，然后返回一个`Atom`类型的值。接着就是函数`echo`的具体定义，无论传入什么`Atom`都原样返回。对于函数来说 eliminator 只有一个，那就是对函数的调用，只能借由这唯一的途径来使用定义好的函数。函数的调用语法是，`(`后第一个写函数或指向函数的变量，紧跟着一个或多个参数，最后以`)`收尾：

```pie
> (echo '你好)
(the Atom '你好)
```

以`>`开头的一行表示这是在交互区输入的内容，下面紧跟着的是这行运行后得到的值。`'你好`前面的`the Atom`是对这个值的类型注解。对于在交互区输入的表达式，Pie 都会以 (the 类型 值) 的形式来显示它的值：

```pie
> 18
(the Nat 18)
```

#### Atom

对于`Atom`来说所有符合规则的值都是一个 constructor，而且`Atom`没有对应的 eliminator，因为每个值本身就是它的意义所在。

#### Nat

自然数`Nat`的 constructor 是`zero`和`add1`。`zero`顾名思义得到的是最小自然数零，而`add1`接受一个自然数`n`作为参数，得到一个比`n`大`1`的自然数。所以自然数 1、2、3 可以分别表示成`(add1 zero)`、`(add1 (add1 zero))`、`(add1 (add1 (add1 zero)))`。如果用归纳法来定义`Nat`的话，可以写作：

```code
Nat ::= zero
Nat ::= (add1 Nat)
```

用这种方式定义出来的自然数又叫做[皮亚诺数（Peano number）](https://wiki.haskell.org/Peano_numbers)。当然这种表示方式写起来有些繁琐，所以 Pie 也提供了更便捷的语法：可以直接把自然数写作数字，例如`(add1 (add1 zero))`和`2`就表示的是同一个自然数。对于自然数， Pie 提供了多个可供使用的 eliminator。具体使用哪个取决于要解决问题的复杂度。比如一个类型为`(-> Nat Nat)`的函数`pred`，规定它对于自然数`0`返回`0`，对于其他自然数返回小一的自然数：

```pie
(claim pred
  (-> Nat
    Nat))
(define pred
  (λ (n)
    (which-Nat n
      0
      (λ (n-1)
        n-1))))
```

