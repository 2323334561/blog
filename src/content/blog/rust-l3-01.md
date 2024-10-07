---
title: 函数
pubDatetime: 2024-09-29T9:30:00+08:00
author: shushunie
slug: rust-l3-01
description: ""
---

## 语法

- 函数名和变量名使用[蛇形命名法(snake case)](https://course.rs/practice/naming.html)，例如 `fn add_two() -> {}`

- 函数的位置可以随便放，Rust 不关心我们在哪里定义了函数，只要有定义即可

- 每个函数参数都需要标注类型

### 返回值

返回值的类型需要在函数声明时，使用 `->` 进行标注。如果函数没有返回值，可以不写类型或者使用 `-> ()` 表示返回空。如果函数永远不会返回（抛出错误或死循环等情况），可以使用 `-> !` 表示

rust 函数的返回值就是函数体最后一条表达式的返回值，也可以使用 return 提前返回。想要完全理解这句话需要掌握\*语句（statement）和表达式（expression）\*\*的概念。对于初学者来说记住两种写法即可

1. return + 分号

2. 没有 return + 没有分号
