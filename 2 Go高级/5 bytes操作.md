1. 是否存在某个子slice

   ```go
   func Contains(b, subslice []byte) bool
   ```

2. []byte出现次数

   ```go
   // slice sep 在 s 中出现的次数（无重叠）
   func Count(s, sep []byte) int
   ```

3. Runes类型转换

   ```go
   // 该函数将 []byte 转换为 []rune ，适用于汉字等多字节字符，示例：
   func Funes(s []byte) []rune
   
   b:=[]byte("你好，世界")
   for k,v:=range b{
       fmt.Printf("%d:%s |",k,string(v))
   }
   r:=bytes.Runes(b)
   for k,v:=range r{
       fmt.Printf("%d:%s|",k,string(v))
   }
   ```

4. Reader类型

   ```go
   type Reader struct {
       s        []byte
       i        int64 // 当前读取下标
       prevRune int   // 前一个字符的下标，也可能 < 0
   }
   ```

   bytes 包下的 Reader 类型实现了 io 包下的 Reader, ReaderAt, RuneReader, RuneScanner, ByteReader, ByteScanner, ReadSeeker, Seeker, WriterTo 等多个接口。

   ```go
   func NewReader(b []byte) *Reader
   
   x:=[]byte("你好，世界")
   
   r1:=bytes.NewReader(x)
   d1:=make([]byte,len(x))
   n,_:=r1.Read(d1)
   fmt.Println(n,string(d1))
   
   r2:=bytes.Reader{}
   r2.Reset(x)
   d2:=make([]byte,len(x))
   n,_=r2.Read(d2)
   fmt.Println(n,string(d2))
   ```

   Reader 包含了 8 个读取相关的方法，实现了前面提到的 io 包下的 9 个接口（ReadSeeker 接口内嵌 Reader 和 Seeker 两个接口）

   ```go
   // 读取数据至 b 
   func (r *Reader) Read(b []byte) (n int, err error) 
   // 读取一个字节
   func (r *Reader) ReadByte() (byte, error)
   // 读取一个字符
   func (r *Reader) ReadRune() (ch rune, size int, err error)
   // 读取数据至 w
   func (r *Reader) WriteTo(w io.Writer) (n int64, err error)
   // 进度下标指向前一个字节，如果 r.i <= 0 返回错误。
   func (r *Reader) UnreadByte() 
   // 进度下标指向前一个字符，如果 r.i <= 0 返回错误，且只能在每次 ReadRune 方法后使用一次，否则返回错误。
   func (r *Reader) UnreadRune() 
   // 读取 r.s[off:] 的数据至b，该方法忽略进度下标 i，不使用也不修改。
   func (r *Reader) ReadAt(b []byte, off int64) (n int, err error) 
   // 根据 whence 的值，修改并返回进度下标 i ，当 whence == 0 ，进度下标修改为 off，当 whence == 1 ，进度下标修改为 i+off，当 whence == 2 ，进度下标修改为 len[s]+off.
   // off 可以为负数，whence 的只能为 0，1，2，当 whence 为其他值或计算后的进度下标越界，则返回错误。
   func (r *Reader) Seek(offset int64, whence int) (int64, error)
   ```

5. Buffer类型

   ```go
   type Buffer struct {
       buf      []byte
       off      int   
       lastRead readOp 
   }
   ```

   该类型实现了 io 包下的 ByteScanner, ByteWriter, ReadWriter, Reader, ReaderFrom, RuneReader, RuneScanner, StringWriter, Writer, WriterTo 等接口，可以方便的进行读写操作。

   对象可读取数据为 buf[off : len(buf)], off 表示进度下标，lastRead 表示最后读取的一个字符所占字节数，方便 Unread* 相关操作。

   Buffer 可以通过 3 中方法初始化对象：

   ```go
   a := bytes.NewBufferString("Hello World")
   b := bytes.NewBuffer([]byte("Hello World"))
   c := bytes.Buffer{}
   
   fmt.Println(a)
   fmt.Println(b)
   fmt.Println(c)
   ```

Buffer 包含了 21 个读写相关的方法，大部分同名方法的用法与前面讲的类似，这里只讲演示其中的 3 个方法：

```go
// 读取到字节 delim 后，以字节数组的形式返回该字节及前面读取到的字节。如果遍历 b.buf 也找不到匹配的字节，则返回错误(一般是 EOF)
func (b *Buffer) ReadBytes(delim byte) (line []byte, err error)
// 读取到字节 delim 后，以字符串的形式返回该字节及前面读取到的字节。如果遍历 b.buf 也找不到匹配的字节，则返回错误(一般是 EOF)
func (b *Buffer) ReadString(delim byte) (line string, err error)
// 截断 b.buf , 舍弃 b.off+n 之后的数据。n == 0 时，调用 Reset 方法重置该对象，当 n 越界时（n < 0 || n > b.Len() ）方法会触发 panic.
func (b *Buffer) Truncate(n int)
```



















