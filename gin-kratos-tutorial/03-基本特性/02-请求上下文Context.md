# 请求上下文 Context

> 一句话定位：`*gin.Context` 是一次请求的「工作台」，从对象池取用、请求结束立即重置；跨 goroutine 或跨请求持有它必然读到别人的数据，而真正该传给下游的是 `c.Request.Context()`。

## 本篇目标

- 说清 `*gin.Context` 的结构、生命周期与复用风险。
- 会用 `c.Request.Context()` 把取消与超时传到 GORM、Redis、gRPC。
- 会用 `c.Set/Get/MustGet` 做请求级传值，并知道 `Keys` 的并发边界。
- 掌握 `c.Copy()` 的正确用法，避免异步代码里的数据竞争；熟练读取 Query、Form、Path、Header、Cookie。

## 前置知识

- 学过 Go 的 `context` 包，见 [并发-goroutine-channel-context](../01-Go语言基础/07-并发-goroutine-channel-context.md)。
- 已完成 [01-路由](./01-路由.md)，知道 handler 签名是 `func(*gin.Context)`。

## 一、它是什么

`Context` 把本次请求的所有状态捏在一起：`Request`、`Writer`（响应）、`Params`（路径参数）、`handlers` + `index`（中间件链执行位置）、`Keys`（`Set/Get` 存储）、`Errors`（`c.Error` 收集的错误）。

生命周期是一条闭环：

```text
请求到达 → engine 从 sync.Pool 取 Context → 绑定 Request/Writer → 组装 handlers
→ c.Next() 执行整条链 → 写响应 → reset（清空 Keys/Params/handlers，index=-1）→ 放回池
```

由此得到三条铁律：

- 不要自己 `&gin.Context{}`，必须由框架创建。
- 不要把它存进全局变量或长生命周期结构体，更不要交给 goroutine（除非先 `c.Copy()`）。
- `reset()` 会把 `Keys` 置为 `nil`，handler 返回后再读就是空。

## 二、两个 context 别混用

`c.Request.Context()` 是标准库 `context.Context`，请求结束或客户端断开时被 cancel。它是**唯一**该传给下游的 context：

```go
// app/gateway/internal/handler/user.go
func (h *UserHandler) Get(c *gin.Context) {
    uid, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"code": 40001, "message": "id 非法"})
        return
    }
    user, err := h.svc.GetUser(c.Request.Context(), uid) // 取消、超时、trace 全靠它
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"code": 50000, "message": "查询失败"})
        return
    }
    c.JSON(http.StatusOK, gin.H{"code": 0, "data": user})
}
// repo：GORM 必须带 WithContext，否则超时与取消传不进驱动
func (r *UserRepo) GetByID(ctx context.Context, id int64) (*model.User, error) {
    var u model.User
    if err := r.db.WithContext(ctx).First(&u, id).Error; err != nil {
        return nil, err
    }
    return &u, nil
}
```

- gRPC 客户端把 `ctx` 作为第一个参数透传；Redis 用 `rdb.Get(ctx, key)`，框架会自动处理 deadline 与 cancel。
- 不要用 `c.Set` 冒充 context 交给 service，下游既收不到取消信号，也拿不到 trace；要给请求换 context（如加超时），只能 `c.Request = c.Request.WithContext(ctx)`，因为 service 拿不到 `c`。不同框架的 Context 不通用，跨进程只能靠显式参数或标准库 `context.Context`。

## 三、请求级传值：Set / Get / MustGet

```go
const ctxUserID = "auth.user_id" // 统一前缀，避免不同中间件互相覆盖

func Auth() gin.HandlerFunc {
    return func(c *gin.Context) {
        uid, err := parseToken(c.GetHeader("Authorization"))
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"code": 40100, "message": "未登录"})
            return
        }
        c.Set(ctxUserID, uid)
        c.Next()
    }
}

// handler 中读取
uid := c.GetInt64(ctxUserID) // 不存在时返回 0
raw, ok := c.Get(ctxUserID)  // 需要区分「没设置」时看 ok
```
- `Get` 系列有 `GetString/GetBool/GetInt/GetInt64/GetFloat64/GetDuration/GetTime`，类型不符返回零值，不报错。
- `MustGet` 在 key 不存在时 panic，只用于「一定有」的内部约定。
- `c.Keys` 是普通 map，`Set` 不加锁，只能在 handler 链的同一个 goroutine 里读写。
- 不要往 `Keys` 里塞大对象，它跟着请求活到响应结束。

## 四、并发：c.Copy()

```go
// 错误：handler 返回后 c 被 reset 放回池，goroutine 读到别人的请求
go func() { log.Println("id =", c.Param("id")) }()

// 正确：Copy 出一份独立上下文供后台任务使用
cc := c.Copy()
go func() { log.Println("id =", cc.Param("id")) }() // cc 写响应不会发给客户端
```

`Copy()` 复制了 `Request`、`Writer`、`Params` 与 `Keys`（浅拷贝，值里的指针仍共享）。更稳的做法是进 goroutine 前把值抽成普通变量，并用 `context.WithoutCancel` 保留 trace、避免客户端断开就中断后台任务：

