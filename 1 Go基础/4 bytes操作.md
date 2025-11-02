### 一 定义

在Go中，`byte` 是 `uint8` 的 **别名（alias）**，代表一个 **8 位无符号整数（0~255）**。

```go
type byte = uint8
```

### 二 类型关系

| 类型     | 定义           | 用途                                        |
| -------- | -------------- | ------------------------------------------- |
| `byte`   | `uint8`        | 表示原始字节（如文件、网络、字符串等）      |
| `rune`   | `int32`        | 表示 Unicode 字符（UTF-8 编码中的一个字符） |
| `string` | 只读的字节序列 | 一般是 UTF-8 编码的文本                     |
| `[]byte` | 字节切片       | 可变的二进制或文本数据                      |

### 三 基本操作

- string与byte转换

  ```go
  s := "hello"
  b := []byte(s)
  fmt.Println(b) // [104 101 108 108 111]
  s := string(b)
  fmt.Println(s) 
  
  ```

- 是否存在某个子slice

  ```go
  func Contains(b, subslice []byte) bool
  ```

- []byte出现次数

  ```go
  // slice sep 在 s 中出现的次数（无重叠）
  func Count(s, sep []byte) int
  ```

- Runes类型转换

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

  









