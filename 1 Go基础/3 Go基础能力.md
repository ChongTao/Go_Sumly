# 字符串操作 

string 类型可以看成是一种特殊的slice类型，以下是常见的字符串的场景。

1. 字符串比较

   ```go
   func Compare(a, b string) int // 比较两个字符串的大小
   func EqualFold(s, t string) bool // 计算s与t是否相等，忽略字母大小写
   ```

2. 是否存在某个字符或子串

   ```go
   // 子串 substr 在 s 中，返回 true
   func Contains(s, substr string) bool
   
   // chars 中任何一个 Unicode 代码点在 s 中，返回 true
   func ContainsAny(s, chars string) bool
   
   // Unicode 代码点 r 在 s 中，返回 true
   func ContainsRune(s string, r rune) bool
   ```

3. 字符串匹配

   ```go
   // 查找子串出现次数
   func Count(s, sep string) int
   ```

4. 字符串分割

   ```go
   func Fields(s string) []string
   func FieldsFunc(s string, f func(rune) bool) []string
   fmt.Printf("Fields are: %q", strings.Fields("  foo bar  baz   "))  // Fields are: ["foo" "bar" "baz"]
   fmt.Println(strings.FieldsFunc("  foo bar  baz   ", unicode.IsSpace))
   ```

   > Fields用一个或多个连续空格分隔字符串s，返回字符串的数组(slice)。如果字符串s只包含空格，则返回空列表([]string的长度为0)。

   ```go
   func Split(s, sep string) []string { return genSplit(s, sep, 0, -1) }
   func SplitAfter(s, sep string) []string { return genSplit(s, sep, len(sep), -1) }
   func SplitN(s, sep string, n int) []string { return genSplit(s, sep, 0, n) }
   func SplitAfterN(s, sep string, n int) []string { return genSplit(s, sep, len(sep), n) }
   
   fmt.Printf("%q\n", strings.Split("foo,bar,baz", ","))       // ["foo" "bar" "baz"]
   fmt.Printf("%q\n", strings.SplitAfter("foo,bar,baz", ","))  // ["foo," "bar," "baz"]
   ```

   这四个函数都是通过sep进行分割，分割[]string，如果sep为空，相当于分割一个个UTF-8字符，如`Split（"abc", ""）`，得到的是[a b c]。

   Split(s, sep)和SplitN(s, sep, -1)等价；SplitAfter(s, sep)和SplitAfterN(s, sep, -1)等价。

5. 字符串是否有某个前缀或后缀

   ```go
   // s 中是否以 prefix 开始
   func HasPrefix(s, prefix string) bool {
     return len(s) >= len(prefix) && s[0:len(prefix)] == prefix
   }
   // s 中是否以 suffix 结尾
   func HasSuffix(s, suffix string) bool {
     return len(s) >= len(suffix) && s[len(s)-len(suffix):] == suffix
   }
   ```

6. 字符或者子串在字符串中出现的位置

   ```go
   // 在 s 中查找 sep 的第一次出现，返回第一次出现的索引
   func Index(s, sep string) int
   // 在 s 中查找字节 c 的第一次出现，返回第一次出现的索引
   func IndexByte(s string, c byte) int
   // chars 中任何一个 Unicode 代码点在 s 中首次出现的位置
   func IndexAny(s, chars string) int
   // 查找字符 c 在 s 中第一次出现的位置，其中 c 满足 f(c) 返回 true
   func IndexFunc(s string, f func(rune) bool) int
   // Unicode 代码点 r 在 s 中第一次出现的位置
   func IndexRune(s string, r rune) int
   
   // 有三个对应的查找最后一次出现的位置
   func LastIndex(s, sep string) int
   func LastIndexByte(s string, c byte) int
   func LastIndexAny(s, chars string) int
   func LastIndexFunc(s string, f func(rune) bool) int
   ```

7. 字符串join操作

   将字符串数组连接起来使用join方法。

   ```go
   func Join(a []string, sep string) string
   ```

8. 字符串重复几次

   ```go
   func Repeat(s string, count int) string
   ```

9. 字符串替换

   ```go
   func Map(mapping func(rune) rune, s string) string
   ```

   Map 函数，将 s 的每一个字符按照 mapping 的规则做映射替换，如果 mapping 返回值 <0 ，则舍弃该字符。该方法只能对每一个字符做处理，但处理方式很灵活，可以方便的过滤，筛选汉字等。

   ```go
   // 用 new 替换 s 中的 old，一共替换 n 个。
   // 如果 n < 0，则不限制替换次数，即全部替换
   func Replace(s, old, new string, n int) string
   // 该函数内部直接调用了函数 Replace(s, old, new , -1)
   func ReplaceAll(s, old, new string) string
   ```

