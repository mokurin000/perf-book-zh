# 机器码

当你有小段非常热的代码时，检查生成的机器码以查看是否存在任何低效之处（例如可消除的
[边界检查]）可能是值得的。[Compiler Explorer] 网站是在小型代码片段上进行此操作的
优秀资源。[`cargo-show-asm`] 是一个可用于完整 Rust 项目的替代工具。

[边界检查]: bounds-checks.md
[Compiler Explorer]: https://godbolt.org/
[`cargo-show-asm`]: https://github.com/pacak/cargo-show-asm

与之相关的是，[`core::arch`] 模块提供了对架构特定内建函数的访问，其中许多与
SIMD 指令有关。

[`core::arch`]: https://doc.rust-lang.org/core/arch/index.html
