# 02-API速查表

> 一句话定位：按主题把 Hertz、GORM、go-redis、Kitex 的高频 API 压缩成“用途 | API | 说明”表格，写代码时直接查。

## 本篇目标

- 一张表查完服务启动、路由、取值、绑定、响应、中间件与错误处理。
- 覆盖 Hertz Client、GORM、go-redis、Kitex 的常用方法签名。
- 标注易错点，例如 `WithHostPorts` 单参数、`Cookie` 返回 `[]byte`。

## 前置知识

- 已读 [Engine 详解与启动参数](../02-Hertz入门/05-Engine详解与启动参数.md) 与 [请求上下文 Context](../03-基本特性/02-请求上下文Context.md)。
- API 以 Hertz v0.10.6 为准；Kitex 以 v0.13+ 官方文档为准，升级后注意签名变化。

## 一、服务启动

| 用途 | API | 说明 |
| --- | --- | --- |
| 默认服务器 | `server.Default(opts ...config.Option) *server.Hertz` | 内置 Recovery 等默认中间件 |
| 纯净服务器 | `server.New(opts ...config.Option) *server.Hertz` | 不含默认中间件，需自行加 Recovery |
| 监听地址 | `server.WithHostPorts(hp string) config.Option` | 单个字符串，如 `":8080"`，不是可变参数 |
| 网络库 | `server.WithTransport(t func(*config.Options) network.Transporter) config.Option` | 显式选 netpoll 或标准库 |
| 网络类型 | `server.WithNetwork(nw string) config.Option` | `tcp4`/`tcp6`/`unix` 等 |
| TLS | `server.WithTLS(cfg *tls.Config) config.Option` | 开启 HTTPS |
| 请求体上限 | `server.WithMaxRequestBodySize(bs int) config.Option` | 默认 4MB，含 multipart |
| 405 处理 | `server.WithHandleMethodNotAllowed(b bool) config.Option` | 区分 404 与 405 |
| 退出等待 | `server.WithExitWaitTime(timeout time.Duration) config.Option` | 优雅退出等待在途请求 |
| 启动并处理信号 | `h.Spin()` | 阻塞运行，推荐生产使用 |
| 仅运行 | `h.Run() error` | 阻塞运行，但不处理信号与优雅退出 |
| 主动关闭 | `h.Shutdown(ctx context.Context) error` | 配合自定义信号处理 |
| 路由表 | `h.Routes() route.RoutesInfo` | 排查 404 时逐条核对 |

## 二、路由

| 用途 | API | 说明 |
| --- | --- | --- |
| 注册方法 | `h.GET/POST/PUT/DELETE/PATCH/HEAD/OPTIONS(path string, handlers ...app.HandlerFunc)` | 返回路由组，可链式继续注册 |
| 任意方法 | `h.Any(path string, handlers ...app.HandlerFunc)` | 所有方法都匹配 |
| 指定方法 | `h.Handle(method, path string, handlers ...app.HandlerFunc)` | 动态方法名 |
| 路由组 | `h.Group(relativePath string, handlers ...app.HandlerFunc) *route.RouterGroup` | 组内共享前缀与中间件 |
| 路径参数 | `"/users/:id"` + `c.Param("id")` | `:` 匹配单段 |
| 通配参数 | `"/files/*filepath"` + `c.Param("filepath")` | `*` 匹配剩余路径 |
| 静态目录 | `h.Static(relativePath, root string)` | 映射目录到 URL 前缀 |
| 单文件 | `h.StaticFile(relativePath, filepath string)` | 常用于 favicon、robots.txt |
| 目录配置 | `h.StaticFS(relativePath string, fs *app.FS)` | 可设 `Root/IndexNames/GenerateIndexPages` |
| 404 | `h.NoRoute(handlers ...app.HandlerFunc)` | SPA 回退也放这里 |
| 405 | `h.NoMethod(handlers ...app.HandlerFunc)` | 需先开 `WithHandleMethodNotAllowed` |

## 三、请求取值