10. 大小写转换

    ```go
    func ToLower(s string) string
    func ToLowerSpecial(c unicode.SpecialCase, s string) string
    func ToUpper(s string) string
    func ToUpperSpecial(c unicode.SpecialCase, s string) string
    ```

11. 标题处理

    ```go
    // 将s每个单词的首个字母大写
    func Title(s string) string
    // 将s的每个字母大写
    func ToTitle(s string) string
    // 将s的每个字母大写，并将一些特殊字母转换为对于的特殊大小字母
    func ToTitleSpecial(c unicode.SpecialCase, s string) string
    ```

12. 修剪

    ```go
    // 将 s 左侧和右侧中匹配 cutset 中的任一字符的字符去掉
    func Trim(s string, cutset string) string
    // 将 s 左侧的匹配 cutset 中的任一字符的字符去掉
    func TrimLeft(s string, cutset string) string
    // 将 s 右侧的匹配 cutset 中的任一字符的字符去掉
    func TrimRight(s string, cutset string) string
    // 如果 s 的前缀为 prefix 则返回去掉前缀后的 string , 否则 s 没有变化。
    func TrimPrefix(s, prefix string) string
    // 如果 s 的后缀为 suffix 则返回去掉后缀后的 string , 否则 s 没有变化。
    func TrimSuffix(s, suffix string) string
    // 将 s 左侧和右侧的间隔符去掉。常见间隔符包括：'\t', '\n', '\v', '\f', '\r', ' ', U+0085 (NEL)
    func TrimSpace(s string) string
    // 将 s 左侧和右侧的匹配 f 的字符去掉
    func TrimFunc(s string, f func(rune) bool) string
    // 将 s 左侧的匹配 f 的字符去掉
    func TrimLeftFunc(s string, f func(rune) bool) string
    // 将 s 右侧的匹配 f 的字符去掉
    func TrimRightFunc(s string, f func(rune) bool) string
    ```



# byte操作

在Go中，`byte` 是 `uint8` 的 **别名（alias）**，代表一个 **8 位无符号整数（0~255）**。

```go
type byte = uint8
```

byte与其它类型关系如下：

| 类型     | 定义           | 用途                                        |
| -------- | -------------- | ------------------------------------------- |
| `byte`   | `uint8`        | 表示原始字节（如文件、网络、字符串等）      |
| `rune`   | `int32`        | 表示 Unicode 字符（UTF-8 编码中的一个字符） |
| `string` | 只读的字节序列 | 一般是 UTF-8 编码的文本                     |
| `[]byte` | 字节切片       | 可变的二进制或文本数据                      |

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


# 占位符

Go 语言中的 **占位符（Verb）** 主要用于 `fmt` 系列函数（如 `fmt.Printf`、`fmt.Sprintf`、`fmt.Fprintf` 等）中，用来格式化输出各种类型的数据，常见占位符如下：

| 类型     | 常用占位符                   | 示例           |
| -------- | ---------------------------- | -------------- |
| 通用     | `%v, %+v, %#v, %T, %%`       | 通用值与类型   |
| 整数     | `%d, %b, %o, %x, %X, %c, %U` | 数字格式化     |
| 浮点     | `%f, %.2f, %e, %E, %g`       | 小数与科学计数 |
| 字符串   | `%s, %q, %x, % X`            | 文本与编码     |
| 指针     | `%p`                         | 内存地址       |
| 宽度控制 | `%6d, %-6d, %06d, %8.2f`     | 对齐与填充     |

示例：

```go
type user struct {
	name string
}

func main() {
	u := user{name: "Go"}
	fmt.Printf("%+v\n", u)       // 格式化输出 {name:Go}
	fmt.Printf("%#v\n", u)       // 输出值的Go语言表示方法 main.user{name:"Go"}
	fmt.Printf("%T\n", u)        // 输出值的类型的Go语言表示 main.user
	fmt.Printf("%t\n", true)     // 输出值的 true 或 false
	fmt.Printf("%b\n", 1024)     //二进制表示
	fmt.Printf("%c\n", 11111111) //数值对应的 Unicode 编码字符
	fmt.Printf("%d\n", 10)       //十进制表示
	fmt.Printf("%o\n", 8)        //八进制表示
	fmt.Printf("%q\n", 22)       //转化为十六进制并附上单引号
	fmt.Printf("%x\n", 1223)     //十六进制表示，用a-f表示
	fmt.Printf("%X\n", 1223)     //十六进制表示，用A-F表示
	fmt.Printf("%U\n", 1233)     //Unicode表示
}
```

