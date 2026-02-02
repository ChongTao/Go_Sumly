# 一 robfig/cron

[**robfig/cron**](https://github.com/robfig/cron) 是一个：

- **基于 cron 表达式的定时调度器**
- 用于 **在 Go 进程内调度任务**
- 到点就触发（不关心任务是否完成）



# 二 使用

## 2.1 创建Cron实例

```go
c := cron.New()
```

- 一个进程里可以有多个 Cron
- **通常只需要一个全局实例**

------

## 2.2 添加定时任务

每注册一个任务，都会生成一个 `Entry`：

```
c.AddFunc("*/5 * * * *", func() {
    fmt.Println("run every 5 minutes")
})
```

## 2.3 启动调度器

```go
c.Start()
defer c.Stop()
```

