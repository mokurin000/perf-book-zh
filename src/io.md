# I/O

## 锁定

Rust 的 [`print!`] 和 [`println!`] 宏每次调用时都会锁定 stdout。如果你重复调用这些
宏，手动锁定 stdout 可能会更好。

[`print!`]: https://doc.rust-lang.org/std/macro.print.html
[`println!`]: https://doc.rust-lang.org/std/macro.println.html

例如，将这段代码：
```rust
# let lines = vec!["one", "two", "three"];
for line in lines {
    println!("{}", line);
}
```
改为：
```rust
# fn blah() -> Result<(), std::io::Error> {
# let lines = vec!["one", "two", "three"];
use std::io::Write;
let mut stdout = std::io::stdout();
let mut lock = stdout.lock();
for line in lines {
    writeln!(lock, "{}", line)?;
}
// 当 `lock` 被丢弃时，stdout 解锁
# Ok(())
# }
```
stdin 和 stderr 在重复操作时同样可以锁定。

## 缓冲

Rust 文件 I/O 默认是无缓冲的。如果你对文件或网络套接字进行大量小而重复的读取或写入
调用，请使用 [`BufReader`] 或 [`BufWriter`]。它们在内存中维护输入和输出缓冲区，
从而最大限度地减少所需的系统调用次数。

[`BufReader`]: https://doc.rust-lang.org/std/io/struct.BufReader.html
[`BufWriter`]: https://doc.rust-lang.org/std/io/struct.BufWriter.html

例如，将这个无缓冲写入器代码：
```rust
# fn blah() -> Result<(), std::io::Error> {
# let lines = vec!["one", "two", "three"];
use std::io::Write;
let mut out = std::fs::File::create("test.txt")?;
for line in lines {
    writeln!(out, "{}", line)?;
}
# Ok(())
# }
```
改为：
```rust
# fn blah() -> Result<(), std::io::Error> {
# let lines = vec!["one", "two", "three"];
use std::io::{BufWriter, Write};
let mut out = BufWriter::new(std::fs::File::create("test.txt")?);
for line in lines {
    writeln!(out, "{}", line)?;
}
out.flush()?;
# Ok(())
# }
```
[**示例 1**](https://github.com/rust-lang/rust/pull/93954),
[**示例 2**](https://github.com/nnethercote/dhat-rs/pull/22/commits/8c3ae26f1219474ee55c30bc9981e6af2e869be2).

显式调用 [`flush`] 并非严格必要，因为当 `out` 被丢弃时会自动刷新。但是，在这种情况下，
刷新时发生的任何错误都会被忽略，而显式刷新将使该错误显现出来。

[`flush`]: https://doc.rust-lang.org/std/io/trait.Write.html#tymethod.flush

忘记缓冲在写入时更为常见。无缓冲和有缓冲的写入器都实现了 [`Write`] trait，这意味着
向无缓冲写入器写入的代码与向有缓冲写入器写入的代码基本相同。相比之下，无缓冲读取器
实现了 [`Read`] trait，而有缓冲读取器实现了 [`BufRead`] trait，这意味着从无缓冲读取器
读取的代码与从有缓冲读取器读取的代码是不同的。例如，使用无缓冲读取器逐行读取文件很困难，
但使用有缓冲读取器通过 [`BufRead::read_line`] 或 [`BufRead::lines`] 就很简单。
因此，很难像上面写入器的例子那样为读取器写出修改前后如此相似的示例。

[`Write`]: https://doc.rust-lang.org/std/io/trait.Write.html
[`Read`]: https://doc.rust-lang.org/std/io/trait.Read.html
[`BufRead`]: https://doc.rust-lang.org/std/io/trait.BufRead.html
[`BufRead::read_line`]: https://doc.rust-lang.org/std/io/trait.BufRead.html#method.read_line
[`BufRead::lines`]: https://doc.rust-lang.org/std/io/trait.BufRead.html#method.lines

最后，请注意缓冲也适用于 stdout，因此在多次写入 stdout 时，你可能希望将手动锁定*和*
缓冲结合起来。

## 从文件读取行

[本节]解释了在使用 [`BufRead`] 逐行读取文件时如何避免过多的分配。

[本节]: heap-allocations.md#reading-lines-from-a-file
[`BufRead`]: https://doc.rust-lang.org/std/io/trait.BufRead.html

## 以原始字节形式读取输入

内置的 [String] 类型在内部使用 UTF-8，当你将输入读入其中时，UTF-8 验证会增加一个
虽小但非零的开销。如果你只想处理输入字节而不关心 UTF-8（例如处理 ASCII 文本时），
可以使用 [`BufRead::read_until`]。

[String]: https://doc.rust-lang.org/std/string/struct.String.html
[`BufRead::read_until`]: https://doc.rust-lang.org/std/io/trait.BufRead.html#method.read_until

还有专门用于读取[面向字节的数据行]和处理[字节字符串]的 crate。

[面向字节的数据行]: https://github.com/Freaky/rust-linereader
[字节字符串]: https://github.com/BurntSushi/bstr
