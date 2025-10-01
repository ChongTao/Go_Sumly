# 1 Gin 特性

- **快速**：路由不使用反射，基于Radix树，内存占用少。
- **中间件**：HTTP请求，可先经过一系列中间件处理，例如：Logger，Authorization，GZIP等。这个特性和 NodeJs 的 `Koa` 框架很像。中间件机制也极大地提高了框架的可扩展性。
- **异常处理**：服务始终可用，不会宕机。Gin 可以捕获 panic，并恢复。而且有极为便利的机制处理HTTP请求过程中发生的错误。
- **JSON**：Gin可以解析并验证请求的JSON。这个特性对`Restful API`的开发尤其有用。
- **路由分组**：例如将需要授权和不需要授权的API分组，不同版本的API分组。而且分组可嵌套，且性能不受影响。
- **渲染内置**：原生支持JSON，XML和HTML的渲染。

# 2 安装Go & Gin

```shell
go get -u github.com/gin-gonic/gin
```

在代码中导入

```go
import "github.com/gin-gonic/gin"
```

```go
func main() {

	// 定义一个实例
	r := gin.Default()
	//声明一个路由
	r.GET("/", func(c *gin.Context) {
		c.String(200, "hello world")
	})
	// 程序运行，监听端口号8080
	r.Run(":8080")
}
```

## 2.1 路由

路由方法有 **GET, POST, PUT, PATCH, DELETE** 和 **OPTIONS**，还有**Any**，可匹配以上任意类型的请求。

```go
// 无参数
r.GET("/", func(c *gin.Context)) {
	c.String(http.StatusOK, "Who are you?")
}

// 路径参数
r.GET("/user/:name", func(c *gin.Context) {
	name := c.Param("name")
	c.String(http.StatusOK, "Hello %s", name)
})

// query参数
// 匹配users?name=xxx&role=xxx，role可选
r.GET("/users", func(c *gin.Context) {
	name := c.Query("name")
	role := c.DefaultQuery("role", "teacher")
	c.String(http.StatusOK, "%s is a %s", name, role)
})

// POST请求
r.POST("/form", func(c *gin.Context) {
	username := c.PostForm("username")
	password := c.DefaultPostForm("password", "000000") // 可设置默认值

	c.JSON(http.StatusOK, gin.H{
		"username": username,
		"password": password,
	})
})

// POST Map参数
r.POST("/post", func(c *gin.Context) {
	ids := c.QueryMap("ids")
	names := c.PostFormMap("names")

	c.JSON(http.StatusOK, gin.H{
		"ids":   ids,
		"names": names,
	})
})

// 重定向
r.GET("/redirect", func(c *gin.Context) {
    c.Redirect(http.StatusMovedPermanently, "/index")
})

r.GET("/goindex", func(c *gin.Context) {
	c.Request.URL.Path = "/"
	r.HandleContext(c)
})
```

分组路径，

```go
// group routes 分组路由
defaultHandler := func(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"path": c.FullPath(),
	})
}
// group: v1
v1 := r.Group("/v1")
{
	v1.GET("/posts", defaultHandler)
	v1.GET("/series", defaultHandler)
}
// group: v2
v2 := r.Group("/v2")
{
	v2.GET("/posts", defaultHandler)
	v2.GET("/series", defaultHandler)
}
```

上传文件

```go
	r.POST("/upload", func(c *gin.Context) {
		file, _ := c.FormFile("file")
		c.String(http.StatusOK, file.Filename)
	})
```

