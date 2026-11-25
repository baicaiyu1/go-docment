# 01-Gin迁移到Hertz

> 一句话定位：把 Gin 的路由、绑定、中间件与错误处理逐项映射到 Hertz，用“先跑通、再统一、后切流”的渐进策略完成迁移。

## 本篇目标

- 用一张对照表看懂 `gin.Context` 与 `*app.RequestContext` 的能力映射。
- 通过逐项代码对照与渐进迁移路线，把路由、绑定、中间件、错误处理搬到 Hertz。

## 前置知识

- 已读 [请求上下文](../03-基本特性/02-请求上下文Context.md)、[参数绑定与校验](../03-基本特性/03-参数绑定与校验.md)、[中间件](../03-基本特性/05-中间件.md)、[错误处理](../03-基本特性/06-错误处理.md)。
- 熟悉 Gin 的 `gin.Context`、`ShouldBind`、`r.Group` 与中间件写法。

## 一、先接受三个事实

1. Gin 构建在 `net/http` 上；Hertz 的请求/响应是 `protocol.Request/Response`，默认网络库是 Netpoll，语义更接近 fasthttp。
2. `*gin.Context` 每请求新建；`*app.RequestContext` 来自对象池，handler 返回后会被复用，**不能跨 goroutine 持有**。
3. Hertz 的 `Bind` 不会自动写 400，也没有 `AbortWithError`；错误响应必须自己写。

## 二、核心 API 对照表

| 能力 | Gin | Hertz |
| --- | --- | --- |
| 上下文 | `*gin.Context` | `*app.RequestContext` |
| Handler | `func(*gin.Context)` | `func(context.Context, *app.RequestContext)` |
| 中间件 | `gin.HandlerFunc` | `app.HandlerFunc` |
| 继续链 | `c.Next()` | `c.Next(ctx)` |
| 中止并返回 JSON | `c.AbortWithStatusJSON(code, obj)` | 同名同参数 |
| 记录错误并中止 | `c.AbortWithError(code, err)` | 无；用 `c.Error(err)` + `c.AbortWithStatus(code)` |
| 路径参数 | `c.Param("id")` | `c.Param("id")` |
| Query | `c.Query` / `c.DefaultQuery` | 同名同参数 |
| 表单 | `c.PostForm` / `c.DefaultPostForm` | 同名同参数 |
| 原始 Body | `c.GetRawData()`，`([]byte, error)` | `c.GetRawData()`，`[]byte`；或 `c.Body()` |
| Header | `c.GetHeader(k)`，string | `c.GetHeader(k)`，`[]byte` |
| Cookie | `c.Cookie(k)`，`(string, error)` | `c.Cookie(k)`，`[]byte` |
| 自动绑定 | `c.ShouldBind(&req)` | `c.Bind(&req)` |
| JSON / Query 绑定 | `ShouldBindJSON` / `ShouldBindQuery` | `BindJSON` / `BindQuery` |
| 绑定并校验 | `ShouldBind` + `binding` tag | `BindAndValidate` + `validate` tag |
| JSON / 文本 / 字节 | `c.JSON` / `c.String` / `c.Data` | 同名同参数 |
| 重定向 | `c.Redirect(code, url)` | `c.Redirect(code, []byte(url))` |
| 路由组 | `r.Group("/api")` | `h.Group("/api")` |
| 静态目录 | `r.Static("/assets", "./assets")` | `h.Static("/assets", "./assets")` |
| 静态 FS | `r.StaticFS("/assets", http.Dir("./assets"))` | `h.StaticFS("/assets", &app.FS{Root: "./assets"})` |
| 404 / 405 | `r.NoRoute` / `r.HandleMethodNotAllowed` | `h.NoRoute` / `WithHandleMethodNotAllowed(true)` + `h.NoMethod` |
| 启动 | `r.Run(":8080")` | `h.Spin()` |
| 路由语法 | `:id`、`*action` | `:id`、`*path` |

## 三、逐项代码对照

