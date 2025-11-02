Go 的 I/O 模型核心思想是：**一切输入输出都是流（stream），通过 Reader/Writer 抽象实现解耦。**

# 1 Reader接口

Reader接口的定义如下：

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

- 从数据源中读取最多 `len(p)` 字节，存入 `p`；

- 返回实际读取的字节数 `n`；

- 若无数据可读返回 `io.EOF`。

```go
// 可以从标准输入读取
data, err := ReadFrom(os.Stdin, 11)

// 从文件读取，其中 file 是 os.File 的实例
data, err = ReadFrom(file, 9)

// 从字符串读取
data, err := ReadFrom(strings.NewReader("from string"), 12)
```

实现者：`os.File`、`bytes.Buffer`、`net.Conn`、`strings.Reader` 等。

# 2 Writer接口

Writer接口的定义如下：

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

- 将 `p` 中的数据写入目标（文件、网络、内存等）；

- 返回成功写入的字节数。

在fmt库中，有一组函数：Fprintf/Fprint/Fprintln，它们接收一个 io.Wrtier 类型参数（第一个参数）

```go
func Fprintf(w io.Writer, format string, a ...any) (n int, err error) {
	p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}
```

fmt.Fprintf将内容输出到标准输出中。

# 3 实现io.Reader接口和io.Writer接口的类型

上述可知os.File和os.Stdin/Stdout实现该两个接口，在os包中定义如下：

```go
var (
	Stdin  = NewFile(uintptr(syscall.Stdin), "/dev/stdin")
	Stdout = NewFile(uintptr(syscall.Stdout), "/dev/stdout")
	Stderr = NewFile(uintptr(syscall.Stderr), "/dev/stderr")
)
```

上述可知Stdin、Stdout、Stderr只是三个NewFile文件类型的标识。当前实现io.Reader和io.Writer接口的类型如下：

- os.File 同时实现了 io.Reader 和 io.Writer
- strings.Reader 实现了 io.Reader
- bufio.Reader/Writer 分别实现了 io.Reader 和 io.Writer
- bytes.Buffer 同时实现了 io.Reader 和 io.Writer
- bytes.Reader 实现了 io.Reader
- compress/gzip.Reader/Writer 分别实现了 io.Reader 和 io.Writer
- crypto/cipher.StreamReader/StreamWriter 分别实现了 io.Reader 和 io.Writer
- crypto/tls.Conn 同时实现了 io.Reader 和 io.Writer
- encoding/csv.Reader/Writer 分别实现了 io.Reader 和 io.Writer
- mime/multipart.Part 实现了 io.Reader
- net/conn 分别实现了 io.Reader 和 io.Writer(Conn接口定义了Read/Write)。

# 4 ReaderAt接口

```go
type ReaderAt interface {
    ReadAt(p []byte, off int64) (n int, err error)
}
```

ReaderAt从基本输入源的偏移量off开始，将len(p)个字节读取到p中，返回读取的字节数n。

```go
func main() {
	reader := strings.NewReader("Go语言")
	p := make([]byte, 6)
	n, err := reader.ReadAt(p, 2)
	if err != nil {
		panic(err)
	}
	fmt.Printf("%s，%d\n", p, n)
}
```

# 5 WriterAt接口

```go
type WriterAt interface {
	WriteAt(p []byte, off int64) (n int, err error)
}
```

WriteAt从p中将len(p)个字节写入到偏移量off的基本数据流中，返回从p中被写入的字节数n。

```go
func main() {
	file, err := os.Create("writeAt.txt")
	if err != nil {
		panic(err)
	}

	defer file.Close()
	file.WriteString("Goland语言")
	n, err := file.WriteAt([]byte("Go语言"), 30)
	if err != nil {
		panic(err)
	}
	fmt.Println(n)
}
```

# 6 ReaderFrom接口

```go
type ReaderFrom interface {
	ReadFrom(r Reader) (n int64, err error)
}
```

ReadFrom从r中读取数据，直到EOF或者发生错误，返回读取的字节数n。

```go
func main() {
	file, err := os.Open("writeAt.txt")
	if err != nil {
		panic(err)
	}

	defer file.Close()
	writer := bufio.NewWriter(os.Stdout)
	writer.ReadFrom(file)
	writer.Flush()
}
```

# 7 WriterTo接口

```go
type WriterTo interface {
    WriteTo(w Writer) (n int64, err error)
}
```

WriteTo将数据写入w中，直到没有数据可写或者发生错误，返回写入字节数n。

# 8 Seeker接口

```go
type Seeker interface {
	Seek(offset int64, whence int) (int64, error)
}
```

Seek设置下一次Read或Write的偏移量为offset，其中whence：0表示相对于文件的起始，1表示相对于当前的偏移，2表示表示文件结尾处，返回新的偏移量和错误。

Seek方法用于设置偏移量，可以从某个特定位置开始操作数据流。

```go
reader := strings.NewReader("Go语言")
reader.Seek(-3, io.SeekEnd)
r, _, _ := reader.ReadRune()
```

# 9 Closer接口

```go
type Closer interface {
    Close() error
}
```

Close方法用于关闭数据流。文件（os.FIle）、压缩包、数据库连接、Socket等需要手动关闭的资源都需要实现Closer接口。

# 10 ByteReader和ByteWriter

```go
type ByteReader interface {
    ReadByte() (c byte, err error)
}

type ByteWriter interface {
    WriteByte(c byte) error
}
```

读或写一个字节。

在标准库中，有如下类型实现了 io.ByteReader 或 io.ByteWriter:

- bufio.Reader/Writer 分别实现了io.ByteReader 和 io.ByteWriter
- bytes.Buffer 同时实现了 io.ByteReader 和 io.ByteWriter
- bytes.Reader 实现了 io.ByteReader
- strings.Reader 实现了 io.ByteReader

```go
func main() {
	var ch byte
	fmt.Scanf("%c\n", &ch)

	buffer := new(bytes.Buffer)
	err := buffer.WriteByte(ch)
	if err == nil {
		fmt.Println("write byte")
		newCh, _ := buffer.ReadByte()
		fmt.Printf("read byte: %c\n", newCh)
	} else {
		fmt.Println(err)
	}
}
```

代码从标准输入接收一个字节，调用buffer的WriteByte将字节写入buffer中，通过ReadByte读取该字节。



























