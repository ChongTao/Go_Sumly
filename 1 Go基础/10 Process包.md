Go 中的 **`os.Process`** 是操作系统进程管理的核心结构之一，位于标准库 **`os`** 包下，用于表示和控制一个正在运行的系统进程。它提供了创建、等待、终止等功能，是 Go 在系统编程中管理外部程序的关键工具。

### 一 基本定义

```go
type Process struct {
	Pid int

	// state contains the atomic process state.
	//
	// This consists of the processStatus fields,
	// which indicate if the process is done/released.
	state atomic.Uint32

	// Used only when handle is nil
	sigMu sync.RWMutex // avoid race between wait and signal

	// handle, if not nil, is a pointer to a struct
	// that holds the OS-specific process handle.
	// This pointer is set when Process is created,
	// and never changed afterward.
	// This is a pointer to a separate memory allocation
	// so that we can use runtime.AddCleanup.
	handle *processHandle

	// cleanup is used to clean up the process handle.
	cleanup runtime.Cleanup
}
```

`os.Process` 表示一个已启动或附加的进程对象。

### 二 创建与获取进程

- 创建新进程

  ```go
  func StartProcess(name string, argv []string, attr *ProcAttr) (*Process, error)
  ```

  **`name`**：可执行文件路径

  **`argv`**：命令行参数（`argv[0]` 通常是程序名）

  **`attr`**：进程属性（标准输入输出、环境变量、工作目录等）

  示例：

  ```go
  package main
  
  import (
      "fmt"
      "os"
  )
  
  func main() {
      attr := &os.ProcAttr{
          Files: []*os.File{os.Stdin, os.Stdout, os.Stderr}, // 重定向标准流
      }
  
      proc, err := os.StartProcess("/bin/ls", []string{"ls", "-l"}, attr)
      if err != nil {
          fmt.Println("启动失败:", err)
          return
      }
  
      fmt.Println("启动进程 PID:", proc.Pid)
      state, err := proc.Wait() // 等待执行完
      fmt.Println("退出状态:", state)
  }
  
  ```

### 三 os.ProcAttr结构

```go
type ProcAttr struct {
    // 如果 Dir 非空，子进程会在创建 Process 实例前先进入该目录。（即设为子进程的当前工作目录）
    Dir string
    // 如果 Env 非空，它会作为新进程的环境变量。必须采用 Environ 返回值的格式。
    // 如果 Env 为 nil，将使用 Environ 函数的返回值。
    Env []string
    // Files 指定被新进程继承的打开文件对象。
    // 前三个绑定为标准输入、标准输出、标准错误输出。
    // 依赖底层操作系统的实现可能会支持额外的文件对象。
    // nil 相当于在进程开始时关闭的文件对象。
    Files []*File
    // 操作系统特定的创建属性。
    // 注意设置本字段意味着你的程序可能会执行异常甚至在某些操作系统中无法通过编译。这时候可以通过为特定系统设置。
    // 看 syscall.SysProcAttr 的定义，可以知道用于控制进程的相关属性。
    Sys *syscall.SysProcAttr
}
```

示例：修改环境与工作目录

```go
attr := &os.ProcAttr{
    Dir: "/tmp",
    Env: []string{"FOO=bar"},
    Files: []*os.File{os.Stdin, os.Stdout, os.Stderr},
}
```

### 四 Process方法

| 方法                                                 | 说明                                     |
| ---------------------------------------------------- | ---------------------------------------- |
| `func (p *Process) Kill() error`                     | 强制终止进程（发送 SIGKILL）             |
| `func (p *Process) Signal(sig os.Signal) error`      | 向进程发送信号（Unix）                   |
| `func (p *Process) Wait() (*os.ProcessState, error)` | 等待进程退出并返回状态                   |
| `func (p *Process) Release() error`                  | 释放进程资源但不等待退出（避免僵尸进程） |

### 五 ProcessState（进程状态）

```go
type ProcessState struct {
    // 不可导出
}
```

| 方法         | 说明                                               |
| ------------ | -------------------------------------------------- |
| `Exited()`   | 是否已退出                                         |
| `Success()`  | 是否正常退出（退出码为0）                          |
| `Pid()`      | 进程ID                                             |
| `String()`   | 简要描述                                           |
| `Sys()`      | 返回底层系统信息（Unix 下是 `syscall.WaitStatus`） |
| `ExitCode()` | 退出码（Go 1.12+）                                 |

示例：

```go
state, _ := proc.Wait()
fmt.Println("退出码:", state.ExitCode())
fmt.Println("成功退出:", state.Success())
```

### 六 与 `exec` 包的关系

虽然 `os.StartProcess` 很底层，但大多数情况下我们使用更高级的 `os/exec` 封装。

```go
import "os/exec"

cmd := exec.Command("ls", "-l")
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr
cmd.Run()
```

`exec.Command` 内部其实也是基于 `os.StartProcess` 实现的，但提供了更易用的接口（如直接传字符串参数、支持输出捕获等）。