# 一 Ristretto

[Ristretto](https://github.com/dgraph-io/ristretto)是由 **Dgraph Labs** 开发的高性能 Go 本地缓存库，目标是：在保持高命中率（hit ratio）的前提下，提供极高的吞吐性能和并发安全。

它的设计灵感来自 **现代缓存系统（如 Caffeine、ARC、TinyLFU）**，是一种在 **性能、内存利用率和准确性** 之间平衡得非常好的方案。

# 二 Ristretto 解决的问题

Ristretto 的出现主要是为了解决传统缓存（如 `go-cache`、`bigcache`）在高并发和复杂负载下的几个痛点

| 问题               | 传统方案缺陷                    | Ristretto 解决方式                                 |
| ------------------ | ------------------------------- | -------------------------------------------------- |
| **并发性能差**     | `sync.RWMutex` 锁竞争严重       | 使用 **分段 + 原子操作 + Ring Buffer**             |
| **内存利用率低**   | 无法根据 key/value 大小控制容量 | 使用 **Cost-based eviction（基于代价的淘汰策略）** |
| **淘汰策略不智能** | LRU 仅考虑最近使用，命中率低    | 使用 **TinyLFU + Sampled LFU**，命中率更高         |
| **统计不准确**     | 实时更新代价高                  | 使用 **近似计数（Approximate Counters）** 提升性能 |

# 三 实现原理

Ristretto 的底层实现非常有意思，它融合了多个算法和工程技巧：

## 3.1 写入流程（Write Pipeline）

当你调用 `Set(key, value, cost)` 时：

1. 请求进入 **ring buffer**（一个固定大小的写队列）；
2. 异步后台 goroutine 处理这些写请求；
3. 将 key 哈希后映射到 **多个分段（shard）** 中；
4. 同时记录 key 的 “cost”（可代表内存大小、权重等）；
5. 若超出 `MaxCost`，执行 **淘汰策略**（TinyLFU）；

这样可以极大降低锁争用，提高写性能。

## 3.2 淘汰算法（Eviction Policy）

Ristretto 使用了一种混合算法：

> **TinyLFU + Sampled LFU**

- **TinyLFU**：使用滑动窗口记录访问频率，结合布隆过滤器原理，仅保留常用项；
- **Sampled LFU**：从缓存中随机抽样一部分数据，通过比较频率选择要淘汰的项；
- 这种方式既高效又接近最优命中率。

## 3.3 Cost-based Capacity

传统缓存一般用「数量」限制，比如最多存 N 条；但 Ristretto 可以用「cost」来限制内存用量，例如：

```go
MaxCost = 1 << 30 // 1GB
```

每次 `Set()` 时，你可以指定每个 key 的 cost，比如：

```go
cache.Set("user:1", userData, int64(len(userData)))
```

这样可以更精确地控制整体内存占用。

## 3.4 统计与并发优化

- **Approximate Counter**：类似 HyperLogLog，降低实时统计开销；
- **Ring Buffer + Atomic 操作**：减少锁竞争；
- **Sharded Map**：多 segment 并发安全地操作不同区域。

# 四 示例

```go
package main

import (
    "fmt"
    "github.com/dgraph-io/ristretto"
)

func main() {
    cache, _ := ristretto.NewCache(&ristretto.Config{
        NumCounters: 1e7,     // 频率估算用计数器数量（越大越精确）
        MaxCost:     1 << 30, // 最大内存代价（如 1GB）
        BufferItems: 64,      // 写缓冲区大小
    })

    cache.Set("key", "value", 1)
    cache.Wait() // 等待写入完成

    value, found := cache.Get("key")
    if found {
        fmt.Println(value)
    }
}
```