```go
uid := c.GetInt64(ctxUserID)
ctx, cancel := context.WithTimeout(context.WithoutCancel(c.Request.Context()), 5*time.Second)
defer cancel()
go func(uid int64, ctx context.Context) { _ = orderSvc.AsyncCreate(ctx, uid) }(uid, ctx)
```

## 五、取消与超时传播

```go
// app/gateway/internal/middleware/timeout.go
func Timeout(d time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        ctx, cancel := context.WithTimeout(c.Request.Context(), d)
        defer cancel() // 及时释放 timer
        c.Request = c.Request.WithContext(ctx)
        c.Next()
    }
}

// handler：判断下游返回的错误，不要自己轮询
user, err := h.svc.GetUser(c.Request.Context(), uid)
switch {
case errors.Is(err, context.DeadlineExceeded):
    c.AbortWithStatusJSON(http.StatusGatewayTimeout, gin.H{"code": 50400, "message": "下游超时"})
case errors.Is(err, context.Canceled):
    return // 客户端已断开，写响应没有意义
case err != nil:
    c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"code": 50000, "message": "查询失败"})
default:
    c.JSON(http.StatusOK, gin.H{"code": 0, "data": user})
}
```

Gin 不会因为 context 到期就杀掉 handler，超时**只能靠下游配合**：所有 IO 都带 ctx，才会在阻塞点返回。

## 六、读取请求数据速查

| 来源 | 方法 | 备注 |
| --- | --- | --- |
| Query | `c.Query` / `c.DefaultQuery` / `c.GetQuery` / `c.QueryArray("tag")` | `GetQuery` 返回 `(string, bool)` |
| Form | `c.PostForm` / `c.DefaultPostForm` | 兼容 urlencoded 与 multipart |
| Path / Header / Cookie | `c.Param("id")` / `c.GetHeader(...)` / `c.Cookie("sid")` | Header key 大小写不敏感 |
| Body / IP | `c.GetRawData()` / `c.ClientIP()` | Body 只能读一次，优先用绑定 |

## 完整示例

```go
// app/gateway/cmd/main.go
package main

import (
    "cmp"
    "log"
    "net/http"

    "github.com/gin-gonic/gin"
)

const ctxRequestID = "meta.request_id"

func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        id := cmp.Or(c.GetHeader("X-Request-Id"), "generated")
        c.Set(ctxRequestID, id)
        c.Header("X-Request-Id", id)
        c.Next()
    }
}

func main() {
    r := gin.New()
    r.Use(gin.Recovery(), RequestID())
    r.GET("/echo/:name", func(c *gin.Context) {
        cc := c.Copy()
        go func() {
            log.Printf("async name=%s rid=%s", cc.Param("name"), cc.GetString(ctxRequestID))
        }()
        c.JSON(http.StatusOK, gin.H{"code": 0, "data": gin.H{
            "name": c.Param("name"), "q": c.DefaultQuery("q", "none"), "rid": c.GetString(ctxRequestID),
        }})
    })
    if err := r.Run("127.0.0.1:8080"); err != nil { panic(err) }
}
```

Windows PowerShell：

```powershell
go run .
curl.exe -i "http://127.0.0.1:8080/echo/tom?q=hi" -H "X-Request-Id: req-1001"
```

Linux / macOS bash：

```bash
go run .
curl -i "http://127.0.0.1:8080/echo/tom?q=hi" -H "X-Request-Id: req-1001"
```

## 常见坑

1. **把 `c` 丢进 goroutine**：handler 返回后 `c` 被复用，异步代码读到别人的数据，用 `c.Copy()` 或提前抽值。
2. **用 `c.Set` 传 context**：service 要的是 `context.Context`，用 `c.Request.Context()`，否则超时与 trace 全断。
3. **GORM 漏写 `WithContext`**：请求取消后 SQL 仍在跑，慢查询会拖垮连接池。
4. **`MustGet` 用在不一定存在的 key 上**：key 缺失直接 panic，对外接口变 500。
5. **客户端断开后继续写响应**：`context.Canceled` 时直接 return。
6. **先 `GetRawData` 再 `ShouldBindJSON`**：Body 已被消费，绑定得到空对象。

## 小结

- `*gin.Context` 池化复用，只能在当前请求的 goroutine 内使用。
- 传给 GORM/Redis/gRPC 的永远是 `c.Request.Context()`，改 context 要回写 `c.Request`。
- 请求级传值用 `Set/Get`，key 加前缀；异步任务用 `c.Copy()` 或提前抽值，超时靠 context 向下游传播。

## 练习

1. 写一个中间件把 `X-Request-Id` 写入上下文，在 handler 与后台日志里各打印一次。
2. 故意在 goroutine 里直接读 `c.Param("id")`，并发压测后观察现象，再改成 `c.Copy()`。
3. 给 `/slow` 加 200ms 超时，让下游 `time.Sleep(1s)`，验证返回 504 而不是 200。

## 官方文档

- Gin Context API：<https://pkg.go.dev/github.com/gin-gonic/gin#Context>
- Go `context` 包：<https://pkg.go.dev/context>
- Gin 自定义中间件：<https://gin-gonic.com/zh-cn/docs/middleware/custom/>
