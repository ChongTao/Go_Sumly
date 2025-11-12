# 一 FreeCache

[FreeCache](https://github.com/coocood/freecache) 由 **coocood**开源，最初用于性能敏感的 Web 服务。与 `go-cache` 不同，它专注于：

- **极致性能**（无 GC 压力）
- **高并发安全（lock-free 读）**
- **精确的内存控制（固定容量）**
- **内置 TTL 支持**

它的目标是：提供一个能在 **数百万条缓存、数十万 QPS 并发** 下仍能保持稳定延迟的 Go 本地缓存。

# 二 设计目标与解决问题

| 问题           | FreeCache 的解决方案                                  |
| -------------- | ----------------------------------------------------- |
| Go GC 导致卡顿 | 缓存全部放入 **预分配的字节环形缓冲区**，避免 GC 管理 |
| map 扩容阻塞   | 使用 **固定大小分片数组 + 环形存储区**，不扩容        |
| 写锁竞争       | 分段（Sharding） + **读无锁**                         |
| 缺乏 TTL       | 每条记录内置过期时间戳，精准过期                      |
| 不可控内存增长 | 缓存大小在初始化时固定                                |

# 三 实现原理

## 3.1 数据结构

```go
type Cache struct {
	locks    [segmentCount]sync.Mutex
	segments [segmentCount]segment  // 分片
}
```

每个 segment 独立处理读写，内部使用一个 **环形缓冲区（ring buffer）** 存放键值对：

```text
+---------------------------------------------+
| keyLen | valLen | expireAt | key | value | ...
+---------------------------------------------+
```

- 所有条目都连续存储在 `[]byte` 内；

- 通过偏移量访问，不创建 map[string]interface{}；

- 内存由程序自行管理，而非 Go GC。

## 3.2 分片

- 默认 **256 个 segment**；

- 每个 segment 独立锁，减少竞争；

- 通过 hash(key) 决定落在哪个 segment。

## 3.3 环形缓冲区（Ring Buffer）

- 每个 segment 有一个固定大小的 `byte` 数组；
- 写入时按顺序写入；
- 当空间用完时，从头覆盖旧数据；
- 不会额外分配内存，因此延迟稳定。

##  3.4 哈希索引与查找

- 每个 segment 内维护一个索引表（基于哈希桶）；
- 每个桶记录多个条目偏移量；
- 查找 key 时在桶中匹配（通过 key 比较）；
- 时间复杂度 O(1) 均摊。

## 3.5 过期与删除机制

- 每条记录包含 `expireAt`；
- 获取时懒删除（访问时检测是否过期）；
- 过期条目在下次写入时覆盖；
- 无独立清理线程，因此无需锁和 GC。

# 四 示例

```go
package main

import (
    "fmt"
    "time"
    "github.com/coocood/freecache"
)

func main() {
    // 初始化缓存：100MB
    cacheSize := 100 * 1024 * 1024
    cache := freecache.NewCache(cacheSize)

    // 设置缓存，过期时间 5 秒
    cache.Set([]byte("name"), []byte("Tom"), 5)

    // 读取缓存
    entry, err := cache.Get([]byte("name"))
    if err == nil {
        fmt.Println("Found:", string(entry))
    }

    // 模拟过期
    time.Sleep(6 * time.Second)
    _, err = cache.Get([]byte("name"))
    fmt.Println("Expired:", err == freecache.ErrNotFound)

    // 获取统计信息
    fmt.Printf("Hit Rate: %.2f%%\n", cache.HitRate()*100)
}
```

