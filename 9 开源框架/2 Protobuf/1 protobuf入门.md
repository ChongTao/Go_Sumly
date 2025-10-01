# 1 Protocol Buffers简介

protobuf即protocol Buffers，是一种轻便高效的结构化数据存储格式，与语言、平台无关，可扩展可序列化。protobuf 性能和效率大幅度优于 JSON、XML 等其他的结构化数据格式。protobuf 是以二进制方式存储的，占用空间小，但也带来了可读性差的缺点。protobuf 在通信协议和数据存储等领域应用广泛。

Protobuf 在 `.proto` 定义需要处理的结构化数据，可以通过 `protoc` 工具，将 `.proto` 文件转换为 C、C++、Golang、Java、Python 等多种语言的代码，兼容性好，易于使用



# 2 安装

安装流程参考[windows安装protoc、protoc-gen-go、protoc-gen-go-grpc-CSDN博客](https://blog.csdn.net/YouMing_Li/article/details/134895264)

# 3 使用

## 3.1 定义消息类型

```protobuf
syntax = "proto3";

// 一定要指定包路径
option go_package="./;main";

package main;

// 关键字定义，Student是类型名，name, male，scores是类的字段和类型。
// 后面的数字被称为标识符，每个字段都需要提供唯一的标识符，用来在消息的二进制格式中识别各个字段。
message Student {
    string name = 1;
    bool male = 2;
    repeated int32 scores = 3;
}
```

示例

```go
package main

import (
	"github.com/golang/protobuf/proto"
	"log"
)

func main() {
	test := &Student{
		Name:   "test1",
		Male:   false,
		Scores: []int32{1, 2, 3},
	}

	data, err := proto.Marshal(test)
	if err != nil {
		log.Fatal(err)
	}
	
	newTest := &Student{}
	err = proto.Unmarshal(data, newTest)
	if err != nil {
		log.Fatal(err)
	}
}
```

## 3.2 字段类型

### 3.2.1 标量型数据

| proto类型 | go类型  | 备注                          |
| :-------- | :------ | :---------------------------- |
| float     | float32 |                               |
| double    | float64 |                               |
| int64     | int64   |                               |
| uint64    | uint64  |                               |
| sint64    | int64   | 适合负数                      |
| fixed64   | uint64  | 固长编码，适合大于2^56的值    |
| sfixed64  | int64   | 固长编码                      |
| int32     | int32   |                               |
| uint32    | uint32  |                               |
| sint32    | int32   | 适合负数                      |
| fixed32   | uint32  | 固长编码，适合大于2^28的值    |
| sfixed32  | int32   | 固长编码                      |
| bool      | bool    |                               |
| bytes     | []byte  | 任意字节序列，长度不超过 2^32 |
| string    | string  | UTF8 编码，长度不超过 2^32    |

标量类型如果没有被赋值，则不会被序列化，解析时，会赋予默认值。

- strings：空字符串
- bytes：空序列
- bools：false
- 数值类型：0

### 3.2.2 枚举类型

枚举类型适用于提供一组预定义的值，选择其中一个。例如我们将性别定义为枚举类型。

```protobuf
message Person {
  string name = 1;
  enum Gender {
    FEMAL = 0;
    MALE = 1;
  }
  
  Gender gender = 2;
  repeated int32 scores = 3;
}
```

枚举类型的第一个选项标识符必须是0，此外运行不同枚举值赋予相同的标识符，被称为别名，但是需要打开allow_alias选项。

```protobuf
message EnumAllowAlias {
  enum Status {
    option allow_alias = true;
    UNKONE = 0;
    STATUS = 1;
    RUNNING = 1;
  }
}
```

## 3.3 定义服务

如果消息类型是用来远程通行（Remote Rrocedure Call, RPC）, 可以在.proto文件中定义RPC服务接口。

```protobuf
service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```

