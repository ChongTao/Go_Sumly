# 1 RPC简介

RPC(remote procedure call)：是一种通信协议，使客户端可以像调用本地函数一样调用远程服务器上的方法，底层会自动处理序列化、反序列化、网络通信等细节。

![Big Image](https://geektutu.com/post/quick-go-rpc/rpc-procedure.jpg)

# 2 gRPC

[gRPC](https://grpc.io/) 是一个高性能、开源、通用的RPC框架，由Google推出，基于[HTTP2](https://http2.github.io/)协议标准设计开发，默认采用[Protocol Buffers](https://developers.google.com/protocol-buffers/)数据序列化协议，支持多种开发语言。gRPC提供了一种简单的方法来精确的定义服务，并且为客户端和服务端自动生成可靠的功能库。

gRPC核心组件包括： ProtoBuf、HTTP/2和gRPC Runtime。

gRPC比较快原因：

- Protobuf (二进制)：比JSON小、编解码快、编译期检查。
- HTTP/2 多路复用：一个TCP连接多个并发。
- 长连接+连接复用：RTT成本低、连接复用

## 2.1 步骤

1. 定义 `.proto` 文件([protobuf](../2 Protobuf/protobuf介绍.md))，描述服务接口和消息结构（类似接口定义语言 IDL）。

   ```protobuf
   syntax = "proto3";
   
   package helloworld;
   
   service Greeter {
     rpc SayHello (HelloRequest) returns (HelloReply);
     rpc StreamHello (HelloRequest) returns (stream HelloReply);
     rpc Chat (stream HelloRequest) returns (stream HelloReply);
   }
   
   message HelloRequest {
     string name = 1;
   }
   
   message HelloReply {
     string message = 1;
   }
   ```

2. 使用 `protoc` 工具生成客户端与服务端的代码。

   ```shell
   protoc --go_out=. --go-grpc_out=. helloworld.proto
   ```

3. 服务端实现接口，客户端直接调用。

   ```go
   type server struct {
       pb.UnimplementedGreeterServer
   }
   
   func (s *server) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloReply, error) {
       return &pb.HelloReply{Message: "Hello " + req.Name}, nil
   }
   
   func main() {
       lis, _ := net.Listen("tcp", ":50051")
       grpcServer := grpc.NewServer()
       pb.RegisterGreeterServer(grpcServer, &server{})
       grpcServer.Serve(lis)
   }
   ```

4. 客户端调用

   ```go
   func main() {
       conn, _ := grpc.Dial("localhost:50051", grpc.WithInsecure())
       defer conn.Close()
   
       client := pb.NewGreeterClient(conn)
       resp, _ := client.SayHello(context.Background(), &pb.HelloRequest{Name: "Go"})
       fmt.Println(resp.Message)
   }
   ```

## 2.2 gRPC客户端

### 2.2.1 原始gRPC Client

**直接使用官方 gRPC SDK 生成的 Client Stub** 如下（**注意**：gRPC的Client不是一个连接，而是一个“连接管理池”）：

```go
conn, _ := grpc.Dial("localhost:50051", grpc.WithInsecure())
defer conn.Close()

client := pb.NewGreeterClient(conn)
resp, _ := client.SayHello(context.Background(), &pb.HelloRequest{Name: "Go"})
```

其特点是强类型、高性能、无需额外开发。其中最新要使用`grpc.DialContext`，带有上下文。

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

conn, err := grpc.DialContext(
    ctx,
    "localhost:50051",
    grpc.WithBlock(),
)
```

### 2.2.2 NewClient

使用`DialContext`出现以下几个问题：

- ClientConn的生命周期容易被误用：每次请求都会创建一个连接。
- Dial语义不清晰：Dial不等于连接，每次创建不代表连接上。
- 客户端出事逻辑分散：超时、TLS、拦截器、负载均衡分散。

因此Google官网推荐使用`NewClient`作为客户端构造函数将`Dial`细节全部封装，如下示例：

```go
	conn, err := grpc.NewClient("127.0.0.1:8000",
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin":{}}]}`),
		grpc.WithKeepaliveParams(keepalive.ClientParameters{
			Time:    30 * time.Second,
			Timeout: 30 * time.Second,
		}),
	)
	if err != nil {
		logger.Errorf(meta, "create grpc client error: %v", err)
		return err
	}
    defer conn.Close()

	// 创建客户端
    client := pb.NewMyServiceClient(conn)
    // 执行方法
    resp, err := client.MyMethod(context.Background(), &pb.MyRequest{})
```

支持`NewClient`的配置

```go
// 拦截器
grpc.WithUnaryInterceptor(myUnaryInterceptor)
// 负载均衡
grpc.WithDefaultServiceConfig(`{
  "loadBalancingPolicy": "round_robin"
}`)
```

### 2.2.3 原理流程

1. 解析目标地址

   - 解析用户传入的地址（如 `localhost:50051` 或域名）。

   - 支持不同的 **resolver**（解析器），比如 DNS、负载均衡等。

2. 创建 HTTP/2 连接

   - gRPC 基于 HTTP/2 协议，因此需要先和目标服务器建立 **HTTP/2 TCP 连接**。

   - 通过 `net.Dial` 或 TLS 建立安全连接（ALPN 协商 HTTP/2）。

3. 初始化 Channel 对象

   - gRPC 把连接抽象为一个 **Channel**，后续所有 RPC 调用都通过它发送。

   - Channel 内部有多路复用功能，可以同时跑多个 RPC（HTTP/2 的 Stream 特性）。

4. 启动流控、健康检测、KeepAlive

   - **流量控制**（使用 HTTP/2 的窗口机制）

   - **心跳检测**（KeepAlive）

   - **连接重试与负载均衡**

5. 生成 Stub（客户端代理对象）

   - `NewClient` 通常返回的是一个经过 protobuf 描述的 **Stub 对象**，它会封装序列化、反序列化、Header 处理等逻辑。

   - 调用 `stub.MethodName(req)` 就会：
     - **序列化** `req` → Protobuf 二进制
     - 通过 HTTP/2 Stream 发送到服务端
     - 等待服务端返回的响应数据
     - **反序列化** 回响应对象

### 2.2.4 使用推荐

1. 复用客户端，在服务每次请求可以复用gRPC客户端，原因：

   - **创建客户端是昂贵的**: 每次调用 `grpc.Dial`（或 `grpc.NewClient`）都会建立新的 TCP & HTTP/2 连接，性能差，耗时高。

   - **gRPC Channel 是线程安全的**，gRPC会自动管理底层的TCP连接，维护多个子连接。

   - **避免资源浪费**，重复创建ClientConn会产生大量连接，泄露fd，触发“too many open files”，此外频繁建立连接会导致服务端负载过高。


2. gRPC 承载并发，一个 gRPC 连接（HTTP/2）可以承载非常高的并发，通常是 *几十到几千* 个并发请求。原因：

   - gRPC 基于 **HTTP/2 多路复用**


   - 同一个 TCP 连接上可以跑 **多个 stream（每个 stream = 一个 RPC 调用）**


   - stream 之间不会互相阻塞（除非极端 HEAD-OF-LINE 情况，被 gRPC 内核处理）

在应用启动时创建客户端连接，业务逻辑中直接复用同一个客户端 Stub。

1. 使用 `sync.Once` 单例模式，确保全局只创建一次连接，所有 RPC 共享同一个连接，充分利用 HTTP/2 多路复用。

   ```go
   var (
       grpcConn *grpc.ClientConn
       once sync.Once
   )
   
   func GetConn() *grpc.ClientConn {
       once.Do(func() {
           var err error
           grpcConn, err = grpc.NewClient(address, []grpc.DialOption{
               grpc.WithTransportCredentials(insecure.NewCredentials()),
               grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin":{}}]}`),
               grpc.WithKeepaliveParams(keepalive.ClientParameters{
                   Time:    30 * time.Second,
                   Timeout: 30 * time.Second,
               }),
           })
           if err != nil {
               panic(err)
           }
       })
       return grpcConn
   }
   
   conn := GetConn()
   client := service.NewServiceClient(conn)
   resp, err := client.Exec(timeoutCtx, req)
   ```

2. 显式缓存，将 `ClientConn`缓存，懒加载。

   ```go
   type ClientPool struct {
       mu    sync.Mutex
       conns map[string]*grpc.ClientConn
   }
   
   func (p *ClientPool) Get(addr string) (*grpc.ClientConn, error) {
       p.mu.Lock()
       defer p.mu.Unlock()
   
       if conn, ok := p.conns[addr]; ok {
           return conn, nil
       }
   
       conn, err := grpc.DialContext(context.Background(), addr)
       if err != nil {
           return nil, err
       }
   
       p.conns[addr] = conn
       return conn, nil
   }
   ```



# 3 gRPC客户端开源框架

| 方案                    | 是否多连接        | 是否自动回收 | 并发能力 | 延迟表现 | 易用性 | 备注                  |
| ----------------------- | ----------------- | ------------ | -------- | -------- | ------ | --------------------- |
| **go-grpc-pool**        | ✔ 多连接池        | ✔ idle 回收  | ⭐⭐⭐⭐     | ⭐⭐⭐⭐     | ⭐⭐⭐    | 高并发推荐            |
| **flyaways/pool**       | ✔ 可多连接        | ✔ idle 回收  | ⭐⭐⭐      | ⭐⭐⭐      | ⭐⭐     | 需自己包装            |
| **gRPC 单连接多路复用** | ❌ 单连接          | ❌            | ⭐⭐⭐⭐⭐    | ⭐⭐⭐⭐⭐    | ⭐⭐⭐⭐⭐  | 默认最佳实践          |
| **go-zero**             | ❌（但可自动重连） | ❌            | ⭐⭐⭐⭐     | ⭐⭐⭐⭐     | ⭐⭐⭐⭐⭐  | 建议 go-zero 用户使用 |

## 3.1 go‑grpc‑pool

- go‑grpc‑pool 是 Go 语言下为 gRPC 客户端提供连接池（connection pool）的开源库。它的目标是让你 **复用 gRPC 连接，而不是对每个 RPC 都 `Dial + Close`**。
- 通过连接池，你可以设定：最小连接数、最大连接数、空闲连接超时时间、获取连接超时等参数。池子会管理连接的创建、回收、空闲关闭等逻辑。
- 调用方通过 `pool.Get()`（或对应 API）获取连接，使用完毕后 `pool.Release(conn)`（或等价方法）归还连接，而不是 `conn.Close()`。这样，底层连接可以被多个 RPC 重用。

示例：

```go
func main() {
    // 创建 pool
    factory := func() (*grpc.ClientConn, error) {
        return grpc.Dial(
            "server:50051",
            grpc.WithTransportCredentials(insecure.NewCredentials()),
            grpc.WithBlock(),
            grpc.WithTimeout(5*time.Second),
        )
    }

    pool, err := grpcpool.NewPool(factory, grpcpool.PoolOptions{
        MinConn:     2,
        MaxConn:     10,
        IdleTimeout: 5 * time.Minute,
        // 其他选项...
    })
    if err != nil {
        log.Fatalf("failed to create grpc pool: %v", err)
    }
    defer pool.Close()

    // 每次 RPC 调用
    conn, err := pool.Get()
    if err != nil {
        log.Fatalf("failed to get grpc conn: %v", err)
    }
    defer pool.Put(conn)  // 注意不是 conn.Close()

    client := pb.NewYourServiceClient(conn)
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    resp, err := client.YourMethod(ctx, &pb.YourRequest{...})
    if err != nil {
        // handle error
    }
    // use resp
}
```

| 优点                           | 解释                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| **减少连接开销**               | 避免每次 RPC 都新建 TCP + HTTP/2 + TLS 等连接开销，大幅提高性能和吞吐 |
| **控制连接数量 / 并发连接数**  | 可以设定池的最大连接数 / 空闲超时时间，避免过多连接占用资源  |
| **连接复用**                   | 池化 + HTTP/2 多路复用，让多个 RPC 共用连接资源，提升效率    |
| **适合高并发 / 高频 RPC 场景** | 对于每秒有大量 RPC 请求的服务，连接池能极大降低负载和延迟    |
| **连接自动管理**               | 空闲连接会回收、失败连接可重建，无需手动管理 Dial / Close    |

注意事项：

- **HTTP/2 & gRPC 本身就支持多路复用** — 如果你只是单服务、单连接，且并发不高，好用单连接 + reuse 也足够。池化不一定带来显著收益。
- **连接过多可能造成资源浪费** — 如果池配置不当（MaxConn 很大 / IdleTimeout 很短 / 连接复用率不高），可能会浪费内存 / FD / CPU。
- **管理复杂度增加** — 需要注意 `Get()` / `Put()` 是否匹配，防止连接泄漏或重复关闭。
- **部分池不一定活跃维护** — 不是所有开源池都有很活跃的维护社区；依赖前需要评估稳定性与兼容性。
- **Load‑balancing & 连接健康检查有限** — 池化只是连接复用，不等同于服务发现 + 健康检查 + 自动切换节点。对于多实例 + 负载均衡，你可能仍需自己管理。

## 3.2 go‑zero

- go-zero 提供了 `zrpc` 模块，用来作为 gRPC Client 管理抽象。你通过配置 `zrpc.MustNewClient(...)` / `zrpc.RpcClientConf` 就能拿到一个复用连接。
- 你不需要为每一次 RPC 都调用 `grpc.Dial` — 框架帮你管理连接生命周期。
- 支持直连单服务，也支持多个 endpoint／集群，甚至服务发现（如 etcd）方式，方便服务扩展和负载均衡。

> 简而言之：go‑zero 的推荐方式，就是让一个连接（ClientConn）复用，而不是你手动每次 Dial + Close。

示例：

```go
func main() {
    // 构造客户端配置
    clientConf := zrpc.RpcClientConf{}
    // 可填 Target (单节点) / Endpoints (集群) / Etcd (服务发现)
    clientConf.Target = "127.0.0.1:50051"
    // 如果需要 TLS／凭证／LB，可在配置里加入

    // 创建 gRPC 客户端连接（底层复用 ClientConn）
    conn := zrpc.MustNewClient(clientConf)
    // 从 zrpc.Client 获取 grpc.ClientConnInterface
    c := greet.NewGreetClient(conn.Conn())

    // 发起 RPC 调用
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    resp, err := c.Ping(ctx, &greet.Request{Message: "Hello"})
    if err != nil {
        log.Fatalf("RPC failed: %v", err)
    }
    fmt.Println("Reply:", resp.Message)

    // 程序结束时（可选）关闭 underlying connection
    conn.Close()
}
```

# 4 JSON-RPC

**JSON-RPC** 是一种基于 **JSON 格式** 的远程过程调用（RPC）协议。它不依赖 HTTP、本身也不限定传输层，可以运行在：

- HTTP / HTTPS
- WebSocket
- TCP / Unix Socket

目前主流版本是 **JSON-RPC 2.0**（2010年发布）。

请求格式：

```json
{
  "jsonrpc": "2.0",           // 固定版本
  "method": "sayHello",       // 调用函数的名称
  "params": {"name": "Tom"},  // 调用的参数
  "id": 1                     // 请求 ID，用于匹配响应
}
```

服务器响应：

```json
{
  "jsonrpc": "2.0",         // 固定版本
  "result": "Hello, Tom!",  // 成功结果
  "id": 1                   // 请求 ID
}
```
