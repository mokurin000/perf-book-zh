# 哈希

`HashSet` 和 `HashMap` 是两种广泛使用的类型，有一些方法可以让它们更快。

## 替代哈希器

默认的哈希算法未指定，但在撰写本文时，默认算法是一种名为 [SipHash 1-3] 的算法。
这种算法质量很高——提供了很高的碰撞防护能力——但相对较慢，特别是对于整数这样的短键。

[SipHash 1-3]: https://en.wikipedia.org/wiki/SipHash

如果性能分析表明哈希是热点，且 [HashDoS 攻击]对你的应用不是问题，那么使用具有更快
哈希算法的哈希表可以带来巨大的速度提升。
- [`rustc-hash`] 提供了 `FxHashSet` 和 `FxHashMap` 类型，可作为 `HashSet` 和
  `HashMap` 的直接替代品。它的哈希算法质量较低但速度非常快，尤其适用于整数键，
  并且已被发现在 rustc 中的性能优于所有其他哈希算法。（[`fxhash`] 是相同算法和类型的
  较旧、维护较少的实现。）
- [`fnv`] 提供了 `FnvHashSet` 和 `FnvHashMap` 类型。它的哈希算法质量比 `rustc-hash`
  更高，但稍慢一些。
- [`ahash`] 提供了 `AHashSet` 和 `AHashMap`。它的哈希算法可以利用某些处理器上可用的
  AES 指令支持。

[HashDoS 攻击]: https://en.wikipedia.org/wiki/Collision_attack
[`rustc-hash`]: https://crates.io/crates/rustc-hash
[`fxhash`]: https://crates.io/crates/fxhash
[`fnv`]: https://crates.io/crates/fnv
[`ahash`]: https://crates.io/crates/ahash

如果哈希性能在程序中很重要，值得尝试多种替代方案。例如，在 rustc 中观察到了以下结果。
- 从 `fnv` 切换到 `fxhash` 带来了[高达 6% 的速度提升][fnv2fx]。
- 尝试从 `fxhash` 切换到 `ahash` 导致了[1-4% 的减速][fx2a]。
- 尝试从 `fxhash` 切换回默认哈希器导致了[4-84% 的减速][fx2default]！

[fnv2fx]: https://github.com/rust-lang/rust/pull/37229/commits/00e48affde2d349e3b3bfbd3d0f6afb5d76282a7
[fx2a]: https://github.com/rust-lang/rust/issues/69153#issuecomment-589504301
[fx2default]: https://github.com/rust-lang/rust/issues/69153#issuecomment-589338446

如果你决定普遍使用某种替代方案，例如 `FxHashSet`/`FxHashMap`，很容易在某些地方
意外地使用 `HashSet`/`HashMap`。你可以[使用 Clippy] 来避免这个问题。

[使用 Clippy]: linting.md#disallowing-types

有些类型不需要哈希。例如，你可能有一个包装整数的 newtype，且整数值是随机的或接近随机。
对于此类类型，哈希值的分布与值本身的分布不会有太大差异。在这种情况下，
[`nohash_hasher`] crate 可能会很有用。

[`nohash_hasher`]: https://crates.io/crates/nohash-hasher

哈希函数设计是一个复杂的主题，已超出本书的范围。[`ahash` 文档]中有很好的讨论。

[`ahash` 文档]: https://github.com/tkaitchuck/aHash/blob/master/compare/readme.md

## 逐字节哈希

当你用 `#[derive(Hash)]` 注解一个类型时，生成的 `hash` 方法将分别哈希每个字段。
对于某些哈希函数，将类型转换为原始字节并将字节作为流进行哈希可能会更快。这对于满足
某些属性（例如没有填充字节）的类型是可行的。

[`zerocopy`] 和 [`bytemuck`] crate 都提供了 `#[derive(ByteHash)]` 宏，用于生成
执行这种逐字节哈希的 `hash` 方法。[`derive_hash_fast`] crate 的 README 提供了
关于此技术的更多细节。

[`zerocopy`]: https://crates.io/crates/zerocopy
[`bytemuck`]: https://crates.io/crates/bytemuck
[`derive_hash_fast`]: https://crates.io/crates/derive_hash_fast

这是一项高级技术，性能效果高度依赖于哈希函数和被哈希类型的具体结构。请仔细测量。