### 3.1 启动与路由

Gin：

```go
r := gin.Default()
r.GET("/api/v1/users/:id", getUser)
r.POST("/api/v1/users", createUser)
_ = r.Run(":8080")
```

Hertz：

```go
h := server.Default(server.WithHostPorts(":8080"))
h.GET("/api/v1/users/:id", getUser)
h.POST("/api/v1/users", createUser)
h.Spin()
```

### 3.2 请求取值与绑定

Gin：

```go
id := c.Param("id")
page := c.DefaultQuery("page", "1")
name := c.PostForm("name")
raw, _ := c.GetRawData()
token := c.GetHeader("Authorization")
if err := c.ShouldBindJSON(&req); err != nil {
	c.JSON(400, gin.H{"message": err.Error()})
	return
}
```

Hertz：

```go
id := c.Param("id")
page := c.DefaultQuery("page", "1")
name := c.PostForm("name")
raw := c.GetRawData() // 单返回值，Gin 是 ([]byte, error)
token := string(c.GetHeader("Authorization")) // []byte
if err := c.BindAndValidate(&req); err != nil {
	c.JSON(consts.StatusBadRequest, utils.H{"message": err.Error()})
	return
}
```

迁移时把 `binding` 改成 `validate`、`uri` 改成 `path`，并按 [参数绑定与校验](../03-基本特性/03-参数绑定与校验.md) 注入 go-playground validator；`Bind` 的错误必须自己处理。

### 3.3 响应

Gin：

```go
c.JSON(200, gin.H{"code": 0, "data": user})
c.Data(200, "text/csv", buf)
c.Redirect(302, "/new")
```

Hertz：

```go
c.JSON(consts.StatusOK, utils.H{"code": 0, "data": user})
c.Data(consts.StatusOK, "text/csv", buf)
c.Redirect(consts.StatusFound, []byte("/new"))
```

`utils.H` 就是 `map[string]interface{}`；Hertz 的 `Redirect` 不终止 handler，记得 `Abort` 后 `return`。

### 3.4 中间件

Gin：

```go
func Auth() gin.HandlerFunc {
	return func(c *gin.Context) {
		if c.GetHeader("Authorization") == "" {
			c.AbortWithStatusJSON(401, gin.H{"code": 401})
			return
		}
		c.Set("uid", "u-1")
		c.Next()
	}
}
```

Hertz：

```go
func Auth() app.HandlerFunc {
	return func(ctx context.Context, c *app.RequestContext) {
		if len(c.GetHeader("Authorization")) == 0 {
			c.AbortWithStatusJSON(consts.StatusUnauthorized, utils.H{"code": 401})
			return
		}
		c.Set("uid", "u-1")
		c.Next(ctx)
	}
}
```

两边都靠 `index` 推进 handler 链，**真正的中断是 `Abort`**；不调用 `Next` 不会让后续 handler 消失，但会丢掉当前中间件的后置逻辑，洋葱顺序随之错乱。

### 3.5 错误处理

Gin：

```go
if err != nil {
	_ = c.Error(err)
	c.AbortWithStatusJSON(500, gin.H{"code": 500})
}
```

Hertz：

```go
if err != nil {
	c.Error(err)
	c.AbortWithStatusJSON(consts.StatusInternalServerError, utils.H{"code": 500})
}
```

更推荐 `c.Error(err)` + 全局错误中间件统一出参，见 [错误处理](../03-基本特性/06-错误处理.md)。

### 3.6 静态文件

Gin：

```go
r.Static("/assets", "./public")
r.StaticFile("/favicon.ico", "./public/favicon.ico")
```

Hertz：

```go
h.Static("/assets", "./public")
h.StaticFile("/favicon.ico", "./public/favicon.ico")
h.StaticFS("/assets/spa", &app.FS{Root: "./web/dist", IndexNames: []string{"index.html"}})
```

## 四、渐进迁移策略