| 用途 | API | 说明 |
| --- | --- | --- |
| Query | `c.Query(key string) string` | 不存在返回空串 |
| Query 默认值 | `c.DefaultQuery(key, defaultValue string) string` | 空值也会用默认值 |
| Query 判存在 | `c.GetQuery(key string) (string, bool)` | 区分“空”与“不存在” |
| 路径参数 | `c.Param(key string) string` | 对应 `:id` / `*path` |
| 表单 | `c.PostForm(key string) string` | `application/x-www-form-urlencoded` 或 multipart |
| 表单默认值 | `c.DefaultPostForm(key, defaultValue string) string` | 不存在时返回默认值 |
| 表单判存在 | `c.GetPostForm(key string) (string, bool)` | 注意与 Query 分离 |
| Query+Form | `c.FormValue(key string) []byte` | 先 Query 后 Form |
| Header | `c.GetHeader(key string) []byte` | 返回 `[]byte`，比较前转 string |
| Cookie | `c.Cookie(key string) []byte` | 返回 `[]byte`，不是 `(string, error)` |
| 原始 Body | `c.GetRawData() []byte` | 单返回值 |
| Body | `c.Body() ([]byte, error)` | 与 `GetRawData` 二选一 |
| 流式 Body | `c.RequestBodyStream() io.Reader` | 大 Body 场景 |
| 客户端 IP | `c.ClientIP() string` | 受代理头影响，需可信代理配置 |
| 请求方法/路径 | `c.Method() []byte` / `c.Path() []byte` | 返回 `[]byte` |
| 请求级存储 | `c.Set(key string, value any)` / `c.Get(key string) (any, bool)` | 类型断言后使用 |
| 字符串存储 | `c.GetString(key string) string` / `c.GetInt` / `c.GetBool` | 取不到返回零值 |

## 四、绑定

| 用途 | API | 说明 |
| --- | --- | --- |
| 自动绑定 | `c.Bind(obj any) error` | 按 Content-Type/tag 选择来源，不写响应 |
| 绑定并校验 | `c.BindAndValidate(obj any) error` | 需注入 validator |
| JSON | `c.BindJSON(obj any) error` | 只读 Body |
| Query | `c.BindQuery(obj any) error` | 只读 Query |
| Form | `c.BindForm(obj any) error` | 只读表单 |
| Path | `c.BindPath(obj any) error` | tag 用 `path` |
| Header | `c.BindHeader(obj any) error` | tag 用 `header` |
| Protobuf | `c.BindProtobuf(obj any) error` | Content-Type 为 protobuf |
| 按 Content-Type | `c.BindByContentType(obj any) error` | 不自动校验 |

## 五、响应

| 用途 | API | 说明 |
| --- | --- | --- |
| JSON | `c.JSON(code int, obj any)` | 自动设置 `application/json` |
| 字符串 | `c.String(code int, format string, values ...any)` | 格式化输出 |
| 字节 | `c.Data(code int, contentType string, data []byte)` | 自定义 Content-Type |
| 状态码 | `c.Status(code int)` / `c.SetStatusCode(code int)` | 只设置状态 |
| 响应头 | `c.Header(key, value string)` | 设置响应头 |
| 重定向 | `c.Redirect(code int, uri []byte)` | 第二个参数是 `[]byte`，且不终止 handler |
| 文件 | `c.File(filepath string)` | 发送文件 |
| 附件 | `c.FileAttachment(filepath, filename string)` | 设置下载文件名 |
| 直接写 Body | `c.SetBodyString(s string)` / `c.Write(p []byte)` | 更底层 |
| 流式响应 | `c.SetBodyStream(r io.Reader, size int)` | 大响应或 SSE |
| 渲染器 | `c.Render(code int, r render.Render)` | HTML/Protobuf 等 |

## 六、中间件

