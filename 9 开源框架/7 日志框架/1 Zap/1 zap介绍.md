# 1 Zap简介

**[Zap](https://github.com/uber-go/zap)** 是由 Uber 开发的 Go 语言日志库，定位 **高性能、结构化日志框架**。适用于需要高性能日志输出的 Go 服务、微服务或大规模系统，特点：

1. **高性能** — 基于预分配对象和零分配（Zero-allocation）设计，写日志开销极小。
2. **结构化日志** — 支持 JSON 格式输出，便于日志分析、聚合和搜索。
3. **灵活配置** — 支持多种输出方式（stdout、文件、网络等），可配置日志级别。
4. **丰富的字段类型** — 支持多种类型（字符串、int、float、时间、对象等）的字段输出。

# 2 架构设计

Zap 采用 **Logger + Core + Encoder + Syncer** 的设计模式：

```lua
+---------------------+
|       Logger        |  <- 提供接口和方法，如 Info, Error
+---------------------+
           |
           v
+---------------------+
|        Core         |  <- 日志处理核心，决定级别、输出策略
+---------------------+
           |
           v
+---------------------+        +---------------------+
|      Encoder        | -----> |   JSON / Console    |  <- 日志格式化
+---------------------+        +---------------------+
           |
           v
+---------------------+
|       Syncer        |  <- 日志写入目标（文件、stdout、网络）
+---------------------+
```

- **Logger**：应用使用的主要接口，支持 Info, Warn, Error, DPanic, Fatal 等级别。

- **Core**：日志核心，用于判断日志级别、格式化和输出。

- **Encoder**：编码器，支持 `JSONEncoder`（结构化日志）和 `ConsoleEncoder`（控制台文本日志）。

- **Syncer**：日志输出目标，负责将日志写入文件、标准输出或其他 io.Writer。



# 3 Zap的两种模型

1. **Sugared Logger（糖果接口）**

   ```go
   sugar := zap.NewExample().Sugar()
   sugar.Infof("User %s logged in", userName)
   sugar.Infow("User login",
       "user", userName,
       "ip", ipAddress,
   )
   ```

2. Structured Logger（结构化 Logger）

   ```go
   logger := zap.NewExample()
   logger.Info("User login",
       zap.String("user", userName),
       zap.String("ip", ipAddress),
   )
   ```

# 4 功能

**日志级别**：

- Debug, Info, Warn, Error, DPanic, Panic, Fatal
- 可以动态调整输出级别

**字段类型**：

- 基本类型：`zap.String`, `zap.Int`, `zap.Bool`, `zap.Float64`
- 时间类型：`zap.Time`
- 对象类型：`zap.Any`，可以输出复杂对象

**输出方式**：

- 控制台 (Console)
- JSON (适合 ELK / Loki / Grafana 等日志系统)
- 文件、Rolling log（可与 lumberjack 搭配实现文件切割）

**采样（Sampling）**：

- 支持日志采样，防止高频日志刷爆磁盘或网络

- 示例：

  ```
  cfg := zap.NewProductionConfig()
  cfg.Sampling = &zap.SamplingConfig{
      Initial:    100,
      Thereafter: 100,
  }
  ```

**日志切割 / 文件轮转**：

- Zap 本身不处理文件切割，但可与 [`lumberjack`](https://github.com/natefinch/lumberjack) 配合
- 生产环境推荐这种组合

**Stacktrace / Caller**：

- 可记录调用函数和文件行号
- 支持错误堆栈输出

# 5 示例

```go
import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

func main() {
    cfg := zap.Config{
        Level:       zap.NewAtomicLevelAt(zap.InfoLevel),
        Development: false,
        Encoding:    "json", // "console" 或 "json"
        EncoderConfig: zapcore.EncoderConfig{
            TimeKey:        "ts",
            LevelKey:       "level",
            NameKey:        "logger",
            CallerKey:      "caller",
            MessageKey:     "msg",
            StacktraceKey:  "stacktrace",
            LineEnding:     zapcore.DefaultLineEnding,
            EncodeLevel:    zapcore.LowercaseLevelEncoder,
            EncodeTime:     zapcore.ISO8601TimeEncoder,
            EncodeDuration: zapcore.StringDurationEncoder,
            EncodeCaller:   zapcore.ShortCallerEncoder,
        },
        OutputPaths:      []string{"stdout", "./app.log"},
        ErrorOutputPaths: []string{"stderr"},
    }

    logger, _ := cfg.Build()
    defer logger.Sync()
    
    logger.Info("Service started", zap.String("version", "1.0.0"))
}
```





