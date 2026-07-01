# 并行

Rust 为安全的并行编程提供了出色的支持，这可以带来巨大的性能提升。有多种方式可以
在程序中引入并行，而最佳方式在很大程度上取决于程序的设计。

话虽如此，对并行的深入讨论已超出本书的范围。

如果你对基于线程的并行感兴趣，[`rayon`] 和 [`crossbeam`] crate 的文档是一个很好的
起点。[Rust Atomics and Locks][Atomics] 也是一份优秀的资源。

[`rayon`]: https://crates.io/crates/rayon
[`crossbeam`]: https://crates.io/crates/crossbeam
[Atomics]: https://marabos.nl/atomics/

如果你对细粒度的数据并行感兴趣，这篇[博客文章]对截至 2025 年 11 月 Rust 中
SIMD 支持的状况进行了很好的概述。

[博客文章]: https://shnatsel.medium.com/the-state-of-simd-in-rust-in-2025-32c263e5f53d

