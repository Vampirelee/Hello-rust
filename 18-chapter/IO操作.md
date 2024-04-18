# 输入与输出

Rust 标准库中的输入和输出的特性是围绕 3 个特型组织的，即 Read、BufRead 和 Write。

- Read 实现了 Read 的值具有面向字节的输入方法。它们叫作读取器。
- 实现了 BufRead 的值是缓冲读取器。它们支持 Read 的所有方法，外加读取文本行等方法。
- 实现了 Write 的值能支持面向字节和 UTF-8 文本的输出。它们叫作写入器。

下图展示了这 3 个特型以及几个读取器类型和写入器类型的示例。

![](./images/1.png)

## 读取器和写入器

你的程序可以从读取器中读取一些字节。例如：

- 使用 `std::fs::File::open()` 打开一个文件。
- `std::net::TcpStream`，用于通过网络接收数据。
- `std::io::stdin()`, 用于从进程的标准输入流中进行读取
- `std::io::Cursor<&[u8]>`和`std::io::Cursor<Vec<u8>>`，它们是从已存在于内存中的字节数组或向量中进行“读取”的读取器。

写入器则可以向其中写入一些字节。例如：

- 使用 `std::fs::File::create(filename)` 创建一个文件。
- `std::net::TcpStream`，用于通过网络发送数据。
- `std::io::stdout()` 和 `std::io::stderr()`，它们是进程的标准输出流和标准错误流。
- `Vec<u8>` 也是一个写入器，它的 write 方法会把内容追加到向量
- `std::io::Cursor<Vec<u8>` 它与 `Vec<u8>` 很像，但允许读取数据和写入数据，并能在向量中寻找不同的位置
- `std::io::Cursor<&mut [u8]>`，它与`std::io::Cursor<Vec<u8>>` 很像，不过不能增长缓冲区，因为它只是一些现有字节数组的切片。

由于读取器和写入器都有标准特型（`std::io::Read` 和`std::io::Write`），因此编写适用于各种输入通道或输出通道的泛型代码是很常见的。例如，下面是一个将所有字节从任意读取器复制到任意写入器的函数：

```rust
use std::io::{self, ErrorKind, Read, Write};

const DEFAULT_BUF_SIZE: usize = 8 * 1024;
pub fn copy<R: ?Sized, W: ?Sized>(reader: &mut R, writer: &mut W) -> io::Result<u64>
where
    R: Read,
    W: Write,
{
    let mut buf = [0; DEFAULT_BUF_SIZE];
    let mut written = 0;
    loop {
        let len = match reader.read(&mut buf) {
            Ok(0) => return Ok(written),
            Ok(len) => len,
            Err(ref e) if e.kind() == ErrorKind::Interrupted => continue,
            Err(e) => return Err(e),
        };
        writer.write_all(&buf[..len])?;
        written += len as u64;
    }
}
```

这是 Rust 标准库中提供的 `std::io::copy()` 函数的实现。它使用一个缓冲区来减少系统调用的次数，并在遇到 `Interrupted` 错误时重试。

### 读取器

`std::io::Read` 有以下几个读取数据的方法。所有这些方法都需要对读取器本身进行可变引用。

- `reader.read(&mut buffer)` 读取

  从数据源中读取一些字节并将他们存储在给定的 buffer 中。buffer 参数的类型是 `&mut[u8]`, 此方法最多会读取 `buffer.len()` 个字节。

  返回的类型是 `io::Result<u64>`, 它是 `Result<u64, io::Error>`的类型别名。成功时，这个 u64 的值是已读取的字节数，可能等于或小于 buffer.len()，就算数据源突然涌入了更多数据也不会超出。Ok(0) 则意味着没有更多输入可以读取了

  出错时，.read() 会返回一个 Err(err)， 其中 err 是一个 io::Error 值。为了对人类友好，io::Error 是可打印的； 为了便于程序处理，它有一个 .kind() 方法，该方法会返回 io::ErrorKind 类型的错误代码。此枚举的成员都有 PermissionDenied 和 ConnectionReset 之类的名称。大多数表示严重的错误，不容忽视，但有一种错误需要特殊处理。 io::ErrorKind::Interrupted 对应于 Unix 错误码 EINTR，表示读取恰好被某种信号中断了。除非程序的设计目标之一就是对信号中断做一些巧妙的处理，否则就应该再次尝试读取。

  .read() 方法是非常底层的，甚至还继承了底层操作系统的某些怪癖。如果你正在为一种新型数据源实现 Read 特型，那么这会给你很大的发挥空间。但如果你想读取一些数据，则会很痛苦。因此，Rust 提供了几个更高级的便捷方法。这些方法是 Read 中的默认是实现，而且都处理了 ErrorKind::Interrupted, 这样就不必我们自己处理了。

