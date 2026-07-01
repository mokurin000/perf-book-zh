# 代码检查

[Clippy] 是一个用于捕获 Rust 代码中常见错误的 lint 集合。它通常是在 Rust 代码上
运行的优秀工具。它还能帮助提升性能，因为许多 lint 与可能导致次优性能的代码模式
有关。

鉴于自动检测问题优于手动检测，本书其余部分将不再提及 Clippy 默认检测到的性能问题。

## 基础

[Clippy]: https://github.com/rust-lang/rust-clippy

安装后，运行非常简单：
```text
cargo clippy
```
完整的性能 lint 列表可以通过访问 [lint 列表]并取消选中除"Perf"以外的所有 lint 组来查看。

[lint 列表]: https://rust-lang.github.io/rust-clippy/master/

除了使代码更快之外，性能 lint 的建议通常还会使代码更简单、更符合语言习惯，因此即使
对于不频繁执行的代码，也值得遵循。

相反，一些非性能的 lint 建议也能提升性能。例如，[`ptr_arg`] 风格 lint 建议将各种容器参数
改为切片，例如将 `&mut Vec<T>` 参数改为 `&mut [T]`。这样做的主要动机是切片提供了更灵活的
API，但也可能因为减少间接访问并为编译器提供更好的优化机会而产生更快的代码。
[**示例**](https://github.com/fschutt/fastblur/pull/3/files)。

[`ptr_arg`]: https://rust-lang.github.io/rust-clippy/master/index.html#ptr_arg

## 禁止使用某些类型

在接下来的章节中，我们将看到，有时值得避免使用某些标准库类型，而采用更快的替代方案。
如果你决定使用这些替代方案，很容易在某些地方意外地错误使用标准库类型。

你可以使用 Clippy 的 [`disallowed_types`] lint 来避免这个问题。例如，要禁止使用标准哈希表
（原因在[哈希]章节中解释），请在代码中添加一个 `clippy.toml` 文件，包含以下内容。
```toml
disallowed-types = ["std::collections::HashMap", "std::collections::HashSet"]
```

[哈希]: hashing.md
[`disallowed_types`]: https://rust-lang.github.io/rust-clippy/master/index.html#disallowed_types
