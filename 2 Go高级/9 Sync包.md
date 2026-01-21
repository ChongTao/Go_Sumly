# 1 Sync包

在 Go 里，`sync` 是 **标准库** 提供的一个并发同步工具包。主要用来解决多协程（goroutines）运行时的**数据安全**和**执行序控制**问题，让我们在并发场景下避免出现数据竞争（data race）或逻辑错误。

## 1.1 sync.Mutex互斥锁

保护共享数据，确保同一时间只有一个 goroutine 可以访问。

- 核心方法：
  - `Lock()`：加锁
  - `Unlock()`：解锁
- 注意事项：
  - **不可重入**：避免重复加锁，否则会死锁，必须成对使用Lock和unlock方法

```go
var mu sync.Mutex
var counter int

func increment() {
    mu.Lock()
    counter++
    mu.Unlock()
}
```

## 1.2 sync.RWMutex读写锁

适合读多写少的场景，多个读可以并发，写必须独占。

- 核心方法：
  - `RLock()` / `RUnlock()`：读锁
  - `Lock()` / `Unlock()`：写锁

```go
var rw sync.RWMutex
var data = make(map[string]string)

func read(key string) string {
    rw.RLock()
    defer rw.RUnlock()
    return data[key]
}

func write(key, value string) {
    rw.Lock()
    defer rw.Unlock()
    data[key] = value
}
```

## 1.3 sync.WaitGroup等待组

等待一组 goroutine 全部完成后再继续执行。

- 核心方法：
  - `Add(delta int)`：向 WaitGroup 的计数器 **增加或减少**值。
  - `Done()`：减少计数器的值，相等于`wg.Add(-1)`。
  - `Wait()`：阻塞当前 goroutine，直到计数器归零。

```go
var wg sync.WaitGroup

func task(id int) {
    defer wg.Done()
    fmt.Println("任务", id, "开始")
    time.Sleep(time.Second)
    fmt.Println("任务", id, "结束")
}

func main() {
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go task(i)
    }
    wg.Wait()
    fmt.Println("全部任务完成")
}
```

### 1.3.1 原理

`sync.WaitGroup` 是 Go 标准库中的一个同步原语，用于等待一组 goroutine 完成。其底层依赖 **原子操作（atomic）** 和 **信号量（runtime_sema）** 实现计数器与阻塞唤醒逻辑。

1. **一个计数器 counter 记录未完成任务数**

2. **Wait 通过信号量阻塞**
3. **Done 将计数器减 1**
4. **当计数器变为 0 时，唤醒 Wait**

### 1.3.2 WaitGroup数据结构（Go 1.25）

```go
type WaitGroup struct {
    // Go 强制禁止复制，使用 noCopy 静态检查。
	noCopy noCopy

	// Bits (high to low):
	//   bits[0:32]  counter
	//   bits[32]    flag: synctest bubble membership
	//   bits[33:64] wait count
	state atomic.Uint64  
	sema  uint32
}
```

- state是64位复合字段类型，内部结构如下:

  ```bash
  | 63     ...    33 | 32 | 31 ... 0 |
  | wait count (31b) | F  | counter  |
  ```

  - counter(bits[0-31]) 表示未完成的任务数据，每次Add(1)增加，每次Done减少，当counter数量为零时表示所有数量都结束，Wait可以返回；如果小于0，触发panic
  - flag：用于Synctest，标记是否处于sync test “bubble” 中。
  -  wait count（bits[33–63]）：表示当前有多少goroutine正在执行`Wait()`，每当 Wait() 被调用，wait count++, 当counter为0时，runtime会通过信号量唤醒所有waiters，wait count被清零

- sema

  - Go runtime 提供的信号量（semaphore）
  - 当 counter > 0 时 Wait() 会调用 `runtime_Semacquire(&wg.sema)` 阻塞当前 goroutine
  - 当 counter == 0 时 Add 或 Done 内部会调用 `runtime_Semrelease(&wg.sema)` 来唤醒 Wait

### 1.3.3 Add原理

1. 原子读取 state（counter 和 wait count）
2. 修改 counter（add delta）
3. 检查是否负数 → panic
4. 如果 counter 变为 0：
   - 唤醒所有 waiters（Wait 阻塞的 goroutine）
5. 如果有 waiter 但 counter 增加了 → panic（Add 与 Wait 并发）

**核心：修改 counter 必须是原子的，为了保证并发安全。**

### 1.3.4 Done原理

```go
wg.Add(-1)
```

- counter--

- 若 counter == 0，则调用 semrelease 唤醒 Wait

### 1.3.5 Wait原理

1. 原子读取 state
2. 如果 counter == 0 → return（无需等待）
3. 否则：
   - 增加 wait count
   - 阻塞在信号量上 `runtime_Semacquire(&wg.sema)`

直到：

- counter 变为 0，Add/Dond 唤醒 Wait
- 信号量释放，Wait 返回

### 1.3.6 WaitGroup禁止Copy

因为 WaitGroup 的内部依赖：

- 状态（state）同步
- runtime 信号量（sema）

如果进行复制`wg1 := wg`，将会发生：

- 两个 WaitGroup 会持有不同的 sema 副本

- 两者对 counter 和 wait count 的修改不再共享

- Wait 永远不会被正确唤醒或永远无法结束

### 1.3.7 常见问题

- Add与Wait并发执行
  - wait count 的增长与 counter 的修改冲突
  - Wait 内部可能错误判断 counter 的初始状态
