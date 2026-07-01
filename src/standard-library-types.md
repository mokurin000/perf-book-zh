# 标准库类型

值得通读常见标准库类型（如 [`Vec`]、[`Option`]、[`Result`] 和 [`Rc`]/[`Arc`]）的
文档，以找到有时可用于提升性能的有趣函数。

[`Vec`]: https://doc.rust-lang.org/std/vec/struct.Vec.html
[`Option`]: https://doc.rust-lang.org/std/option/enum.Option.html
[`Result`]: https://doc.rust-lang.org/std/result/enum.Result.html
[`Rc`]: https://doc.rust-lang.org/std/rc/struct.Rc.html
[`Arc`]: https://doc.rust-lang.org/std/sync/struct.Arc.html

了解标准库类型的高性能替代方案也很有价值，例如 [`Mutex`]、[`RwLock`]、[`Condvar`] 和
[`Once`]。

[`Mutex`]: https://doc.rust-lang.org/std/sync/struct.Mutex.html
[`RwLock`]: https://doc.rust-lang.org/std/sync/struct.RwLock.html
[`Condvar`]: https://doc.rust-lang.org/std/sync/struct.Condvar.html
[`Once`]: https://doc.rust-lang.org/std/sync/struct.Once.html

## `Vec`

创建长度为 `n` 的零填充 `Vec` 的最佳方法是使用 `vec![0; n]`。这很简单，并且可能
[与替代方案一样快或更快]，例如使用 `resize`、`extend` 或任何涉及 `unsafe` 的方式，
因为它可以利用操作系统的协助。

[与替代方案一样快或更快]: https://github.com/rust-lang/rust/issues/54628

[`Vec::remove`] 移除特定索引处的元素并将所有后续元素向左移动一位，复杂度为 O(n)。
[`Vec::swap_remove`] 用最后一个元素替换特定索引处的元素，这不保持顺序，但复杂度为 O(1)。

[`Vec::retain`] 高效地从 `Vec` 中移除多个元素。其他集合类型如 `String`、`HashSet` 和
`HashMap` 也有等效的方法。

[`Vec::remove`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.remove
[`Vec::swap_remove`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.swap_remove
[`Vec::retain`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.retain

## `Option` 和 `Result`

[`Option::ok_or`] 将 `Option` 转换为 `Result`，并接受一个 `err` 参数，当 `Option`
值为 `None` 时使用。`err` 是即时计算的。如果其计算开销很大，应改用
[`Option::ok_or_else`]，它通过闭包惰性地计算错误值。例如，这个：
```rust
# fn expensive() {}
# let o: Option<u32> = None;
let r = o.ok_or(expensive()); // 总是求值 `expensive()`
```
应改为：
```rust
# fn expensive() {}
# let o: Option<u32> = None;
let r = o.ok_or_else(|| expensive()); // 仅在需要时求值 `expensive()`
```
[**示例**](https://github.com/rust-lang/rust/pull/50051/commits/5070dea2366104fb0b5c344ce7f2a5cf8af176b0).

[`Option::ok_or`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.ok_or
[`Option::ok_or_else`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.ok_or_else

[`Option::map_or`]、[`Option::unwrap_or`]、
[`Result::or`]、[`Result::map_or`] 和 [`Result::unwrap_or`] 也有类似的替代方法。

[`Option::map_or`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.map_or
[`Option::unwrap_or`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap_or
[`Result::or`]: https://doc.rust-lang.org/std/result/enum.Result.html#method.or
[`Result::map_or`]: https://doc.rust-lang.org/std/result/enum.Result.html#method.map_or
[`Result::unwrap_or`]: https://doc.rust-lang.org/std/result/enum.Result.html#method.unwrap_or

## `Rc`/`Arc`

[`Rc::make_mut`]/[`Arc::make_mut`] 提供了写时复制语义。它们获取 `Rc`/`Arc` 的可变引用。
如果引用计数大于 1，它们将 `clone` 内部值以确保唯一所有权；否则，它们将修改原始值。
虽然不常用，但偶尔会非常有用。
[**示例 1**](https://github.com/rust-lang/rust/pull/65198/commits/3832a634d3aa6a7c60448906e6656a22f7e35628),
[**示例 2**](https://github.com/rust-lang/rust/pull/65198/commits/75e0078a1703448a19e25eac85daaa5a4e6e68ac).

[`Rc::make_mut`]: https://doc.rust-lang.org/std/rc/struct.Rc.html#method.make_mut
[`Arc::make_mut`]: https://doc.rust-lang.org/std/sync/struct.Arc.html#method.make_mut

## `Mutex`、`RwLock`、`Condvar` 和 `Once`

[`parking_lot`] crate 提供了这些同步类型的替代实现。`parking_lot` 类型的 API 和语义
与标准库中等效类型相似但不完全相同。

`parking_lot` 版本过去在体积、速度和灵活性上始终优于标准库版本，但标准库版本在某些
平台上已经有了很大改进。因此，在切换到 `parking_lot` 之前应该进行测量。

[`parking_lot`]: https://crates.io/crates/parking_lot

如果你决定普遍使用 `parking_lot` 类型，很容易在某些地方意外地使用标准库的等效类型。
你可以[使用 Clippy] 来避免这个问题。

[使用 Clippy]: linting.md#disallowing-types
