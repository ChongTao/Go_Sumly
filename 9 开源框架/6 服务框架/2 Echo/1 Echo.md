# 1 Echo

**Echo** 是一个高性能、极度工程化的 Go Web 框架。其设计理念是统一抽象，内置工程能力。相比与Gin：

- Handler 统一返回 `error`
-  Context 抽象彻底
-  参数绑定 / 校验 / 错误处理是一等公民
-  官方内置大量中间件
-  开箱即用，少“拼框架”

# 2 核心对象

Echo的核心对象：

- echo.Echo：应用实例（路由、配置、启动）
- echo.Context：请求的上下文
- echo.HTTPError：统一错误管理

示例：

```go
import (
    "net/http"

    "github.com/labstack/echo"
)

func main() {
    e := echo.New()
    e.GET("/", func(c echo.Context) error {
        return c.String(http.StatusOK, "Hello, World!")
    })
    e.Logger.Fatal(e.Start(":1323"))
}
```

其中`echo.New()`实例化一个对象

```go
type Echo struct {
    Debug       bool                   // 是否开启debug模型
    Validator   Validator              // 全局参数校验器
    HTTPErrorHandler HTTPErrorHandler  // 错误处理核心
    Binder      Binder                 // 请求数据绑定到结构体
}
```

启动方法包括

```go
e.Start(":8080")
e.StartTLS(":8443", cert, key)
e.Shutdown(ctx)
```

# 3 路由

1. 注册路由

   ```go
   e.GET("/users/:id", handler)
   e.POST("/users", handler)
   ```

2. 路由分组

   ```go
   api := e.Group("/api")
   api.Use(middleware.JWT(...))
   api.GET("/users", handler)
   ```

# 4 echo.Context

Echo的Context可以看做是Gin的Context + Controller。

1. 请求数据读取

   ```go
   // Path参数
   id := c.Param("id")
   
   // Query参数
   q := c.QueryParam("q")
   
   // Header 
   token := c.Request().Header.Get("Authorization")
   ```

2. Body绑定

   ```go	
   var req CreateUserReq
   c.Bind(&req)
   ```

3. 响应输出

   ```go
   c.String(200, "ok")
   c.JSON(200, data)
   c.NoContent(204)
   ```

4. Context数据传递

   ```go
   c.Set("user", user)
   c.Get("user")
   ```

# 5 错误处理机制

1. 错误抛出

   ```go
   echo.NewHTTPError(400, "invalid param")
   ```

2. 错误响应

   ```go
   {
     "message": "invalid param"
   }
   ```

3. 自定义全局错误处理

   ```go
   e.HTTPErrorHandler = func(err error, c echo.Context) {
       if he, ok := err.(*echo.HTTPError); ok {
           c.JSON(he.Code, map[string]string{
               "error": he.Message.(string),
           })
           return
       }
       c.JSON(500, "internal error")
   }
   ```

# 6 中间件

1. 自带中间件

   ```go
   middleware.Logger()
   middleware.Recover()
   middleware.CORS()
   middleware.JWT()
   ```

2. 自定义中间件

   ```go
   func MyMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
       return func(c echo.Context) error {
           // before
           err := next(c)
           // after
           return err
       }
   }
   ```
