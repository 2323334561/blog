---
title: 字符串
pubDatetime: 2024-10-1T11:30:00+08:00
author: shushunie
slug: rust-l3-7
description: ""
---

### 类型

分为 `&str` 和 `String` 类型

&str 是 Rust 的核心语言提供的字符串类型，称为字符串字面量。它是对储存在别处的 UTF-8 编码字符串数据的引用，是不可变的。&str 在编译时会被硬编码进可执行文件中

String 类型由 Rust 标准库提供，而不是编入核心语言，它是一种可增长、可变、可拥有、UTF-8 编码的字符串类型

### 语法

```rust
// &str
let _str1 = "string";

// String
let _str2 = String::from("string");
```

### 类型转换

从 `String` 类型转变为 `&str` 是非常便捷的，而且无损的（性能无损，不会造成重写malloc或者数据移动）。由于Rust实现了自动解引用，String ，因此在很多函数中，如果接收参数是字符串的引用，通常会采用 &str 作为入参，以获取更好的数据兼容性

从 `&str` 转为 `String` 开销较大，因为需要为 &str 的值申请内存。具体可以使用 `String::from` 函数

### 字符串索引

`String` 是一个 `Vec<u8>` 的封装，使用的是 UTF-8 编码。在 UTF-8 编码中，每个英文单词字母占一个字节，每个中文字符占三个字节。所以在对 "hello 中国" 这一字符串进行索引时，"中"的索引为 7～9，在我们对字符串执行操作时，需要注意这一问题

### 常用方法

**修改原字符串**

- `push()` 方法追加字符 char

- `push_str()` 方法追加字符串字面量

- `insert()` 方法插入单个字符 char

- `insert_str()` 方法插入字符串字面量

- `pop()` 方法删除并返回最后一个字符

- `remove()` 删除指定位置字符

- `truncate()` 删除从指定位置开始到结尾的全部字符

- `clear()` 清空字符串

- `replace_range()` 替换索引范围内的字符

**返回新字符串**

- `replace()` 替换所有匹配到的字符串

- `replacen()` 替换匹配到的第 n 个字符串，由第三个参数控制

**字符串连接**

- `+` 或 `+=`

- `add()` 方法

- `format` 方法

**字符串转义**

- 使用 `\` 输出 ASCII 和 Unicode 字符

- 使用 `r#` 可以让字符串中的 `\` 或 `""` 视为普通字符
