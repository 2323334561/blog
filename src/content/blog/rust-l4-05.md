---
title: Macro 宏编程
pubDatetime: 2024-10-13T11:42:00+08:00
author: shushunie
slug: rust-l4-05
description: ""
---

在刚接触 Rust 时，我们就已经使用过宏。`println!` 就是一个最常用的宏，除此之外还有 `vec!` 、`assert_eq!` 都是相当常用的，可以说宏在 Rust 中无处不在

从功能上来讲，宏类似于动态语言中的装饰器，都是通过通过声明式的方式来修改或增强现有的代码。从根本上来说，宏在编译时操作代码，而装饰器在运行时修改函数或类

相对函数来说，由于宏是基于代码再展开成代码，因此实现相比函数来说会更加复杂，再加上宏的语法更为复杂，最终导致定义宏的代码相当地难读，也难以理解和维护

宏相较于函数在调用时多了一个 `!`。除此之外 `println!` 后面跟着的是 `()`，而 `vec!` 后面跟着的是 `[]`，这是因为宏的参数可以使用 `()`、`[]` 以及 `{}`。虽然三种使用形式皆可，但是 Rust 内置的宏都有自己约定俗成的使用方式，例如 `vec![...]`、`assert_eq!(...)` 等

```rust
fn main() {
    println!("aaaa");
    println!["aaaa"];
    println!{"aaaa"}
}
```

在 Rust 中宏分为两大类：

- 声明式宏( *declarative macros* ) `macro_rules!`

- 三种过程宏( *procedural macros* )

  - 派生宏，可以为目标结构体或枚举派生指定的代码，例如 `Debug` 特征

  - 类属性宏(Attribute-like macro)，用于为目标添加自定义的属性

  - 类函数宏(Function-like macro)，看上去就像是函数调用

## **声明式宏** macro_rules!

声明式宏允许我们写出类似 `match` 的代码。与 `match` 不同的是，**宏里的值是一段 Rust 源代码**(字面量)，模式用于跟这段源代码的结构相比较，一旦匹配，传入宏的那段源代码将被模式关联的代码所替换，最终实现宏展开

对于 `macro_rules` 来说，它是存在一些问题的，因此，Rust 计划在未来使用新的声明式宏来替换它：工作方式类似，但是解决了目前存在的一些问题，在那之后，`macro_rules` 将变为 `deprecated` 状态

## 过程宏

从形式上来看，过程宏跟函数较为相像，但过程宏是使用源代码作为输入参数，基于代码进行一系列操作后，再输出一段全新的代码

#### 派生宏

假设我们有一个特征 `HelloMacro`，现在有两种方式让用户使用它

- 使用 `Trait` 语法为每个类型手动实现该特征

- 使用过程宏来统一实现该特征，用户只需要对类型进行标记：`#[derive(HelloMacro)]`

两种方式并没有孰优孰劣，主要在于不同的类型是否可以使用同样的默认特征实现，如果可以，那过程宏的方式可以帮我们减少很多代码实现

```rust
use hello_macro_derive::HelloMacro;

#[derive(HelloMacro)]
struct Sunface;

fn main() {
    Sunface::hello_macro();
}
```

#### 类属性宏

类属性过程宏跟 `derive` 宏类似，但是前者允许我们定义自己的属性。除此之外，`derive` 只能用于结构体和枚举，而类属性宏可以用于其它类型项，例如函数

假设我们在开发一个 `web` 框架，当用户通过 `HTTP GET` 请求访问 `/` 根路径时，使用 `index` 函数为其提供服务

```rust
#[route(GET, "/")]
fn index() {
```

与 `derive` 宏不同，类属性宏的定义函数有两个参数：

- 第一个参数时用于说明属性包含的内容：`Get, "/"` 部分

- 第二个是属性所标注的类型项，在这里是 `fn index() {...}`，注意，函数体也被包含其中

除此之外，类属性宏跟 `derive` 宏的工作方式并无区别

#### 类函数宏

类函数宏可以让我们定义像函数那样调用的宏，从这个角度来看，它跟声明宏 `macro_rules` 较为类似。区别在于，`macro_rules` 的定义形式与 `match` 匹配非常相像，而类函数宏的定义形式则类似于之前讲过的两种过程宏

类函数宏使用方式则类似于函数调用，比如最常见的 `println!`

```rust
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
  ...
}

let sql = sql!(SELECT * FROM posts WHERE id=1);
```