- `reader.read_to_end(&mut buffer)` 读取所有字节

  从这个读取器读取所有剩余的输入，将其追加到 `Vec<u8>`型的 byte_vec 中。返回 `io::Result<usize>`, 即已读取的字节数。此方法对要装入向量中的数据量没有任何限制，因此不要在不受信任的来源上使用它。

- `reader.read_to_string(&mut string)` 读取所有字节并将其追加到 String 中

  如果流不是有效的 UTF-8，则返回 ErrorKind::InvalidData 错误

- `reader.read_exact(&mut buffer)` 精确读满

  读取足够的数据来填充给定的缓冲区。参数类型是 &[u8]。如果读取器在读够 buf.len() 字节之前就耗尽了数据，那么此方法就会返回 ErrorKind::UnexpectedEof 错误。

以上这些是 Read 特型的主要方法。此外，还有 3 个适配器方法可以按值获取 reader，将其转换为迭代器或另一种读取器。

- `reader.bytes()` 适配器，返回一个迭代器，每次迭代返回一个字节。

  返回输入流中各字节的迭代器。条目类型是`io::Result<u8>`，因此每字节都需要进行错误检查。此外，此方法会为每字节调用一次 `reader.read()`，如果没有缓冲，则会非常低效。

- `reader.chain(reader2)` 串联

  返回新的读取器，先生成来自 reader 的所有输入，然后再生成来自 reader2 的所有输入。

- `reader.take(n)` 截断

  返回新的读取器，从与 reader 相同的数据源读取，但仅限于前 n 字节的输入。

> 没有关闭读取器的方法。读取器和写入器通常会实现 Drop 以便自行关闭

### 缓冲读取器

为了提高效率，可以对读取器和写入器进行缓冲，这基本上意味着它们有一块内存（缓冲区），用于保存一些输入数据或输出数据。这可以减少一些系统调用, 如图所示。应用程序会从 BufReader 中读取数据，在本例中是通过调用其 .read_line() 方法实现的。 BufReader 会依次从操作系统获取更大块的输入。

![](./images/2.png)

BufReader 缓冲区的实际大小默认为几千字节，因此一次 read 系统调用就可以服务数百次 .read_line()调用。这很重要，因为系统调用很慢。

缓冲读取器同时实现了 Read 特型和 BufRead 特型，后者添加了以下方法。

- `reader.read_line(&mut line)` 读取一行

  读取一行文本并将其追加到 line，line 是一个 String。行尾的换行符 '\n' 包含在 line 中，返回值是 `io::Result<usize>`，表示已读取的字节数，包括行尾结束符号（如果有的话）。　如果读取器在输入的末尾，则会保持 line 不变并返回 `Ok(0)`。

- `reader.lines()` 文本行迭代器

  返回生成各个输入行的迭代器。条目类型是`io::Result<String>`。字符串中不包含换行符。如果输入带有 Windows 风格的行尾结束符号 "\r\n"，则这两个字符都会被去掉。

- `reader.read_until(stop_byte, &mut byte_vec)` 读取到 stop_byte 为止，`reader.split(stop_byte)` 根据 stop_byte 拆分

  与 `.read_line()` 和 `.lines()` 类似（这两个是以换行符为分隔符的），但这两个方法是面向字节的，会生成 `Vec<u8>` 而不是 `String`。你要自选分隔符`stop_byte`。

### 读取行

下面是一个用于实现 Unix grep 实用程序的函数，该函数会在多行文本（通常是通过管道从另一条命令输入的文本）中搜索给定字符串

```rust
use std::io;
use std::io::prelude::*;


fn grep(target: &str) -> io::Result<()> {
    let stdin = io::stdin();
    for line_result in stdin.lock().lines() {
        let line = line_result?;
        if line.contains(target) {
            println!("{}", line);
        }
    }
    Ok(())
}
```

