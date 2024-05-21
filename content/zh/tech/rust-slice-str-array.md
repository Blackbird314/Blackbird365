+++
title = 'Rust 语法辨析：切片、数组和字符串'
tags = ["rust"]
date = 2024-05-19T12:38:05+08:00
slug = "rust-slice-str-array"
+++

## 何为切片 Slice

Rust 中，[切片(slice)](https://doc.rust-lang.org/reference/types/slice.html)是一种[动态尺寸类型(Dynamically sized type)](https://doc.rust-lang.org/reference/dynamically-sized-types.html)，切片类型的泛型写法是 `[T]`，它是对内存中一系列 `T` 类型元素所组成序列的“视图”。这里的内存，可能是堆(heap)、栈(stack)、静态存储区(Static Storage)，还可能硬编码进可执行程序。特别的，`str` 类型本质上就是符合 `UTF-8` 编码的 `[u8]` 类型。

> UTF-8(8-bit Unicode Transformation Format/Universal Character Set)是在 Unicode 标准基础上定义的一种可变长度字符编码。它可以表示 Unicode 标准中的任何字符，而且其编码中的第一个字节仍与 ASCII 兼容。

slice 类型非常特殊，它并不真的存在，换言之，你不能在代码中声明一个 `[T]` 类型的变量并拥有 `[T]` 内存对象的所有权。以 `str` 为例，它只能以 `&str` `&mut str` `Box<str>` `String` 等形式呈现，前两者是对 `str` 的引用，后两者包含了指向 `str` 的指针。

对于 slice 类型 `[T]` 而言，有三种常见的切片引用：

- `&[T]`：共享切片(shared slice)，是切片的不可变借用，它不拥有 `[T]` 内存对象的所有权，..为了方便，共享切片也被简称为切片..
- `&mut [T]`：可变切片(mutable slice)，可变借用于它指向的 `[T]` 内存对象，同样没有所有权
- `Box<[T]>`：智能指针切片(boxed slice)，`[T]` 内存对象存储在堆(heap)上，Box 切片拥有它的所有权

```Rust
// 一个在堆上分配的数组 [i32; 3] 被自动强转成切片 [i32]
let boxed_array: Box<[i32]> = Box::new([1, 2, 3]);

// 数组形式的共享切片
let slice: &[i32] = &boxed_array[..];
```

虽然 `[T]` 和 `str` 本身都是可变的（不妨试着用 `Box<str>` 调用 `make_ascii_uppercase()` 验证），但它被硬编码进二进制程序时，它是只读/不可变的，这时 Rust 编译器就只允许我们使用它的不可变引用 `&[T]` `&str`。一个常见的例子是字符串字面量，它在程序运行的整个生命周期内都有效，因此绑定它的变量具有静态生命周期，换言之，绑定该字面量的变量类型实际是 `&'static str`。

切片的所有元素总是初始化过的，使用 Rust 中的安全(safe)方法或操作符来访问切片时总是会做越界检查。

上述三种切片引用都是宽指针，包括了指向内存对象的指针和内存对象的尺寸，是普通指针的两倍大。你可能会好奇，为什么切片的引用都是宽指针？这和我们之前提到的动态尺寸类型有关。

## 动态尺寸类型 DST

Rust 中大多数的类型都有一个在编译时就已知的固定尺寸，并实现了 trait `Sized`。只有在运行时才知道尺寸的类型称为动态尺寸类型(dynamically sized type)（DST），或者非正式地称为非固定尺寸类型(unsized type)。切片和[特征对象(trait object)](https://www.zhihu.com/question/581900340/answer/2873592812)是 DSTs 的两个例子。

注意，这里提到的尺寸未知是对类型而言，即 DST(slice, trait object) ..类型的尺寸..无法确定，而非变量值的尺寸。例如，`str` 类型可以是任意长度（只要不超出计算机内存的限制），但具体到一个字符串字面量，其长度是确定无疑且不可更改的。

固定尺寸类型的引用只需要指向该类型内存对象的第一个字节，不需要了解内存对象的尺寸，因为 Rust 在编译时会生成包含类型信息的机器码，对每个固定尺寸类型的数据，Rust 都能知道其类型，从而确定大小。但对于动态尺寸类型，即使知道内存对象的类型，也无法确定应该引用的范围。

## 字符串
