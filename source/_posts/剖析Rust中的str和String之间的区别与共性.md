---
title: 剖析Rust中的str和String之间的区别与共性
date: 2021-03-05 13:17:37
tags:
 - Rust
keywords:
 - Rust
 - String
 - str
categories: Rust
description:
summary_img:
---

之前写代码的时候发现自己其实完全不懂`String`和`str`之间的区别，现在写篇文章来细 🔒

<!-- more -->

首先查看[官方参考文档](https://doc.rust-lang.org/std/string/struct.String.html)中对 String 的描述如下

> The `String` type is the most common string type that has ownership over the contents of the string. It has a close relationship with its borrowed counterpart, the primitive `str`.

我们可以发现 `String` 是拥有字符串内容的所有权的，而 `str` 则被描述为“借用”。查看具体实现的源代码，有如下的实现内容：

```rust
pub struct String {
    vec: Vec<u8>,
}
```

也就是说，String 是确确实实有“拥有”这个字符串内容的。而 `str` 在[参考文档](https://doc.rust-lang.org/std/str/index.html)中被描述为如下的内容：

> Unicode string slices.
>
> _See also the `std::str` primitive module._
>
> The `&str` type is one of the two main string types, the other being `String`. Unlike its `String` counterpart, its contents are borrowed.

string slices 意为字符串切片，那么说明一般 `str` 类型是作为指针，描述一个字符串切片的存在，而作为一个 primitive type，无法找到其具体内容实现，但是可以确定的是，它并没有对指向内容的所有权，但是由于往往作为引用 `&str` 存在，因此有着更高的灵活性与便捷度。

## 关于字符常量

我们常常会使用类似于如下的代码创建 `String` 以及 `&str` 变量：

```rust
let a_string = String::from("驻留字符串");
let a_str = "这是字符串常量";
```

学习 Java 虚拟机时就学习过，虚拟机会将编译得到的字符串`驻留(interned)`，即，提前为不可变字符串在全局区域分配空间，如果有多个相同引用只需要实际创建一次即可。

我们可以看到，不论是上面的 `a_string` 还是下面的 `a_str`，都使用了相应的字符串常量，分别分配了两段内容为`驻留字符串`和`这是字符串常量`的文本于内存中，同时，`String::from`的实现为：

```rust
// String::from
impl From<Box<str>> for String {
    fn from(s: Box<str>) -> String {
        s.into_string()
    }
}
```

继续追溯：

```rust
// str::into_string
pub fn into_string(self: Box<str>) -> String {
    let slice = Box::<[u8]>::from(self);
    unsafe { String::from_utf8_unchecked(slice.into_vec()) }
}
// String::from_utf8_unchecked
pub unsafe fn from_utf8_unchecked(bytes: Vec<u8>) -> String {
    String { vec: bytes }
}
// str::into_vec
pub fn into_vec<A: Allocator>(self: Box<Self, A>) -> Vec<T, A> {
    // N.B., see the `hack` module in this file for more details.
    hack::into_vec(self)
}
// hack::into_vec
pub fn into_vec<T, A: Allocator>(b: Box<[T], A>) -> Vec<T, A> {
    unsafe {
        let len = b.len();
        let (b, alloc) = Box::into_raw_with_allocator(b);
        Vec::from_raw_parts_in(b as *mut T, len, len, alloc)
    }
}
```

我们发现，`String::from`其实是根据字符串常量内容自行 alloc 了相同的空间，并且将内容复制到该空间内（在堆空间内）。

## 总结

对于希望拥有字符串所有权，需要更大程度地掌控字符串，请使用 `String` 类型，而如果是作为一个不会更改的常量存在，则应该使用 `str` 类型进行操作。
