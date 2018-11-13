---
title: "Java 中的 Subtyping"
date: 2018-11-06T01:14:13+08:00
draft: true
author: "Mu Xian Ming"

categories: [P]
tags: [Java, Subtyping]
---

# 概述

类型可以看作是满足某一条件的一类值的集合的名称。编程语言中的静态类型系统是为了（静态地）防止值与集合的错误匹配，从而在程序运行前就发现和类型相关的绝大部分错误。子类型概念的引入是为了让类型系统更宽容也更实用。先通过 Typed Racket 来更好的理解一些通用的概念。下面是用`struct`定义的一个 point 类型`pt`，以及计算`pt`到原点距离的函数`dist-to-origin`：

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
(struct color-pt pt ([c : String])) ;; color-pt 继承自 pt，同时也增加了新属性 c
```

一个很自然的想法是希望类型系统也允许`dist-to-origin`接受`color-pt`这个参数类型，毕竟计算到原点的距离只需要 x，y 两个属性。而这也正是子类型系统的核心思想：允许一个类型的值同时也属于另一个含有更少信息的类型，前者叫做后者的子类型。在上例中`pt`相较于`color-pt`含有更少的信息（没有颜色属性），所以`color-pt`的值同时也属于`pt`类型，`color-pt`是`pt`的一个子类型。引入了子类型这个概念也使得类型系统有了一个新的特性，可替代性：程序中接受某一类型值的位置可以替换成子类型的值而不会造成类型错误。所以 Typed Racket 允许`dist-to-origin`函数接受`color-pt`类型的参数：

```racket
(dist-to-origin (color-pt 3 4 "red"))
;; => 5
```