- counter为负
  - 直接触发panic

## 1.4 sync.Once

在Go语言中，`sync.Once`是一个同步原语，用于确保某个操作只执行一次，即使在并发环境中也是安全的。`sync.Once`类型的变量可以用来保证一个函数只被执行一次，无论有多少个goroutine尝试调用它。

- **用途**：在并发环境下，确保某个操作只执行一次（比如单例模式初始化）。
- 方法：
  - `Do(func)`：无论多少次调用，传入的函数只执行一次。

```go
var once sync.Once

func initConfig() {
    fmt.Println("初始化配置")
}

func main() {
    for i := 0; i < 5; i++ {
        once.Do(initConfig)
    }
}
```

### 1.4.1 Once数据结构

```go
type Once struct {
	_ noCopy
	done atomic.Bool   // 通过原子操作判断是否已运行
	m    Mutex         // 互斥锁，保证只有一个 goroutine 执行初始化函数 f
}
```

### 1.4.2 执行流程

```go
func (o *Once) Do(f func()) {
    if o.done.Load() == 0 {
        o.doSlow(f)
    }
}

// 如果done == 0
func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()

    if o.done.Load() == 0 {
        f()                 // 执行用户逻辑
        o.done.Store(1)     // 标记“已执行”
    }
}
```

> 上述进行双重检查的原因是因为多个 goroutine 同时通过第一次判断 `done == 0`， 但只能让一个执行`f()`，所以进入临界区后需要 **第二次检查**

### 1.4.3 常见问题

- Do 里的函数 panic，会导致 Once 永远无法完成
- 不要在 Do(func) 内再次调用 Do()，可能造成死锁

## 1.5 sync.Cond 条件变量

- **用途**：用于实现复杂的协程通信，比如等待某个条件发生。
- 方法：
  - `Wait()`：等待条件
  - `Signal()`：唤醒一个等待的 goroutine
  - `Broadcast()`：唤醒所有等待的 goroutine

```go
var mu sync.Mutex
var cond = sync.NewCond(&mu)
var ready = false

func worker(id int) {
    mu.Lock()
    for !ready {
        cond.Wait()
    }
    fmt.Println("Worker", id, "开始工作")
    mu.Unlock()
}

func main() {
    for i := 1; i <= 3; i++ {
        go worker(i)
    }
    time.Sleep(time.Second)
    mu.Lock()
    ready = true
    cond.Broadcast()
    mu.Unlock()
}
```



## 1.6 Sync.Map

`sync.Map` 是 Go 语言中提供的一个并发安全的映射（map）类型，它允许在多个 goroutines 之间进行安全的读写操作，而无需额外的同步机制，如互斥锁（mutex）。`sync.Map` 的设计初衷是为了在并发场景下提供比传统的 `map` 加上互斥锁更高效的读写操作。

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var sm sync.Map

	// 存储键值对
	sm.Store("key1", "value1")

	// 读取键值对
	value, ok := sm.Load("key1")
	// 读取键值对
	value, ok := sm.Load("key1")
	if ok {
		fmt.Println("Loaded value:", value)
	} else {
		fmt.Println("Key not found")
	}

	// 读取或存储键值对，如果键不存在则存储它
	_, loaded := sm.LoadOrStore("key2", "value2")
	if loaded {
		fmt.Println("Key already exists")
	} else {
		fmt.Println("Key-value pair stored")
	}

	// 删除键值对
	sm.Delete("key1")

	// 遍历所有的键值对
	sm.Range(func(key, value interface{}) bool {
		fmt.Printf("Key: %v, Value: %v\n", key, value)
		return true // 返回 true 继续遍历，返回 false 停止遍历
	})
}
```

`sync.Map` 的内部实现相对于传统的 `map` 来说更为复杂，它旨在减少锁争用（lock contention）并优化读写性能。其设计考虑到了读多写少的场景，其中读操作通常是无锁的，而写操作则通过精细的锁粒度来减少锁争用。

以下是 `sync.Map` 的主要组件及其作用：

1. **read**：每个 `sync.Map` 实例包含一个 `read` 结构体，用于存储最近读取的键值对和指向相关桶（bucket）的指针。这有助于优化无锁读操作。
2. **dirty**：一个标准的 `map`，用于存储新写入的键值对以及由于并发修改而需要更新的键值对。当 `dirty` 不为空时，它包含了 `sync.Map` 的最新状态。
3. **misses**：一个计数器，记录了在 `read` 中未找到键的次数。当 `misses` 达到一定阈值时，会将 `dirty` 的内容提升（promote）到 `read` 中，并清空 `dirty`，以减少未来读操作的开销。
4. **mu**：一个互斥锁，保护 `dirty` 的读写以及 `read` 和 `dirty` 之间的提升操作。

在 `sync.Map` 的实现中，读操作通常不需要获取锁，它会先尝试从 `read` 中读取键值对。如果 `read` 中没有所需的键值对，或者 `read` 和 `dirty` 之间的状态不一致（由于并发写操作），则会尝试获取 `mu` 锁并从 `dirty` 中读取或更新键值对。写操作则需要获取 `mu` 锁来修改 `dirty`。

这种设计使得 `sync.Map` 在高并发读、低并发写的场景中表现良好，但在写操作非常频繁的情况下，性能可能不如使用单个互斥锁保护的 `map`。因此，在选择使用 `sync.Map` 还是传统的 `map` 加锁时，需要根据实际的使用场景进行权衡。
