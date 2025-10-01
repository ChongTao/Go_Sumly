# 1 RPC简介

rpc(remote procedure call)：让你像调用本地函数一样，去调用一个位于远程机器上的函数或者方法。

![Big Image](https://geektutu.com/post/quick-go-rpc/rpc-procedure.jpg)

RPC 模型是一个典型的客户端-服务器模型(Client-Server, CS)，相比于调用本地的接口，RPC 还需要知道的是服务器端的地址信息。

# 2 示例

Go语言的RPC包路径为net/rpc，方法定义如下：

```go
func (t *T) MethodName(argType T1, replyType *T2) error
```

- 方法类型（T）是导出的（首字母大写）
- 方法名（MethodName）是导出的
- 方法有2个参数(argType T1, replyType *T2)，均为导出/内置类型
- 方法的第2个参数一个指针(replyType *T2)
- 方法的返回值类型是 error

`net/rpc` 对参数个数的限制比较严格，方法只能有两个可序列化的参数，第一个参数是调用者提供的请求参数，第二个参数是返回给调用者的响应参数，服务端需要将计算结果写在第二个参数中。如果调用过程中发生错误，会返回 error 给调用者。

1. 创建服务端并启动

```go
package main

import (
	"log"
	"net/http"
	"net/rpc"
)

type Result struct {
	Num, Ans int
}

type Cal int

func (cal *Cal) Square(num int, result *Result) error {
	result.Num = num
	result.Ans = num * num
	return nil
}

func main() {
	// 发布Cal中满足RPC注册条件方法
	rpc.Register(new(Cal))
	// 注册用于处理RPC消息HTTP Handle
	rpc.HandleHTTP()
	log.Printf("Serving RPC server at port %d", 8080)

	// 监听端口，等待RPC请求
	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal("Error serving:", err)
	}
}

```

2. 客户端远程调用

```go
package main

import (
	"log"
	"net/rpc"
)

type Result struct {
	Num, Ans int
}

func main() {

	// 创建HTTP客户端，并创建连接
	clint, _ := rpc.DialHTTP("tcp", "127.0.0.1:8080")
	var result Result
	// 调用远程方法，传递参数和结果
	if err := clint.Call("Cal.Square", 12, &result); err != nil {
		log.Fatal("Failed to call Cal.Square", err)
	}
	log.Printf("Square result: %d^2 = %d", result.Num, result.Ans)
}
```

此外也可以异步调用

```go
asyncCall := client.Go("Cal.Square", 2, &result, nil)
log.Printf("%d^2 = %d", result.Num, result.Ans)
<-asyncCall.Done
log.Printf("%d^2 = %d", result.Num, result.Ans)
```

