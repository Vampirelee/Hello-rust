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


