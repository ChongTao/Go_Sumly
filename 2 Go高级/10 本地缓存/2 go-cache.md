# 一 go-cache

[go-cache](https://github.com/patrickmn/go-cache) 是由 **Patrick Mn** 开发的纯 Go 实现的本地内存缓存库。它并不是追求极限性能的方案，而是为了让开发者在不引入外部依赖（如 Redis）的情况下，**快速拥有一个可过期、线程安全、可清理的本地缓存。**

# 二 适用场景与设计目标

### 2.1 设计目标

- 简单：几行代码即可完成缓存管理；
- 轻量：基于 Go 原生 map + RWMutex；
- 安全：并发安全；
- 可靠：支持 TTL 过期时间与自动清理；
- 通用：支持任何类型的对象（接口类型存储）。

### 2.2 典型使用场景

- 小型服务的请求级缓存；
- 配置、Session、Token 缓存；
- 计算结果缓存（如 Embedding 或中间结果）；
- 降低数据库 / 外部 API 调用频率。

# 三 实现原理

`go-cache` 的实现思想非常直接，可以认为是：一个“带过期机制的 sync.Map”

## 3.1 核心结构

```go
type Cache struct {
    defaultExpiration time.Duration // 默认过期时间
    items             map[string]Item
    mu                sync.RWMutex
    janitor           *janitor // 清理后台任务
}

type Item struct {
    Object     interface{}
    Expiration int64
}
```

## 3.2 过期时间机制

- 每个 Item 都有自己的过期时间戳（Unix 时间）。
- 获取时检查是否过期：
  - 如果过期 → 删除并返回未命中；
  - 如果未过期 → 返回缓存内容。

## 3.3 后台清理协程（Janitor）

- 启动时自动启动一个 goroutine：

  ```go
  for {
      time.Sleep(cleanupInterval)
      c.DeleteExpired()
  }
  ```

## 3.4 并发安全

- 通过 `sync.RWMutex` 保护 `map` 的读写；
- 支持高并发读，写入时锁住 map。

# 四 示例

```go
package main

import (
    "fmt"
    "time"
    "github.com/patrickmn/go-cache"
)

func main() {
    // 创建缓存：默认过期时间 5分钟，清理周期 10分钟
    c := cache.New(5*time.Minute, 10*time.Minute)

    // 设置键值
    c.Set("token_123", "abc123", cache.DefaultExpiration)

    // 获取键值
    token, found := c.Get("token_123")
    if found {
        fmt.Println("Found:", token)
    }

    // 设置带自定义过期时间
    c.Set("temp", "hello", 3*time.Second)
    time.Sleep(4 * time.Second)
    _, found = c.Get("temp")
    fmt.Println("Expired:", !found)

    // 删除键
    c.Delete("token_123")
}
```

