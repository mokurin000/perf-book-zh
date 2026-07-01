# 日志与调试

有时日志代码或调试代码会显著拖慢程序。要么是日志/调试代码本身很慢，要么是为日志/调试
提供数据的数据收集代码很慢。请确保在未启用日志/调试功能时，不会为了日志/调试目的而执行
不必要的工作。
[**示例 1**](https://github.com/rust-lang/rust/pull/50246/commits/2e4f66a86f7baa5644d18bb2adc07a8cd1c7409d),
[**示例 2**](https://github.com/rust-lang/rust/pull/75133/commits/eeb4b83289e09956e0dda174047729ca87c709fe),
[**示例 3**](https://github.com/rust-lang/rust/pull/147293/commits/cb0f969b623a7e12a0d8166c9a498e17a8b5a3c4).

注意，[`assert!`] 调用始终执行，但 [`debug_assert!`] 调用仅在 dev 构建中运行。
如果你的断言是热点但并非安全必需，请考虑将其改为 `debug_assert!`。
[**示例 1**](https://github.com/rust-lang/rust/pull/58210/commits/f7ed6e18160bc8fccf27a73c05f3935c9e8f672e),
[**示例 2**](https://github.com/rust-lang/rust/pull/90746/commits/580d357b5adef605fc731d295ca53ab8532e26fb).

[`assert!`]: https://doc.rust-lang.org/std/macro.assert.html
[`debug_assert!`]: https://doc.rust-lang.org/std/macro.debug_assert.html