| 用途 | API | 说明 |
| --- | --- | --- |
| 注册中间件 | `h.Use(handlers ...app.HandlerFunc)` | 必须在路由注册前 |
| 继续链 | `c.Next(ctx context.Context)` | 后置逻辑写在它之后 |
| 中止 | `c.Abort()` | 停止后续 handler |
| 中止并设状态 | `c.AbortWithStatus(code int)` | 不写 Body |
| 中止并返回 JSON | `c.AbortWithStatusJSON(code int, obj any)` | 常用错误出口 |
| 是否已中止 | `c.IsAborted() bool` | 中间件判断 |
| 恢复 panic | `recovery.Recovery(...)` | `server.New` 必须自行注册 |
| 全链路超时 | `context.WithTimeout` + 下游超时 | Hertz 无内置超时中间件，需自写 |

## 七、错误处理

| 用途 | API | 说明 |
| --- | --- | --- |
| 记录错误 | `c.Error(err error) *errors.Error` | 返回错误对象，不自动响应 |
| 统一错误出口 | `c.Error(err)` + 错误中间件 | 推荐由中间件统一 `AbortWithStatusJSON` |
| 全局错误处理器 | `server.WithErrorHandler(...)` | v0.10.6 无此选项，新版本以官方文档为准 |

## 八、客户端 Client

| 用途 | API | 说明 |
| --- | --- | --- |
| 创建客户端 | `client.NewClient(opts ...config.ClientOption) (*client.Client, error)` | 全局复用，不要每请求新建 |
| GET | `c.Get(ctx, dst []byte, url string, opts ...config.RequestOption) (int, []byte, error)` | `dst` 可为 nil |
| POST | `c.Post(ctx, dst []byte, url string, args *protocol.Args, opts ...config.RequestOption) (int, []byte, error)` | 表单参数用 `protocol.Args` |
| 通用请求 | `c.Do(ctx, req *protocol.Request, resp *protocol.Response) error` | 完全自定义 |
| 超时请求 | `c.DoTimeout(ctx, req, resp, timeout time.Duration) error` | 单次超时 |
| 截止时间 | `c.DoDeadline(ctx, req, resp, deadline time.Time) error` | 绝对时间 |
| 连接数 | `client.WithMaxConnsPerHost(n int)` | 每 Host 最大连接 |
| 空闲连接 | `client.WithMaxIdleConnDuration(d time.Duration)` | 空闲回收 |
| 等待连接 | `client.WithMaxConnWaitTimeout(d time.Duration)` | 连接池满时排队上限 |
| 关闭空闲连接 | `c.CloseIdleConnections()` | 发布或下线时调用 |

## 九、GORM

| 用途 | API | 说明 |
| --- | --- | --- |
| 打开数据库 | `gorm.Open(mysql.Open(dsn), &gorm.Config{})` | 建议开 `TranslateError: true` |
| 自动迁移 | `db.AutoMigrate(&User{})` | 仅适合开发/小项目 |
| 创建 | `db.Create(&user)` | 回填主键 |
| 查一条 | `db.First(&user, id)` | 主键查询；`Take` 不排序 |
| 查多条 | `db.Find(&users)` | 条件用 `Where` |
| 条件 | `db.Where("status = ?", 1).Find(&users)` | 避免字符串拼接 |
| 更新 | `db.Model(&user).Updates(map[string]any{...})` | 结构体更新忽略零值 |
| 保存 | `db.Save(&user)` | 全字段更新，慎用 |
| 删除 | `db.Delete(&user)` | 有 `DeletedAt` 时为软删除 |
| 硬删除 | `db.Unscoped().Delete(&user)` | 绕过软删除 |
| 事务 | `db.Transaction(func(tx *gorm.DB) error { ... })` | 返回错误自动回滚 |
| 预加载 | `db.Preload("Orders").Find(&users)` | 解决 N+1 |
| 作用域 | `db.Scopes(Paginate(page, size)).Find(&users)` | 复用查询片段 |
| 计数 | `db.Model(&User{}).Count(&n)` | 生成 COUNT |
| 连接池 | `sqlDB.SetMaxOpenConns/SetMaxIdleConns/SetConnMaxLifetime` | 从 `db.DB()` 取 `*sql.DB` |
| 关闭 | `sqlDB.Close()` | 进程退出时 |

## 十、go-redis

