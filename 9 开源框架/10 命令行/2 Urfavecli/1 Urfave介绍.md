**urfave/cli** 是 Go 生态中另一个非常成熟、轻量的 **CLI 框架**。

## 1 核心概念

**`cli.App`**是urfave/cli 核心对象

```go
func main() {
    app := &cli.App {
        Name: "myapp",
        Usage: "命令行示例应用"
    }
    app.Run(os.Arags)
}
```

- Name：命令名
- Usage：命令的使用介绍
- Description：命令详细描述。
- Version：版本号
- Flags：定义全局flags

## 2 核心对象：cli.Command

关键字段包括：

- Name：命令名
- Aliases：命令别名
- Usage：简短说明
- Description：详细说明
- Flags：命令级flag
- Action：执行逻辑

Urfave的flags是结构体，而不是方法的调用

```go
&cli.StringFlag{
    Name:    "name",
    Aliases: []string{"n"},
    Usage:   "用户名",
    Required: true,
}
```

## 3 示例

 ```go
 app := &cli.App{
     Name:  "app",
     Usage: "示例 CLI",
     Flags: []cli.Flag{
         &cli.StringFlag{
             Name:     "name",
             Aliases:  []string{"n"},
             Usage:    "用户名",
             Required: true,
         },
     },
     Action: func(ctx *cli.Context) error {
         fmt.Println("hello", ctx.String("name"))
         return nil
     },
 }
 
 app.Run(os.Args)
 ```

