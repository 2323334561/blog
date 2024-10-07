---
title: 模块化
pubDatetime: 2024-10-7T13:12:00+08:00
author: shushunie
slug: rust-l3-11
description: ""
---

## Package 和 Crate

#### create

Crate 是 Rust 编译器在编译时最小的代码单位。无论是使用 `rustc` 还是 `cargo` 编译，Rust 都会将每个源文件视为一个 crate

Crate 可以有两种形式：

- **二进制 crate**：可以被编译为可执行程序，必须包含一个 `main` 函数。例如，命令行应用程序或 Web 服务器

- **库 crate**：不包含 `main` 函数，提供函数和类型供其他项目使用，例如 `rand` crate 用于生成随机数

#### package

`Package` 就是一个项目，因此它包含有独立的 `Cargo.toml` 文件，以及因为功能性被组织在一起的一个或多个包。一个 `Package` 只能包含**一个**库(library)类型的包，但是可以包含**多个**二进制可执行类型的包

#### 举例

📦 my-project\
 ┣ 📂 src\
 ┃ ┣ 🦀 main.rs\
 ┃ ┣ 🦀 lib.rs\
 ┃ ┗ 📂 bin\
 ┃ ┣ 🦀 main1.rs\
 ┃ ┗ 🦀 main2.rs\
 ┣ 📂 tests\
 ┃ ┗ 🧪 some_integration_tests.rs\
 ┣ 📂 benches\
 ┃ ┗ 🧪 simple_bench.rs\
 ┣ 📂 examples\
 ┃ ┗ 📜 simple_example.rs\
 ┣ ⚙️ Cargo.lock\
 ┗ 📜 Cargo.toml

- 唯一库包：`src/lib.rs`
- 默认二进制包：`src/main.rs`，编译后生成的可执行文件与 `Package` 同名
- 其余二进制包：`src/bin/main1.rs` 和 `src/bin/main2.rs`，它们会分别生成一个文件同名的二进制可执行文件
- 集成测试文件：`tests` 目录下
- 基准性能测试 `benchmark` 文件：`benches` 目录下
- 项目示例：`examples` 目录下

## 模块 Module

#### 创建模块

在 Rust 中，每个 `.rs` 文件都可以看作一个模块。文件内部也可以通过 `mod` 关键字来定义子模块，并且模块之间可以嵌套

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }
}

```

#### 引用模块

可以使用相对路径或绝对路径来引用模块

- **绝对路径**，当前 creat 的 src 目录作为根路径，路径名以包名或者 `crate` 作为开头
- **相对路径**，当前模块作为根路径，路径名以 `self`，`super` 或模块的名称作为开头
  - `self`，表示当前模块路径，可以直接省略
  - `super`，表示父模块路径，类似文件系统中的 `..`
  - 模块的名称，表示匹配的模块的路径

Rust 出于安全的考虑，默认情况下，所有的类型都是私有化的，包括函数、方法、结构体、枚举、常量。父模块完全无法访问子模块中的私有项，但是子模块却可以访问父模块。因此，在进行链式调用时，需要为模块中的内容项添加 `pub` 关键字

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // 绝对路径
    crate::front_of_house::hosting::add_to_waitlist();

    // 相对路径
    front_of_house::hosting::add_to_waitlist();
}
```

#### 限制可见性

`pub(in crate::a)` 的写法可以限制模块的可见性

- `pub` 意味着可见性无任何限制
- `pub(crate)` 表示在当前包可见
- `pub(self)` 在当前模块可见
- `pub(super)` 在父模块可见
- `pub(in <path>)` 表示在某个路径代表的模块中可见，其中 `path` 必须是父模块或者祖先模块

## 使用 use

如果代码中，通篇都是 `crate::front_of_house::hosting::add_to_waitlist` 这样的函数调用形式，代码重复度太高，可读性很差。在 Rust 中，可以使用 `use` 关键字把路径提前引入到当前作用域中，随后的调用就可以省略该路径，极大地简化了代码

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use front_of_house::hosting::add_to_waitlist;

pub fn eat_at_restaurant() {
    add_to_waitlist();
    add_to_waitlist();
    add_to_waitlist();
}
```

#### use 使用技巧

`use` 引用的路径可以使用绝对路径或相对路径

当出现同名模块时，可以使用 `as` 关键字重命名

当外部的模块项 `A` 被引入到当前模块中时，它的可见性自动被设置为私有的。如果想从当前模块直接引入模块 `A` ，可以使用 `pub` 关键字导出引入的模块 `A`

在 `Cargo.toml` 文件 `[dependencies]` 区域声明的第三方包，比如 `rand = "0.8.3"` 可以直接引入

项目中，一个大型模块通常包含多个字模块项，可以使用 `{}` 来一起引入进来，减少 `use` 语句代码重复

使用 \* 可以直接引入模块下的所有公共项

```rust
use crate::front_of_house::hosting;
use front_of_house::hosting::add_to_waitlist;

use std::fmt::Result;
use std::io::Result as IoResult;

pub use crate::front_of_house::hosting;

use rand::Rng;

use std::collections::{HashMap,BTreeMap,HashSet};
use std::{cmp::Ordering, io};

use std::collections::*;
```