# struct的tag类型

在 Go 中，结构体字段可以带一个或多个「标签（tag）」：

```go
type StructName struct {
    FieldName FieldType `key1:"value1" key2:"value2"`
}
```

这些标签是供**反射机制（reflect）**读取的元信息，Go 自身不会解析或验证内容。不同的库（`encoding/json`, `yaml`, `gorm`, `validator` 等）会根据各自的 `key` 去解释相应部分。

| 标签       | 来源/用途                   | 示例                              | 说明                        |
| ---------- | --------------------------- | --------------------------------- | --------------------------- |
| `json`     | 标准库 `encoding/json`      | `json:"user_id,omitempty"`        | 控制 JSON 的序列化/反序列化 |
| `xml`      | 标准库 `encoding/xml`       | `xml:"User>ID"`                   | 控制 XML 标签和层级         |
| `yaml`     | 第三方库 `gopkg.in/yaml.v3` | `yaml:"user_name"`                | 控制 YAML 序列化字段名      |
| `form`     | Web 框架（如 Gin）          | `form:"username"`                 | 控制表单绑定字段名          |
| `db`       | ORM/SQLx                    | `db:"user_id"`                    | 控制数据库字段名映射        |
| `gorm`     | GORM 框架                   | `gorm:"primaryKey;autoIncrement"` | 定义数据库列约束            |
| `validate` | go-playground/validator     | `validate:"required,min=3"`       | 字段验证规则                |
| `bson`     | MongoDB 驱动                | `bson:"_id,omitempty"`            | BSON 数据映射               |

所有这些标签最终都会被反射获取：

```go
t := reflect.TypeOf(User{})
f, _ := t.FieldByName("Name")
fmt.Println(f.Tag.Get("json")) // 读取 json 标签
```

`json` 标签是最常用的一种，主要用于控制字段与 JSON 编解码时的映射关系。它的形式是

```go
`json:"name[,option1,option2,...]"`
```

- `"name"` 是 JSON 字段名；

- 逗号后可以带一些选项（如 `omitempty`, `string`）。

| 标签写法                | 说明                                       | 示例                                | 输出效果                   |
| ----------------------- | ------------------------------------------ | ----------------------------------- | -------------------------- |
| `json:"id"`             | 指定字段对应的JSON 名                      | `ID int json:"id"`                  | `{ "id": 1 }`              |
| `json:"name,omitempty"` | 值为零值时忽略该字段                       | `Name string json:"name,omitempty"` | 当 Name 为空字符串时不输出 |
| `json:"-"`              | 完全忽略该字段                             | `Password string json:"-"`          | 永远不出现在 JSON 中       |
| `json:",omitempty"`     | 保留字段名默认转换规则（小写），但忽略零值 | `Score int json:",omitempty"`       | 输出 `score` 字段（非零）  |
| `json:"age,string"`     | 数字字段以字符串编码/解码                  | `Age int json:"age,string"`         | 输出 `"age":"18"`          |
| `json:"-"` + 其他       | 无效，`-` 优先级最高，字段始终被忽略       | —                                   | —                          |

序列化示例：

```go
package main

import (
    "encoding/json"
    "fmt"
)

type User struct {
    ID       int     `json:"id"`
    Name     string  `json:"name,omitempty"`
    Age      int     `json:"age,string"`
    Password string  `json:"-"`           // 永远不输出
    Score    float64 `json:",omitempty"`  // 使用默认名 "score"
}

func main() {
    u := User{ID: 1, Age: 25, Score: 0}
    data, _ := json.Marshal(u)
    fmt.Println(string(data))   // {"id":1,"age":"25"}
}
```

反序列化示例：

```go
jsonData := `{"id":10, "age":"30"}`
var u User
json.Unmarshal([]byte(jsonData), &u)
fmt.Println(u)  // {ID:10 Name:"" Age:30 Password:"" Score:0}
```

> 字段首字母小写不生效，`encoding/json` 只能访问导出字段（首字母大写），如果不添加json字段，json字段名和struct结构体内字段原名一致。
