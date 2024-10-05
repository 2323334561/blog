---
title: 枚举和模式匹配
pubDatetime: 2024-10-5T16:12:00+08:00
author: shushunie
slug: rust-l3-10
description: ""
---

## 使用 Vector 储存列表

vector 允许我们在一个单独的数据结构中储存多于一个的值，它在内存中彼此相邻地排列所有的值，它的类型是 `Vec<T>`

#### 新建 vector

创建一个新的空 vector，可以调用 `Vec::new` 函数

Rust 也提供了 `vec!` 宏，这个宏会根据我们提供的值来创建一个新的 vector

```rust
let vector1 = Vec::new();
let vector2 = vec![1, 2, 3];
```

#### 更新 vector

对于新建一个 vector 并向其增加元素，可以使用 `push` 方法

```rust
let vector1 = vec![1, 2, 3];
vector1.push(4);
```

#### 读取 vector 元素

有两种方法引用 vector 中储存的值：通过索引或使用 `get` 方法。使用 `&` 和 `[]` 会得到一个索引位置元素的引用。使用索引作为参数调用 `get` 方法时，会得到一个可以用于 `match` 的 `Option<&T>`

Rust 提供了两种引用元素的方法的原因是当尝试使用现有元素范围之外的索引值时可以选择让程序如何运行。例如，索引可能来源于用户输入的数字。如果它们不慎输入了一个过大的数字那么程序就会得到 `None` 值，你可以告诉用户当前 vector 元素的数量并再请求它们输入一个有效的值

```rust
let a = vector1[0];

let b = vector1.get(1);
match b {
    Some(value) => {
        println!("a = {}", value);
    }
    None => {
        println!("None");
    }
}
```

如果想要依次访问 vector 中的每一个元素，可以使用`for` 循环

```rust
for i in vector2 {
    println!("{i}")
}
```

#### 结合枚举使用

vector 只能储存相同类型的值，这是很不方便的。而枚举的成员都被定义为相同的枚举类型，所以当需要在 vector 中储存不同类型值时，我们可以定义并使用一个枚举

假如我们想要从电子表格的一行中获取值，而这一行的有些列包含数字，有些包含浮点值，还有些是字符串。我们可以定义一个枚举，其成员会存放这些不同类型的值，同时所有这些枚举成员都会被当作相同类型：那个枚举的类型。接着可以创建一个储存枚举值的 vector，这样最终就能够储存不同类型的值了

```rust
enum table_content {
    int(i32),
    float(f64),
    text(String),
}

let table1 = vec![
    table_content::float(1.1),
    table_content::text("text".to_string()),
];
```

## 使用 Hash Map 储存键值对

`HashMap<K, V>` 类型储存了一个键类型 `K` 对应一个值类型 `V` 的映射。它通过一个 **哈希函数**（_hashing function_）来实现映射，决定如何将键和值放入内存中。很多编程语言支持这种数据结构，不过通常有不同的名字：哈希、map、对象、哈希表或者关联数组

像 vector 一样，哈希 map 将它们的数据储存在堆上，这个 `HashMap` 的键类型是 `String` 而值类型是 `i32`。类似于 vector，哈希 map 是同质的：所有的键必须是相同类型，值也必须都是相同类型

#### 新建一个哈希 map

可以使用 `new` 创建一个空的 `HashMap`，并使用 `insert` 增加元素

```rust
use std::collections::HashMap;

let mut hash = HashMap::new();

hash.insert("blue".to_string(), 10);
hash.insert("yellow".to_string(), 20);
```

#### 访问哈希 map 中的值

可以通过 `get` 方法并提供对应的键来从哈希 map 中获取值

`get` 方法返回 `Option<&V>`，调用 `copied` 方法可以将其类型转为`Option<&i32>`，接着调用 `unwrap_or` 在值为 `None` 时设置默认值

```rust
let team_name = "blue".to_string();
let blue_score = hash.get(&team_name).copied().unwrap_or(0);
```

可以使用与 vector 类似的方式来遍历哈希 map 中的每一个键值对，也就是 `for` 循环

遍历哈希 map 会以任意顺序进行

```rust
for (key, value) in hash {
    println!("{key}: {value}");
}
```

#### 哈希 map 和所有权

对于像 `i32` 这样的实现了 `Copy` trait 的类型，其值可以拷贝进哈希 map。对于像 `String` 这样拥有所有权的值，其值将被移动而哈希 map 会成为这些值的所有者

```rust
let mut hash2 = HashMap::new();
let mut test_string = "test".to_string();

hash2.insert("key".to_string(), test_string);
// test_string.push('a') // 所有权发生了转移，无法使用
```

#### 更新哈希 map

如果我们插入了一个键值对，接着用相同的键插入一个不同的值，与这个键相关联的旧值将被替换

想要实现只在键没有对应值时插入键值对，需要使用 `entry` API。它获取我们想要检查的键作为参数，返回一个枚举，代表了可能存在也可能不存在的值。`Entry` 的 `or_insert` 方法在键对应的值存在时就返回这个值的可变引用，如果不存在则将参数作为新值插入并返回新值的可变引用。`entry` 配合 `or_insert` 即可实现没有对应值时插入键值对

另一个常见的哈希 map 的应用场景是找到一个键对应的值并根据旧的值更新它。我们可以先使用 `or_insert` 获取到旧值。将这个可变引用储存在 `count` 变量中，所以为了赋值必须首先使用星号（`*`）解引用 `count`。这个可变引用在 `for` 循环的结尾离开作用域，这样所有这些改变都是安全的并符合借用规则

```rust
let mut hash2 = HashMap::new();
let text = "hello world wonderful world";

for word in text.split_whitespace() {
    let count = hash2.entry(word).or_insert(1);
    *count += 1;
}
```
