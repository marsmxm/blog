---
title: "Subtyping"
date: 2018-11-13T01:14:13+08:00
draft: true
author: "Mu Xian Ming"

categories: [P]
tags: [Typed Racket, Java, Subtyping]
---

## 概述

类型可以看作是满足某一条件的一类值的集合的名称。编程语言中的静态类型系统是为了（静态地）防止值与类型的错误匹配，从而在程序运行前就发现和类型相关的绝大部分错误。**子类型（Subtyping）**概念的引入是为了增加类型系统的表达力同时也增强了它的实用性。先通过 Typed Racket 来更好的理解一些通用的概念。下面是用`struct`定义的一个 point 类型`pt`，以及计算`pt`到原点距离的函数`dist-to-origin`：

```racket
(struct pt ([x : Real] [y : Real]))

(: dist-to-origin (-> pt Real))
(define (dist-to-origin pt)
  (sqrt (+ (sqr (pt-x pt))
           (sqr (pt-y pt)))))

(dist-to-origin (pt 3 4))
;; => 5
```

第三行`(: dist-to-origin (-> pt Real))`用来声明`dist-to-origin`的参数和返回值类型分别为`pt`和`Real`。如果现在有另一个 point 类型`color-pt`，除横纵坐标（x，y）外增加了一个颜色属性:

```racket
;; color-pt 继承自 pt，同时也增加了新属性 c
(struct color-pt pt ([c : String]))
```

一个很自然的想法是希望类型系统也允许`dist-to-origin`接受`color-pt`这个参数类型，毕竟计算到原点的距离只需要 x，y 两个属性就足够。这也正是子类型系统的核心思想：允许一个类型的值同时也属于另一个含有更少信息的类型，前者叫做后者的子类型。在上例中`pt`相较于`color-pt`含有更少的信息（没有颜色属性），所以`color-pt`的值同时也属于`pt`类型，`color-pt`是`pt`的一个子类型。引入了子类型这个概念也使得类型系统有了一个新的特性，可替代性：程序中接受某一类型值的位置可以替换成子类型的值而不会造成类型错误。所以 Typed Racket 允许`dist-to-origin`函数接受`color-pt`类型的参数：

```racket
(dist-to-origin (color-pt 3 4 "red"))
;; => 5
```

## Depth Subtyping

为了增强类型系统的表达力，Typed Racket，ML 等语言引入了多态（polymorphism），或者更准确的说参数多态（parametric polymorphism）的概念，所以就有了含有类型参数的类型（类似的概念在 Java 中被叫做 **generic**，而 polymorphism 则被 Java 的设计者用来指代 OOP 的一个核心特性，**dynamic dispatch**）。比如`(Listof t)`和`'a list`分别是 Typed Racket 和 ML 中的列表类型，`t`和`'a`是类型参数，用来表示列表中元素的类型。一个很自然的问题是，列表会因为所含元素的类型的父子关系而也有了相同的父子关系吗？比如在 Typed Racket 中，`(Listof color-pt)`是`(Listof pt)`的子类型吗？答案是，对于列表类型来说，这个父子关系是成立的，使得这个关系成立的规则就叫做 **Depth Subtyping**。用一个例子来说明：

```racket
(: sum-of-dist (-> (Listof pt) Real))
(define (sum-of-dist pt-list)
  (foldl (lambda ([p : pt] [accu : Real])
           (+ (dist-to-origin p) accu))
         0
         pt-list))

(sum-of-dist (list (pt 3 4) (pt 3 4)))
;; => 10

(sum-of-dist 
  (list (color-pt 3 4 "red")
        (color-pt 3 4 "green")))
;; => 10
```

用`sum-of-dist`来对列表中所有点到原点的距离求和。虽然声明的参数类型是`(Listof pt)`，它也接受了`(Listof color-pt)`这个类型的参数，进而说明`(Listof color-pt)`是`(Listof pt)`的子类型。对于列表来说，使得 depth subtyping 成立的一个关键特性是它的**不可更改性（immutability）**。