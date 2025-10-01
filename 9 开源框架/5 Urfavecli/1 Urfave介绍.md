命令行工具

```go
func main() {
    app := &cli.App {
        Name: "myapp",
        Usage: "命令行示例应用"
    }
    app.Run(os.Arags)
}
```

Urfave/cli支持全局参数、命令行参数以及标志位等多种形式。

全局参数：

```go
app.Flags = []cli.Flag {
    &cli.StringFlag {
        Name: "config",
        Value: "default.conf",
        Usage: "指定配置路径"
    }
}
```

命令参数

 ```go
 cmd := &cli.Command {
     Name: "serve",
     Usage: "服务",
     Flags: []cli.Flag {
         &cli.IntFlag {
             Name: "port",
             Value: "8080",
             Usage: "指定端口",
         }
     },
     Action: func(c *cli.Context) err {
         port := c.Int("port")
         return nil
     }
 }
 ```

