# 1 NopCloser函数

```go
func NopCloser(r io.Reader) io.ReadCloser {
	return io.NopCloser(r)
}
```

在标准库 net/http 包中的 NewRequest，接收一个 io.Reader 的 body，而实际上，Request 的 Body 的类型是 io.ReadCloser，因此，代码内部进行了判断，如果传递的 io.Reader 也实现了 io.ReadCloser 接口，则转换，否则通过ioutil.NopCloser 包装转换一下。相关代码如下：

```go
rc, ok := body.(io.ReadCloser)
if !ok && body != nil {
    rc = ioutil.NopCloser(body)
}
```

# 2 ReadAll函数

```go
func ReadAll(r io.Reader) ([]byte, error)
```

通过 bytes.Buffer 中的 [ReadFrom](http://docscn.studygolang.com/src/bytes/buffer.go?s=5385:5444#L144) 来实现读取所有数据的。该函数成功调用后会返回 err == nil 而不是 err == EOF。

# 3 ReadDir函数

读取目录并放回排序的文件和子目录名。

```go
func listAll(path string, curHier int) {
	fileInfos, err := ioutil.ReadDir(path)
	if err != nil {
		fmt.Println(err)
		return
	}

	for _, info := range fileInfos {
		if info.IsDir() {
			for tmpHier := curHier; tmpHier > 0; tmpHier-- {
				fmt.Printf("|\t")
			}
			fmt.Println(info.Name, "\\")
			listAll(path+"/"+info.Name(), curHier+1)
		} else {
			for tmpHier := curHier; tmpHier > 0; tmpHier-- {
				fmt.Printf("|\t")
			}
			fmt.Println(info.Name)
		}
	}
}

func main() {
	dir := os.Args[1]
	listAll(dir, 0)
}
```

# 4 ReadFile和WriteFile函数

ReadFile读取整个文件的内容，会先判断文件的大小，给bytes.Buffer一个预定义容量，避免额外分配内存。

```go
func ReadFile(filename string) ([]byte, error)
```

> ReadFile 从 filename 指定的文件中读取数据并返回文件的内容。成功的调用返回的err 为 nil 而非 EOF。因为本函数定义为读取整个文件，它不会将读取返回的 EOF 视为应报告的错误。(同 ReadAll )

WriteFile将data写入filename，当文件不存在时会根据perm指定的权限进行创建一个,文件存在时会先清空文件内容。对于 perm 参数，我们一般可以指定为：0666，具体含义 os 包中讲解。

```go
func WriteFile(filename string, data []byte, perm os.FileMode) error
```

# 5 TempDir和TempFile函数

操作系统中一般都会提供临时目录，比如 linux 下的 /tmp 目录（通过 os.TempDir() 可以获取到)。有时候，我们自己需要创建临时目录，比如 Go 工具链源码中（src/cmd/go/build.go），通过 TempDir 创建一个临时目录，用于存放编译过程的临时文件：

```go
b.work, err = ioutil.TempDir("", "go-build")
```

第一个参数如果为空，表明在系统默认的临时目录（ os.TempDir ）中创建临时目录；第二个参数指定临时目录名的前缀，该函数返回临时目录的路径。

相应的，TempFile 用于创建临时文件。如 gofmt 命令的源码中创建临时文件：

```go
f1, err := ioutil.TempFile("", "gofmt")
```

参数和 ioutil.TempDir 参数含义类似。

# 6 Discard变量

Discard 对应的类型（`type devNull int`）实现了 io.Writer 接口，同时，为了优化 io.Copy 到 Discard，避免不必要的工作，实现了 io.ReaderFrom 接口。

devNull 在实现 io.Writer 接口时，只是简单的返回（标准库文件：[src/pkg/io/ioutil.go](http://docscn.studygolang.com/pkg/io/ioutil/#pkg-variables))。

```go
    func (devNull) Write(p []byte) (int, error) {
        return len(p), nil
    }
```

而 ReadFrom 的实现是读取内容到一个 buf 中，最大也就 8192 字节，其他的会丢弃（当然，这个也不会读取）。















