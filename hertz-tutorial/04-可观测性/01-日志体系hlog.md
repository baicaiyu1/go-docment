# 日志体系 hlog
> 一句话定位：从 hlog 默认实现到 zap/logrus 替换，讲清日志“怎么写、写到哪、怎么检索”。
## 本篇目标
- 理解 `pkg/common/hlog` 的接口分层与全局函数
- 掌握日志级别、格式化、输出重定向与滚动切割
- 把默认 logger 换成 zap / logrus，业务代码零改动
- 写请求日志中间件，用 trace id 串起一次请求
- 建立生产日志规范：结构化、脱敏、采样、分级
## 前置知识
- 已完成 [03-基本特性/05-中间件](../03-基本特性/05-中间件.md)
- 了解 Go 的 `io.Writer`、`context.Context`
- 知道 zap 或 logrus 的基本用法
## 一、hlog 在 Hertz 中的位置
日志抽象在 `pkg/common/hlog`。框架内部（读包错误、启动信息）与业务代码都调用它，因此**替换一次实现，两边同时生效**：

| 能力 | 用法 | 说明 |
| --- | --- | --- |
| 全局函数 | `hlog.Info`、`hlog.Errorf`、`hlog.CtxInfof` | 调用当前注入的 logger |
| 管理函数 | `SetLogger`、`SetOutput`、`SetLevel` | 替换实现、重定向、调级别 |

接口分为 `Logger`、`FormatLogger`、`CtxLogger`、`Control`，四者组合为 `FullLogger`：

```go
type FullLogger interface {
	Logger        // Trace/Debug/Info/Notice/Warn/Error/Fatal
	FormatLogger  // Xxxf
	CtxLogger     // CtxXxxf(ctx, ...)
	Control       // SetLevel / SetOutput
}
```

自定义实现必须满足 `FullLogger`，否则 `SetLogger` 编译不通过。只增强部分行为时，用嵌入已有实现的方式（见第四节）。
## 二、默认实现与日志级别
```go
h := server.Default()
h.GET("/ping", func(ctx context.Context, c *app.RequestContext) {
	hlog.Info("收到 ping 请求")
	hlog.CtxInfof(ctx, "method=%s path=%s", c.Request.Method(), c.Request.Path())
	c.String(consts.StatusOK, "pong")
})
```

级别共 7 档，从低到高：

| 级别 | 典型用途 |
| --- | --- |
| `LevelTrace` | 框架级细粒度排查，生产关闭 |
| `LevelDebug` | 开发调试、关键分支入参 |
| `LevelInfo` | 启动、配置加载、状态变更 |
| `LevelNotice` | 需关注但不一定是问题 |
| `LevelWarn` | 可恢复异常、降级、重试 |
| `LevelError` | 请求失败、依赖调用失败 |
| `LevelFatal` | 不可恢复错误，通常伴随退出 |

```go
hlog.SetLevel(hlog.LevelInfo)
```

语义是“高于阈值才输出”：设为 Info 时 Trace/Debug 被丢弃。生产常再关一个噪声开关：

```go
hlog.SetSilentMode(true) // 关闭 “error when reading request headers” 之类日志
```
## 三、输出到文件与滚动切割
```go
f, err := os.OpenFile("./logs/app.log", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
if err != nil {
	panic(err)
}
hlog.SetOutput(f)
```

重定向后终端不再输出。开发时常需双写：

```go
hlog.SetOutput(io.MultiWriter(f, os.Stdout))
```

hlog 不负责滚动切割（以官方文档为准），生产交给 lumberjack：

```bash
go get gopkg.in/natefinch/lumberjack.v2
```

```go
rotator := &lumberjack.Logger{
	Filename:   "logs/app.log",
	MaxSize:    200,  // 单文件 MB
	MaxBackups: 30,   // 保留份数
	MaxAge:     30,   // 保留天数
	Compress:   true,
	LocalTime:  true,
}
hlog.SetOutput(rotator)
```

“按天切割”两种做法：应用内按大小轮转 + 采集侧按天归档（推荐）；或应用只写 stdout，交给 Loki/ELK 切分。不要自己在业务里按 0 点 `Reopen` 文件，多副本、重启、时区都会让逻辑变复杂。
## 四、接入 zap / logrus
社区实现统一在 `hertz-contrib/logger`：

| 目录 | 日志库 | 导入路径 |
| --- | --- | --- |
| `zap` | uber-go/zap | `github.com/hertz-contrib/logger/zap` |
| `logrus` | sirupsen/logrus | `github.com/hertz-contrib/logger/logrus` |
| `zerolog` | rs/zerolog | `github.com/hertz-contrib/logger/zerolog` |
| `slog` | 标准库 | `github.com/hertz-contrib/logger/slog` |
| `accesslog` | 访问日志中间件 | `github.com/hertz-contrib/logger/accesslog` |

```bash
go get github.com/hertz-contrib/logger/zap
```

```go
import (
	"github.com/cloudwego/hertz/pkg/common/hlog"
	hertzzap "github.com/hertz-contrib/logger/zap"
)

func init() {
	// 自定义 core 时通过选项注入，选项名以官方 README 为准
	hlog.SetLogger(hertzzap.NewLogger())
	hlog.SetLevel(hlog.LevelInfo)
}
```

