# 类型大小

缩小频繁实例化的类型有助于提升性能。

例如，如果内存使用量很高，可以使用 [DHAT] 等堆分析器来识别热点分配点和涉及的类型。
缩小这些类型可以降低峰值内存使用量，并通过减少内存流量和缓存压力来改善性能。

[DHAT]: https://www.valgrind.org/docs/manual/dh-manual.html

此外，大于 128 字节的 Rust 类型会通过 `memcpy` 而非内联代码进行复制。如果 `memcpy`
在性能分析结果中占据了不可忽略的比例，DHAT 的"复制分析"模式将准确告诉你热点 `memcpy`
调用在哪里以及涉及哪些类型。将这些类型缩小到 128 字节或更少，可以通过避免 `memcpy`
调用和减少内存流量来使代码更快。

## 测量类型大小

[`std::mem::size_of`] 给出类型的大小（以字节为单位），但你通常还需要知道精确的内存布局。
例如，一个 enum 可能因为单个过大的变体而大得出奇。

[`std::mem::size_of`]: https://doc.rust-lang.org/std/mem/fn.size_of.html

`-Zprint-type-sizes` 选项正好能做到这一点。它在 release 版本的 rustc 上未启用，
因此你需要使用 nightly 版本的 rustc。以下是通过 Cargo 调用的一种方式：
```text
RUSTFLAGS=-Zprint-type-sizes cargo +nightly build --release
```
以下是 rustc 的直接调用方式：
```text
rustc +nightly -Zprint-type-sizes input.rs
```
它将打印出所有使用中类型的大小、布局和对齐的详细信息。例如，对于这个类型：
```rust
enum E {
    A,
    B(i32),
    C(u64, u8, u64, u8),
    D(Vec<u32>),
}
```
它会打印以下内容，以及一些内置类型的信息。
```text
print-type-size type: `E`: 32 bytes, alignment: 8 bytes
print-type-size     discriminant: 1 bytes
print-type-size     variant `D`: 31 bytes
print-type-size         padding: 7 bytes
print-type-size         field `.0`: 24 bytes, alignment: 8 bytes
print-type-size     variant `C`: 23 bytes
print-type-size         field `.1`: 1 bytes
print-type-size         field `.3`: 1 bytes
print-type-size         padding: 5 bytes
print-type-size         field `.0`: 8 bytes, alignment: 8 bytes
print-type-size         field `.2`: 8 bytes
print-type-size     variant `B`: 7 bytes
print-type-size         padding: 3 bytes
print-type-size         field `.0`: 4 bytes, alignment: 4 bytes
print-type-size     variant `A`: 0 bytes
```
输出显示了以下内容。
- 类型的大小和对齐。
- 对于 enum，discriminant 的大小。
- 对于 enum，每个变体的大小（从最大到最小排序）。
- 所有字段的大小、对齐和顺序。（注意编译器已重新排序了变体 `C` 的字段以最小化
  `E` 的大小。）
- 所有填充的大小和位置。

另外，[top-type-sizes] crate 可用于以更紧凑的形式显示输出。

[top-type-sizes]: https://crates.io/crates/top-type-sizes

一旦你知道了热类型的布局，有多种方法可以缩小它。

## 字段顺序

Rust 编译器会自动排序 struct 和 enum 中的字段以最小化它们的大小（除非指定了
`#[repr(C)]` 属性），因此你无需担心字段顺序。但还有其他方法可以最小化热类型的大小。

## 更小的 Enum

