---
title: 流程控制
pubDatetime: 2024-09-30T10:30:00+08:00
author: shushunie
slug: rust-l3-02
description: ""
---

# 条件

使用 if 进行条件控制，当不满足 if 条件时，执行 else，如果有多个条件可以使用 else if。if 代码块中的语法和函数类似，最后一个表达式如果不带分号可以作为该块的返回值

需要注意的是 rust 的 if 条件不会进行自动类型转换，即条件表达式的结果必须是布尔类型

```rust
let a = 1;
let b = 2;

if a == b {
    println!("a == b")
} else {
    println!("a != b")
}
```

rust 可以直接将 if 表达式的结果绑定到变量上。一般在其他语言中，这样操作需要单独编写一个函数，在函数体中使用 if 判断，根据不同条件返回不同值，最后在定义变量时调用这个函数

```rust
let _c = if a > b { a } else { b };
```

# 循环

### loop

loop 循环没有条件语句，是一个无限循环，需要在循环体中手动使用 continue 和 break 进行控制（相当于 while(true)）。这里 break 关键字还有类似 return 的效果，跟在 break 后的表达式作为 loop 的返回值

```rust
let i = 1;

loop {
    if i > 3 {
        break;
    }
}

println!("loop end")
```

多个 loop 循环嵌套时，每层 loop 中的 break 只会跳出当前循环。如果需要退出外层循环，需要先给循环添加 label 标签，在 break 语句后接上循环标签即可退出相应循环

```rust
'loop2: loop {
    let mut i = 1;

    loop {
        if i > 3 {
            break 'loop2;
        }

        i += 1;
    }
}
```

### while

和其他语言类似，在判定一些简单条件时，比 loop 更简洁

```rust
let mut i = 1;

while i < 3 {
    i += 1;
}

println!("while end")
```

### for

类似 js 的 for in 的语法简化版，功能基本一致。适用于遍历集合。rust 可以快捷创建一个 range，语法如下

```rust
for i in 1..4 {
    println!("{}", i);
}

// 包含最后一个元素
for i in 1..=4 {
    println!("{}", i);
}
```

使用 for in 的另一个好处是可以在集合或 range 后进行链式调用，比如反向遍历、获取索引值

```rust
// 反向遍历
for i in (1..=4).rev() {
    println!("{}", i)
}

// 索引值
for (value, index) in (1..=4).enumerate() {
    println!("value: {}, index: {}", value, index);
}
```
