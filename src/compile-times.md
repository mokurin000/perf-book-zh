# 编译时间

虽然本书主要讨论改善 Rust 程序的性能，但本节讨论的是减少 Rust 程序的编译时间，
因为这是许多人感兴趣的相关话题。

[最小化编译时间]一节讨论了通过构建配置选择来减少编译时间的方法。本节其余部分讨论
需要修改程序代码来减少编译时间的方法。

[最小化编译时间]: build-configuration.md#minimizing-compile-times

有关更多编译时间缩减技术，请参阅 Corrode 的全面列表：[更快 Rust 编译时间的技巧][Tips]。

[Tips]: https://corrode.dev/blog/tips-for-faster-rust-compile-times/

## 可视化

Cargo 有一个功能可以让你可视化程序的编译过程。使用以下命令构建：
```text
cargo build --timings
```
完成后，它将打印一个 HTML 文件的名称。在 Web 浏览器中打开该文件。它包含一个
[甘特图]，显示程序中各个 crate 之间的依赖关系。这可以显示你的 crate 图中有多少
并行度，从而指示是否有任何使编译串行化的大型 crate 应该被拆分。有关如何阅读图表的
更多详细信息，请参阅[文档][timings]。

[甘特图]: https://en.wikipedia.org/wiki/Gantt_chart
[timings]: https://doc.rust-lang.org/nightly/cargo/reference/timings.html

## 宏

有些宏会生成大量代码。这些代码随后需要时间来编译。Rust 编译器的 `-Zmacro-stats` 标志
可以帮助识别此类情况。

例如，如果你只想测量项目中的一个叶子 crate：
```text
cargo +nightly rustc -- -Zmacro-stats
```
编译器将打印由过程宏和声明式宏生成的代码量的信息。前者通常更值得关注。

或者，如果你想测量项目中的所有 crate：
```text
RUSTFLAGS="-Zmacro-stats" cargo +nightly build
```
要查看生成的代码本身，可以使用 [cargo-expand]。

[cargo-expand]: https://github.com/dtolnay/cargo-expand

不必担心产生少量代码的宏，但如果宏生成的代码量可与手写代码量相当，则可能可以完全
移除该宏的使用，或用更廉价的替代方案替换。
[**示例**](https://nnethercote.github.io/2025/06/26/how-much-code-does-that-proc-macro-generate.html).

或者，也可以修改宏以生成更少的代码。
[**示例 1**](https://github.com/bevyengine/bevy/issues/19873),
[**示例 2**](https://nnethercote.github.io/2025/08/16/speed-wins-when-fuzzing-rust-code-with-derive-arbitrary.html).

## LLVM IR

Rust 编译器使用 [LLVM] 作为其后端。LLVM 的执行可能占据编译时间的很大一部分，
特别是当 Rust 编译器的前端生成大量需要 LLVM 花费长时间优化的 [IR] 时。

[LLVM]: https://llvm.org/
[IR]: https://en.wikipedia.org/wiki/Intermediate_representation

这些问题可以通过 [`cargo llvm-lines`] 来诊断，它显示哪些 Rust 函数导致生成了最多的
LLVM IR。泛型函数通常是最重要的，因为它们可以在大型程序中实例化数十次甚至数百次。

[`cargo llvm-lines`]: https://github.com/dtolnay/cargo-llvm-lines/

如果泛型函数导致 IR 膨胀，有几种修复方法。最简单的方法是让函数更小。
[**示例 1**](https://github.com/rust-lang/rust/pull/72166/commits/5a0ac0552e05c079f252482cfcdaab3c4b39d614),
[**示例 2**](https://github.com/rust-lang/rust/pull/91246/commits/f3bda74d363a060ade5e5caeb654ba59bfed51a4).

另一种方法是将函数的非泛型部分移到一个单独的非泛型函数中，该函数将只被实例化一次。
这是否可行取决于泛型函数的具体情况。当可行时，非泛型函数通常可以整洁地编写为泛型函数
内部的函数，如 [`std::fs::read`] 的代码所示：
```rust,ignore
pub fn read<P: AsRef<Path>>(path: P) -> io::Result<Vec<u8>> {
    fn inner(path: &Path) -> io::Result<Vec<u8>> {
        let mut file = File::open(path)?;
        let size = file.metadata().map(|m| m.len()).unwrap_or(0);
        let mut bytes = Vec::with_capacity(size as usize);
        io::default_read_to_end(&mut file, &mut bytes)?;
        Ok(bytes)
    }
    inner(path.as_ref())
}
```
[`std::fs::read`]: https://doc.rust-lang.org/std/fs/fn.read.html

[**示例**](https://github.com/rust-lang/rust/pull/72013/commits/68b75033ad78d88872450a81745cacfc11e58178).

有时像 [`Option::map`] 和 [`Result::map_err`] 这样的通用工具函数会被实例化多次。
将它们替换为等价的 `match` 表达式有助于编译时间。

[`Option::map`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.map
[`Result::map_err`]: https://doc.rust-lang.org/std/result/enum.Result.html#method.map_err

这类更改对编译时间的影响通常很小，但偶尔也可能很大。
[**示例**](https://github.com/servo/servo/issues/26585).

此类更改还可以减小二进制体积。
