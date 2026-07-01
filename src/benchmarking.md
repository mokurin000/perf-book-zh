# 基准测试

基准测试通常涉及比较执行相同任务的两个或多个程序的性能。有时这可能需要比较两个或
更多不同的程序，例如 Firefox vs Safari vs Chrome。有时涉及比较同一程序的两个不同版本。
后一种情况让我们能够可靠地回答"这个改动是否加快了速度？"这个问题。

基准测试是一个复杂的话题，全面覆盖已超出本书的范围，但以下是基本要点。

首先，你需要用于测量的工作负载。理想情况下，你应该有各种能够代表程序实际使用情况的
工作负载。使用真实世界输入的工作负载最好，但[微基准测试]和[压力测试]适度使用也很有用。

[微基准测试]: https://stackoverflow.com/questions/2842695/what-is-microbenchmarking
[压力测试]: https://en.wikipedia.org/wiki/Stress_testing_(software)

其次，你需要一种运行工作负载的方法，这也将决定所使用的指标。
- Rust 内置的[基准测试]是一个简单的起点，但它们使用了不稳定的特性，因此仅适用于
  nightly Rust。
- [Criterion] 和 [Divan] 是更复杂的替代方案。
- [Hyperfine] 是一个优秀的通用基准测试工具。
- [Bencher] 可以在 CI（包括 GitHub CI）上进行持续基准测试。
- [Gungraun] 提供了带有高精度测量的 `cargo bench` 集成。
- 也可以使用自定义的基准测试框架。例如，[rustc-perf] 就是用于对 Rust 编译器进行
  基准测试的框架。

[基准测试]: https://doc.rust-lang.org/nightly/unstable-book/library-features/test.html
[Criterion]: https://github.com/bheisler/criterion.rs
[Divan]: https://github.com/nvzqz/divan
[Hyperfine]: https://github.com/sharkdp/hyperfine
[Bencher]: https://github.com/bencherdev/bencher
[Gungraun]: https://github.com/gungraun/gungraun
[rustc-perf]: https://github.com/rust-lang/rustc-perf/

至于指标，有很多选择，正确的指标取决于被基准测试的程序的性质。例如，对批处理程序
有意义的指标可能对交互式程序没有意义。墙钟时间在许多情况下是一个明显的选择，因为它
与用户感知一致。然而，它可能存在高方差的问题。特别是，内存布局的微小变化可能导致
显著但短暂的性能波动。因此，其他方差较低的指标（如周期数或指令计数）可能是合理的
替代方案。

汇总来自多个工作负载的测量结果也是一个挑战，有多种方法可以做到，但没有一种方法
明显优于其他。

好的基准测试是困难的。话虽如此，不要过于追求完美的基准测试设置，特别是在你开始
优化程序时。平庸的基准测试远胜于没有基准测试。对你正在测量的内容保持开放的心态，
随着你了解程序的性能特征，可以逐步改进基准测试。
