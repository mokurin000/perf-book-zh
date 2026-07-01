# 通用技巧

本书前面的章节讨论了 Rust 特有的技术。本节简要概述一些通用的性能原则。

只要避免了明显的陷阱（例如[使用非 release 构建]），Rust 代码通常运行速度快且内存占用小。
特别是如果你习惯于 Python 和 Ruby 等动态类型语言，或 Java 和 C# 等带有垃圾收集器的
静态类型语言时，这一点尤为明显。

[使用非 release 构建]: build-configuration.md

优化后的代码通常比未优化的代码更复杂，编写起来也更费力。因此，只值得优化热代码。

最大的性能提升通常来自算法或数据结构的改变，而非底层优化。
[**示例 1**](https://github.com/rust-lang/rust/pull/53383/commits/5745597e6195fe0591737f242d02350001b6c590),
[**示例 2**](https://github.com/rust-lang/rust/pull/54318/commits/154be2c98cf348de080ce951df3f73649e8bb1a6).

编写与现代硬件配合良好的代码并不总是容易的，但值得努力。例如，尽可能减少缓存未命中
和分支预测错误。

大多数优化带来的速度提升都很小。虽然单个小的速度提升并不明显，但如果你能做足够的优化，
它们累积起来的效果就很可观。

不同的分析器各有其优势。使用多种分析器是好的做法。

当性能分析表明某个函数是热点时，通常有两种加速方法：(a) 让函数运行得更快，和/或
(b) 减少对它的调用。

消除愚蠢的减速通常比引入巧妙的加速更容易。

除非必要，避免计算。惰性/按需计算通常是一个胜利。
[**示例 1**](https://github.com/rust-lang/rust/pull/36592/commits/80a44779f7a211e075da9ed0ff2763afa00f43dc),
[**示例 2**](https://github.com/rust-lang/rust/pull/50339/commits/989815d5670826078d9984a3515eeb68235a4687).

复杂的通用情况通常可以通过乐观地检查更简单的常见特殊情况来避免。
[**示例 1**](https://github.com/rust-lang/rust/pull/68790/commits/d62b6f204733d255a3e943388ba99f14b053bf4a),
[**示例 2**](https://github.com/rust-lang/rust/pull/53733/commits/130e55665f8c9f078dec67a3e92467853f400250),
[**示例 3**](https://github.com/rust-lang/rust/pull/65260/commits/59e41edcc15ed07de604c61876ea091900f73649).
特别地，当小规模占主导时，专门处理包含 0、1 或 2 个元素的集合通常是一个胜利。
[**示例 1**](https://github.com/rust-lang/rust/pull/50932/commits/2ff632484cd8c2e3b123fbf52d9dd39b54a94505),
[**示例 2**](https://github.com/rust-lang/rust/pull/64627/commits/acf7d4dcdba4046917c61aab141c1dec25669ce9),
[**示例 3**](https://github.com/rust-lang/rust/pull/64949/commits/14192607d38f5501c75abea7a4a0e46349df5b5f),
[**示例 4**](https://github.com/rust-lang/rust/pull/64949/commits/d1a7bb36ad0a5932384eac03d3fb834efc0317e5).

类似地，处理重复数据时，通常可以使用一种简单的数据压缩形式，即为常见值使用紧凑表示，
然后对不常见的值回退到辅助表。
[**示例 1**](https://github.com/rust-lang/rust/pull/54420/commits/b2f25e3c38ff29eebe6c8ce69b8c69243faa440d),
[**示例 2**](https://github.com/rust-lang/rust/pull/59693/commits/fd7f605365b27bfdd3cd6763124e81bddd61dd28),
[**示例 3**](https://github.com/rust-lang/rust/pull/65750/commits/eea6f23a0ed67fd8c6b8e1b02cda3628fee56b2f).

当代码处理多种情况时，测量各情况的频率并优先处理最常见的情况。

当处理具有高局部性的查找时，在数据结构前放置一个小型缓存可能是一个胜利。

优化后的代码通常具有非显而易见的结构，这意味着解释性注释非常有价值，特别是那些引用
性能分析测量的注释。像"99% 的情况下这个向量有 0 或 1 个元素，所以先处理这些情况"
这样的注释可以很有启发性。
