+++
title = 'Rust 语法辨析：切片和字符串'
tags = ["rust", "字符串"]
date = 2024-05-19T12:38:05+08:00
slug = "rust-slice"
+++

## 何为切片 Slice

Rust 中，[切片(slice)](https://doc.rust-lang.org/reference/types/slice.html)属于原始数据类型 *primitive type*，是一种[动态尺寸类型(Dynamically sized type)](https://doc.rust-lang.org/reference/dynamically-sized-types.html)。切片类型的泛型写法是 `[T]`，它是对内存中一系列 `T` 类型元素所组成序列的“视图(View)”。这里的内存，可能是堆(Heap)、栈(Stack)、只读数据区(Literals)。特别的，`str` 类型本质上就是符合 `UTF-8` 编码的 `[u8]` 类型，它就是字符串切片。

> UTF-8(8-bit Unicode Transformation Format/Universal Character Set)是在 Unicode 标准基础上定义的一种可变长度字符编码。它可以表示 Unicode 标准中的任何字符，而且其编码中的第一个字节仍与 ASCII 兼容。

Slice 类型非常特殊，在代码层面，它并不真的存在。换言之，你不能在代码中声明一个 `[T]` 或 `str` 类型的变量并拥有内存对象的所有权。以 `str` 为例，它只能以 `&str` `&mut str` `Box<str>` `String` 等形式呈现，前两者是对 `str` 的引用，后两者包含了指向 `str` 的指针。

对于 slice 类型 `[T]` 而言，有三种常见的切片引用：

- `&[T]`：共享切片(shared slice)，是切片的不可变借用，它不拥有 `[T]` 内存对象的所有权。..为了方便，共享切片也被简称为切片..
- `&mut [T]`：可变切片(mutable slice)，可变借用于它指向的 `[T]` 内存对象，同样没有所有权
- `Box<[T]>`：智能指针切片(boxed slice)，`[T]` 内存对象存储在堆(heap)上，Box 切片拥有它的所有权

```Rust
// 一个在堆上分配的数组 [i32; 3] 被自动强转成切片 [i32]
let boxed_array: Box<[i32]> = Box::new([1, 2, 3]);

// 数组形式的共享切片
let slice: &[i32] = &boxed_array[..];
```

虽然 `[T]` 和 `str` 本身都是可变的（不妨试着用 `Box<str>` 调用 `make_ascii_uppercase()` 验证），但某些情况下是只读/不可变的，这时 Rust 编译器只允许我们使用它的不可变引用 `&[T]` `&str`。一个常见的例子是字符串字面量，它被硬编码进可执行程序，在程序运行的整个生命周期内都有效，因此绑定它的变量具有静态生命周期，换言之，绑定该字面量的变量类型实际是 `&'static str`。（这并不意味着有 `'static` 生命周期的 `str` 类型就不可变，仍然有办法构造出具有 `'static` 生命周期的 `&mut str`）

切片的所有元素总是初始化过的，使用 Rust 中的安全(safe)方法或操作符来访问切片时总是会做越界检查。

有些编程语言（如 C 语言）会在字符串末尾添加一个零字符 `\0`，并记录起始地址。要确定字符串的长度，程序必须从起始位置开始遍历原始字节，直到找到这个零字节。但 Rust 采用的方法不同：它用来访问字符串的 `&str` 引用是宽指针，包括了字符串起始地址（裸指针）和所需字节数，这比追加零字节更好，因为计算在编译时就提前完成。

事实上，上述三种切片引用都是宽指针，均包括了指向内存对象的指针和内存对象的尺寸，是普通指针的两倍大小。你可能会好奇，为什么切片的引用都是宽指针？这和我们之前提到的动态尺寸类型有关。

## 动态尺寸类型 DST

Rust 中大多数的类型都有一个在编译时就已知的固定尺寸，并实现了 Trait `Sized`。只有在运行时才知道尺寸的类型称为动态尺寸类型(dynamically sized type)（DST），或者非正式地称为非固定尺寸类型(unsized type)。切片和[特征对象(Trait object)](https://www.zhihu.com/question/581900340/answer/2873592812)是 DSTs 的两个例子。

注意，这里提到的尺寸未知是对类型而言，即 DST(slice, Trait object) ..类型的尺寸..无法确定，而非变量值的尺寸。例如，`str` 类型可以是任意长度（只要不超出计算机内存的限制），但具体到一个字符串字面量 `"Hello World!"`，其长度在编译时是确定无疑且不可更改的。

固定尺寸类型的引用只需要指向内存对象的第一个字节，不需要知道内存对象的尺寸，因为 Rust 在编译时会生成包含类型信息的机器码，对每个固定尺寸类型的数据，Rust 都能知道其类型，从而确定大小。但对于动态尺寸类型，即使知道内存对象的类型(比如明确是 `str` 类型)，由于尺寸可以是任意值，仍无法确定应该引用的内存范围，因而必须使用宽指针。

编译器在编译时需要计算局部变量和参数所需的内存，并相应地为每个栈帧分配空间。`[1..4]` 是切片语法，会从变量 `a` 的内存对象中截取一部分。编译器无法从切片语法中确定结果切片的大小，因此下面的代码报错：

```Rust
fn slice_out_of_array() {
    let a: [u8; 5] = [1, 2, 3, 4, 5];
    let nice_slice = a[1..4]; // 报错：[u8] doesn't have a size known at compile-time
    assert_eq!([2, 3, 4], nice_slice)
}
```

## `String` 字符串

如前所述，Rust `str` 是符合 `UTF-8` 规范的一串 `[u8]` 数据，同理 `String` 类型是基于 `Vec<u8>` 的封装，二者堆内存分配策略一致：`2->4->8`，如果容量不够，下次申请的为前一次的 2 倍。和 `Vec<u8>` 一样，`String` 类型变量的内存对象存储在堆上，且拥有它的所有权。

`String` 类型在标准库中的定义：

```Rust
pub struct String {
    vec: Vec<u8>,
}
```

可以看出，`String` 类型定义中的 `vec` 字段是私有的。这意味着我们不能直接创建字符串实例，只能通过封装的方法来创建。之所以保持私有，是因为并非所有 `[u8]` 字节流都符合 `UTF-8` 标准，与底层 `u8` 字节的直接交互可能会破坏字符串数据。通过这种受控访问，编译器可以确保 `String` 数据始终有效。以下是两种初始化 `String` 的方式：

```Rust
let hello_world: &str = "hello world"; // hello_world 指向只读数据区

let s: String = String::from(hello);
let s: String = hello.to_string(); // 发生了变量遮蔽

let world: &str = &s[6..] // world 指向堆
```

显然，`&str` 类型可以指向堆，也可以指向只读数据区，还可以指向栈：只需将分配到栈上的字节数组转换为 `&str` 类型，这时 `str` 自然是栈上的内存对象：

```Rust
use std::str;

let x: &[u8] = &[b'a', b'b', b'c'];
let stack_str: &str = str::from_utf8(x).unwrap();
```

作为存储在栈上的宽指针，`String` 类型包括三部分：指针、长度和容量，相比于 `&str` 类型仅增加了一个容量字段，因为 `String` 指向堆内存，所以运行过程中它的长度可以动态改变。

![alt text](/images/str-pointer.png "s 是 String 类型，world 是 &str 类型")

`&String` 类型还可以被隐式的转换为 `&str` 类型，因此当函数参数为 `&str` 类型时，不仅能传入 `&str` 变量，也可以传入 `&String` 变量，这样的函数使用更加灵活。

## `Box<str>` 字符串

`Box<str>` 类型是 `Box<[T]>` 的子集，如前所述，它是一个智能指针/宽指针，将 `str` 放在堆上。不同于 `&str` `&mut str`，`Box<str>` 拥有内存对象的所有权。相比 `String` 类型，`Box<str>` 缺少 `capacity` 字段，这意味着无法修改 `Box<str>` 中 `str` 的长度，只能改变 `str` 中每个字符的值。

## 总结

`[T]` `str` 类型数据可以存储在以下三种位置：

- Heap 堆：`Box<T>` `String` 类型
- Stack 栈：如前所述
- 只读数据区：绑定的字符串字面量 `"hello"` 直接被硬编码进可执行程序中，运行时载入内存的只读数据区

一图以蔽之，Rust 字符串内存模型如下：

![alt text](/images/rust-str-model.webp)
