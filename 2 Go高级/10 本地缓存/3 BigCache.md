# 一 BigCache

BigCache 由 **Allegro.pl**（欧洲大型电商平台）开发并开源。他们的 Go 服务需要在内存中缓存大量小对象（数百万级）， 但发现传统缓存方案（如 `map[string]interface{}` 或 `go-cache`）在大规模数据下会频繁触发 **GC（垃圾回收）**，导致延迟抖动和性能下降。于是，他们开发了 [**BigCache**](https://github.com/allegro/bigcache) —— 一个 **高并发、安全且无 GC 压力** 的本地缓存库。

# 二 设计目标与解决的问题

| 问题                      | BigCache 的解决方式                                          |
| ------------------------- | ------------------------------------------------------------ |
| **GC 压力过大**           | 数据存储为 **连续字节数组（[]byte）**，不将每个 key/value 都变成单独对象，从而减少 GC 负担。 |
| **高并发写入冲突**        | 使用 **sharding 分段锁机制**，每个分段独立锁，减少全局锁竞争。 |
| **TTL（过期）管理效率低** | 每个分段维护独立的时间索引，过期检查时批量清理而非逐项遍历。 |
| **序列化开销大**          | 用户可自行管理序列化（如 JSON、protobuf），内部存储纯字节。  |
| **map 扩容阻塞**          | 内部分片结构固定容量，避免 map 频繁扩容引发性能波动。        |

# 三 实现原理

1. 分片（Sharding）机制

   - 默认分为 **1024 个分片（segment）**。

   - 每个分片内部是一个 map-like 结构，有自己的锁。

   - 读写操作通过哈希（如 `fnv32(key) % 1024`）选择对应的分片。

   - 优点：减少锁竞争、并发安全。

     ```go
     segmentID := fnv32(key) % segmentCount
     cache.shards[segmentID].Set(key, entry)
     ```

2.  数据结构设计

   - 数据按字节形式连续存储在环形缓冲区中。

   - 避免创建大量小对象，从而显著降低 GC 负担。

3.  过期与清理机制

   - 每个 entry 带时间戳。
   - 读取时进行懒惰删除（lazy expiration），即过期条目在被访问时清理。
   - 后台 goroutine 也可周期性清理旧数据（默认配置可关）。

4.  内存管理

   - 每个分片维护环形 buffer，当写入超出容量时，会覆盖旧数据（类似循环队列）。
   - 因此 BigCache **不会频繁分配新内存**，也不会因为 GC 卡顿。

![ringbuf内数据格式](https://cdn.xiaobaidebug.top/1708825671005.png)

# 四 示例

```go
package main

import (
    "fmt"
    "time"
    "github.com/allegro/bigcache/v3"
)

func main() {
    config := bigcache.Config{
        Shards:             1024,             // 分片数量
        LifeWindow:         10 * time.Minute, // 缓存有效期
        CleanWindow:        1 * time.Minute,  // 清理间隔
        MaxEntriesInWindow: 1000 * 10 * 60,   // 预估的最大缓存数
        MaxEntrySize:       500,              // 单条缓存最大字节数
        Verbose:            false,
    }

    cache, _ := bigcache.NewBigCache(config)

    // 写入缓存
    cache.Set("user:1", []byte("Tom"))

    // 读取缓存
    entry, err := cache.Get("user:1")
    if err == nil {
        fmt.Println(string(entry))
    }

    // 删除缓存
    cache.Delete("user:1")
}
```

