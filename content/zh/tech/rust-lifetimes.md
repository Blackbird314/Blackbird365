+++
title = "Rust 核心语法：生命周期与借用检查"
date = "2024-05-24T17:37:27+08:00"
tags = ["Rust"]
slug = "rust-lifetimes"
+++

## 从重借用开始

上文提到，借用检查不允许对一个实例的多个可变引用，也不能同时存在共享和可变引用。但对解引用得到的影子变量进行借用（重借用）却是可行的：

```Rust
let mut s = String::from("ABC");
let r1 = &mut s;
{
    let r2 = &mut *r1; // 重借用
    r2.push('2');
    println!("{}", r2); // r2 的作用域到此结束
}
println!("{}", r1); // r1 的作用域到此结束
```

这段代码的大括号内，同时存在 `r1` `r2` 两个指向同一变量 `s` 的可变引用，但编译器不会报错。这是因为编译器看到 `*r1` 的时候，通常很难确定解引用得到的实际对象是什么，所以借用检查不会把 `*r1` 跟 `s` 当成同一个对象，自然不会报错。

关于重借用，唯一的限制是不能在 `r2` 的生命周期中使用 `r1`，这样就不会违反借用规则：

```Rust
let mut s = String::from("ABC");
let r1 = &mut s;
{
    let r2 = &mut *r1;

    let l = r1.len(); // 错误 Cannot borrow `*r1` as immutable because it is also borrowed as mutable
    println!("{}", l);
    r2.push('2');
    r1.push('3'); // 错误 Cannot borrow `*r1` as mutable more than once at a time
    println!("{:?}", r1); // 错误 Cannot borrow `r1` as immutable because it is also borrowed as mutable
    println!("{}", r2);
}
println!("{}", r1);
```

你可能会好奇，明明传入方法的是引用类型，为什么前两条报错信息中会显示 `*r1`？这是因为发生了隐式重借用，`r1.len()` 实际上是 `String::len(&*r1)`，同理 `r1.push('3')` 实际上是 `String::push(&*r1, '3')`。

隐式重借用并非多此一举，`len()` 和 `push()` 的方法签名分别是 `pub fn len(&self) -> usize` `pub fn push(&mut self, ch: char)`。没有隐式重借用，可变引用 `r1` 将无法调用 `len()`，而 `r1.push('3')` 会转移可变引用 `r1` 的所有权，导致 `r1` 之后无法使用。

事实上，隐式重借用几乎无处不在：

```Rust
let mut s = String::from("ABC");
let r1 = &mut s;
// 不标注 r2 的类型，会 Move 而非隐式重借用，之后 r1 失效
let r2 = r1;
// 手动标注 r2 的类型，会进行非隐式重借用，比如调用函数时传参
let r3: &mut String = r2; // 相当于 let r3: &mut String = &mut *r2;
println!("{:?}", r3);
println!("{:?}", r2); // 打印 r3 r2 的顺序不能颠倒
```

对共享引用 `&T`，可以认为发生了重借用, 也可以认为直接发生 `Copy`，因为效果完全一样。

### 手写重借用

下面这两种情况[^1]，`from()` 函数不会自动重借用：

```Rust
struct X;

impl From<&mut i32> for X {
    fn from(i: &mut i32) -> Self {
        X
    }
}

let mut i = 4;
let r = &mut i;

fn from_auto_reborrow<'a, F, T: From<&'a mut F>>(f: &'a mut F) -> T {
    T::from(f)
}
let x: X = from_auto_reborrow(r); // 可以自动重借用
let x: X = from_auto_reborrow(r); // 可以自动重借用

fn from<F, T: From<F>>(f: F) -> T {
    T::from(f)
}
let x: X = from(&mut *r); // 必须显式重借用, 创建 x 的 reborrow 不会 move x
let x: X = from(r); // 此处不会自动重借用, 导致 Move x
let x: X = from(r); // 编译失败
```

```Rust
struct I(i32);

struct X1;
impl From<&mut I> for X1 {
    fn from(p: &mut I) -> X1 {
        p.0 = 1;
        X1
    }
}
// 必须引入这个中间函数
fn x1(p: &mut I) -> X1 {
    X1::from(p)
}

// value used here after move
fn from_twice_fail(p: &mut I) {
    let x11 = X1::from(p); // p Move
    let x12 = X1::from(p); // 编译失败
}

fn from_twice(p: &mut I) {
    let x11 = x1(p); // 此处不会自动重借用, 导致 Move p
    let x12 = x1(p); // 编译通过
}
```

[^1]: 参考：<https://rustcc.cn/article?id=28fedcbc-d0c9-41e1-8d95-de73a578a078>