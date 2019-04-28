---
title: "依赖类型与定理证明"
date: 2019-04-14T00:56:13+08:00
draft: true
author: "Mu Xian Ming"

categories: [P]
tags: [dependent type, idris, pie, coq]
---

## 什么是依赖类型

程序语言中的类型描述的是程序的行为，可以保证程序在运行时不会出现某一类的错误行为。一般的静态类型语言对类型和值有明确的区分，对于类型信息可以出现在哪也有严格的限制。而在支持依赖类型（dependent type）的语言里，类型和值之间的界限变得模糊，类型可以像值一样被计算（即所谓的 first-class 类型 [^1]），同时值也可以出现在类型信息里，依赖类型的“依赖”两个字指的就是类型可以依赖于值。因此类型和值的关系不再是单向的，两者变得可以互相描述了。这样的好处一是类型系统变得更加强大，可以检测并阻止更多的错误类型让程序更可靠；二是有了依赖类型之后，我们甚至可以让计算机像运行传统的计算类程序一样来运行数学证明 [^2]。

为了对依赖类型有一个更直观的理解，举一个假想的例子。假如 Java 中加入了对依赖类型的支持，那么我们以 Java 的数组类型为例，可以让它包含更多的信息，例如数组的长度：

```java
String[3] threeNames = {"张三", "赵四", "成吉思汗"}; // 虚构的语法
```

这么做的好处是，所有围绕于数组类型的函数或类现在都可以在类型信息中包含更具体的对行为的描述。比如合并两个数组的函数`concat`的签名可以写成：

```java
String[n + m] concat(String[n] start, String[m] end) {...} // 虚构的语法
```

这里通过类型就可以知道`concat`返回的数组的长度是两个参数数组长度的和。这样不仅程序变得更易读，所有可以借由数组长度反映出来的程序错误都能在运行前被检测出来（后面会有实例说明）。再举一个 first-class 类型的例子，下面是用 Idris 写的一个程序片段，Idris 是一个支持依赖类型、和 Hashkell 非常接近的语言。

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

最后一个函数`valToString`的类型和`getStringOrInt`的类似，区别只在于它的第二个参数类型依赖于参数`x`的值。如果`x`是`True`，参数`val`的类型就是`Int`，又因为返回值类型是`String`，所以需要用`cast`函数做一个类型转换；若`x`等于`False`，参数`val`的类型和返回值类型一致，就可以直接返回。

