---
title: test 测试
pubDatetime: 2024-10-15T11:39:00+08:00
author: shushunie
slug: rust-l4-07
description: ""
---

## 常用宏

- `#[cfg(test)]` 只有在运行 `cargo test` 时，带有 `#[cfg(test)]` 标记的代码才会被编译和执行

- `#[test]` 标记测试为函数，运行 `cargo test` 时自动执行它们

- `#[should_panic(expected = "xxx")]` 测试函数的 `panic` 信息

- `#[ignore]` 忽略当前测试函数

## 单元测试

单元测试目标是测试某一个代码单元(一般都是函数)，验证该单元是否能按照预期进行工作，例如测试一个 `add` 函数，验证当给予两个输入时，最终返回的和是否符合预期。Rust 单元测试的惯例是将测试代码的模块跟待测试的正常代码放入同一个文件中

```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        assert_eq!(add_two(2), 4);
    }
}
```

## 集成测试

集成测试的代码在一个单独的目录下，它们使用跟其它模块一样的方式去调用你想要测试的代码，只能调用通过 `pub` 定义的 `API`

一个标准的 Rust 项目，在它的根目录下会有一个 `tests` 目录用来存放集成测试。该录一般来说需要手动创建，该目录在项目的根目录下，跟 `src` 目录同级。在 `tests` 目录下文件不需要使用 `#[cfg(test)]` 宏来表明它是测试代码

```rust
use adder;

#[test]
fn it_adds_two() {
    assert_eq!(4, adder::add_two(2));
}
```

#### 共享测试模块

在集成测试代码中，我们可能有需要复用的模块。如果我们创建一个 `tests/common.rs` 文件用于存放公共测试模块，这个模块会被当做集成测试函数运行。正确的方式是使用 `tests/common/mod.rs` 文件，通过这种方式创建的文件不会被视作测试文件，执行`cargo test`时不会执行该文件

## 文档测试

在 Rust 中，`///` 开头的注释用于编写文档注释（documentation comments），这些注释是为生成文档而设计的。文档注释中的 Rust 代码在执行 cargo test 时将作为测试代码执行

````rust
/// `add_one` 将指定值加1
///
/// # Examples
///
/// ```
/// let arg = 5;
/// let answer = world_hello::compute::add_one(arg);
///
/// assert_eq!(6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
````

#### panic 文档测试

文档测试中的用例可以添加 should_panic 来实现类似 `#[should_panic]` 宏类似的效果

````rust
/// # Panics
///
/// The function panics if the second argument is zero.
///
/// ```rust,should_panic
/// // panics on division by zero
/// world_hello::compute::div(10, 0);
/// ```
````
