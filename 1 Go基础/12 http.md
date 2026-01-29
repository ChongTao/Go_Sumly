# 1 Http

在 Go 语言中，**HTTP 功能主要由标准库 `net/http` 提供**，它可以用于处理http请求、响应等功能。

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello, Go HTTP!")
    })

    http.ListenAndServe(":8080", nil)
}
```

- `http.HandleFunc`：注册路由和处理函数
- ListenAndServe(":8080", nil)：启动服务器

上述的流程可以简化成：创建TCP Listener -> 接受客户端连接 -> 每个连接创建goroutine ->解析HTTP请求->调用Handler->返回响应。

http有四个核心类型：

| 类型                  | 作用        |
| :-------------------- | :---------- |
| `http.Server`         | HTTP 服务器 |
| `http.Request`        | HTTP 请求   |
| `http.ResponseWriter` | HTTP 响应   |
| `http.Handler`        | 请求处理器  |

## 1.1 Http Handler

Go使用**Handler**接口中处理请求：

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

业务可以自定义Handler:

```go
type MyHandler struct{}

func (h MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("自定义 Handler"))
}

func main() {
    http.ListenAndServe(":8080", MyHandler{})
}
```

## 1.2 路由

```go
http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello"))
})

http.HandleFunc("/login", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Login"))
})
```

匹配规则：

- 更长的路径优先
- 静态路径 > 前缀匹配
- `/api/` 会匹配 `/api/x`, `/api/user/123` 等

## 1.3 Request

```go
type Request struct {
    Method string          // 请求的方法 http.Methodxxx
    URL    *url.URL
    Header Header
    Body   io.ReadCloser
    Form   url.Values
    PostForm url.Values
}
```

请求报文读取

```go
var req struct {
    Name string `json:"name"`
}

json.NewDecoder(r.Body).Decode(&req)
```

## 1.4 ResponseWriter

```go
// 写Header
w.WriteHeader(http.StatusCreated)

// 写响应体
w.Write([]byte("OK"))
```

## 1.5 中间件

```go
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 前置逻辑
        next.ServeHTTP(w, r)
        // 后置逻辑
    })
}
```

## 1.6 Client

```go
client := &http.Client{
    Timeout: 5 * time.Second,
}

req, _ := http.NewRequest("POST", url, body)
req.Header.Set("Authorization", "Bearer token")

resp, _ := client.Do(req)
defer resp.Body.Close()
```



# 2 httptest

`httptest` 是 Go 标准库 `net/http/httptest` 包，用于在本地 **模拟 HTTP 服务器、HTTP 请求与响应环境**，非常适合用于：

- HTTP 服务器接口的单元测试

- HTTP 客户端请求的单元测试
- 不依赖真实网络环境的测试
- 捕获/检查 HTTP 响应内容

- Mock 下游 HTTP 服务

## 2.1. httptest.NewRecorder

- 模拟一个 HTTP ResponseWriter

- 用于测试 HTTP handler

类似一个“假的响应对象”，用于读取、断言 handler 响应的

```go
func TestHandler(t *testing.T) {
    req := httptest.NewRequest("GET", "/hello", nil)
    w := httptest.NewRecorder()

    HelloHandler(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("unexpected status: %v", w.Code)
    }

    if w.Body.String() != "hello world" {
        t.Fatalf("unexpected body: %v", w.Body.String())
    }
}
```

## 2.2 httptest.NewServer

创建一个真实可访问的本地 HTTP Server（分配随机端口）。

```go
func TestClient(t *testing.T) {
    // 创建一个 mock server
    ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "pong")
    }))
    defer ts.Close()

    // 使用客户端请求 mock server
    resp, err := http.Get(ts.URL)
    if err != nil {
        t.Fatal(err)
    }

    body, _ := io.ReadAll(resp.Body)

    if string(body) != "pong\n" {
        t.Fatalf("unexpected response: %s", body)
    }
}
```

## 2.3 httptest.NewUnstartedServer()

- 创建一个 server 但不自动启动可在启动前自定义内部 *listener*、tls 配置等

```go
ts := httptest.NewUnstartedServer(handler)
// 自定义 TLS 等...
ts.StartTLS()
defer ts.Close()
```



