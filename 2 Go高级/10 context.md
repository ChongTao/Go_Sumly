# 一 Context

在 Go 语言中，`context.Context` 是一个**用来在多个goroutine之间传递“控制信号”和请求级数据的接口**。它主要解决三个问题：

1. **取消信号（Cancellation）**：一个操作不需要继续了，可以通知所有相关 goroutine 停止工作。
2. **超时与截止时间（Timeout / Deadline）**：比如：一个请求最多处理 2 秒，超时就自动取消。
3. **请求范围内的数据传递（Request-scoped values）**：如：requestId、用户信息、traceId 等。

Context使用场景：

- HTTP / RPC 请求处理
- 并发任务的生命周期管理
- 数据库、缓存、外部服务调用
- 微服务调用链路追踪

# 二 Context 接口定义

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)  
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

- `Deadline`返回一个`time.Time`，表示操作的截止日期。
- `Done`返回一个`<-chan struct{}`，当context被取消或超时时，该通道会被关闭。
- `Err`方法在`Done`通道关闭后返回取消的错误原因。
- `Value`方法返回与此context关联的值，如果该值不存在则返回`nil`。

# 三 使用

```go
// 1. 创建根Context
ctx := context.Background() // 主流程、服务启动
ctx := context.TODO()

//2. 创建可取消
ctx, cancel := context.WithCancel(context.Background())
defer cancle()  // 方法执行完后，取消上下文   

//3. 创建超时取消的上下文
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)  // 3s后取消
defer cancle()   // 一定要调用 cancel()，否则会造成定时器泄漏

// 4. 带值的 Context
type traceIDKeyType struct{}
ctx = context.WithValue(ctx, traceIDKeyType{}, "trace-123")
```

# 四 常见问题

## 4.1 Done()是什么

```go
Done() <-chan struct{}
```

返回一个 **只读 channel**，当Context被取消时，Channel会被Close，监听方式

```go
select {
    case <-ctx.Done():
	  return ctx.Err()
    case data := <-ch:
      ...
}
```

