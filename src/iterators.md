# 迭代器

## `collect` 和 `extend`

[`Iterator::collect`] 将迭代器转换为集合（如 `Vec`），这通常需要一次分配。如果之后
只是再次迭代该集合，则应避免调用 `collect`。

[`Iterator::collect`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect

因此，从函数返回 `impl Iterator<Item=T>` 这样的迭代器类型通常比返回 `Vec<T>` 更好。
请注意，如[这篇博客文章]所述，这些返回类型有时需要额外的 lifetime 标注。
[**示例**](https://github.com/rust-lang/rust/pull/77990/commits/660d8a6550a126797aa66a417137e39a5639451b).

[这篇博客文章]: https://blog.katona.me/2019/12/29/Rust-Lifetimes-and-Iterators/

类似地，你可以使用 [`extend`] 用迭代器扩展现有集合（如 `Vec`），而不是将迭代器收集到
`Vec` 中再使用 [`append`]。

[`extend`]: https://doc.rust-lang.org/std/iter/trait.Extend.html#tymethod.extend
[`append`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.append

最后，在编写迭代器时，如果可能的话，实现 [`Iterator::size_hint`] 或
[`ExactSizeIterator::len`] 方法通常是有价值的。使用该迭代器的 `collect` 和 `extend`
调用可能会进行更少的分配，因为它们预先知道了迭代器产生的元素数量。

[`Iterator::size_hint`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.size_hint
[`ExactSizeIterator::len`]: https://doc.rust-lang.org/std/iter/trait.ExactSizeIterator.html#method.len

## 链式操作

[`chain`] 非常方便，但也可能比单个迭代器慢。对于热门的迭代器，如果可能的话，
最好避免使用。
[**示例**](https://github.com/rust-lang/rust/pull/64801/commits/5ca99b750e455e9b5e13e83d0d7886486231e48a).

类似地，[`filter_map`] 可能比先使用 [`filter`] 再使用 [`map`] 更快。

[`chain`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.chain
[`filter_map`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter_map
[`filter`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter
[`map`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map

## 块迭代

当需要块迭代器且已知块大小正好能整除切片长度时，请使用更快的 [`slice::chunks_exact`]
而不是 [`slice::chunks`]。

当不确定块大小是否能正好整除切片长度时，使用 `slice::chunks_exact` 并结合
[`ChunksExact::remainder`] 或手动处理多余元素仍然可能更快。
[**示例 1**](https://github.com/johannesvollmer/exrs/pull/173/files),
[**示例 2**](https://github.com/johannesvollmer/exrs/pull/175/files).

同样的情况也适用于相关的迭代器：
- [`slice::rchunks`]、[`slice::rchunks_exact`] 和 [`RChunksExact::remainder`]；
- [`slice::chunks_mut`]、[`slice::chunks_exact_mut`] 和 [`ChunksExactMut::into_remainder`]；
- [`slice::rchunks_mut`]、[`slice::rchunks_exact_mut`] 和 [`RChunksExactMut::into_remainder`].

[`slice::chunks`]: https://doc.rust-lang.org/stable/std/primitive.slice.html#method.chunks
[`slice::chunks_exact`]: https://doc.rust-lang.org/stable/std/primitive.slice.html#method.chunks_exact
[`ChunksExact::remainder`]: https://doc.rust-lang.org/stable/std/slice/struct.ChunksExact.html#method.remainder

[`slice::rchunks`]: https://doc.rust-lang.org/stable/std/primitive.slice.html#method.rchunks
[`slice::rchunks_exact`]: https://doc.rust-lang.org/stable/std/primitive.slice.html#method.rchunks_exact
[`RChunksExact::remainder`]: https://doc.rust-lang.org/stable/std/slice/struct.RChunksExact.html#method.remainder

[`slice::chunks_mut`]: https://doc.rust-lang.org/stable/std/primitive.slice.html#method.chunks_mut
[`slice::chunks_exact_mut`]: https://doc.rust-lang.org/stable/std/primitive.slice.html#method.chunks_exact_mut
[`ChunksExactMut::into_remainder`]: https://doc.rust-lang.org/stable/std/slice/struct.ChunksExactMut.html#method.into_remainder

[`slice::rchunks_mut`]: https://doc.rust-lang.org/stable/std/primitive.slice.html#method.rchunks_mut
[`slice::rchunks_exact_mut`]: https://doc.rust-lang.org/stable/std/primitive.slice.html#method.rchunks_exact_mut
[`RChunksExactMut::into_remainder`]: https://doc.rust-lang.org/stable/std/slice/struct.RChunksExactMut.html#method.into_remainder

## `copied`

当迭代整数等小型数据类型的集合时，使用 `iter().copied()` 可能比 `iter()` 更好。
消费该迭代器的代码将按值而非按引用接收整数，LLVM 在这种情况下可能生成更好的代码。
[**示例 1**](https://github.com/rust-lang/rust/issues/106539),
[**示例 2**](https://github.com/rust-lang/rust/issues/113789).

这是一项高级技术。你可能需要检查生成的机器码以确定它是否有效。有关如何执行此操作的
详细信息，请参阅[机器码](machine-code.md)章节。
