# 包装类型

Rust 有多种"包装"类型，例如 [`RefCell`] 和 [`Mutex`]，它们为值提供特殊的行为。
访问这些值可能需要花费不可忽略的时间。如果多个这样的值通常一起被访问，将它们放在
单个包装器内可能会更好。

[`RefCell`]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[`Mutex`]: https://doc.rust-lang.org/std/sync/struct.Mutex.html

例如，像这样的 struct：
```rust
# use std::sync::{Arc, Mutex};
struct S {
    x: Arc<Mutex<u32>>,
    y: Arc<Mutex<u32>>,
}
```
也许更好的表示方式是这样：
```rust
# use std::sync::{Arc, Mutex};
struct S {
    xy: Arc<Mutex<(u32, u32)>>,
}
```
这是否有助于提升性能将取决于值的具体访问模式。
[**示例**](https://github.com/rust-lang/rust/pull/68694/commits/7426853ba255940b880f2e7f8026d60b94b42404).
