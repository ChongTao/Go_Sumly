# 1 Select 

`select` 用于**同时监听多个 channel 操作**，即从所有“当前可执行”的 case 中，随机选择一个执行；如果都不可执行，则阻塞（除非有 default）

## 1.1 select 的底层结构

Go 编译器会把`select`转换为对`runtime.selectgo`的调用，即编译期完成：

- case 数组构造
- channel 指针、elem 指针整理
- 是否有 default 的标记

每个 `case` 在编译期会被整理成 runtime 可识别的结构`scase`：

```go
type scase struct {
    typ   uint8           // case 类型：send/recv/default
    chan  *hchan          // 对应的 channel
    elem  unsafe.Pointer  // 要发送/接收的数据指针
}
```

它们组成一个数组，然后进行 **shuffle 随机排序**。

在运行时，对`selectgo`的调用

```go
func selectgo(
    cases []scase,    // 上述编译期的数组
    order []uint16,
    pc0 *uintptr,
    nsends, nrecvs int,
    block bool,
) (chosen int, recvOK bool)

```

其流程如下：

1. **把每个 case 转换为 scase 数组结构体**
2. **随机打乱 case 顺序（防止饥饿）**
3. 按顺序尝试“是否立即可执行”
   - recv：有没有数据可读
   - send：channel 是否可写
4. 若有可执行 case → 立即执行
5. 若都不可执行 → 将当前 goroutine **挂到对应 channel 的等待队列**
   - sendq：发送等待队列
   - recvq：接收等待队列
6. 挂好后当前 goroutine **阻塞**（park）
7. 其他 goroutine 唤醒它后继续执行

## 1.2 为什么 select 是随机选择 case

因为 Go 想避免以下情况：

```go
select {
case <-ch1:   // 永远排第一
case <-ch2:
}
```

如果不随机：

- ch1 只要经常 ready，会导致 ch2 永远无法被 select 选择（饥饿问题）
- 这在生产环境是灾难，select 会倾向某一个 case

因此 Go runtime 会随机打乱 case 顺序，即使顺序写死，也无法保证顺序执行。

```go
for i := range order {
    j := fastrandn(uint32(i + 1))
    order[i], order[j] = order[j], order[i]
}
```

