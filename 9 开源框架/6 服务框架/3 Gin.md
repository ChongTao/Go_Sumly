```go
go get -u github.com/gin-gonic/gin
```

创建一个简单的Gin应用程序。

```go
func main() {
    r := gin.Default()
    r.GET("/", func(c *gin.Context) {
        c.String(200, "Hello, Gin!")
    })
    r.Run(":8080")
}
```

go run example.go