# 堆分配

堆分配的成本中等偏高。具体细节取决于所使用的分配器，但每次分配（和释放）通常涉及
获取全局锁、执行一些非平凡的数据结构操作，以及可能执行系统调用。小分配不一定比大
分配更便宜。值得了解哪些 Rust 数据结构和操作会引起分配，因为避免它们可以极大地
提升性能。

[Rust 容器速查表]提供了常见 Rust 类型的可视化展示，是以下各节的优秀伴侣。

[Rust 容器速查表]: https://docs.google.com/presentation/d/1q-c7UAyrUlM-eZyTo1pd8SZ0qwA_wYxmPZVOQkoDmH4/

## 性能分析

如果通用分析器显示 `malloc`、`free` 及相关函数是热点，那么尝试降低分配率和/或使用
替代分配器可能是值得的。

[DHAT] 是在降低分配率时使用的优秀分析器。它适用于 Linux 和其他一些 Unix 系统。
它精确定位热点分配位置及其分配率。确切结果会有所不同，但 rustc 的经验表明，每百万
条指令减少 10 次分配可以带来可测量的性能改进（例如约 1%）。

[DHAT]: https://www.valgrind.org/docs/manual/dh-manual.html

以下是 DHAT 的一些示例输出。
```text
AP 1.1/25 (2 children) {
  Total:     54,533,440 bytes (4.02%, 2,714.28/Minstr) in 458,839 blocks (7.72%, 22.84/Minstr), avg size 118.85 bytes, avg lifetime 1,127,259,403.64 instrs (5.61% of program duration)
  At t-gmax: 0 bytes (0%) in 0 blocks (0%), avg size 0 bytes
  At t-end:  0 bytes (0%) in 0 blocks (0%), avg size 0 bytes
  Reads:     15,993,012 bytes (0.29%, 796.02/Minstr), 0.29/byte
  Writes:    20,974,752 bytes (1.03%, 1,043.97/Minstr), 0.38/byte
  Allocated at {
    #1: 0x95CACC9: alloc (alloc.rs:72)
    #2: 0x95CACC9: alloc (alloc.rs:148)
    #3: 0x95CACC9: reserve_internal<syntax::tokenstream::TokenStream,alloc::alloc::Global> (raw_vec.rs:669)
    #4: 0x95CACC9: reserve<syntax::tokenstream::TokenStream,alloc::alloc::Global> (raw_vec.rs:492)
    #5: 0x95CACC9: reserve<syntax::tokenstream::TokenStream> (vec.rs:460)
    #6: 0x95CACC9: push<syntax::tokenstream::TokenStream> (vec.rs:989)
    #7: 0x95CACC9: parse_token_trees_until_close_delim (tokentrees.rs:27)
    #8: 0x95CACC9: syntax::parse::lexer::tokentrees::<impl syntax::parse::lexer::StringReader<'a>>::parse_token_tree (tokentrees.rs:81)
  }
}
```
描述此示例中的所有内容超出了本书的范围，但应该清楚的是，DHAT 提供了关于分配的丰富
信息，例如分配发生的位置和频率、分配的大小、存活时间以及访问频率。

## `Box`

[`Box`] 是最简单的堆分配类型。`Box<T>` 值是一个分配在堆上的 `T` 值。

[`Box`]: https://doc.rust-lang.org/std/boxed/struct.Box.html

有时值得将一个 struct 或 enum 中的一个或多个字段装箱，以使类型更小。（有关更多信息，
请参阅[类型大小](type-sizes.md)章节。）

除此之外，`Box` 很直接，没有太多优化空间。

## `Rc`/`Arc`

[`Rc`]/[`Arc`] 类似于 `Box`，但堆上的值附带两个引用计数。它们允许值共享，
这可以成为减少内存使用的有效方式。

[`Rc`]: https://doc.rust-lang.org/std/rc/struct.Rc.html
[`Arc`]: https://doc.rust-lang.org/std/sync/struct.Arc.html

