---
title: 引用与借用
pubDatetime: 2024-09-30T10:30:00+08:00
author: shushunie
slug: rust-l3-05
description: ""
---

## 引用与借用

rust 中我们将获取变量的引用，称之为借用(borrowing)。正如现实生活中，如果一个人拥有某样东西，你可以从他那里借来。当你使用完毕，必须还回去，我们并不拥有它

#### 不可变引用

使用 `&` 符号可以创建不可变引用

下面的代码，我们用 `s1` 的引用作为参数传递给 `calculate_length` 函数，而不是把 `s1` 的所有权转移给该函数。因为并不拥有这个值，当引用离开作用域后，其指向的值也不会被丢弃

```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

#### 可变引用

使用 `&mut` 可以创建可变引用

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

#### 引用规则

- 可变引用同时只能存在一个

- 可变引用与不可变引用不能同时存在

引用规则背后的目的是防止**数据竞争**（data races）。数据竞争是一种并发错误，可能发生在多个线程同时访问同一块内存，并且至少有一个线程正在修改数据的情况下。数据竞争会导致未定义行为，这种行为很可能超出我们的预期，而上述规则可以让Rust 在编译期就避免数据竞争
