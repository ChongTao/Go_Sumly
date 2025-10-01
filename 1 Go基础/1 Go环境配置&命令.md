## 1 Go 安装

最新版本下载地址[All releases - The Go Programming Language](https://golang.google.cn/dl/)，并安装。下载完后配置环境变量GOROOT和GOPATH环境变量。

设置阿里云镜像

> go env -w GOPROXY=https://mirrors.aliyun.com/goproxy/,direct



## 2 Goland下载

Goland下载：[Download GoLand: A Go IDE with extended support for JavaScript, TypeScript, and databases](https://www.jetbrains.com.cn/go/download/?section=windows)



## 3 常见命令

1. `go run`：直接编译运行Go程序，例如`go run main.go`
2. `go build`：编译Go程序，生成可执行文件（默认与包名相同）。例如`go build`（当前目录）或者`go build -o myapp`（指定输出文件名）
3. `go install`：编译并安装可执行文件到`$GOPATH/bin`或`$GOBIN`目录
4. `go get`：用于下载并安装第三方包或工具。`go get gitbub.com/gin-gonic/gin`，可通过`-u`升级包，`-d`仅下载不安装。
5. `go mod`：模块管理命令，用于初始化项目、管理依赖等。
   - `go mod init <module-name>`：初始化新模块
   - `go mod tidy`：自动添加缺失依赖、移除无用的依赖
   - `go mod vendor`：将依赖复制到项目的`vendor`目录，用于离线构建。
6. `go test`：运行测试文件（以_test.go结尾）中的测试函数。例如`go test`（当前包）或者`go test ./..`
7. `go fmt/goimports`：格式化代码，统一风格
8. `go doc`：查看包或者函数的文档。
9. `go clean`：清理编译生成的临时文件、可执行文件等。