假如我们想完善这个 grep 程序，让它支持在磁盘上搜索文件。可以把这个函数变成泛型函数

```rust
fn grep(target: &str, reader: R) -> io::Result<()>
where R: BufRead
{
    for line_result in reader.lines() {
        let line = line_result?;
        if line.contains(target) {
            println!("{}", line);
        }
    }
    Ok(())
}
```

### 收集行

有些读取器方法（包括 .lines()）会返回生成 Result 值的迭代器。当你第一次想要将文件的所有行都收集到一个大型向量中时，就会遇到如何摆脱 Result 的问题

```rust
// 正确，但不是你想要的
let results: Vec<io::Result<String>> = reader.lines().collect();
// 错误：不能把Result的集合转换成Vec<String>
let lines: Vec<String> = reader.lines().collect();
```

第二次尝试无法编译：遇到这些错误怎么办？最直观的解决方法是编写一个 for 循环并检查每个条目是否有错

```rust
let mut lines = vec![];
for line_result in reader.lines() {
    lines.push(line_result?);
}
```

这固然没错，但这里最好还是用 .collect()，事实上确实可以做到。只要知道该请求哪种类型就可以了：

```rust
let lines = reader.lines().collect::<io::Result<Vec<String>>>()?;
```

标准库中包含了 Result 对 FromIterator 的实现，这个实现让一切成为可能：

```rust
impl<T, E, C> FromIterator<Result<T, E>> for Result<C, E>
  where C: FromIterator<T>
{
  //...
}
```

假设 C 是任意集合类型，比如` Vec` 或 `HashSet`。只要已经知道如何从 T 值的迭代器构建出 C，就可以从生成 `Result<T, E>` 值的迭代器构建出`Result<C, E>`。只需从迭代器中提取各个值并从 Ok 结果构建出集合即可，但一旦看到 Err，就停止并将其传出。

换句话说，`io::Result<Vec<String>>` 是一种集合类型，因此 `.collect()` 方法可以创建并填充该类型的值。

### 写入器

如前所述，输入主要是用方法完成的，而输出略有不同。

我们一直在使用 `println!()` 生成纯文本输出，还有不会在行尾添加换行符的 `print!()` 宏，以及会写入标准错误流的 `eprintln!` 宏和 `eprint!` 宏。所有这些宏的格式化代码都和 `format!` 宏一样

要将输出发送到写入器，请使用 write!() 宏和 writeln!() 宏。它们和 print!() 和 println!() 类似，但有两点区别

- 一是每个 `write` 宏都接受一个额外的写入器作为第一参数。
- 二是它们会返回 `Result`，因此必须处理错误。这就是为什么要在每行末尾使用 `?` 运算符

Write 特型的主要方法如下：

- `writer.write(&buffer)` 写入

  将切片 buf 中的一些字节写入底层流。此方法会返回`io::Result<usize>`。成功时，这给出了已写入的字节数，如果流突然提前关闭，那么这个值可能会小于 `buf.len()`。

  > 和 Reader::read() 一样，这是一个要避免直接使用的底层方法。

- `writer.write_all(&buffer)` 写入所有字节

  将切片 buf 中的所有字节都写入。返回 `Result<()>`

- `writer.flush()` 刷新缓冲区

  将可能被缓冲在内存中的数据刷新到底层流中。返回 `Result<()>`

  > 请注意，虽然 println! 宏和 eprintln! 宏会自动刷新 stdout 流和 stderr 流的缓冲区，但 print! 宏和 eprint! 宏不会。使用它们时，可能要手动调用 flush()。

与读取器一样，写入器也会在被丢弃时自动关闭。

正如 `BufReader::new(reader)` 会为任意读取器添加缓冲区一样，`BufWriter::new(writer)` 也会为任意写入器添加缓冲区

```rust
let file = File::create("tmp.txt")?;
let writer = BufWriter::new(file);
```

要设置缓冲区的大小，请使用 `BufWriter::with_capacity(size, writer)`。

当丢弃 `BufWriter` 时，所有剩余的缓冲数据都将写入底层写入器。但是，如果在此写入过程中发生错误，则错误会被忽略。（由于错误发生在 `BufWriter` 的 `.drop()` 方法内部，因此没有合适的地方来报告。）为了确保应用程序会注意到所有输出错误，请在丢弃带缓冲的写入器之前将它手动 `.flush()` 一下。
