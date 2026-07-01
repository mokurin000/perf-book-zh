# 边界检查

默认情况下，在 Rust 中对切片和向量等容器类型的访问会涉及边界检查。这些检查可能会
影响性能，例如在热循环中，尽管这种情况比你想象的更少见。

有几种安全的方法可以修改代码，让编译器知道容器的长度，从而优化掉边界检查。

- 使用迭代替换循环中的直接元素访问。
- 不要在循环内对 `Vec` 进行索引，而是在循环前创建 `Vec` 的一个切片，然后在循环内
  对该切片进行索引。
- 对索引变量的范围添加断言。
[**示例 1**](https://github.com/rust-random/rand/pull/960/commits/de9dfdd86851032d942eb583d8d438e06085867b),
[**示例 2**](https://github.com/image-rs/jpeg-decoder/pull/167/files).

让这些方法生效可能有些棘手。[Bounds Check Cookbook] 对此主题有更详细的讨论。

[Bounds Check Cookbook]: https://github.com/Shnatsel/bounds-check-cookbook/

作为最后的手段，可以使用不安全的 [`get_unchecked`] 和 [`get_unchecked_mut`] 方法。

[`get_unchecked`]: https://doc.rust-lang.org/std/primitive.slice.html#method.get_unchecked
[`get_unchecked_mut`]: https://doc.rust-lang.org/std/primitive.slice.html#method.get_unchecked_mut

