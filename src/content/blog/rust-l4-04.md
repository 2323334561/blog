---
title: 特征 Trait
pubDatetime: 2024-10-13T11:42:00+08:00
author: shushunie
slug: rust-l4-04
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

#### 关联类型

关联类型通常与 trait 一起使用，通过 `type` 关键字在 trait 中定义，例如

```rust
trait Iterator {
    type Item; // 关联类型

    fn next(&mut self) -> Option<Self::Item>; // 使用关联类型
}

```

在这个例子中，`Iterator` trait 定义了一个关联类型 `Item`，表示迭代器生成的值的类型，`Self::Item` 表示实现者要提供的具体类型

关联类型的效果有点类似泛型，它们的主要区别在于使用场景不同。泛型适合用于希望同一个逻辑适用于多种类型时。关联类型则适合用于某个 trait 的实现对某一类型有固定的要求，例如，Rust 的 `Iterator` trait

## 常见特性

#### Deref 解引用

常规引用是一个指针类型，包含了目标数据存储的内存地址。对常规引用使用 `*` 操作符，就可以通过解引用的方式获取到内存地址对应的数据值

```rust
fn main() {
    let x = 5;
    let y = &x;

    assert_eq!(5, x);
    assert_eq!(5, y);
    assert_eq!(5, *y);
}
```

这里 `y` 就是一个常规引用，包含了值 `5` 所在的内存地址，然后通过解引用 `*y`，我们获取到了值 `5`。如果你试图执行 `assert_eq!(5, y);`，代码就会无情报错，因为你无法将一个引用与一个数值做比较

解引用应作用于地址，如果我们对一个结构体进行解引用理论上应该无法执行，但实际上并非如此。Rust 提供了 `Deref` 特征，实现了这个特征的结构体可以支持解引用

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```

当我们对智能指针 `Box` 进行解引用时，实际上 Rust 为我们调用了 `*(y.deref())`。通常情况下 `deref` 方法会返回一个引用（内存地址），配合解引用就可以获取到需要的值

看到这里你可能有点疑惑，函数传参时都是直接传引用值（内存地址），为什么可以直接使用内存地址进行操作呢？

对于函数和方法的传参，Rust 提供了一个极其有用的隐式转换：`Deref` 转换。如果一个类型实现了 `Deref` 特征，那它的引用在传给函数或方法时，会根据参数签名来决定是否进行隐式的 `Deref` 转换。另外`Deref` 可以支持连续的隐式转换，直到找到适合的形式为止

```rust
fn main() {
    let s = MyBox::new(String::from("hello world"));
    display(&s)
}

fn display(s: &str) {
    println!("{}",s);
}
```

`Deref` 并不是没有缺点。如果你不知道某个类型是否实现了 `Deref` 特征，那么在看到某段代码时，并不能在第一时间反应过来该代码发生了隐式的 `Deref` 转换。事实上，不仅仅是 `Deref`，在 Rust 中还有各种 `From/Into` 等等会给阅读代码带来一定负担的特征

#### Drop 释放资源

在 Rust 中，变量超出作用域后编译器将帮你自动插入内存释放代码，这个操作实际上由 `Drop` 特征进行控制

Rust 自动为几乎所有类型都实现了 `Drop` 特征，因此就算你不手动为结构体实现 `Drop`，它依然会调用默认实现的 `drop` 函数，同时再调用每个字段的 `drop` 方法

```rust
struct Foo;

impl Drop for Foo {
    fn drop(&mut self) {
        println!("Dropping Foo!")
    }
}

fn main() {
    let _foo = Foo;
    println!("Running!");
}
```

`Drop` 特征中的 `drop` 方法借用了目标的可变引用，而不是拿走了所有权。理论上来说，就算手动提前调用了 `drop`，由于没有发生所有权转移，根据所有权规则目标值依然是可以使用的。但 `drop` 方法本身的作用是释放内存，实际使用已经被 `drop` 的值会访问一块已经被释放的内存。为了避免这个问题 Rust 禁止手动调用 `drop` 方法
