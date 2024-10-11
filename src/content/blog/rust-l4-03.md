---
title: 生命周期
pubDatetime: 2024-10-11T17:35:00+08:00
author: shushunie
slug: rust-l4-03
description: ""
---

## 生命周期检查

```rust
fn print_age() {
    let my_age;              //---------------+-- 'my_age
    {                        //               |
        let age = 12;        //------+ 'age   |
        my_age = &age;       //      |        |
    }                        //------+        |
    println!("{}", my_age);  //               |
}                            //---------------+
```

在这个例子中，借用检查器会发现 `my_age` 引用的 `age` 生命周期已经结束，而 `my_age` 还在试图使用这个引用，形成了一个悬垂引用。如果执行这段代码，编译器会提示 `age does not live long enough`

引用的生命周期不能比实际数据更长，否则会造成悬垂引用。为了避免悬垂引用问题，Rust 的**借用检查器（BC）**在编译阶段会检查所有引用的生命周期，并确保它们在使用时指向的内存是有效的

## 生命周期标注

#### 无法确定的生命周期

```rust
struct Cat {
    name: String,
    age: u8
}

fn main() {
    let kitty = Cat { name: "Kitty".to_string(), age: 12 };
    let nancy = Cat { name: "Nancy".to_string(), age: 16 };

    let boss = boss_cat(&kitty, &nancy);
    println!("{}", boss.name);
}

fn boss_cat(c1: &Cat, c2: &Cat) -> &Cat {
    if c1.age > c2.age {
        c1
    } else {
        c2
    }
}
```

`boss_cat` 函数接受两个 `Cat` 的引用，比较他们的年龄大小，返回年纪大的作为 boss。这段代码从其他编程语言的角度来看是没有问题的， 但 Rust 的 BC 会提示错误

```powershell
$ cargo run
error[E0106]: missing lifetime specifier
   |
14 | fn boss_cat(c1: &Cat, c2: &Cat) -> &Cat {
   |                 ----      ----     ^ expected named lifetime parameter
   |
   = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `c1` or `c2`
```

错误信息表明：这个函数的返回类型包含一个借来的值，但是签名没有说明它是从’ c1 ’还是’ c2 ’借来的。对于 `boss` 这个变量来说，他的值是 `boss_cat` 函数的调用结果，可能是 `kitty` 或 `nancy` 的引用。它的生命周期存在两种可能，不够明确，在 Rust 眼中这不符合规范

#### 函数签名中的生命周期标注

为了告诉 BC 明确的生命周期，我们需要添加「生命周期标注」。具体写法如下

```rust

```

`boss_cat<'a>` 后面的 `'a` 就是生命周期标注的写法。它使用「单引号」后面加上任何小写字母（当然你也可以用大写字母，但 Rust 会建议你不要这么做）。虽然生命周期标注的名称可以用任意字母组合，不过通常大家会用简单的 `'a` 来表示

函数签名中的生命周期标注和泛型的类似，可以作用于函数参数以及返回值。本例中 `'a` 标注标记了参数 `c1`、`c2` 以及返回值，它表示 `c1`、`c2` 以及返回值的声明周期都是相同的 `'a`（可以这么理解但不严谨），至于 'a 的具体声明周期有多长，取决于实际传入的参数

#### 结构体定义中的生命周期标注

如果你定义了一个没有生命周期标注的结构体，其中包含了引用，Rust 会抛出编译错误，要求你指定引用的生命周期

这是因为在结构体中如果成员不使用引用类型，则在创建实例的时候会发生所有权转移，这时实例和成员的生命周期可以保持一致。成员引用类型，结构体本身的生命周期和成员的生命周期是不一样的，Rust 编译器目前无法自动推断 `title` 的生命周期和实例的生命周期之间的关系

```rust
struct Book<'a> {
    // title: &str // 无法自动推断引用的生命周期
    title: &'a str
}

fn main() {
    let book_ref;
    {
        let book_title = String::from("Rust Programming");
        book_ref = Book { title: &book_title };
        // rustc: `book_title` does not live long enough borrowed value does not live long enough
    }
}

```

在结构体中使用生命周期标注，其形式也和泛型类似。以上方代码为例，`'a` 表示成员 `title` 的生命周期和创建结构体时为 `title` 赋值的变量相当。在创建结构体时，`book_title` 被要求和 `book_ref` 的生命周期一致，否则就会报错

#### 特殊标注

`'static` 是一个特殊的生命周期标注，表示这个生命周期会持续整个程序，直到程序结束为止。

全局变量和字符串字面量就使用了这个标记，只不过编译器会自动推断，所以可以省略。在某些特殊情况下需要用到手动 `'static`，比如异步或者跨线程的情况

#### 生命周期约束

如果在同一个函数中需要多个标注，通常会依次使用 `'b`、`'c` 这样按顺序使用

```rust
fn boss_cat<'a, 'b>(c1: &'a Cat, c2: &'b Cat) -> &'a Cat {
    if c1.age > c2.age {
        c1
    } else {
        c2
    }
}
```

这个函数使用了 `'a`、`'b` 两个标注，它表示返回值的生命周期和 `c1` 相关联。这样是无法明确返回值的生命周期的，因此这段代码无法正常运行。我们可以通过给 `'b` 添加约束： `'b: 'a` ，表示生命周期 `'a >='b`

```rust
fn boss_cat<'a, 'b: 'a>(c1: &'a Cat, c2: &'b Cat) -> &'a Cat {
    // ... 略 ...
}
```
