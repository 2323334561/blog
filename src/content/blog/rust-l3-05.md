---
title: 引用
pubDatetime: 2024-09-30T10:30:00+08:00
author: shushunie
slug: rust-l3-5
description: ""
---

# 引用与借用

rust 中我们将创建一个引用的行为称为 **借用**（_borrowing_）。正如现实生活中，如果一个人拥有某样东西，你可以从他那里借来。当你使用完毕，必须还回去，我们并不拥有它

### 语法

正如变量默认是不可变的，引用也一样。（默认）不允许修改引用的值 `let a = &s;`

添加 mut 关键字可以创建可变引用 `let b = &mut s;`

如果你有一个对该变量的可变引用，你就不能再创建对该变量的引用。这一限制以一种非常小心谨慎的方式允许可变性，防止同一时间对同一数据存在多个可变引用。新 Rustacean 们经常难以适应这一点，因为大部分语言中变量任何时候都是可变的。这个限制的好处是 Rust 可以在编译时就避免数据竞争。**数据竞争**（_data race_）类似于竞态条件，它可由这三个行为造成：

- 两个或更多指针同时访问同一数据。

- 至少有一个指针被用来写入数据。

- 没有同步数据访问的机制。

数据竞争会导致未定义行为，难以在运行时追踪，并且难以诊断和修复；Rust 避免了这种情况的发生，因为它甚至不会编译存在数据竞争的代码！

### 悬垂指针

在具有指针的语言中，很容易通过释放内存时保留指向它的指针而错误地生成一个 悬垂指针（dangling pointer），所谓悬垂指针是其指向的内存可能已经被分配给其它持有者。相比之下，在 Rust 中编译器确保引用永远也不会变成悬垂状态：当你拥有一些数据的引用，编译器确保数据不会在其引用之前离开作用域