| 用途 | API | 说明 |
| --- | --- | --- |
| 创建客户端 | `redis.NewClient(&redis.Options{Addr, Password, DB, PoolSize})` | 全局复用 |
| 连通性 | `rdb.Ping(ctx).Err()` | 启动时探活 |
| 写入 | `rdb.Set(ctx, key, value, expiration)` | `expiration=0` 表示不过期 |
| 读取 | `rdb.Get(ctx, key).Result()` | 不存在返回 `redis.Nil` |
| 过期 | `rdb.Expire(ctx, key, d)` | 返回是否设置成功 |
| 删除 | `rdb.Del(ctx, keys...)` | 大 key 用 `UNLINK` |
| 不存在才写 | `rdb.SetNX(ctx, key, value, expiration)` | 简易分布式锁 |
| 管道 | `pipe := rdb.Pipeline(); pipe.Set(...); pipe.Exec(ctx)` | 减少 RTT |
| 脚本对象 | `redis.NewScript(src)` + `script.Run(ctx, rdb, keys, args...)` | 自动 `EVALSHA` 回退 |
| 直接执行脚本 | `rdb.Eval(ctx, script, keys, args...)` | 调试方便 |
| 通用客户端 | `redis.NewUniversalClient(&redis.UniversalOptions{...})` | 单机/哨兵/集群统一 |
| 关闭 | `rdb.Close()` | 进程退出时 |

## 十一、Kitex

| 用途 | API | 说明 |
| --- | --- | --- |
| 创建客户端 | `client.NewClient(serviceName string, opts ...client.Option) (client.Client, error)` | 全局单例 |
| 创建服务端 | `server.NewServer(opts ...server.Option) server.Server` | 配合生成的 handler |
| 直连地址 | `client.WithHostPorts(addrs ...string) client.Option` | 调试或静态地址 |
| 服务发现 | `client.WithResolver(r discovery.Resolver) client.Option` | 如 Nacos Resolver |
| 服务端监听 | `server.WithServiceAddr(addr net.Addr) server.Option` | 如 `net.ResolveTCPAddr` |
| 注册中心 | `server.WithRegistry(r registry.Registry) server.Option` | 如 Nacos Registry |
| RPC 超时 | `client.WithRPCTimeout(d time.Duration) client.Option` | 非幂等接口慎配重试 |
| 连接超时 | `client.WithConnectTimeout(d time.Duration) client.Option` | 建连阶段 |
| 启动 | `svr.Run() error` | 阻塞运行 |
| 关闭 | `svr.Stop() error` | 优雅停机 |

## 使用建议

- 先查表确认方法名与返回值，再回正文看上下文；不要凭印象写 `c.Cookie` 的返回类型。
- 表格只列高频 API，完整选项看 [官方文档索引](01-官方文档索引.md) 的配置说明。
- 升级版本后优先核对 `GetRawData`、`Cookie`、`WithHostPorts` 这类签名易错项。

## 小结

- Hertz 的 API 与 Gin 相似，但 `[]byte` 返回值、单参数 `WithHostPorts`、无 `AbortWithError` 是高频坑。
- 数据层与 RPC 层只列了最常用方法，复杂查询与治理选项仍需查官方文档。
- 把本表当作索引，而不是完整参考；真正的行为以源码与官方文档为准。

## 延伸阅读

- [官方文档索引](01-官方文档索引.md)
- [版本演进与选型](03-版本演进与选型.md)
- [Gin 迁移到 Hertz](../14-迁移与FAQ/01-Gin迁移到Hertz.md)
- [常见问题 FAQ](../14-迁移与FAQ/03-常见问题FAQ.md)

## 相关章节

- [Engine 详解与启动参数](../02-Hertz入门/05-Engine详解与启动参数.md)
- [路由](../03-基本特性/01-路由.md)
- [请求上下文 Context](../03-基本特性/02-请求上下文Context.md)
- [参数绑定与校验](../03-基本特性/03-参数绑定与校验.md)
- [响应与渲染](../03-基本特性/04-响应与渲染.md)
- [GORM 核心用法与模型设计](../09-MySQL与ORM/02-GORM核心用法与模型设计.md)
- [go-redis 接入与连接池](../10-Redis/01-go-redis接入与连接池.md)
