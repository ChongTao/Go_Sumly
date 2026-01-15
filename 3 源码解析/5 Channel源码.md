# 1 Channel的底层原理

chan是runtime管理的数据结构，其本质是：**一个受 mutex 保护的队列 + 等待队列（sendq / recvq）**，是goroutine通信的重要组成，核心目标：

- goroutine 间 **安全传值**
- 支持 **阻塞 / 非阻塞**
- 支持 **select 多路复用**
- 支持 **关闭语义**

## 1.1 hchan结构

```go
type hchan struct {
    qcount   uint           // 当前缓冲区中元素个数
    dataqsiz uint           // 环形队列大小（cap）
    buf      unsafe.Pointer // 指向环形缓冲区
    elemsize uint16         // 单个元素大小
    closed   uint32         // 是否已关闭

    sendx    uint           // 发送索引
    recvx    uint           // 接收索引

    recvq    waitq          // 等待接收的 goroutine 队列
    sendq    waitq          // 等待发送的 goroutine 队列

    lock     mutex          // 保护整个 channel
}
```

channel 本质是：**环形队列 + 等待队列 + mutex**

## 1.2 makechan

```go
func makechan(t *chantype, size int) *hchan
```

负责Channel的创建，返回一个hchan的指针。

- t参数表示元素的类型，

```go
type ChanType struct {
	Type
	Elem *Type
	Dir  ChanDir
}
```

- size表示Channel buffer槽位有多少，没有指定默认为0。

## 1.3 chansend

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool)
```

`chansend`是编译器解析到`c<- x`代码时插入，其本质是将元素传递到hchan的`rightbuffer`中。

发送的 5 种情况：

1. 有等待的 receiver（无论是否 buffered）

   ```go
   if recv := c.recvq.dequeue(); recv != nil {
       // 直接把数据拷贝到 receiver 栈
   }
   ```

2. buffered 且 buf 未满

   ```go
   if c.qcount < c.dataqsiz {
       // 放入 buf[sendx]
   }
   ```

3. 非 buffered，且无 receiver

   - 当前 goroutine 入 `sendq`

   - **park（阻塞）**

   ```go
   gopark(...)
   ```

4. buffered，但 buf 已满

   - 同样阻塞

   - 加入sendq


5. channel 已关闭

   ```go
   if c.closed != 0 {
       panic("send on closed channel")
   }
   ```

## 1.4 chanrev

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool)
```

`chanrecv`是编译器解析到`x <-c`代码时插入，其本质是将元素出队。

接收的 6 种情况：

1. buf 中有数据

   ```go
   if c.qcount > 0 {
       // 从 buf[recvx] 取
   }
   ```

2. 有等待的 sender（unbuffered）
   - 直接从 sender 栈拷贝


3. buf 为空，但 sendq 非空（buffered）

   - 取 sender 的值

   - sender 的值 **补到 buf**

   - sender 唤醒


4. channel 已关闭 + buf 有数据
   - 继续读数据


5. channel 已关闭 + buf 为空

   ```go
   // 返回零值 + false
   return true, false
   ```

6. buf 为空 + 没 sender + block=true

   - 当前 goroutine 入 recvq

   - park


## 1.5 close(channel) 源码行为

```go
func closechan(c *hchan)
```
