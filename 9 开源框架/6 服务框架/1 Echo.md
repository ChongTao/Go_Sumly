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

Echo 提倡通过中间件或处理程序 (handler) 返回 HTTP 错误集中处理。集中式错误处理程序允许我们从统一位置将错误记录到外部服务，并向客户端发送自定义 HTTP 响应。

你可以返回一个标准的 `error` 或者 `echo.*HTTPError`。

```go
    e.Use(func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            return echo.NewHTTPError(http.StatusUnauthorized)
        }
    })
}
```

可以使用 `Context#Bind(i interface{})` 将请求内容体绑定至 go 的结构体。默认绑定器支持基于 Content-Type 标头包含 application/json，application/xml 和 application/x-www-form-urlencoded 的数据。

```go
// User
User struct {
  Name  string `json:"name" form:"name" query:"name"`
  Email string `json:"email" form:"email" query:"email"`
}

// Handler
func(c echo.Context) (err error) {
  u := new(User)
  if err = c.Bind(u); err != nil {
    return
  }
  return c.JSON(http.StatusOK, u)
}

type CustomBinder struct {}

func (cb *CustomBinder) Bind(i interface{}, c echo.Context) (err error) {
    // 你也许会用到默认的绑定器
    db := new(echo.DefaultBinder)
    if err = db.Bind(i, c); err != echo.ErrUnsupportedMediaType {
        return
    }

    return
}

// 查询参数
func(c echo.Context) error {
    name := c.QueryParam("name")
    return c.String(http.StatusOK, name)
})
```

响应

```go
func(c echo.Context) error {
  return c.String(http.StatusOK, "Hello, World!")
}
```



