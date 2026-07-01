# 构建配置

你可以大幅改变 Rust 程序的性能，而无需修改代码，只需更改其构建配置。每个 Rust 程序
都有许多可能的构建配置。所选配置将影响编译后代码的多个特征，例如编译时间、运行时速度、
内存使用、二进制体积、可调试性、可分析性以及编译后的程序将运行的架构。

大多数配置选择会在改善一个或多个特征的同时，使一个或多个其他特征变差。例如，一种常见
的权衡是接受更长的编译时间以换取更高的运行时速度。适合你程序的正确选择取决于你的需求
和程序的具体情况，而与性能相关的选择（大部分都是）应通过基准测试来验证。

值得仔细阅读本章以了解所有构建配置选项。不过，对于没有耐心或容易忘记的人来说，
[`cargo-wizard`] 封装了这些信息，可以帮助你选择合适的构建配置。

请注意，Cargo 仅查看工作区根目录下 `Cargo.toml` 文件中的 profile 设置。依赖项中定义的
profile 设置将被忽略。因此，这些选项主要与二进制 crate 相关，而非库 crate。

[`cargo-wizard`]: https://github.com/Kobzol/cargo-wizard

## Release 构建

单个最重要的构建配置选择很简单但[容易忽视]：当你需要高性能时，请确保使用的是
[release 构建]而非 [dev 构建]。这通常通过向 Cargo 指定 `--release` 标志来完成。

[容易忽视]: https://users.rust-lang.org/t/why-my-rust-program-is-so-slow/47764/5
[release 构建]: https://doc.rust-lang.org/cargo/reference/profiles.html#release
[dev 构建]: https://doc.rust-lang.org/cargo/reference/profiles.html#dev

Dev 构建是默认的。它们适合调试，但未进行优化。如果你运行 `cargo build` 或 `cargo run`，
就会生成 dev 构建。（或者，不带额外选项运行 `rustc` 也会生成未优化的构建。）

考虑以下 `cargo build` 运行的最终输出行。
```text
Finished dev [unoptimized + debuginfo] target(s) in 29.80s
```
此输出表明已生成 dev 构建。编译后的代码将放置在 `target/debug/` 目录中。`cargo run`
将运行 dev 构建。

相比之下，release 构建经过了更多优化，省略了调试断言和整数溢出检查，并省略了调试信息。
比 dev 构建快 10-100 倍是常见的！如果你运行 `cargo build --release` 或
`cargo run --release`，就会生成 release 构建。（或者，`rustc` 有多个优化构建选项，
如 `-O` 和 `-C opt-level`。）由于额外的优化，这通常比 dev 构建需要更长的时间。

考虑以下 `cargo build --release` 运行的最终输出行。
```text
Finished release [optimized] target(s) in 1m 01s
```
此输出表明已生成 release 构建。编译后的代码将放置在 `target/release/` 目录中。
`cargo run --release` 将运行 release 构建。

有关 dev 构建（使用 `dev` profile）和 release 构建（使用 `release` profile）之间差异的
更多详细信息，请参阅 [Cargo profile 文档]。

[Cargo profile 文档]: https://doc.rust-lang.org/cargo/reference/profiles.html

Release 构建中使用的默认构建配置选项在编译时间、运行时速度和二进制体积等上述特征之间
提供了良好的平衡。但有许多可能的调整，如下节所述。

## 最大化运行时速度

以下构建配置选项主要旨在最大化运行时速度。其中一些还可能减小二进制体积。

### Codegen Units

