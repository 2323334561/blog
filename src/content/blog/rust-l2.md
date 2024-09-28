---
title: 变量和可变类型
pubDatetime: 2024-09-28T23:30:00+08:00
author: shushunie
slug: rust-l2
description: ''
---

# 变量声明

### [let](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/37e9af67-addd-40de-916a-2f44cf260684) 关键字

Bind a value to a variable

将值绑定到变量

```rust
// let
fn immutable() {
    let a = 5;
    // a = 10 // cannot mutate immutable variable 'a'
    println!("不可变变量 {}", a);
}
```

### [const](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/e5ce725c-2ceb-4d70-b03d-d8616c607709) 关键字

Compile-time constants, compile-time evaluable functions, and raw pointers

编译时常量、编译时可计算函数和原始指针

在本章中 const 用于声明常量，关于常量需要注意以下几点


- 使用 const 关键字进行声明

- 名称需要使用大写字母，否则编译器会报 Warning

- 声明时必须指定数据类型

- 必须使用常量表达式进行赋值，即必须是编译期能计算出的值

- 不支持直接修改

- 不支持 mut 关键字

- 不支持重定义（遮蔽）

```rust
// 常量
fn constant() {
    // const number: i32 = 5; // Constant number' should have UPPER_SNAKE_CASE name
    const NUMBER: i32 = 5;
    println!("常量 {}", NUMBER);
}
```

### [static](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/691e3386-54db-4f0c-b677-75747f4214c2) 关键字

A static item is a value which is valid for the entire duration of your program (a `'static` lifetime)

静态项是在程序的整个持续时间（ `'static`生命周期”）内有效的值

在本章中 static 用于声明静态变量，静态变量有以下特性


- 使用 static 关键字进行声明

- 名称需要使用大写字母，否则编译器会报 Warning

- 声明时必须指定数据类型

- 必须使用常量表达式进行赋值，即必须是编译期能计算出的值

- 不支持直接修改

- 支持使用 mut 关键字后修改

- 常量不支持重定义（遮蔽）

```rust
// static
fn _static() {
    static NUMBER: i32 = 5;
    println!("静态变量 {}", NUMBER)
}
```

# 特性说明

### [mut](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/5cbfe9ca-ddc5-4701-835f-0fb09891d765) 关键字

A mutable variable, reference, or pointer.

可变变量、引用或指针

本章中 mut 关键字与 let 或 static 关键字组合使用，表示变量的值是可变的

```rust
// mut
fn mutable() {
    let mut a = 5;
    println!("可变变量 {}", a);
    a = 10;
    println!("可变变量 {}", a);
}
```

### shadowing [遮蔽](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/77d503fe-2141-4756-80d3-ac82e9e9f5bf)（重定义）

遮蔽指：使用关键字声明变量后，在后续代码中再次声明同一变量。仅使用 let 声明的变量支持这一操作，变量被遮蔽后的值基于最新声明

```rust
// 遮蔽
fn shadowing() {
    let a = 3;
    let a = 5;
    println!("变量遮蔽 {}", a)
}
```

### [内联](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/296de632-a9f9-43d2-8269-aa64ad52d19f)与[引用](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/04c875f8-130d-43ff-a34d-d31e8aa9ae2e)

常量在编译时被内联
```rust
const a: i32 = 1
println("原始代码 {}", a)
```
```rust
const a: i32 = 1
println("内联 {}", 1)
```
静态变量在编译时会指向引用
```rust
static a: i32 = 1
println("原始代码 {}", a)
```
```rust
static a: i32 = 1 // 内存地址 0x123
println("引用 {}", a) // 指向 ↑
```

### 特性对比

|            | let | const  | static  |
| :--------- | :-- | :----- | :------ |
| 名称要求大写     |     | ✔      | ✔       |
| 必须声明类型     |     | ✔      | ✔       |
| 必须使用常量赋值   |     | ✔      | ✔       |
| 支持直接修改     | ✔   |        |         |
| 支持 mut 关键字 | ✔   |        | ✔       |
| 支持重定义      | ✔   |        |         |
| 其他         |     | 编译时被内联 | 编译时指向引用 |