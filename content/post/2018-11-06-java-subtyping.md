---
title: "关于 Java 的子类型"
date: 2018-11-06T01:14:13+08:00
draft: true
author: "Mu Xian Ming"

autoCollapseToc: true
categories: [P]
tags: [Java]
---

类型可以看作是满足某一条件的一类值的集合的名称。编程语言中的静态类型系统是为了（静态地）防止值与集合的错误匹配。对于所谓的面向对象语言来说，类型系统最主要的任务是避免将消息发送给没法理解它的对象。更直白的说就是，避免找不到 field 或 method 的问题。

- Letting an expression that has one type also have another type that has less information is the idea of subtyping
- 