## 一 常见问题

### 1.1 对已经关闭channel进行读写

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

### 1.2 对未初始化channel进行读写

channel 是引用类型，仅声明 `var ch chan int` 时，ch 的值是 `nil`；必须通过 `make(chan int)` 创建底层结构（队列、锁、缓冲区等）后才能使用。

**Go 运行时检查**

- 对 `nil` channel 的读写操作被设计为“永久阻塞”；
- 这样可避免 runtime 不确定行为；
- `close(nil)` 直接 panic 是为了防止误操作。

| 操作                    | 结果     | 是否阻塞 | 是否 panic | 说明                             |
| ----------------------- | -------- | -------- | ---------- | -------------------------------- |
| **读（`<-ch`）**        | 永远阻塞 | ✅ 是     | ❌ 否       | 因为没有任何实际通道可以接收数据 |
| **写（`ch <- v`）**     | 永远阻塞 | ✅ 是     | ❌ 否       | 因为没有通道可供发送             |
| **关闭（`close(ch)`）** | panic    | ❌ 否     | ✅ 是       | 因为不能关闭 `nil` 通道          |

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

  