logrus 只需换成 `hertzlogrus.NewLogger()`。替换后业务代码完全不用改，仍写 `hlog.Infof`。

只增强部分行为时，用嵌入避免实现 20 多个方法：

```go
type Logger struct {
	hlog.FullLogger
}

func New() *Logger { return &Logger{FullLogger: hertzzap.NewLogger()} }

func (l *Logger) CtxErrorf(ctx context.Context, format string, args ...interface{}) {
	l.FullLogger.CtxErrorf(ctx, "[error] "+format, args...)
}
```

注意：默认 logger 基于标准库 `log.Logger`，文件名/行号依赖调用深度。外面再包一层自己的 `Log` 函数，行号可能指向包装函数；强依赖行号时优先直接调用 hlog，或给 zap 配 `AddCallerSkip`（示例示意）。
## 五、请求日志中间件
```go
func AccessLog() app.HandlerFunc {
	return func(ctx context.Context, c *app.RequestContext) {
		start := time.Now()
		c.Next(ctx)
		hlog.CtxTracef(ctx,
			"status=%d cost=%d method=%s full_path=%s client_ip=%s host=%s",
			c.Response.StatusCode(), time.Since(start).Microseconds(),
			c.Request.Header.Method(), c.Request.URI().PathOriginal(),
			c.ClientIP(), c.Request.Host())
	}
}

h.Use(AccessLog())
```

生产版建议补充：`trace_id`/`span_id`、`uid`/`tenant`、`bytes_in/out`、命中的路由模板、业务错误码。路径用 `PathOriginal()`，聚合指标用路由模板，避免 `/user/123` 与 `/user/456` 被当成两个维度。
## 六、trace id 串联
推荐顺序：

1. 用 `hertz-contrib/requestid` 或自建中间件读取/生成 `X-Request-ID`
2. 写入响应头，方便客户端反馈
3. 日志统一用 `hlog.CtxInfof(ctx, ...)`
4. 调用下游时继续透传

```go
import "github.com/hertz-contrib/requestid"

h.Use(requestid.New()) // 选项以该库 README 为准
h.Use(AccessLog())
```

如果已接入 OpenTelemetry，日志优先带 trace_id/span_id，request id 作为业务兜底关联键。关键点是第一个参数必须是请求链路里的 ctx；很多“日志没有 trace id”的问题，根因都是中途换成了 `context.Background()`。
## 七、生产日志规范
| 规范 | 做法 | 反例 |
| --- | --- | --- |
| 结构化 | JSON 输出，字段固定 | 拼接长字符串 |
| 分级 | Info 记状态，Error 记失败原因 | 全用 Info |
| 脱敏 | 手机号/身份证/密码/token 统一处理 | 打印整个请求体 |
| 采样 | 高频成功日志按比例，错误全量 | 1 万 QPS 每请求 3 条 Info |
| 限长 | 超长 body 截断（如 4KB） | 打印上传文件内容 |
| 不吞错 | `Errorf("查询用户失败: %v", err)` | `Error("失败了")` |
| 不打敏感头 | 过滤 Authorization、Cookie | 原样打印全部 header |

```go
func maskPhone(s string) string {
	if len(s) != 11 {
		return "***"
	}
	return s[:3] + "****" + s[7:]
}
```

采样不要在每条日志上用 `rand` 判断，成本高且不可复现。成功日志按 1%~10% 采样、错误全量，或在采集侧（Vector/Fluent Bit）做采样。
## 常见坑
1. `SetOutput` 后终端没日志：重定向是替换而非追加，需用 `io.MultiWriter`。
2. 设了 `LevelInfo` 看不到 Debug：级别是“高于阈值才输出”，排障时临时调低。
3. 日志没有 trace id：用了 `hlog.Infof` 而不是 `CtxInfof`，或把 ctx 换成了 Background。
4. 接入 zap 后文件名行号偏移：多包了一层，需要 `AddCallerSkip` 或直接调用 hlog。
5. 日志文件无限增长打满磁盘：只 `SetOutput` 没做轮转。
6. 多副本写同一个 NFS 文件：互相覆盖，容器应统一走 stdout 采集。
## 小结
- hlog 是“接口 + 全局函数 + 可替换实现”，替换不影响业务代码
- 级别、输出、静默模式是最常用的三个运行时开关
- 切割交给 lumberjack 或采集系统，不要手写定时器
- AccessLog + Ctx 系列方法，是 trace id 串联的最低成本方案
- 生产日志的重点是“结构化、可聚合、可脱敏、可采样”
## 练习
1. 把默认 logger 换成 zap，JSON 输出到 `logs/app.log`，同时保留终端输出。
2. 写 AccessLog 中间件，额外记录 `X-UID` 与响应字节数。
3. 模拟 1 万次请求，验证成功日志采样 10%、错误全量。
4. 在 handler 里用 `context.Background()` 打日志，复现 trace id 丢失并修复。
## 官方文档
- 日志：<https://www.cloudwego.io/zh/docs/hertz/tutorials/observability/log/>
- 日志扩展：<https://www.cloudwego.io/zh/docs/hertz/tutorials/framework-exten/log/>
- hertz-contrib/logger：<https://github.com/hertz-contrib/logger>
- 访问日志中间件：<https://www.cloudwego.io/zh/docs/hertz/tutorials/third-party/middleware/access-log/>
