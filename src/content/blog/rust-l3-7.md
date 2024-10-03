---
title: 结构体
pubDatetime: 2024-10-3T10:40:00+08:00
author: shushunie
slug: rust-l3-7
description: ""
---

# 定义和实例化

定义结构体，需要使用 `struct` 关键字并为整个结构体提供一个名字

创建一个实例需要以结构体的名字开头，接着在大括号中使用 `key: value` 键 - 值对的形式提供字段，其中 key 是字段的名字，value 是需要存储在字段中的数据值

从结构体的实例中获取某个特定的值，可以使用点号。如果结构体的实例是可变的，我们可以使用点号并为对应的字段赋值

```rust
struct User {
    name: String,
    active: bool,
    email: String,
}

let mut user1 = User {
    name: "user1".to_string(),
    active: true,
    email: "124@163.com".to_string(),
};

user1.name = "user_1".to_string();
```

### 字段初始化简写语法

如果参数名与字段名都完全相同，可以使用 **字段初始化简写语法**（_field init shorthand_）

```rust
fn create_user2(active: bool, email: String) -> User {
    User {
        name: "user2".to_string(),
        active,
        email,
    }
}
```

### 结构体更新语法

使用旧实例的大部分值但改变其部分值来创建一个新的结构体实例，可以使用 **结构体更新语法**（_struct update syntax_）实现（类似 js 的拓展运算符）

注意：如果更新的字段中包括没有实现 `Copy trait` 的类型（如 `String`、`Vec<T>` 等），会发生所有权转移。这会导致旧结构体失效

```rust
User {
    name: "user2".to_string(),
    ..user2
}
```

### 元组结构体

当你想给整个元组取一个名字，并使元组成为与其他元组不同的类型时，元组结构体是很有用的。要定义元组结构体，以 `struct` 关键字和结构体名开头并后跟元组中的类型

```rust
struct A(i32, String, i32, f64);
```

### 类单元结构体

没有任何字段的结构体称为 **类单元结构体**（_unit-like structs_），因为它们类似于 `()`，即 一节中提到的 unit 类型。类单元结构体常常在你想要在某个类型上实现 trait 但不需要在类型中存储数据的时候发挥作用

```rust
struct B;
```

# 方法语法

为结构体定义方法，需要使用`impl` 块（`impl` 是 *implementation* 的缩写），并将方法的声明放入块中。每个结构体都允许拥有多个 `impl` 块

方法的第一个参数必须有一个名为 `self` 的`Self` 类型的参数（关键字 `Self` 代指在 `impl` 关键字后出现的类型），即 `self: &Self`，可以使用 `&self` 进行缩写

与字段同名的方法将被定义为只返回字段中的值，这样的方法被称为 *getters*

```rust
impl User {
    fn get_name(self: &Self) -> String {
        format!("{}user", self.name)
    }

    fn active(&self) -> bool {
        self.active
    }
}
```

### 关联函数

我们可以定义不以 `self` 为第一参数的函数，称为 **关联函数**（_associated functions_）

使用结构体名和 `::` 语法来调用关联函数。我们已经使用了一个这样的函数：在 `String` 类型上定义的 `String::from` 函数

不是方法的关联函数经常被用作返回一个结构体新实例的构造函数，这些函数的名称通常为 `new` （new 不是关键字）

```rust
impl User {
    fn new(option: User) -> User {
        User {
            name: "user".to_string(),
            active: true,
            email: "1234@163.com".to_string(),
            ..option
        }
    }
}
```
