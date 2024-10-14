---
title: Macro 宏编程
pubDatetime: 2024-10-14T17:41:00+08:00
author: shushunie
slug: rust-l4-06
description: ""
---

## 迭代器 Iterator

迭代器允许我们迭代一个连续的集合，例如数组、动态数组 `Vec`、`HashMap` 等，在此过程中，只需关心集合中的元素如何处理，而无需关心如何开始、如何结束、按照什么样的索引去访问等问题

```javascript
let arr = [1, 2, 3];
for (let i = 0; i < arr.length; i++) {
  console.log(arr[i]);
}
```

上述代码是 js 中的 `for` 循环，通过设置索引的开始点和结束点，然后再通过索引去访问元素 `arr[i]`。Rust 中使用 `for in` 循环，不需要通过索引就能直接遍历数组。新版本的 js 也已经支持 `for in` 循环，在写法上能方便许多。使用这种循环形式，需要被遍历的数据实现相应的迭代器，在 Rust 中我们说需要被遍历的元素实现 `Iterator` 特征

#### next 方法

迭代器内部是通过调用 next 方法来进行遍历的，我们也可以手动调用 next 方法

```rust
fn main() {
    let arr = [1, 2, 3];
    let mut arr_iter = arr.into_iter();

    assert_eq!(arr_iter.next(), Some(1));
    assert_eq!(arr_iter.next(), Some(2));
    assert_eq!(arr_iter.next(), Some(3));
    assert_eq!(arr_iter.next(), None);
}
```

#### 实现迭代器

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

```

实现自己的迭代器非常简单，但是 `Iterator` 特征中，不仅仅是只有 `next` 一个方法。但其它方法都具有默认实现，所以无需像 `next` 这样手动去实现，而且这些默认实现的方法其实都是基于 `next` 方法实现的，下面列举四个常用方法

- `zip` 把两个迭代器合并成一个迭代器，新迭代器中，每个元素都是一个元组，由之前两个迭代器的元素组成。例如将**形如** `[1, 2, 3, 4, 5]` 和 `[2, 3, 4, 5]` 的迭代器合并后，新的迭代器形如 `[(1, 2),(2, 3),(3, 4),(4, 5)]`

- `map` 是将迭代器中的值经过映射后，转换成新的值[2, 6, 12, 20]

- `filter` 对迭代器中的元素进行过滤，若闭包返回 `true` 则保留元素[6, 12]，反之剔除

- `enumerate` 产生一个新的迭代器，其中每个元素均是元组 `(索引，值)`

#### Iterator 和 IntoIterator 的区别

`Iterator` 就是迭代器特征，只有实现了它才能称为迭代器，才能调用 `next`

`IntoIterator` 强调的是某一个类型如果实现了该特征，它可以通过 `into_iter`，`iter` 等方法变成一个迭代器
