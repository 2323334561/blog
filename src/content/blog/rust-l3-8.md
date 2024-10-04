---
title: 枚举和模式匹配
pubDatetime: 2024-10-4T16:44:00+08:00
author: shushunie
slug: rust-l3-8
description: ""
---

## 枚举的定义

假设我们要处理 IP 地址。目前被广泛使用的两个主要 IP 标准：IPv4（version four）和 IPv6（version six）。这是我们的程序可能会遇到的所有可能的 IP 地址类型：所以可以 **枚举** 出所有可能的值，这也正是此枚举名字的由来

可以通过在代码中定义一个 `IpAddrKind` 枚举来表现这个概念并列出可能的 IP 地址类型，`V4` 和 `V6`。这被称为枚举的 **成员**（_variants_）。枚举成员要求使用大驼峰写法，否测编译器会发出警告

```rust
enum IpAddrKind {
    V4,
    V6,
}
```

#### 枚举值的使用

枚举的成员位于其标识符的命名空间中，并使用两个冒号分开。在 `IpAddrKind::V4` 和 `IpAddrKind::V6` 都是 `IpAddrKind` 类型的。例如，接着可以定义一个函数来获取任何 `IpAddrKind`

可以使用 `impl` 来为结构体定义方法那样在枚举上定义方法

```rust
enum IpAddrKind {
    V4,
    V6,
}

impl IpAddrKind {
    fn get_ip_kind(&self) -> &IpAddrKind {
        self
    }
}

struct IP {
    address: String,
    kind: IpAddrKind,
}

let _home_ip = IP {
    address: "1111".to_string(),
    kind: IpAddrKind::V4,
};
```

#### 枚举成员关联数据

上述代码是将枚举作为结构体的一部分，可以将数据直接放进每一个枚举成员来简化代码

```rust
enum IpAddrKind {
    V4(String),
    V6(String),
}

let _home_ip = IpAddrKind::V4("222222".to_string());
```

## match 控制流结构

`match` 是 Rust 中一个非常强大的控制流运算符，它的作用有点像开关开关机的按钮，或者我们在其他语言中常见的 `switch` 语句，但功能更强大

`match` 关键字后跟随的是模式列表，每一个模式后面跟着 `=>`，然后是与该模式匹配时应执行的代码。如果需要在匹配的同时执行额外的逻辑，比如打印出信息，你可以在匹配的代码块中使用大括号

`match` 分支必须覆盖了所有的可能性。如果无法覆盖所有情况，rust 编译器会报错

当 `match` 运行时，它会依次检查每一个模式是否匹配传入的值。一旦找到匹配的模式，执行该分支后就会结束，不会再检查后续的分支

```rust
enum Drink {
    Water,
    Coffee,
    Tea(String),
}

impl Drink {
    fn is_water(&self) -> bool {
        match self {
            Drink::Water => true,
            Drink::Coffee => false,
            Drink::Tea(_) => false,
        }
    }
}
```

#### 绑定值模式

如果枚举成员关联了数据，模式匹配可以时可以操作对应的数据

```rust
impl Drink {
    fn print_drink(&self) {
        match self {
            Drink::Water => println!("water"),
            Drink::Coffee => println!("coffee"),
            Drink::Tea(kind) => println!("{} tea", kind),
        }
    }
}
```

#### 通配模式和 \_ 占位符

和 `switch` 语句类似， `match` 语段中可以使用 `other` 来匹配剩余所有可能的值

如果你不需要使用匹配到 `other` 时的具体值，可以使用 `_` 来替代 `other`

如果不需要执行任何操作，可以使用空元组 `()` 作为 `_` 分支的代码

```rust
fn add_hat() {}
fn remove_hat() {}
fn move_player(num: i32) {}
fn do_nothing() {}

let dice_roll = 5;

match dice_roll {
    1 => add_hat(),
    2 => remove_hat(),
    other => move_player(other),
}

match dice_roll {
    1 => add_hat(),
    2 => remove_hat(),
    _ => do_nothing(),
}
```

#### option 枚举

在许多编程语言中，像 C++、Java、Python 和 JavaScript 等，都存在一种特殊的值，通常称为 `null`、`nil` 或 `undefined`，用来表示没有值。如果程序员不小心直接使用了这些 `null` 值，就会导致运行时错误。Rust 的 `Option` 则通过编译时检查的方式，强制程序员处理值的存在与否

`Option<T>` 枚举是如此有用以至于它甚至被包含在了 prelude 之中，你不需要将其显式引入作用域。另外，它的成员也是如此，可以不需要 `Option::` 前缀来直接使用 `Some` 和 `None`

```rust
let arr: [i32; 4] = [0, 1, 2, 3];

fn get_arr_member(array: &[i32], index: usize) -> Option<i32> {
    if index < array.len() {
        Some(array[index])
    } else {
        None
    }
}

// 报错
let num = 2 + get_arr_member(&arr, 2);
```

## if let 控制流

前文中提到 `match` 必须覆盖所有可能性。`if let` 语法可以让我们处理只匹配一个模式的值而忽略其他模式的情况，可以认为 `if let` 是 `match` 的一个语法糖

`if let` 可以结合 `else` 使用，这样使用效果和 `other` 类似

```rust
let some_value = Some(5);

if let Some(v) = some_value {
    println!("The value is: {}", v);
} else {
    println!("No value found.");
}
```
