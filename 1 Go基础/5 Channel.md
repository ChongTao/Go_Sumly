# 一 Channel 介绍

在Go语言中，**管道（Channel）**是一种特殊的类型，用于在不同的goroutine之间传递数据，是Go并发模型的核心，使用的是 **CSP（Communicating Sequential Processes）** 模型：**不要通过共享内存来通信，而要通过通信来共享内存。**具有以下特点：

1. **类型安全**：Channel类型安全的管道，每一个Channel都是一个固定的类型，在编译期决定，不允许发送错类型。
2. **线程安全**：Channel 在本质上是线程安全的队列（有锁 + 条件变量），其底层结构包含循环队列buffer、读等待队列(sudog结构)、写等待队列(sudog)、互斥锁、Channel是否关闭标志等。
3. **有缓存和无缓存**：无缓存发送和接受必须同时发生，否则会阻塞；有缓存需等待队列是否满或空才阻塞
4. **关闭后的 channel 行为非常特殊**：将继续接受剩余的数据、读到底后会返回零；写入会panic。
5. **无需管理生命周期**：Go GC 会自动回收 channel 对象。

# 二 Channel

## 2.1 无缓冲Channel

无缓冲 channel 不包含任何队列，其通信是严格同步的。

```go
ch := make(chan int) // 无缓冲 channel
```

其特点是：

- 发送和接收必须同步“同时发生”，无缓冲 channel 的 `send/receive` 操作不会独立进行，必须成对出现。
- 没有数据存储能力，channel内不保存任何数据，发送值会直接交给接收者。
- 天然同步原语，无缓冲 channel 可以用来实现goroutine之间的事件通知，控制执行顺序等。

```go
func main() {
    ch := make(chan int)

    go func() {
        ch <- 1
    }()

    v := <-ch
    fmt.Println(v)
}

```

## 2.2 有缓冲Channel

内部包含一个循环队列，用于缓存数据。

```go
ch := make(chan int, 3) // 带缓冲区，容量为 3
```

其特点是：

- 异步通信，缓冲区未满时，发送方可以立即返回，而不需要接收者已就绪。
- 具有数据存储能力，可以缓存 N 条数据，存储行为类似队列。
- 阻塞条件变化，**发送阻塞：队列满时**；**接收阻塞：队列空时**

```go
func main() {
    ch := make(chan int, 2)

    ch <- 10
    ch <- 20

    fmt.Println(<-ch) // 10
    fmt.Println(<-ch) // 20
}
```

## 2.3 无缓冲和有缓冲区别

1. 通信方式不同

   | 类型               | 通信方式                    |
   | ------------------ | --------------------------- |
   | **无缓冲 Channel** | 同步（send 必须等 recv）    |
   | **有缓冲 Channel** | 异步（buffer 未满即可发送） |

2. 阻塞条件不同

   | 操作 | 无缓冲               | 有缓冲       |
   | ---- | -------------------- | ------------ |
   | send | 永远阻塞直到有接收者 | 队列满时阻塞 |
   | recv | 永远阻塞直到有发送者 | 队列空时阻塞 |

3. 适用场景不同

   | 类型       | 典型用途                   |
   | ---------- | -------------------------- |
   | **无缓冲** | 同步、顺序控制、事件通知   |
   | **有缓冲** | 队列、限流、高吞吐任务管道 |



# 三 常见问题

## 3.1 对已经关闭channel进行读写

1. 对已经关闭的channel进行写入

   ```go
   ch := make(chan int)
   close(ch)
   ch <- 1 // ❌ 写入已关闭的通道 panic: send on closed channel
   ```

   当channel被关闭后，不能在有新的数据被发送，否则panic， 原因如下：

   ```go
   func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
       if c.closed != 0 {
   		unlock(&c.lock)
   		panic(plainError("send on closed channel"))
   	}
   }
   ```

2. 对已经关闭的channel进行读取

   ```go
   ch := make(chan int)
   close(ch)
   
   v, ok := <-ch
   fmt.Println(v, ok)  // 0 false
   ```

   对一个已关闭的通道进行读取时，不会发送阻塞，如果通道中仍有未读取的数据，会依次读出，直到缓冲区的数据全部被读取完，再次读取默认类型的值返回，ok=false， 原因如下：

   ```go
   func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
   	if c.closed != 0 {
   		if c.qcount == 0 {
   			if raceenabled {
   				raceacquire(c.raceaddr())
   			}
   			unlock(&c.lock)
   			if ep != nil {
   				typedmemclr(c.elemtype, ep)
   			}
   			return true, false
   		}
   		// The channel has been closed, but the channel's buffer have data.
   	} 
   }
   ```

| 操作类型            | 通道状态         | 行为                 | 说明                   |
| ------------------- | ---------------- | -------------------- | ---------------------- |
| 写入（`ch <- v`）   | 打开             | 正常发送             | 可能阻塞（无缓冲或满） |
| 写入（`ch <- v`）   | 已关闭           | **panic**            | 不允许写入已关闭通道   |
| 读取（`v := <-ch`） | 打开             | 可能阻塞             | 等待数据               |
| 读取（`v := <-ch`） | 已关闭但仍有数据 | 读取剩余数据         | 不阻塞                 |
| 读取（`v := <-ch`） | 已关闭且无数据   | 返回零值，`ok=false` | 不阻塞                 |

## 3.2 对未初始化channel进行读写

channel 是引用类型，仅声明 `var ch chan int` 时，ch 的值是 `nil`；必须通过 `make(chan int)` 创建底层结构（队列、锁、缓冲区等）后才能使用。

**Go 运行时检查**

- 对 `nil` channel 的读写操作被设计为“永久阻塞”；
- 这样可避免 runtime 不确定行为；
- `close(nil)` 直接 panic 是为了防止误操作。

- 读操作（阻塞）

  ```go
  var ch chan int
  
  fmt.Println("准备读")
  v := <-ch     // 永远阻塞在这里
  fmt.Println(v)
  ```

  原因是：

  ```go
  func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
  	if c == nil {
  		if !block {
  			return
  		}
  		gopark(nil, nil, waitReasonChanReceiveNilChan, traceBlockForever, 2)
  		throw("unreachable")
  	}
  }
  ```

- 写操作（阻塞）

  ```go
  var ch chan int
  
  fmt.Println("准备写")
  ch <- 1        // 永远阻塞
  fmt.Println("写入完成")
  
  ```

  未初始化的`chan`此时是等于`nil`，当它不能阻塞的情况下，直接返回 `false`，表示写 `chan` 失败；当`chan`能阻塞的情况下，则直接阻塞 `gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2) `, 然后调用`throw(s string)`抛出错误,其中`waitReasonChanSendNilChan`就是刚刚提到的报错`"chan send (nil chan)"`

  ```go
  func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
  	if c == nil {
  		if !block {
  			return false
  		}
  		gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2)
  		throw("unreachable")
  	}
  }
  ```

- 关闭操作（panic）

  ```go
  var ch chan int
  close(ch)
  ```