有了一个直观的感受后，为了更详细准确地阐释依赖类型以及基于依赖类型的定理证明，本文将要用到一个叫做 [Pie](https://github.com/the-little-typer/pie) 的语言。它是为了 Friedman 和 Christiansen 的[《The Little Typer》](http://thelittletyper.com/)这本书而被开发出来的包含依赖类型所有核心功能的一个小巧简洁（完整的[参考手册](https://docs.racket-lang.org/pie/index.html)只有不到 3000 字）的语言。

### 准备工作

Pie 是一个用 [Racket](https://racket-lang.org/) 开发的语言，所以它的开发环境也是基于 DrRacket。具体的安装步骤很简单，可以参照[官方网站](http://thelittletyper.com)上的指示。这里说几个让开发环境更易用的设置。一是 DrRacket 安装成功后可以在 View 菜单设置一下 layout 和 toolbar：

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

`x1 x2 ...`是函数的参数，`e1 e2 ...`是作为函数体的一个或多个表达式，函数体的最后一个表达式的值同时也是函数的返回值。另外`lambda`关键字也可以用希腊字母`λ`来代替 [^3]，让程序看起来更简洁。下面是一个完整的函数定义示例：

```pie
(claim echo
  (-> Atom
    Atom))
(define echo
  (λ (any-atom)
    any-atom))
```

这里先是声名这个函数的类型：接受一个`Atom`类型的值作为参数，然后返回一个`Atom`类型的值。接着就是函数`echo`的具体定义，无论传入什么`Atom`都原样返回。对于函数来说 eliminator 只有一个，那就是对函数的调用，只能借由这唯一的途径来使用定义好的函数。函数的调用语法是，`(函数名或匿名函数 参数1 参数2 ...)`：

```pie
> (echo '你好)
(the Atom '你好)
```

以`>`开头的一行表示这是在交互区输入的内容，下面紧跟着的是这行运行后得到的值。`'你好`前面的`the Atom`是对这个值的类型注解。对于在交互区输入的表达式，Pie 都会以 (the 类型 值) 的形式来显示它的值：

```pie
> 18
(the Nat 18)
```

另外，(the 类型 ...) 表达式也可以用来在解释器无法判断当前表达式的类型的情况下给予解释器一个提示：

![example-the](/dt/ex1.png)

#### Atom

对于`Atom`来说所有符合规则的值都是一个 constructor，而且`Atom`没有对应的 eliminator，因为每个值本身就是它的意义所在。

#### Nat

自然数`Nat`的 constructor 是`zero`和`add1`。`zero`顾名思义得到的是最小自然数零，而`add1`接受一个自然数`n`作为参数，得到一个比`n`大`1`的自然数。所以自然数 1、2、3 可以分别表示成`(add1 zero)`、`(add1 (add1 zero))`、`(add1 (add1 (add1 zero)))`。如果用归纳法来定义`Nat`的话，可以写作：

```code
Nat ::= zero
Nat ::= (add1 Nat)
```

用这种方式定义出来的自然数又叫做[皮亚诺数（Peano number）](https://wiki.haskell.org/Peano_numbers)。当然这种表示方式写起来有些繁琐，所以 Pie 也提供了更便捷的语法：可以直接把自然数写作数字，例如`(add1 (add1 zero))`和`2`就表示的是同一个自然数。对于自然数， Pie 提供了多个可供使用的 eliminator。具体使用哪个取决于要解决的问题。比如现在要写一个类型为`(-> Nat Nat)`的函数`pred`，规定它对于自然数`0`返回`0`，对于其他自然数返回比自身小一的数：

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

函数`pred`用到的 eliminator 是 [which-Nat](https://docs.racket-lang.org/pie/index.html#%28def._%28%28lib._pie%2Fmain..rkt%29._which-.Nat%29%29)。`which-Nat`的使用方式是`(which-Nat target base step)`，它的值由三个参数决定。如果用`X`指代`which-Nat`的返回值类型，那么`target`的类型是`Nat`，`base`的类型是`X`，而`step`是一个类型为`(-> Nat X)`的函数。

![which-Nat](/dt/which-Nat.png)

当`target`等于`0`时，`base`的值即为`which-Nat`的值；如果`target`不为`0`，即`target`可以表示成形如`(add1 n-1)`的自然数（这里的`n-1`是一个普通的标识符，不是`n`减`1`的表达式），这时`which-Nat`的值等于`step`函数作用于`n-1`（即比`target`小`1`的自然数）所得到的值。

在上述函数`pred`中，`which-Nat`的`target`是`pred`的参数`n`，`base`是`0`，`step`是函数`(λ (n-1) n-1)`。所以当`n`等于`0`时，`which-Nat`返回`base`的值，`0`；当`n`大于`0`或者说`n`等于`(add1 n-1)`时，`which-Nat`返回的就是`(step n-1)`的值，即参数`n-1`本身。

```pie
> (pred 10086)
(the Nat 10085)
> (pred 0)
(the Nat 0)
```

如果熟悉所谓的函数式语言，可以看出来`which-Nat`其实就是这些语言里的 pattern matching（虽然并不完全等同，后边会说到区别在哪）。如果用 Idris 实现`pred`的话，会是这个样子：

```Idris
pred : Nat -> Nat
pred Z = Z
pred (S k) = k
```

在 Idris 里，`Z`和`S k`分别是`Nat`类型的两个 constructor，等同于 Pie 里的`zero`和`(add1 n)`。后两行的两个 pattern matching 的分支也对应于`which-Nat`的`base`和`step`。前面说到的`which-Nat`和 pattern matching 的区别指的是，在常规的 pattern matching 中可以对函数递归调用，而 Pie 并不允许用户定义的函数对自身的递归调用，这个限制的目的是为了保证所有的函数都是 [**total**](https://en.wikipedia.org/wiki/Partial_function#Total_function) 的。举实现自然数加法运算的函数为例，在支持递归调用的语言比如 Idris 中可以这样实现（递归在最后一行）：

```Idris
plus : Nat -> Nat -> Nat
plus Z m = m
plus (S k) m = S (plus k m)
```

如果 Pie 允许函数的递归调用，完全可以用`which-Nat`实现同样的逻辑：

```Pie
(claim +
  (-> Nat Nat
    Nat))
(define +
  (λ (n m)
    (which-Nat n
      m
      (λ (n-1)
        (add1 (+ n-1 m)))))) ; error: Unknown variable +
```

不幸的是如果把上面这段程序输入到定义区域，DrRacket 会在最后一行提示 “Unknown variable +”。所以我们只能改成使用“内置”了递归的 eliminator，`rec-Nat`：

```Pie
(claim +
  (-> Nat Nat
    Nat))
(define +
  (λ (n m)
    (rec-Nat n
      m
      (λ (n-1 n-1+m)
        (add1 n-1+m)))))
```

[rec-Nat](https://docs.racket-lang.org/pie/index.html#%28def._%28%28lib._pie%2Fmain..rkt%29._rec-.Nat%29%29) 和`which-Nat`类似，接受的三个参数也是`target`、`base`和`step`，而且当`target`等于`0`时同样把`base`作为自身的值返回。

![rec-Nat](/dt/rec-Nat.png)

不同的是`rec-Nat`的`step`参数类型是`(-> Nat X X)`。当`target`可以表示成`(add1 n-1)`的非零自然数时，`step`的两个参数分别是`n-1`和`(rec-Nat n-1 base step)`。也就是说`rec-Nat`内置了对自身的递归调用，作为`rec-Nat`的使用者只需要知道`step`的**第一个参数是比当前的非零`target`小一的自然数，第二个参数等于把第一个参数作为新的`target`传递给递归的`rec-Nat`所得到的值**。对于上面的加法例子，`step`的第二个参数就是比`n`小一的自然数与`m`的和。

为了加深对`rec-Nat`的理解，我们可以模拟一下解释器对`(+ 2 1)`的求值过程。当解释器遇到表达式`(+ 2 1)`时，首先会判断这是一个函数调用，所以第一步把函数名替换成实际的函数定义：

```pie
((λ (n m)
    (rec-Nat n
      m
      (λ (n-1 n-1+m)
        (add1 n-1+m))))
 2 1)
```

然后将函数体中的`n`和`m`分别替换成`2`和`1`：

```pie
(rec-Nat 2
  1
  (λ (n-1 n-1+m)
    (add1 n-1+m)))
```

因为`target`等于非零自然数`2`，也就是`(add1 1)`，所以`rec-Nat`的值就等于对`step`函数调用后的值（将`step`的函数体`(add1 n-1+m)`中的`n-1+m`替换成对`rec-Nat`的递归调用）：

```pie
(add1
  (rec-Nat 1
    1
    (λ (n-1 n-1+m)
      (add1 n-1+m))))
```

这时内层的`rec-Nat`的`target`参数等于`(add1 zero)`仍然不为零，所以继续将`rec-Nat`表达式替换成调用`step`所得到的值：

```pie
(add1
  (add1
    (rec-Nat 0
      1
      (λ (n-1 n-1+m)
        (add1 n-1+m)))))
```

现在最里层的`rec-Nat`的`target`等于`0`，所以它的值就等于`base`的值`1`:

```pie
(add1
  (add1
    1))
```

这样就得到了最后的结果`3`。

#### 柯里化 Currying

虽然前面介绍函数时说函数是可以接受多个参数的（`(lambda (x1 x2 ...) e1 e2 ...)`），但其实本质上 Pie 的函数只接受一个参数。用户之所以可以定义出接受多个参数的函数，是因为 Pie 的解释器会对函数作一个叫做 [currying](https://en.wikipedia.org/wiki/Currying) 的处理。比如这四个函数：

```pie
(claim currying-test1
  (-> Atom Atom
    Atom))
(define currying-test1
  (λ (a b)
    b))

(claim currying-test2
  (-> Atom Atom
    Atom))
(define currying-test2
  (λ (a)
    (λ (b)
      b)))

(claim currying-test3
  (-> Atom
    (-> Atom
      Atom)))
(define currying-test3
  (λ (a b)
    b))

(claim currying-test4
  (-> Atom
    (-> Atom
      Atom)))
(define currying-test4
  (λ (a)
    (λ (b)
      b)))
```

这几个函数无论是被声明成“接受两个`Atom`参数，返回值类型为`Atom`”（currying-test1、currying-test2 的情况），还是“接受一个`Atom`参数，返回一个类型为`(-> Atom Atom)`的函数”（currying-test3、currying-test4 的情况），也无论声明类型的形式和函数定义的形式是否一致（1、4 一致，2、3 不一致），在 Pie 的解释器看来都是完全相同的：

```pie
> currying-test1
(the (→ Atom Atom
       Atom)
  (λ (a b)
    b))

> currying-test2
(the (→ Atom Atom
       Atom)
  (λ (a b)
    b))

> currying-test3
(the (→ Atom Atom
       Atom)
  (λ (a b)
    b))

> currying-test4
(the (→ Atom Atom
       Atom)
  (λ (a b)
    b))
```

知道了这一点我们就可以运用所谓的 [partial application](https://en.wikipedia.org/wiki/Partial_application) 来从已有的函数生成出其他“固定”了某些参数的值的新函数。比如可以从前述的加法函数`+`得到把参数加一的新函数：

```pie
(claim plus-one
  (-> Nat
    Nat))
(define plus-one
  (+ 1))

> (plus-one 2)
(the Nat 3)
```

用了比较长的篇幅来介绍 Pie 的几个基础类型，接下来要说的就是 Pie 语言里的几种依赖类型。

### 依赖于类型的类型：List

如果熟悉 Java 的 generics 或者 ML 的 polymorphism 的话，应该会很容易理解 Pie 的 [List](https://docs.racket-lang.org/pie/index.html#%28part._.Lists%29) 类型。在 Pie 中，若`E`是一个类型则`(List E)`是一个 List 类型，代表一类元素类型都是`E`的 List。`(List Nat)`、`(List (-> Nat Atom))`和`(List (List Atom))`都是 List 类型，但是`(List 1)`和`(List '你好)`不是合法的类型。

List 有两个 constructor：`nil`和`::` [^4]。`nil`构造一个空 List；`::`接受两个参数`e`、`es`，如果`e`和`es`的类型分别为`E`和`(List E)`，则`(:: e es)`构造的是类型为`(List E)`比`es`多一个元素`e`的 List。List 类型可以用归纳法描述如下：

```code
(List E) ::= nil
(List E) ::= (:: E (List E))
```

可以看出来 List 的 constructor 和`Nat`的很相似：`nil`对应于`zero`，`::`对应于`add1`。下面是构造一个`(List Atom)`的示例：

```pie
(claim philosophers
  (List Atom))
(define philosophers
  (:: 'Descartes
    (:: 'Hume
      (:: 'Kant nil))))
```



[^1]: 类型可以出现在普通的表达式中，比如可以把类型作为参数传递给函数，函数也可以把类型像值一样返回。
[^2]: 比如著名的[四色定理](https://en.wikipedia.org/wiki/Four_color_theorem)的证明就是在 1976 年由计算机的定理证明程序来辅助推导得出的。
[^3]: 在 DrRacket 中可以通过快捷键 ⌘-\ 输入字母 λ。
[^4]: `::` 读作 cons（/ˈkɑnz/），继承自 Lisp 语言。