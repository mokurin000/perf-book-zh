# 内联

进入和退出热门的、未被内联的函数通常占用了不可忽略的执行时间。内联这些函数可以消除
这些进出开销，并允许编译器进行额外的底层优化。最好的情况下，整体效果虽小，但却是
轻松获得的速度提升。

有四种可用于 Rust 函数的内联属性。
- **无**。编译器将自行决定函数是否应该被内联。这取决于优化级别、函数大小、函数是否是
  泛型，以及内联是否跨越 crate 边界等因素。
- **`#[inline]`**。这建议函数应该被内联。
- **`#[inline(always)]`**。这强烈建议函数应该被内联。
- **`#[inline(never)]`**。这强烈建议函数不应该被内联。

内联属性并不保证函数一定被内联或不被内联，但在实践中，`#[inline(always)]` 会在除最
特殊情况外的所有情况下导致内联。

内联是不可传递的。如果一个函数 `f` 调用了一个函数 `g`，并且你希望这两个函数在 `f` 的
调用点一起被内联，那么这两个函数都应该用内联属性标记。

## 简单情况

最适合内联的是 (a) 非常小的函数，或 (b) 只有一个调用点的函数。即使没有内联属性，
编译器通常也会自行内联这些函数。但编译器并不总能做出最佳选择，因此有时需要属性。
[**示例 1**](https://github.com/rust-lang/rust/pull/37083/commits/6a4bb35b70862f33ac2491ffe6c55fb210c8490d),
[**示例 2**](https://github.com/rust-lang/rust/pull/50407/commits/e740b97be699c9445b8a1a7af6348ca2d4c460ce),
[**示例 3**](https://github.com/rust-lang/rust/pull/50564/commits/77c40f8c6f8cc472f6438f7724d60bf3b7718a0c),
[**示例 4**](https://github.com/rust-lang/rust/pull/57719/commits/92fd6f9d30d0b6b4ecbcf01534809fb66393f139),
[**示例 5**](https://github.com/rust-lang/rust/pull/69256/commits/e761f3af904b3c275bdebc73bb29ffc45384945d).

Cachegrind 是判断函数是否被内联的好工具。查看 Cachegrind 的输出时，你可以通过函数
的首行和末行*是否*标记了事件计数来判断函数是否已被内联。例如：
```text
      .  #[inline(always)]
      .  fn inlined(x: u32, y: u32) -> u32 {
700,000      eprintln!("inlined: {} + {}", x, y);
200,000      x + y
      .  }
      .  
      .  #[inline(never)]
400,000  fn not_inlined(x: u32, y: u32) -> u32 {
700,000      eprintln!("not_inlined: {} + {}", x, y);
200,000      x + y
200,000  }
```
添加内联属性后应再次测量，因为效果可能不可预测。有时它没有效果，因为附近一个原本
被内联的函数不再被内联了。有时它会拖慢代码。内联还会影响编译时间，特别是跨 crate
内联，这涉及复制函数的内部表示。

## 困难情况

有时你有一个较大的函数，并且有多个调用点，但只有一个调用点是热点。你希望内联热点
调用点以提升速度，但不要内联冷调用点以避免不必要的代码膨胀。处理此问题的方法是将
函数拆分为始终内联和永不内联的变体，后者调用前者。

例如，这个函数：
```rust
# fn one() {};
# fn two() {};
# fn three() {};
fn my_function() {
    one();
    two();
    three();
}
```
将变成这两个函数：
```rust
# fn one() {};
# fn two() {};
# fn three() {};
// 在热点调用点使用此函数。
#[inline(always)]
fn inlined_my_function() {
    one();
    two();
    three();
}

// 在冷调用点使用此函数。
#[inline(never)]
fn uninlined_my_function() {
    inlined_my_function();
}
```
[**示例 1**](https://github.com/rust-lang/rust/pull/53513/commits/b73843f9422fb487b2d26ac2d65f79f73a4c9ae3),
[**示例 2**](https://github.com/rust-lang/rust/pull/64420/commits/a2261ad66400c3145f96ebff0d9b75e910fa89dd).

## 外提

内联的逆操作是*外提*：将很少执行的代码移到一个单独的函数中。你可以为这类函数添加
`#[cold]` 属性，告诉编译器该函数很少被调用。这可以为热路径生成更好的代码。
[**示例 1**](https://github.com/Lokathor/tinyvec/pull/127),
[**示例 2**](https://crates.io/crates/fast_assert).
