### 一 介绍

Go 的 **`flag` 包** 是标准库中用于解析命令行参数的工具，支持定义、绑定、解析各种类型的命令行标志（flag）。示例如下：

```go
import (
	"flag"
	"fmt"
)

var (
	intflag  int
	strflag  string
	boolflag bool
)

func init() {
	flag.IntVar(&intflag, "intflag", 0, "int flag")
	flag.StringVar(&strflag, "strflag", "", "string flag")
	flag.BoolVar(&boolflag, "boolflag", false, "bool flag")
}
func main() {
	flag.Parse()
	fmt.Println("int flag:", intflag)
	fmt.Println("str flag:", strflag)
	fmt.Println("bool flag:", boolflag)
}
```

通过在命令行输入以下命令，执行命令，就可以对变量赋值

```cmd
go run main.go -intflag=5 
```

