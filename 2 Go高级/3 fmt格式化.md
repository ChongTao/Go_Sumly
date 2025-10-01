fmt 包实现了格式化I/O函数，类似于C的 printf 和 scanf. 格式“占位符”衍生自C，但比C更简单。

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

## 占位符

| 占位符 | 描述       | 示例                                                         |
| ------ | ---------- | ------------------------------------------------------------ |
| %v     | 默认格式   | Printf("%v", site)，在打印结构体时，“加号”标记（%+v）会添加字段名 |
| %#v    | Go表示     | Printf("#v", site)  main.user{name:"Go"}                     |
| %T     | Go表示     | main.user                                                    |
| %t     | true/false |                                                              |
| %s     | 字符串输出 |                                                              |

## Scanning

Scan、Scanf 和 Scanln 从 os.Stdin 中读取； Fscan、Fscanf 和 Fscanln 从指定的 io.Reader 中读取； Sscan、Sscanf 和 Sscanln 从实参字符串中读取。 Scanln、Fscanln 和 Sscanln 在换行符处停止扫描，且需要条目紧随换行符之后； Scanf、Fscanf 和 Sscanf 需要输入换行符来匹配格式中的换行符；其它函数则将换行符视为空格。

Scanf、Fscanf 和 Sscanf 根据格式字符串解析实参，类似于 Printf。例如，%x 会将一个整数扫描为十六进制数，而 %v 则会扫描该值的默认表现格式。

## Print序列函数

```go
func Print(a ...interface{}) (n int, err error) {
    return Fprint(os.Stdout, a...)
}
```