基于 [radix tree](http://en.wikipedia.org/wiki/Radix_tree) ，Echo 的路由查询速度非常快。路由使用 [sync pool](https://golang.org/pkg/sync/#Pool) 来重用内存，实现无 GC 开销下的零动态内存分配。

通过特定的 HTTP 方法，url 路径和一个匹配的处理程序 (handler) 可以注册一个路由。例如，下面的代码则展示了一个注册路由的例子：它包括 `Get` 的访问方式， `/hello` 的访问路径，以及发送 `Hello World` HTTP 响应的处理程序。

```go
// 业务处理
func hello(c echo.Context) error {
      return c.String(http.StatusOK, "Hello, World!")
}

// 路由
e.GET("/hello", hello)
```

你可以用 `Echo.Any(path string, h Handler)` 来为所有的 HTTP 方法发送注册 处理程序(handler)；如果仅需要为某个特定的方法注册路由，可使用 `Echo.Match(methods []string, path string, h Handler)`。

Echo 通过 `func(echo.Context) error` 定义 handler 方法，其中 `echo.Context` 已经内嵌了 HTTP 请求和响应接口。



## 1.1. 中间件

中间件是一个函数，嵌入在HTTP 的请求和响应之间。它可以获得 `Echo#Context` 对象用来进行一些特殊的操作， 比如记录每个请求或者统计请求数。

Action的处理在所有的中间件运行完成之后。

### 1.1.1. 中间件级别

#### Root Level (Before router)

`Echo#Pre()` 用于注册一个在路由执行之前运行的中间件，可以用来修改请求的一些属性。比如在请求路径结尾添加或者删除一个'/'来使之能与路由匹配。

下面的这几个内建中间件应该被注册在这一级别：

- AddTrailingSlash
- RemoveTrailingSlash
- MethodOverride

*注意*: 由于在这个级别路由还没有执行，所以这个级别的中间件不能调用任何 `echo.Context` 的 API。

#### Root Level (After router)

大部分时间你将用到 `Echo#Use()` 在这个级别注册中间件。 这个级别的中间件运行在路由处理完请求之后，可以调用所有的 `echo.Context` API。

#### Group Level

当在路由中创建一个组的时候，可以为这个组注册一个中间件。例如，给 admin 这个组注册一个 BasicAuth 中间件。

*用法*

```go
e := echo.New()
admin := e.Group("/admin", middleware.BasicAuth())
```

也可以在创建组之后用 `admin.Use()`注册该中间件。

#### Route Level

当你创建了一个新的路由，可以选择性的给这个路由注册一个中间件。

*用法*

```go
e := echo.New()
e.GET("/", <Handler>, <Middleware...>)
```

## 1.1. Basic Auth (基本认证) 中间件

Basic Auth 中间件提供了 HTTP 的基本认证方式。

- 对于有效的请求，则继续调用下一个处理程序 (handler) 。
- 对于丢失或无效的请求，则返回 "401 - Unauthorized" 响应。

*用法*

```go
e.Use(middleware.BasicAuth(func(username, password string, c echo.Context) (bool, error) {
    if username == "joe" && password == "secret" {
        return true, nil
    }
    return false, nil
}))
```

### 1.1.1. 自定义配置

*用法*

```go
e.Use(middleware.BasicAuthWithConfig(middleware.BasicAuthConfig{}))
```

### 1.1.2. 配置

```go
BasicAuthConfig struct {
  // Skipper 定义了一个跳过中间件的函数
  Skipper Skipper

  // Validator 是一个用来验证 BasicAuth 是否合法的函数
  // Validator 是必须的.
  Validator BasicAuthValidator

  // Realm 是一个用来定义 BasicAuth 的 Realm 属性的字符串
  // 默认值是 "Restricted"
  Realm string
}
```

*默认配置*

```go
DefaultBasicAuthConfig = BasicAuthConfig{
    Skipper: defaultSkipper,
}
```

Body dump 中间件通常在调试 / 记录的情况下被使用，它可以捕获请求并调用已注册的处理程序 (handler) 响应有效负载。然而，当您的请求 / 响应有效负载很大时（例如上传 / 下载文件）需避免使用它；但如果避免不了，可在 skipper 函数中为端点添加异常。

*用法*

```go
e := echo.New()
e.Use(middleware.BodyDump(func(c echo.Context, reqBody, resBody []byte) {
}))
```

自定义配置

```go
e := echo.New()
e.Use(middleware.BodyDumpWithConfig(middleware.BodyDumpConfig{}))

BodyDumpConfig struct {
  // Skipper 定义了一个跳过中间件的函数
  Skipper Skipper

  // Handler 接收请求和响应有效负载
  // Required.
  Handler BodyDumpHandler
}
```



Body Limit 中间件用于设置请求体的最大长度，如果请求体的大小超过了限制值，则返回 "413 － Request Entity Too Large" 响应。该限制的判断是根据 `Content-Length` 请求标头和实际内容确定的，这使其尽可能的保证安全。

限制可以指定 `4x` 或者 `4xB`，x是 "K, M, G, T, P" 计算机单位的倍数之一。

```go
e := echo.New()
e.Use(middleware.BodyLimit("2M"))

e := echo.New()
e.Use(middleware.BodyLimitWithConfig(middleware.BodyLimitConfig{}))

DefaultBodyLimitConfig = BodyLimitConfig{
  Skipper: defaultSkipper,
}
```

CORS (Cross-origin resource sharing) 中间件实现了 [CORS](http://www.w3.org/TR/cors) 的标准。CORS为Web服务器提供跨域访问控制，从而实现安全的跨域数据传输。

```go
e.Use(middleware.CORS())

e := echo.New()
e.Use(middleware.CORSWithConfig(middleware.CORSConfig{
  AllowOrigins: []string{"https://labstack.com", "https://labstack.net"},
  AllowHeaders: []string{echo.HeaderOrigin, echo.HeaderContentType, echo.HeaderAccept},
}))

// CORSConfig defines the config for CORS middleware.
CORSConfig struct {
  // Skipper defines a function to skip middleware.
  Skipper Skipper

  // AllowOrigin defines a list of origins that may access the resource.
  // Optional. Default value []string{"*"}.
  AllowOrigins []string `json:"allow_origins"`

  // AllowMethods defines a list methods allowed when accessing the resource.
  // This is used in response to a preflight request.
  // Optional. Default value DefaultCORSConfig.AllowMethods.
  AllowMethods []string `json:"allow_methods"`

  // AllowHeaders defines a list of request headers that can be used when
  // making the actual request. This in response to a preflight request.
  // Optional. Default value []string{}.
  AllowHeaders []string `json:"allow_headers"`

  // AllowCredentials indicates whether or not the response to the request
  // can be exposed when the credentials flag is true. When used as part of
  // a response to a preflight request, this indicates whether or not the
  // actual request can be made using credentials.
  // Optional. Default value false.
  AllowCredentials bool `json:"allow_credentials"`

  // ExposeHeaders defines a whitelist headers that clients are allowed to
  // access.
  // Optional. Default value []string{}.
  ExposeHeaders []string `json:"expose_headers"`

  // MaxAge indicates how long (in seconds) the results of a preflight request
  // can be cached.
  // Optional. Default value 0.
  MaxAge int `json:"max_age"`
}

// 默认配置
DefaultCORSConfig = CORSConfig{
  Skipper:      defaultSkipper,
  AllowOrigins: []string{"*"},
  AllowMethods: []string{echo.GET, echo.HEAD, echo.PUT, echo.PATCH, echo.POST, echo.DELETE},
}
```

CSRF (Cross-site request forgery) 跨域请求伪造，也被称为 **one-click attack** 或者 **session riding**，通常缩写为 **CSRF** 或者 **XSRF**， 是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。[[1\]](https://zh.wikipedia.org/wiki/跨站请求伪造#cite_note-Ristic-1) 跟[跨网站脚本](https://zh.wikipedia.org/wiki/跨網站指令碼) (XSS) 相比，**XSS** 利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任。

```go
e.Use(middleware.CSRF())
e := echo.New()
e.Use(middleware.CSRFWithConfig(middleware.CSRFConfig{
  TokenLookup: "header:X-XSRF-TOKEN",
}))

// CSRFConfig defines the config for CSRF middleware.
CSRFConfig struct {
  // Skipper defines a function to skip middleware.
  Skipper Skipper

  // TokenLength is the length of the generated token.
  TokenLength uint8 `json:"token_length"`
  // Optional. Default value 32.

  // TokenLookup is a string in the form of "<source>:<key>" that is used
  // to extract token from the request.
  // Optional. Default value "header:X-CSRF-Token".
  // Possible values:
  // - "header:<name>"
  // - "form:<name>"
  // - "query:<name>"
  TokenLookup string `json:"token_lookup"`

  // Context key to store generated CSRF token into context.
  // Optional. Default value "csrf".
  ContextKey string `json:"context_key"`

  // Name of the CSRF cookie. This cookie will store CSRF token.
  // Optional. Default value "csrf".
  CookieName string `json:"cookie_name"`

  // Domain of the CSRF cookie.
  // Optional. Default value none.
  CookieDomain string `json:"cookie_domain"`

  // Path of the CSRF cookie.
  // Optional. Default value none.
  CookiePath string `json:"cookie_path"`

  // Max age (in seconds) of the CSRF cookie.
  // Optional. Default value 86400 (24hr).
  CookieMaxAge int `json:"cookie_max_age"`

  // Indicates if CSRF cookie is secure.
  // Optional. Default value false.
  CookieSecure bool `json:"cookie_secure"`

  // Indicates if CSRF cookie is HTTP only.
  // Optional. Default value false.
  CookieHTTPOnly bool `json:"cookie_http_only"`
}

DefaultCSRFConfig = CSRFConfig{
  Skipper:      defaultSkipper,
  TokenLength:  32,
  TokenLookup:  "header:" + echo.HeaderXCSRFToken,
  ContextKey:   "csrf",
  CookieName:   "_csrf",
  CookieMaxAge: 86400,
}
```

[Casbin](https://github.com/casbin/casbin) 是 Go 下的强大而高效的开源访问控制库，它为基于各种模型的授权提供支持。到目前为止，Casbin 支持的访问控制模型如下：

- ACL (访问控制列表)
- 超级用户下的ACL
- 没有用户的 ACL： 对于没有身份验证或用户登录的系统尤其有用。
- 没有资源的ACL：过使用 write-article , read-log 等权限，某些方案可以应对一类资源而不是单个资源，它不控制对特定文章或日志的访问。
- RBAC (基于角色的访问控制)
- 具有资源角色的 RBAC： 用户和资源可以同时具有角色 (或组)。
- 具有域 / 租户 (tenants) 的 RBAC ：用户可以针对不同的域 / 租户 (tenants) 具有不同的角色集。
- ABAC (基于属性的访问控制)
- RESTful
- Deny-override：支持允许和拒绝授权，否认覆盖允许。

```go
import (
  "github.com/casbin/casbin"
  casbin_mw "github.com/labstack/echo-contrib/casbin"
)

e := echo.New()
e.Use(casbin_mw.Middleware(casbin.NewEnforcer("casbin_auth_model.conf", "casbin_auth_policy.csv")))

// 自定义配置
e := echo.New()
ce := casbin.NewEnforcer("casbin_auth_model.conf", "")
ce.AddRoleForUser("alice", "admin")
ce.AddPolicy(...)
e.Use(casbin_mw.MiddlewareWithConfig(casbin_mw.Config{
  Enforcer: ce,
}))

// Config defines the config for CasbinAuth middleware.
Config struct {
  // Skipper defines a function to skip middleware.
  Skipper middleware.Skipper

  // Enforcer CasbinAuth main rule.
  // Required.
  Enforcer *casbin.Enforcer
}

DefaultConfig = Config{
  Skipper: middleware.DefaultSkipper,
}
```

Gzip 中间件使用 gzip 方案来对 HTTP 响应进行压缩。

```go
e := echo.New()
e.Use(middleware.GzipWithConfig(middleware.GzipConfig{
  Level: 5,
}))

GzipConfig struct {
  // Skipper defines a function to skip middleware.
  Skipper Skipper

  // Gzip compression level.
  // Optional. Default value -1.
  Level int `json:"level"`
}
```

JWT 提供了一个 JSON Web Token (JWT) 认证中间件。

- 对于有效的 token，它将用户置于上下文中并调用下一个处理程序。
- 对于无效的 token，它会发送 "401 - Unauthorized" 响应。
- 对于丢失或无效的 `Authorization` 标头，它会发送 "400 - Bad Request" 。



```go
e.Use(middleware.JWTWithConfig(middleware.JWTConfig{
  SigningKey: []byte("secret"),
  TokenLookup: "query:token",
}))

// JWTConfig defines the config for JWT middleware.
JWTConfig struct {
  // Skipper defines a function to skip middleware.
  Skipper Skipper

  // Signing key to validate token.
  // Required.
  SigningKey interface{}

  // Signing method, used to check token signing method.
  // Optional. Default value HS256.
  SigningMethod string

  // Context key to store user information from the token into context.
  // Optional. Default value "user".
  ContextKey string

  // Claims are extendable claims data defining token content.
  // Optional. Default value jwt.MapClaims
  Claims jwt.Claims

  // TokenLookup is a string in the form of "<source>:<name>" that is used
  // to extract token from the request.
  // Optional. Default value "header:Authorization".
  // Possible values:
  // - "header:<name>"
  // - "query:<name>"
  // - "cookie:<name>"
  TokenLookup string

  // AuthScheme to be used in the Authorization header.
  // Optional. Default value "Bearer".
  AuthScheme string
}
```

Logger 中间件记录有关每个 HTTP 请求的信息。

```go
e.Use(middleware.LoggerWithConfig(middleware.LoggerConfig{
  Format: "method=${method}, uri=${uri}, status=${status}\n",
}))
LoggerConfig struct {
  // Skipper 定义了一个跳过中间件的函数.
  Skipper Skipper

  // 日志的格式可以使用下面的标签定义。:
  //
  // - time_unix
  // - time_unix_nano
  // - time_rfc3339
  // - time_rfc3339_nano
  // - id (Request ID - Not implemented)
  // - remote_ip
  // - uri
  // - host
  // - method
  // - path
  // - referer
  // - user_agent
  // - status
  // - latency (In microseconds)
  // - latency_human (Human readable)
  // - bytes_in (Bytes received)
  // - bytes_out (Bytes sent)
  // - header:<name>
  // - query:<name>
  // - form:<name>
  //
  // 例如 "${remote_ip} ${status}"
  //
  // 可选。默认值是 DefaultLoggerConfig.Format.
  Format string `json:"format"`

  // Output 是记录日志的位置。
  // 可选。默认值是 os.Stdout.
  Output io.Writer
}

// 默认配置
DefaultLoggerConfig = LoggerConfig{
  Skipper: defaultSkipper,
  Format: `{"time":"${time_rfc3339_nano}","remote_ip":"${remote_ip}","host":"${host}",` +
    `"method":"${method}","uri":"${uri}","status":${status}, "latency":${latency},` +
    `"latency_human":"${latency_human}","bytes_in":${bytes_in},` +
    `"bytes_out":${bytes_out}}` + "\n",
  Output: os.Stdout
}
```

Proxy 提供 HTTP / WebSocket 反向代理中间件。它使用已配置的负载平衡技术将请求转发到上游服务器。

```go
url1, err := url.Parse("http://localhost:8081")
if err != nil {
  e.Logger.Fatal(err)
}
url2, err := url.Parse("http://localhost:8082")
if err != nil {
  e.Logger.Fatal(err)
}
e.Use(middleware.Proxy(&middleware.RoundRobinBalancer{
  Targets: []*middleware.ProxyTarget{
    {
      URL: url1,
    },
    {
      URL: url2,
    },
  },
}))

e := echo.New()
e.Use(middleware.ProxyWithConfig(middleware.ProxyConfig{}))

// ProxyConfig defines the config for Proxy middleware.
ProxyConfig struct {
  // Skipper defines a function to skip middleware.
  Skipper Skipper

  // Balancer defines a load balancing technique.
  // Required.
  // Possible values:
  // - RandomBalancer
  // - RoundRobinBalancer
  Balancer ProxyBalancer
}
```

Recover 中间件从 panic 链中的任意位置恢复程序， 打印堆栈的错误信息，并将错误集中交给 [HTTPErrorHandler](http://go-echo.org/guide/customization/) 处理。

```go
e := echo.New()
e.Use(middleware.RecoverWithConfig(middleware.RecoverConfig{
  StackSize:  1 << 10, // 1 KB
}))

RecoverConfig struct {
  // Skipper defines a function to skip middleware.
  Skipper Skipper

  // Size of the stack to be printed.
  // Optional. Default value 4KB.
  StackSize int `json:"stack_size"`

  // DisableStackAll disables formatting stack traces of all other goroutines
  // into buffer after the trace for the current goroutine.
  // Optional. Default value false.
  DisableStackAll bool `json:"disable_stack_all"`

  // DisablePrintStack disables printing stack trace.
  // Optional. Default value as false.
  DisablePrintStack bool `json:"disable_print_stack"`
}

DefaultRecoverConfig = RecoverConfig{
  Skipper:           defaultSkipper,
  StackSize:         4 << 10, // 4 KB
  DisableStackAll:   false,
  DisablePrintStack: false,
}
```

Request ID 中间件为请求生成唯一的 ID。

```go
e.Use(middleware.RequestIDWithConfig(middleware.RequestIDConfig{
  Generator: func() string {
    return customGenerator()
  },
}))
```