如果 enum 有一个过大的变体，考虑将其中一个或多个字段装箱。例如，你可以将这个类型：
```rust
type LargeType = [u8; 100];
enum A {
    X,
    Y(i32),
    Z(i32, LargeType),
}
```
改为：
```rust
# type LargeType = [u8; 100];
enum A {
    X,
    Y(i32),
    Z(Box<(i32, LargeType)>),
}
```
这会减小类型大小，但代价是需要为 `A::Z` 变体额外进行一次堆分配。如果 `A::Z` 变体
相对较少出现，则此更改更可能带来净性能收益。`Box` 还会使 `A::Z` 的可用性略有下降，
特别是在 `match` 模式中。
[**示例 1**](https://github.com/rust-lang/rust/pull/37445/commits/a920e355ea837a950b484b5791051337cd371f5d),
[**示例 2**](https://github.com/rust-lang/rust/pull/55346/commits/38d9277a77e982e49df07725b62b21c423b6428e),
[**示例 3**](https://github.com/rust-lang/rust/pull/64302/commits/b972ac818c98373b6d045956b049dc34932c41be),
[**示例 4**](https://github.com/rust-lang/rust/pull/64374/commits/2fcd870711ce267c79408ec631f7eba8e0afcdf6),
[**示例 5**](https://github.com/rust-lang/rust/pull/64394/commits/7f0637da5144c7435e88ea3805021882f077d50c),
[**示例 6**](https://github.com/rust-lang/rust/pull/71942/commits/27ae2f0d60d9201133e1f9ec7a04c05c8e55e665).

## 更小的整数

通常可以通过使用更小的整数类型来缩小类型。例如，虽然使用 `usize` 作为索引最为自然，
但将索引存储为 `u32`、`u16` 甚至 `u8`，然后在使用点转换为 `usize` 通常是合理的。
[**示例 1**](https://github.com/rust-lang/rust/pull/49993/commits/4d34bfd00a57f8a8bdb60ec3f908c5d4256f8a9a),
[**示例 2**](https://github.com/rust-lang/rust/pull/50981/commits/8d0fad5d3832c6c1f14542ea0be038274e454524).

## Boxed Slices

Rust 向量包含三个字段：长度、容量和指针。如果你有一个未来不太可能改变的向量，可以
使用 [`Vec::into_boxed_slice`] 将其转换为 *boxed slice*。一个 boxed slice 只包含
两个字段：长度和指针。任何多余的元素容量都会被丢弃，这可能导致重新分配。
```rust
# use std::mem::{size_of, size_of_val};
let v: Vec<u32> = vec![1, 2, 3];
assert_eq!(size_of_val(&v), 3 * size_of::<usize>());

let bs: Box<[u32]> = v.into_boxed_slice();
assert_eq!(size_of_val(&bs), 2 * size_of::<usize>());
```
或者，可以使用 [`Iterator::collect`] 直接从迭代器构建 boxed slice。如果迭代器的
长度是预先已知的，这样可以避免任何重新分配。
```rust
let bs: Box<[u32]> = (1..3).collect();
```
Boxed slice 可以使用 [`slice::into_vec`] 转换为向量，无需任何克隆或重新分配。

[`Vec::into_boxed_slice`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.into_boxed_slice
[`Iterator::collect`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect
[`slice::into_vec`]: https://doc.rust-lang.org/std/primitive.slice.html#method.into_vec

## `ThinVec`

Boxed slices 的一个替代方案是来自 [`thin_vec`] crate 的 `ThinVec`。它在功能上等同于
`Vec`，但将长度和容量与元素（如果有的话）存储在同一个分配中。这意味着
`size_of::<ThinVec<T>>` 只有一个字段大小。

`ThinVec` 在频繁实例化的类型中，对于经常为空的向量是一个不错的选择。它也可以用于
缩小 enum 的最大变体（如果该变体包含 `Vec`）。

[`thin_vec`]: https://crates.io/crates/thin-vec

## 避免回归

如果一个类型非常热门，其大小会影响性能，那么使用静态断言来确保它不会意外地变大
是一个好主意。以下示例使用了来自 [`static_assertions`] crate 的宏。
```rust,ignore
  // 此类型使用频繁。确保它不会无意中变大。
  #[cfg(target_arch = "x86_64")]
  static_assertions::assert_eq_size!(HotType, [u8; 64]);
```
`cfg` 属性很重要，因为类型大小在不同的平台上可能不同。将断言限制在 `x86_64`
（通常是最广泛使用的平台）很可能足以在实践中防止回归。

[`static_assertions`]: https://crates.io/crates/static_assertions