Rust 编译器将 crate 拆分为多个 [codegen units] 以并行化（从而加快）编译。然而，这
可能导致编译器错过一些潜在的优化。通过将单元数设置为 1，你可能能够提高运行时速度并
减小二进制体积，但代价是增加编译时间。将以下行添加到 `Cargo.toml` 文件中：
```toml
[profile.release]
codegen-units = 1
```
<!-- Using `https` for this link triggers "potential security risk" warnings due
to a certificate problem. -->
[**示例 1**](http://likebike.com/posts/How_To_Write_Fast_Rust_Code.html#emit-asm),
[**示例 2**](https://github.com/rust-lang/rust/pull/115554#issuecomment-1742192440).

[codegen units]: https://doc.rust-lang.org/cargo/reference/profiles.html#codegen-units

### 链接时优化

[链接时优化]（LTO）是一种全程序优化技术，可以将运行时速度提高 10-20% 或更多，同时
还能减小二进制体积，但代价是编译时间变长。它有几种形式。

[链接时优化]: https://doc.rust-lang.org/cargo/reference/profiles.html#lto

第一种形式的 LTO 是 *thin local LTO*，一种轻量级的 LTO 形式。默认情况下，编译器对任何
涉及非零优化级别的构建都使用此方式。这包括 release 构建。要显式请求此级别的 LTO，
请在 `Cargo.toml` 文件中添加以下行：
```toml
[profile.release]
lto = false
```

第二种形式的 LTO 是 *thin LTO*，它更激进一些，很可能提高运行时速度并减小二进制体积，
同时增加编译时间。在 `Cargo.toml` 中使用 `lto = "thin"` 来启用它。

第三种形式的 LTO 是 *fat LTO*，它更加激进，可能进一步提高性能并进一步减小二进制体积
（但[并非总是如此]），同时再次增加构建时间。在 `Cargo.toml` 中使用 `lto = "fat"` 来
启用它。

[并非总是如此]: https://github.com/rust-lang/rust/pull/103453

最后，可以完全禁用 LTO，这可能会降低运行时速度并增加二进制体积，但会减少编译时间。
在 `Cargo.toml` 中使用 `lto = "off"`。请注意，这与 `lto = false` 选项不同，如上所述，
后者保持 thin local LTO 启用。

### 替代分配器

可以用替代分配器替换 Rust 程序使用的默认（系统）堆分配器。确切效果取决于具体程序和
所选的替代分配器，但在实践中已经看到了运行时速度的大幅提升和内存使用量的大幅减少。
效果也会因平台而异，因为每个平台的系统分配器都有自己的优点和缺点。使用替代分配器也
可能增加二进制体积和编译时间。

#### jemalloc

Linux 和 Mac 上一个流行的替代分配器是 [jemalloc]，可通过 [`tikv-jemallocator`] crate
使用。要使用它，请在 `Cargo.toml` 文件中添加依赖：
```toml
[dependencies]
tikv-jemallocator = "0.5"
```
然后在 Rust 代码中添加以下内容，例如在 `src/main.rs` 的顶部：
```rust,ignore
#[global_allocator]
static GLOBAL: tikv_jemallocator::Jemalloc = tikv_jemallocator::Jemalloc;
```

此外，在 Linux 上，jemalloc 可以配置为使用[透明大页][THP]（THP）。这可以进一步加速
程序，但可能会以更高的内存使用为代价。

[THP]: https://www.kernel.org/doc/html/next/admin-guide/mm/transhuge.html

通过在构建程序之前适当设置 `MALLOC_CONF` 环境变量（或可能 [`_RJEM_MALLOC_CONF`]）
来实现，例如：
```bash
MALLOC_CONF="thp:always,metadata_thp:always" cargo build --release
```
运行编译后程序的系统也必须配置为支持 THP。有关更多详细信息，请参阅[这篇博客文章]。

[`_RJEM_MALLOC_CONF`]: https://github.com/tikv/jemallocator/issues/65
[这篇博客文章]: https://kobzol.github.io/rust/rustc/2023/10/21/make-rust-compiler-5percent-faster.html

#### mimalloc

另一个适用于许多平台的替代分配器是 [mimalloc]，可通过 [`mimalloc`] crate 使用。
要使用它，请在 `Cargo.toml` 文件中添加依赖：
```toml
[dependencies]
mimalloc = "0.1"
```
然后在 Rust 代码中添加以下内容，例如在 `src/main.rs` 的顶部：
```rust,ignore
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

[jemalloc]: https://github.com/jemalloc/jemalloc
[`tikv-jemallocator`]: https://crates.io/crates/tikv-jemallocator
[better performance]: https://github.com/rust-lang/rust/pull/83152
[mimalloc]: https://github.com/microsoft/mimalloc
[`mimalloc`]: https://crates.io/crates/mimalloc

### CPU 特定指令

如果你不关心二进制文件在较旧（或其他类型）处理器上的兼容性，可以告诉编译器生成特定于
[某个 CPU 架构]的最新（且可能最快的）指令，例如针对 x86-64 CPU 的 AVX SIMD 指令。

[某个 CPU 架构]: https://doc.rust-lang.org/rustc/codegen-options/index.html#target-cpu

要从命令行请求这些指令，请使用 `-C target-cpu=native` 标志。例如：
```bash
RUSTFLAGS="-C target-cpu=native" cargo build --release
```

或者，从 [`config.toml`] 文件（适用于一个或多个项目）请求这些指令，添加以下行：
```toml
[build]
rustflags = ["-C", "target-cpu=native"]
```
[`config.toml`]: https://doc.rust-lang.org/cargo/reference/config.html

这可以提高运行时速度，特别是当编译器在你的代码中发现向量化机会时。

如果你不确定 `-C target-cpu=native` 是否在最佳地工作，请比较 `rustc --print cfg`
和 `rustc --print cfg -C target-cpu=native` 的输出，以查看后者是否正确检测到了
CPU 特性。如果没有，你可以使用 `-C target-feature` 来针对特定特性。

### 配置文件引导优化

配置文件引导优化（PGO）是一种编译模型，你先编译程序，在示例数据上运行它并收集分析
数据，然后使用这些分析数据来指导程序的第二次编译。这可以将运行时速度提高 10% 或更多。
[**示例 1**](https://blog.rust-lang.org/inside-rust/2020/11/11/exploring-pgo-for-the-rust-compiler.html),
[**示例 2**](https://github.com/rust-lang/rust/pull/96978).

这是一项高级技术，需要付出一些努力来设置，但在某些情况下是值得的。有关详细信息，
请参阅 [rustc PGO 文档]。此外，[`cargo-pgo`] 命令使使用 PGO（以及类似的 [BOLT]）
来优化 Rust 二进制文件变得更加容易。

遗憾的是，PGO 不支持托管在 crates.io 上并通过 `cargo install` 分发的二进制文件，
这限制了它的可用性。

[rustc PGO 文档]: https://doc.rust-lang.org/rustc/profile-guided-optimization.html
[`cargo-pgo`]: https://github.com/Kobzol/cargo-pgo
[BOLT]: https://github.com/llvm/llvm-project/tree/main/bolt

## 最小化二进制体积

以下构建配置选项主要旨在最小化二进制体积。它们对运行时速度的影响各不相同。

### 优化级别

你可以请求旨在最小化二进制体积的[优化级别]，方法是在 `Cargo.toml` 文件中添加以下行：
```toml
[profile.release]
opt-level = "z"
```
[优化级别]: https://doc.rust-lang.org/cargo/reference/profiles.html#opt-level

这也可能降低运行时速度。

替代方案是 `opt-level = "s"`，它对最小化二进制体积的追求不那么激进。与 `opt-level = "z"`
相比，它允许[稍多一些的内联]以及循环的向量化。

[稍多一些的内联]: https://doc.rust-lang.org/rustc/codegen-options/index.html#inline-threshold

### panic 时中止

如果你不需要在 panic 时展开堆栈，例如因为你的程序不使用 [`catch_unwind`]，你可以
告诉编译器在 panic 时简单地[中止]。panic 时，你的程序仍会产生回溯信息。

[`catch_unwind`]: https://doc.rust-lang.org/std/panic/fn.catch_unwind.html
[中止]: https://doc.rust-lang.org/cargo/reference/profiles.html#panic

这可能会略微减小二进制体积、略微提高运行时速度，甚至可能略微减少编译时间。在
`Cargo.toml` 文件中添加以下行：
```toml
[profile.release]
panic = "abort"
```

### 剥离符号

你可以通过向 `Cargo.toml` 添加以下行来告诉编译器[剥离] release 构建中的符号：
```toml
[profile.release]
strip = "symbols"
```
[剥离]: https://doc.rust-lang.org/cargo/reference/profiles.html#strip

[**示例**](https://github.com/nnethercote/counts/commit/53cab44cd09ff1aa80de70a6dbe1893ff8a41142).

然而，剥离符号可能会使编译后的程序更难调试和分析。例如，如果剥离后的程序 panic，
生成的回溯信息可能比正常情况下包含更少的有用信息。确切效果取决于平台。

Release 构建不需要剥离调试信息。默认情况下，本地 release 构建不会生成调试信息，
并且标准库的调试信息在 release 构建中已自动剥离[自 Rust 1.77 起]。

[自 Rust 1.77 起]: https://blog.rust-lang.org/2024/03/21/Rust-1.77.0.html#enable-strip-in-release-profiles-by-default

### 其他想法

有关更高级的二进制体积最小化技术，请查阅优秀的 [`min-sized-rust`] 仓库中的全面文档。

[`min-sized-rust`]: https://github.com/johnthagen/min-sized-rust

## 最小化编译时间

以下构建配置选项主要旨在最小化编译时间。

### 链接

编译时间的很大一部分实际上是链接时间，特别是在小改动后重新构建程序时。在某些平台上，
可以选择比默认链接器更快的链接器。

一个选项是 [lld]，可在 Linux 和 Windows 上使用。lld 在 Linux 上已经是默认链接器
[自 Rust 1.90 起]。在 Windows 上还不是默认的，但它应该适用于大多数用例。

[自 Rust 1.90 起]: https://blog.rust-lang.org/2025/09/01/rust-lld-on-1.90.0-stable/

要从命令行指定 lld，请使用 `-C link-arg=-fuse-ld=lld` 标志。例如：
```bash
RUSTFLAGS="-C link-arg=-fuse-ld=lld" cargo build --release
```

[lld]: https://lld.llvm.org/

或者，从 [`config.toml`] 文件（适用于一个或多个项目）指定 lld，添加以下行：
```toml
[build]
rustflags = ["-C", "link-arg=-fuse-ld=lld"]
```
[`config.toml`]: https://doc.rust-lang.org/cargo/reference/config.html

有一个 [GitHub Issue] 在跟踪对 lld 的全面支持。

[GitHub Issue]: https://github.com/rust-lang/rust/issues/39915#issuecomment-618726211

另一个选项是 [mold]，目前在 Linux 上可用。只需将上述指令中的 `lld` 替换为 `mold`。
mold 通常比 lld 更快。
[**示例**](https://davidlattimore.github.io/posts/2024/02/04/speeding-up-the-rust-edit-build-run-cycle.html).
它也更新，可能并非在所有情况下都有效。

[mold]: https://github.com/rui314/mold

最后一个选项是 [wild]，目前仅适用于 Linux。它可能比 mold 还快，但不够成熟。

[wild]: https://github.com/davidlattimore/wild

在 Mac 上，不需要替代链接器，因为系统链接器已经很快。

与本章中的其他选项不同，选择另一个链接器没有任何权衡。只要链接器能正确地用于你的程序
（除非你在做不寻常的事情，否则很可能是这样），替代链接器可以显著加快速度，没有任何
缺点。

### 禁用调试信息生成

虽然 release 构建提供了最佳性能，但许多人在开发时使用 dev 构建，因为它们构建更快。
如果你使用 dev 构建但不经常使用调试器，请考虑禁用调试信息。这可以显著改善 dev 构建
时间，最多可提高 20-40%。
[**示例**](https://kobzol.github.io/rust/rustc/2025/05/20/disable-debuginfo-to-improve-rust-compile-times.html)

要禁用调试信息生成，请将以下行添加到 `Cargo.toml` 文件中：
```toml
[profile.dev]
debug = false
```
请注意，这意味着堆栈跟踪将不包含行信息。如果你希望保留行信息，但不需要完整的调试器
信息，可以使用 `debug = "line-tables-only"` 代替，这仍然能带来大部分编译时间收益。

### 实验性并行前端

如果你使用 nightly Rust，可以启用实验性的[并行前端]。它可能会以减少编译时内存使用为
代价来缩短编译时间。它不会影响生成代码的质量。

[并行前端]: https://blog.rust-lang.org/2023/11/09/parallel-rustc.html

你可以通过将 `-Zthreads=N` 添加到 RUSTFLAGS 来实现，例如：
```bash
RUSTFLAGS="-Zthreads=8" cargo build --release
```

或者，从 [`config.toml`] 文件（适用于一个或多个项目）启用并行前端，添加以下行：
```toml
[build]
rustflags = ["-Z", "threads=8"]
```
[`config.toml`]: https://doc.rust-lang.org/cargo/reference/config.html

除了 `8` 之外的其他值也是可能的，但 `8` 往往是能给出最佳结果的值。

在最好的情况下，实验性并行前端可以将编译时间减少多达 50%。但效果差异很大，取决于
代码的特征及其构建配置，对于某些程序，编译时间没有改善。

### Cranelift Codegen 后端

如果你使用 nightly Rust，可以在[某些平台]上启用 Cranelift codegen 后端。它可能会以
降低生成代码质量为代价来减少编译时间，因此建议用于 dev 构建而非 release 构建。

首先，使用以下 `rustup` 命令安装后端：
```bash
rustup component add rustc-codegen-cranelift-preview --toolchain nightly
```

要从命令行选择 Cranelift，请使用 `-Zcodegen-backend=cranelift` 标志。例如：
```bash
RUSTFLAGS="-Zcodegen-backend=cranelift" cargo +nightly build
```

或者，从 [`config.toml`] 文件（适用于一个或多个项目）指定 Cranelift，添加以下行：
```toml
[unstable]
codegen-backend = true

[profile.dev]
codegen-backend = "cranelift"
```
[`config.toml`]: https://doc.rust-lang.org/cargo/reference/config.html

有关更多信息，请参阅 [Cranelift 文档]。

[某些平台]: https://github.com/rust-lang/rustc_codegen_cranelift#platform-support
[Cranelift 文档]: https://github.com/rust-lang/rustc_codegen_cranelift

## 自定义 Profile

除了 `dev` 和 `release` profile 之外，Cargo 还支持[自定义 profile]。例如，如果你发现
dev 构建的运行时速度不够，而 release 构建的编译时间对于日常开发来说太慢，那么创建
一个介于 `dev` 和 `release` 之间的自定义 profile 可能会很有用。

[自定义 profile]: https://doc.rust-lang.org/cargo/reference/profiles.html#custom-profiles

## 总结

构建配置涉及许多选择。以下要点将上述信息总结为一些建议。

- 如果你想最大化运行时速度，请考虑以下所有选项：
  `codegen-units = 1`、`lto = "fat"`、替代分配器和 `panic = "abort"`。
- 如果你想最小化二进制体积，请考虑 `opt-level = "z"`、
  `codegen-units = 1`、`lto = "fat"`、`panic = "abort"` 和 `strip = "symbols"`。
- 无论哪种情况，如果不需要广泛的架构支持，请考虑 `-C target-cpu=native`；如果
  与你的分发机制兼容，请考虑 `cargo-pgo`。
- 如果你所在的平台支持更快的链接器，请始终使用它，因为这样做没有任何缺点。
- 如果你需要有关这些选择的额外帮助，请使用 `cargo-wizard`。
- 对所有更改逐一进行基准测试，以确保它们具有预期的效果。

最后，[这个 issue] 跟踪了 Rust 编译器自身构建配置的演变。Rust 编译器的构建系统比
大多数 Rust 程序更奇特、更复杂。尽管如此，这个 issue 可能有助于说明如何将构建配置
选择应用于大型程序。

[这个 issue]: https://github.com/rust-lang/rust/issues/103595
