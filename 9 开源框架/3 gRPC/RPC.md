# 1 RPC简介

RPC(remote procedure call)：是一种通信协议，使客户端可以像调用本地函数一样调用远程服务器上的方法，底层会自动处理序列化、反序列化、网络通信等细节。

![Big Image](https://geektutu.com/post/quick-go-rpc/rpc-procedure.jpg)

# 2 gRPC

[gRPC](https://grpc.io/) 是一个高性能、开源、通用的RPC框架，由Google推出，基于[HTTP2](https://http2.github.io/)协议标准设计开发，默认采用[Protocol Buffers](https://developers.google.com/protocol-buffers/)数据序列化协议，支持多种开发语言。gRPC提供了一种简单的方法来精确的定义服务，并且为客户端和服务端自动生成可靠的功能库。

工作原理：

1. 定义 `.proto` 文件([protobuf](../2 Protobuf/protobuf介绍.md))，描述服务接口和消息结构（类似接口定义语言 IDL）。

2. 使用 `protoc` 工具生成客户端与服务端的代码。

3. 服务端实现接口，客户端直接调用。

示例：

-  `.proto` 文件：

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

- 生成 Go 代码：

```bash
protoc --go_out=. --go-grpc_out=. helloworld.proto
```

- 服务端实现

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

- 客户端调用

  ```go
  func main() {
      conn, _ := grpc.Dial("localhost:50051", grpc.WithInsecure())
      defer conn.Close()
  
      client := pb.NewGreeterClient(conn)
      resp, _ := client.SayHello(context.Background(), &pb.HelloRequest{Name: "Go"})
      fmt.Println(resp.Message)
  }
  ```

# 3 JSON-RPC

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