然而，如果用于很少共享的值，它们可能会通过堆分配那些原本不会堆分配的值来增加
分配率。
[**示例**](https://github.com/rust-lang/rust/pull/37373/commits/c440a7ae654fb641e68a9ee53b03bf3f7133c2fe).

与 `Box` 不同，对 `Rc`/`Arc` 值调用 `clone` 不涉及分配。相反，它只是增加
引用计数。

## `Vec`

[`Vec`] 是一个堆分配的类型，在优化分配数量和/或最小化浪费的空间方面有很大的空间。
要做到这一点，需要理解其元素是如何存储的。

[`Vec`]: https://doc.rust-lang.org/std/vec/struct.Vec.html

一个 `Vec` 包含三个字段：长度、容量和指针。如果容量非零且元素大小非零，指针将指向
堆分配的内存；否则，它将不指向已分配的内存。

即使 `Vec` 本身不是堆分配的，其元素（如果存在且大小非零）始终是堆分配的。如果存在
大小非零的元素，存储这些元素的内存可能大于实际需要，为未来的额外元素提供空间。
存在的元素数量是长度，而无需重新分配即可容纳的元素数量是容量。

当向量需要增长超过当前容量时，元素将被复制到一个更大的堆分配中，旧的堆分配将被释放。

### `Vec` 增长

通过常见方式（[`vec![]`](https://doc.rust-lang.org/std/macro.vec.html)
或 [`Vec::new`] 或 [`Vec::default`]）创建的新空 `Vec` 的长度和容量均为零，
不需要堆分配。如果你反复将单个元素推入 `Vec` 的末尾，它将定期重新分配。增长策略
未指定，但在撰写本文时，它使用一种准倍增策略，导致以下容量：0、4、8、16、32、64
等等。（它直接从 0 跳到 4，而不是经过 1 和 2，因为这在实践中[避免了许多分配]。）
随着向量的增长，重新分配的频率将呈指数级下降，但可能浪费的额外容量将呈指数级增加。

[`Vec::new`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.new
[`Vec::default`]: https://doc.rust-lang.org/std/default/trait.Default.html#tymethod.default
[避免了许多分配]: https://github.com/rust-lang/rust/pull/72227

这种增长策略对于可增长的数据结构是典型的，在一般情况下是合理的，但如果你预先知道
向量的大致长度，通常可以做得更好。如果你有一个热门的向量分配点（例如热门的
[`Vec::push`] 调用），值得使用 [`eprintln!`] 打印该点的向量长度，然后进行一些
后处理（例如使用 [`counts`]）来确定长度分布。例如，你可能有许多短向量，或者数量
较少但非常长的向量，优化分配点的最佳方式会相应变化。

[`Vec::push`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.push
[`eprintln!`]: https://doc.rust-lang.org/std/macro.eprintln.html
[`counts`]: https://github.com/nnethercote/counts/

### 短 `Vec`

如果你有许多短向量，可以使用 [`smallvec`] crate 中的 `SmallVec` 类型。
`SmallVec<[T; N]>` 是 `Vec` 的直接替代品，它可以在 `SmallVec` 自身内部存储 `N` 个
元素，然后在元素数量超过该值时切换到堆分配。（另请注意，`vec![]` 字面量必须替换为
`smallvec![]` 字面量。）
[**示例 1**](https://github.com/rust-lang/rust/pull/50565/commits/78262e700dc6a7b57e376742f344e80115d2d3f2),
[**示例 2**](https://github.com/rust-lang/rust/pull/55383/commits/526dc1421b48e3ee8357d58d997e7a0f4bb26915).

[`smallvec`]: https://crates.io/crates/smallvec

`SmallVec` 在适当使用时能可靠地降低分配率，但使用它并不能保证性能提升。对于正常操作，
它比 `Vec` 稍慢，因为它必须始终检查元素是否在堆上分配。此外，如果 `N` 较大或 `T` 较大，
则 `SmallVec<[T; N]>` 本身可能比 `Vec<T>` 更大，并且复制 `SmallVec` 值会更慢。
与往常一样，需要进行基准测试来确认优化是否有效。

如果你有许多短向量*并且*精确知道它们的最大长度，那么来自 [`arrayvec`] crate 的
`ArrayVec` 是比 `SmallVec` 更好的选择。它不需要回退到堆分配，这使其稍微快一些。
[**示例**](https://github.com/rust-lang/rust/pull/74310/commits/c492ca40a288d8a85353ba112c4d38fe87ef453e).

[`arrayvec`]: https://crates.io/crates/arrayvec

### 更长的 `Vec`

如果你知道向量的最小或确切大小，可以使用 [`Vec::with_capacity`]、[`Vec::reserve`] 或
[`Vec::reserve_exact`] 预留特定容量。例如，如果你知道向量将增长到至少 20 个元素，
这些函数可以通过一次分配立即提供一个容量至少为 20 的向量，而逐个推送元素将导致四次
分配（容量分别为 4、8、16 和 32）。
[**示例**](https://github.com/rust-lang/rust/pull/77990/commits/a7f2bb634308a5f05f2af716482b67ba43701681).

[`Vec::with_capacity`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.with_capacity
[`Vec::reserve`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.reserve
[`Vec::reserve_exact`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.reserve_exact

如果你知道向量的最大长度，上述函数还可以让你避免不必要地分配多余空间。类似地，
[`Vec::shrink_to_fit`] 可用于最小化浪费的空间，但请注意它可能导致重新分配。

[`Vec::shrink_to_fit`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.shrink_to_fit

## `String`

[`String`] 包含堆分配的字节。`String` 的表示和操作与 `Vec<u8>` 非常相似。许多与增长
和容量相关的 `Vec` 方法在 `String` 中都有对应方法，例如 [`String::with_capacity`]。

[`String`]: https://doc.rust-lang.org/std/string/struct.String.html
[`String::with_capacity`]: https://doc.rust-lang.org/std/string/struct.String.html#method.with_capacity

[`smallstr`] crate 中的 `SmallString` 类型类似于 `SmallVec` 类型。

[`smallstr`]: https://crates.io/crates/smallstr

[`smartstring`] crate 中的 `String` 类型是 `String` 的直接替代品，对于少于三个字段
大小的字符的字符串，它避免堆分配。在 64 位平台上，这是任何小于 24 字节的字符串，
包括包含 23 个或更少 ASCII 字符的所有字符串。
[**示例**](https://github.com/djc/topfew-rs/commit/803fd566e9b889b7ba452a2a294a3e4df76e6c4c).

[`smartstring`]: https://crates.io/crates/smartstring

请注意，`format!` 宏会生成一个 `String`，这意味着它执行一次分配。如果你可以使用
字符串字面量来避免 `format!` 调用，就可以避免这次分配。
[**示例**](https://github.com/rust-lang/rust/pull/55905/commits/c6862992d947331cd6556f765f6efbde0a709cf9).
[`std::format_args`] 和/或 [`lazy_format`] crate 可能对此有所帮助。

[`std::format_args`]: https://doc.rust-lang.org/std/macro.format_args.html
[`lazy_format`]: https://crates.io/crates/lazy_format

## 哈希表

[`HashSet`] 和 [`HashMap`] 是哈希表。它们在分配方面的表示和操作与 `Vec` 类似：
它们有一个连续的堆分配，用于保存键和值，并在表增长时根据需要重新分配。许多与增长
和容量相关的 `Vec` 方法在 `HashSet`/`HashMap` 中都有对应方法，例如
[`HashSet::with_capacity`]。

[`HashSet`]: https://doc.rust-lang.org/std/collections/struct.HashSet.html
[`HashMap`]: https://doc.rust-lang.org/std/collections/struct.HashMap.html
[`HashSet::with_capacity`]: https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.with_capacity

## `clone`

对包含堆分配内存的值调用 [`clone`] 通常涉及额外的分配。例如，对非空 `Vec` 调用
`clone` 需要为元素进行新的分配（但请注意，新 `Vec` 的容量可能与原始 `Vec` 的容量
不同）。例外情况是 `Rc`/`Arc`，其中 `clone` 调用仅增加引用计数。

[`clone`]: https://doc.rust-lang.org/std/clone/trait.Clone.html#tymethod.clone

[`clone_from`] 是 `clone` 的替代方案。`a.clone_from(&b)` 等同于 `a = b.clone()`，
但可能避免不必要的分配。例如，如果你想在现有 `Vec` 之上克隆另一个 `Vec`，现有 `Vec`
的堆分配将尽可能被重用，如下例所示。
```rust
let mut v1: Vec<u32> = Vec::with_capacity(99);
let v2: Vec<u32> = vec![1, 2, 3];
v1.clone_from(&v2); // v1 的分配被重用
assert_eq!(v1.capacity(), 99);
```
尽管 `clone` 通常会引起分配，但在许多情况下使用它是合理的，并且通常可以使代码更简单。
使用性能分析数据来确定哪些 `clone` 调用是热点，值得花精力避免。

[`clone_from`]: https://doc.rust-lang.org/std/clone/trait.Clone.html#method.clone_from

有时 Rust 代码最终包含不必要的 `clone` 调用，原因可能是 (a) 程序员错误，或 (b) 代码
变更使得以前必要的 `clone` 调用变得不必要。如果你看到一个看似不必要的热点 `clone`
调用，有时可以简单地将其移除。
[**示例 1**](https://github.com/rust-lang/rust/pull/37318/commits/e382267cfb9133ef12d59b66a2935ee45b546a61),
[**示例 2**](https://github.com/rust-lang/rust/pull/37705/commits/11c1126688bab32f76dbe1a973906c7586da143f),
[**示例 3**](https://github.com/rust-lang/rust/pull/64302/commits/36b37e22de92b584b9cf4464ed1d4ad317b798be).

## `to_owned`

[`ToOwned::to_owned`] 为许多常见类型实现。它从借用的数据创建拥有的数据，通常通过
克隆，因此经常导致堆分配。例如，它可以用于从 `&str` 创建 `String`。

[`ToOwned::to_owned`]: https://doc.rust-lang.org/std/borrow/trait.ToOwned.html#tymethod.to_owned

有时可以通过在 struct 中存储对借用数据的引用而不是拥有副本来避免 `to_owned` 调用
（以及相关的调用如 `clone` 和 `to_string`）。这需要在 struct 上添加 lifetime 标注，
使代码变得复杂，只有在性能分析和基准测试表明值得时才应进行。
[**示例**](https://github.com/rust-lang/rust/pull/50855/commits/6872377357dbbf373cfd2aae352cb74cfcc66f34).

## `Cow`

有时代码处理的是借用数据和拥有数据的混合。设想一个错误消息的向量，其中一些是静态
字符串字面量，另一些是用 `format!` 构建的。显而易见的表示是 `Vec<String>`，如下例
所示。
```rust
let mut errors: Vec<String> = vec![];
errors.push("something went wrong".to_string());
errors.push(format!("something went wrong on line {}", 100));
```
这需要调用 `to_string` 将静态字符串字面量提升为 `String`，这会产生一次分配。

相反，你可以使用 [`Cow`] 类型，它可以保存借用或拥有的数据。借用值 `x` 用
`Cow::Borrowed(x)` 包装，拥有值 `y` 用 `Cow::Owned(y)` 包装。`Cow` 还为各种字符串、
切片和路径类型实现了 `From<T>` trait，因此你通常也可以使用 `into`。（或者 `Cow::from`，
它更长但使代码更具可读性，因为类型更清晰。）以下示例综合了所有这些。

[`Cow`]: https://doc.rust-lang.org/std/borrow/enum.Cow.html

```rust
use std::borrow::Cow;
let mut errors: Vec<Cow<'static, str>> = vec![];
errors.push(Cow::Borrowed("something went wrong"));
errors.push(Cow::Owned(format!("something went wrong on line {}", 100)));
errors.push(Cow::from("something else went wrong"));
errors.push(format!("something else went wrong on line {}", 101).into());
```
`errors` 现在保存了借用和拥有数据的混合，无需任何额外分配。此示例涉及 `&str`/`String`，
但其他配对如 `&[T]`/`Vec<T>` 和 `&Path`/`PathBuf` 也是可行的。

[**示例 1**](https://github.com/rust-lang/rust/pull/37064/commits/b043e11de2eb2c60f7bfec5e15960f537b229e20),
[**示例 2**](https://github.com/rust-lang/rust/pull/56336/commits/787959c20d062d396b97a5566e0a766d963af022).

上述所有内容适用于数据不可变的情况。但 `Cow` 也允许在需要修改时将借用数据提升为
拥有数据。[`Cow::to_mut`] 将获取拥有值的可变引用，必要时进行克隆。这称为"写时复制"
（clone-on-write），也就是 `Cow` 名称的由来。

[`Deref`]: https://doc.rust-lang.org/std/ops/trait.Deref.html
[`Cow::to_mut`]: https://doc.rust-lang.org/std/borrow/enum.Cow.html#method.to_mut

这种写时复制行为在你有一些主要是只读但偶尔需要修改的借用数据（如 `&str`）时很有用。

[**示例 1**](https://github.com/rust-lang/rust/pull/50855/commits/ad471452ba6fbbf91ad566dc4bdf1033a7281811),
[**示例 2**](https://github.com/rust-lang/rust/pull/68848/commits/67da45f5084f98eeb20cc6022d68788510dc832a).

最后，因为 `Cow` 实现了 [`Deref`]，你可以直接对其包含的数据调用方法。

`Cow` 使用起来可能有点棘手，但通常值得付出努力。

## 重用集合

有时你需要分阶段构建如 `Vec` 这样的集合。通常修改单个 `Vec` 比构建多个 `Vec` 然后
合并它们更好。

例如，如果你有一个可能被多次调用的函数 `do_stuff`，它生成一个 `Vec`：
```rust
fn do_stuff(x: u32, y: u32) -> Vec<u32> {
    vec![x, y]
}
```
更好的做法是修改传入的 `Vec`：
```rust
fn do_stuff(x: u32, y: u32, vec: &mut Vec<u32>) {
    vec.push(x);
    vec.push(y);
}
```
有时值得保留一个可重用的"主力"集合。例如，如果每次循环迭代都需要一个 `Vec`，
你可以在循环外声明 `Vec`，在循环体内使用它，然后在循环体末尾调用 [`clear`]（清空
`Vec` 而不影响其容量）。这避免了分配，代价是掩盖了每次迭代对 `Vec` 的使用与其他
迭代无关的事实。
[**示例 1**](https://github.com/rust-lang/rust/pull/77990/commits/45faeb43aecdc98c9e3f2b24edf2ecc71f39d323),
[**示例 2**](https://github.com/rust-lang/rust/pull/51870/commits/b0c78120e3ecae5f4043781f7a3f79e2277293e7).

[`clear`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.clear

类似地，有时值得在 struct 中保留一个主力集合，以便在被重复调用的一个或多个方法中
重用。

## 从文件读取行

[`BufRead::lines`] 使得逐行读取文件变得容易：
```rust
# fn blah() -> Result<(), std::io::Error> {
# fn process(_: &str) {}
use std::io::{self, BufRead};
let mut lock = io::stdin().lock();
for line in lock.lines() {
    process(&line?);
}
# Ok(())
# }
```
但它生成的迭代器返回 `io::Result<String>`，这意味着它会为文件中的每一行分配内存。

[`BufRead::lines`]: https://doc.rust-lang.org/stable/std/io/trait.BufRead.html#method.lines

另一种方法是使用一个主力 `String` 配合 [`BufRead::read_line`] 循环：
```rust
# fn blah() -> Result<(), std::io::Error> {
# fn process(_: &str) {}
use std::io::{self, BufRead};
let mut lock = io::stdin().lock();
let mut line = String::new();
while lock.read_line(&mut line)? != 0 {
    process(&line);
    line.clear();
}
# Ok(())
# }
```
这将分配次数减少到最多几次，可能仅需一次。（确切次数取决于 `line` 需要重新分配
的次数，这取决于文件中行长的分布。）

这仅在循环体可以操作 `&str` 而非 `String` 时才有效。

[`BufRead::read_line`]: https://doc.rust-lang.org/stable/std/io/trait.BufRead.html#method.read_line

[**示例**](https://github.com/nnethercote/counts/commit/7d39bbb1867720ef3b9799fee739cd717ad1539a).

## 使用替代分配器

也可以在不更改代码的情况下改善堆分配性能，只需使用不同的分配器即可。有关详细信息，
请参阅[替代分配器]一节。

[替代分配器]: build-configuration.md#alternative-allocators

## 避免回归

为确保代码执行的分配数量和/或大小不会意外增加，你可以使用 [dhat-rs] 的*堆使用测试*
功能来编写测试，检查特定代码片段分配了预期的堆内存量。

[dhat-rs]: https://crates.io/crates/dhat
