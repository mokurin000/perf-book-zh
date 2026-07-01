# 性能分析

在优化程序时，你还需要一种方法来确定程序的哪些部分是"热点"（执行频率足以影响运行时）
并值得修改。这最好通过性能分析来实现。

## 分析器

有许多不同的分析器可用，各有其优缺点。以下是在 Rust 程序上成功使用过的分析器的不
完整列表。
- [perf] 是一个使用硬件性能计数器的通用分析器。[Hotspot] 和 [Firefox Profiler]
  适合查看 perf 记录的数据。它适用于 Linux。
- [Instruments] 是 macOS 上随 Xcode 提供的通用分析器。
- [Intel VTune Profiler] 是一个通用分析器。它适用于 Windows、Linux 和 macOS。
- [AMD μProf] 是一个通用分析器。它适用于 Windows 和 Linux。
- [samply] 是一个采样分析器，生成可在 Firefox Profiler 中查看的分析结果。它适用于
  Mac、Linux 和 Windows。
- [flamegraph] 是一个 Cargo 命令，使用 perf/DTrace 分析你的代码，然后在火焰图中
  显示结果。它适用于 Linux 和所有支持 DTrace 的平台（macOS、FreeBSD、NetBSD 和
  可能 Windows）。
- [Cachegrind] 和 [Callgrind] 提供全局、按函数和按源代码行的指令计数以及模拟缓存
  和分支预测数据。它们适用于 Linux 和其他一些 Unix 系统。
- [DHAT] 擅长查找代码中导致大量分配的部分，并提供对峰值内存使用的洞察。它还可以
  用于识别对 `memcpy` 的热点调用。它适用于 Linux 和其他一些 Unix 系统。[dhat-rs]
  是一个实验性的替代方案，功能略弱，需要对 Rust 程序进行少量修改，但适用于所有平台。
- [heaptrack] 和 [bytehound] 是堆分析工具。它们适用于 Linux。
- [`counts`] 支持临时分析，结合使用 `eprintln!` 语句与基于频率的后处理，非常适合
  获得对代码部分的领域特定洞察。它适用于所有平台。
- [Coz] 执行*因果分析*来测量优化潜力，通过 [coz-rs] 支持 Rust。它适用于 Linux。

[perf]: https://perf.wiki.kernel.org/index.php/Main_Page
[Hotspot]: https://github.com/KDAB/hotspot
[Firefox Profiler]: https://profiler.firefox.com/
[Instruments]: https://developer.apple.com/forums/tags/instruments
[Intel VTune Profiler]: https://www.intel.com/content/www/us/en/developer/tools/oneapi/vtune-profiler.html
[AMD μProf]: https://developer.amd.com/amd-uprof/
[samply]: https://github.com/mstange/samply/
[flamegraph]: https://github.com/flamegraph-rs/flamegraph
[Cachegrind]: https://www.valgrind.org/docs/manual/cg-manual.html
[Callgrind]: https://www.valgrind.org/docs/manual/cl-manual.html
[DHAT]: https://www.valgrind.org/docs/manual/dh-manual.html
[dhat-rs]: https://github.com/nnethercote/dhat-rs/
[Valgrind]: https://valgrind.org/
[heaptrack]: https://github.com/KDE/heaptrack
[bytehound]: https://github.com/koute/bytehound
[`counts`]: https://github.com/nnethercote/counts/
[Coz]: https://github.com/plasma-umass/coz
[coz-rs]: https://github.com/plasma-umass/coz/tree/master/rust

## 调试信息

要有效地分析 release 构建，你可能需要启用源代码行调试信息。为此，请将以下行添加到
`Cargo.toml` 文件中：
```toml
[profile.release]
debug = "line-tables-only"
```
有关 `debug` 设置的更多详细信息，请参阅 [Cargo 文档]。

[Cargo 文档]: https://doc.rust-lang.org/cargo/reference/profiles.html#debug

遗憾的是，即使执行了上述步骤，你也不会获得标准库代码的详细性能分析信息。这是因为
已发布版本的 Rust 标准库在构建时未包含调试信息。

最可靠的解决方法是按照[这些说明]构建你自己的编译器和标准库版本，并在仓库根目录的
`bootstrap.toml` 文件中添加以下行：
 ```toml
[rust]
debuginfo-level = 1
```
这很麻烦，但在某些情况下可能值得付出努力。

[这些说明]: https://github.com/rust-lang/rust

或者，不稳定的 [build-std] 功能允许你将标准库作为程序正常编译的一部分进行编译，
使用相同的构建配置。但是，标准库调试信息中的文件名不会指向源代码文件，因为此功能
不会同时下载标准库源代码。因此，这种方法对需要源代码才能完全工作的分析器（如
Cachegrind 和 samply）没有帮助。

[build-std]: https://doc.rust-lang.org/cargo/reference/unstable.html#build-std

## 帧指针

Rust 编译器可能会优化掉帧指针，这会损害堆栈跟踪等性能分析信息的质量。要强制编译器
使用帧指针，请使用 `-C force-frame-pointers=yes` 标志。例如：
```bash
RUSTFLAGS="-C force-frame-pointers=yes" cargo build --release
```

或者，从 [`config.toml`] 文件（适用于一个或多个项目）强制使用帧指针，添加以下行：
```toml
[build]
rustflags = ["-C", "force-frame-pointers=yes"]
```
[`config.toml`]: https://doc.rust-lang.org/cargo/reference/config.html

## 符号反修饰

Rust 使用一种名称修饰形式来编码编译代码中的函数名。如果分析器不知道这一点，其输出
可能包含以 `_ZN` 或 `_R` 开头的符号名称，例如 `_ZN3foo3barE` 或
`_ZN28_$u7b$$u7b$closure$u7d$$u7d$E` 或
`_RMCsno73SFvQKx_1cINtB0_3StrKRe616263_E`

此类名称可以使用 [`rustfilt`] 手动反修饰。

[`rustfilt`]: https://crates.io/crates/rustfilt

如果你在性能分析时遇到符号反修饰问题，可能值得将[修饰格式]从默认的传统格式更改为
较新的 v0 格式。

[修饰格式]: https://doc.rust-lang.org/rustc/codegen-options/index.html#symbol-mangling-version

要从命令行使用 v0 格式，请使用 `-C symbol-mangling-version=v0` 标志。例如：
```bash
RUSTFLAGS="-C symbol-mangling-version=v0" cargo build --release
```

或者，从 [`config.toml`] 文件（适用于一个或多个项目）请求这些指令，添加以下行：
```toml
[build]
rustflags = ["-C", "symbol-mangling-version=v0"]
```
[`config.toml`]: https://doc.rust-lang.org/cargo/reference/config.html

