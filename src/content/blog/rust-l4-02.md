---
title: 泛型
pubDatetime: 2024-10-10T15:23:00+08:00
author: shushunie
slug: rust-l4-02
description: ""
---

## 特征 Trait

Rust 中并没有类的设计，比较接近的是之前介绍过的 `Struct`，搭配 `impl` 可以为 `Struct` 添加一些功能，使用起来就有点像其他编程语言里的类（class）

假设我写了一个 `Cat` 类和一个 `Dog` 类，不同的类通常会有各自的行为，但如果它们有共同的行为的话，在面向对象的世界观里，我们通常会另外创建一个 `Animal` 类，把共同的行为写在里面，然后让 `Cat` 和 `Dog` 类去继承它

面向对象中的继承虽然用起来很方便，但也有它的问题，例如：假设有一个 `Bird` 类，它有飞行功能。如果我想让 `Cat` 也会飞，简单的做法就是让 `Cat` 继承 `Bird` 类。但是，仅仅为了能够飞就必须成为 `Bird` 的后代吗？为了避免这种继承的问题，有些编程语言提供了模块（Module）或接口（Interface）的设计。例如，我们可以把飞行的功能写在飞行模块里，`Cat` 只要挂上这个飞行模块就能飞，而且它依然是 `Cat`，变成一只飞天猫。如果 `Dog` 或 `Fish` 也想要会飞，它们也可以自己拿去用，大家都能开心地做自己，而不需要成为 `Bird` 的后代

Rust 中提供类似接口（interface）或抽象类（abstract class）的概念，即 `Trait`。它允许我们在 Rust 中定义一些行为，然后将这些行为应用到不同的类型上。具体用法如下

```rust
struct Cat {
    name: String,
    age: u8
}

trait Flyable {
    // 默认方法
    fn fly(&self) {
      println!("起飞 🛫");
    };

    // 抽象方法
    fn speed(&self) -> f32;
}

impl Flyable for Cat {
    fn speed(&self) -> f32 {
        15.0
    }
}

let kitty = Cat { name: String::from("Kitty"), age: 18 };
kitty.fly()
```

## 泛型 Generics

我们在编程中，经常有这样的需求：用同一功能的函数处理不同类型的数据，例如两个数的加法，无论是整数还是浮点数，甚至是自定义类型，都能进行支持。在不支持泛型的编程语言中，通常需要为每一种类型编写一个函数

```rust
fn add_i32(a:i32, b:i32) -> i32 {
    a + b
}
fn add_f64(a:f64, b:f64) -> f64 {
    a + b
}

fn main() {
    println!("add i32: {}", add_i32(20, 30));
    println!("add f64: {}", add_f64(1.23, 1.23));
}
```

上述的加法函数代码可以用泛型改造，效果如下

```rust
fn add_number<T>(a: T, b: T) -> T {
    a + b
}
```

## 多态 **Polymorphism**

多态是面向对象编程中的一个概念，指的是同一个接口可以对应不同的类型或行为。多态主要有两种形式，这两种形式 rust 都支持

- **静态多态**（编译时多态，通常通过泛型添加 `Trait` 限制实现）

- **动态多态**（运行时多态，通常通过特征对象 `dyn Trait` 语法实现）

#### 静态多态

前文中提到的泛型加法函数在编译的时候会出现错误信息：`cannot add `T`to`T``。这是由于泛型 T 可以是任意类别，如果使用无法相加的类型这个函数将无法成立。为此我们需要给泛型 T 添加限制

在 Rust 中，类型是否支持加法计算是标准库中的一个 `Trait`。我们可以为泛型添加 `Add trait` 限制，来实现加法函数

```rust
use std::ops::Add;

fn add_number<T: Add<Output = T>>(a: T, b: T) -> T {
    a + b
}
```

泛型的参数在编译时就被替换为具体的类型，没有运行时开销，执行效率好，因此称为静态多态。但编译器需要为每个具体的类型生成对应的代码，会导致编译后的代码量增大

#### 动态多态

使用`dyn Trait` 也可以实现多态，和泛型产生的多态有一定区别。下面以前文中的 `Flyable` 特征来举例，代码表示 `someone` 的类型不确定，但必须实现 `Flyable` 特征

```rust
// 使用泛型
fn bungee<T: Flyable>(someone: &T) {
    someone.fly()
}

// 使用 dyn Trait
fn bungee(someone: &dyn Flyable) {
    someone.fly();
}
```

使用 `dyn` 实现多态，运行时每次调用 `Trait` 中声明的方法，Rust 会通过查找相应的函数指针并执行，被称为动态多态