| 阶段 | 做法 | 验收标准 |
| --- | --- | --- |
| 0 盘点 | 列出路由、中间件、`net/http` 依赖、第三方 SDK | 形成迁移清单与责任人 |
| 1 新接口先上 Hertz | 独立端口/域名，共用 MySQL、Redis；Gin 不动 | 新接口灰度可用 |
| 2 公共能力重写 | JWT、CORS、RequestID、日志、Recovery、Metrics 全部用 Hertz 版 | 两套行为逐项对齐 |
| 3 按域搬迁 | 一个业务域一个 PR，旧路由回源或切到新服务 | 回归通过、错误率不升 |
| 4 双写/灰度 | 网关按 header/cookie/比例分流，离线比对响应 | 一周无差异 |
| 5 下线 Gin | 删除旧进程与适配层 | 监控稳定、成本下降 |

双写只比对响应，不要双写数据库；公共中间件必须重写而不是“套一层”，否则 Context 与指标口径会分裂。

## 五、迁移后验证清单

- [ ] 404/405 与尾斜杠重定向策略一致。
- [ ] Query、Form、JSON、Path 绑定结果一致，含默认值、空值与数组。
- [ ] 校验失败返回 400，错误结构稳定，不泄露内部字段。
- [ ] Header 大小写、Cookie 多值、`Set-Cookie` 属性一致。
- [ ] 中间件顺序、401/403、CORS 预检请求一致。
- [ ] panic 返回 500，堆栈只进日志。
- [ ] 静态资源与 SPA history 回退不吞 API 404。
- [ ] 上传 413 阈值、超时取消传播、优雅退出行为一致。
- [ ] 压测 QPS、P99、内存与 Gin 基线对比可接受。

## 常见坑

1. **把 `*app.RequestContext` 丢进 goroutine**：handler 返回后 Context 被复用，异步任务会读到下一个请求的数据；需要异步先拷贝值。
2. **以为 `c.Bind` 会自动 400**：Gin 的 `Bind` 会，Hertz 的 `Bind` 不会；必须判断错误并 `AbortWithStatusJSON`。
3. **照搬 `binding` 标签**：Hertz 默认校验器不认 `binding`，要改 `validate` 并注入 validator；路径参数 tag 用 `path`。
4. **以为不调用 `Next` 就能中断**：链由 index 推进，`Abort` 才是中断；漏掉 `Next` 只会丢失后置逻辑。
5. **`c.Redirect` 传 string**：第二个参数是 `[]byte`；且 Redirect 不终止 handler，后面必须 `Abort` + `return`。
6. **把 `c.GetHeader` / `c.Cookie` 当 string**：两者返回 `[]byte`，比较前先 `string(...)`。
7. **找不到 `c.AbortWithError`**：Hertz 没有，改用 `c.Error(err)` + `AbortWithStatus`。

## 小结

- API 名字高度相似，真正的差异在 Context 生命周期、绑定错误处理、`[]byte` 返回值与中间件语义。
- 迁移先统一响应、错误与公共中间件，再按业务域搬路由，避免“全局替换 import”式返工。
- 用网关双写与灰度做验证，用清单做回归，保证行为而不只是编译通过。

## 练习

1. 把一篇 Gin 的用户 CRUD 改成 Hertz，保持路由、状态码与 JSON 字段完全一致。
2. 把 Gin 的 JWT 中间件改写成 Hertz 版本，验证未登录 401、放行后能取到 `uid`。

## 相关章节

- [请求上下文 Context](../03-基本特性/02-请求上下文Context.md)
- [参数绑定与校验](../03-基本特性/03-参数绑定与校验.md)
- [中间件](../03-基本特性/05-中间件.md)
- [错误处理](../03-基本特性/06-错误处理.md)
- [适配器与标准库互操作](../03-基本特性/14-适配器与标准库互操作.md)
- [FastHTTP 迁移到 Hertz](02-FastHTTP迁移到Hertz.md)
- [常见问题 FAQ](03-常见问题FAQ.md)
