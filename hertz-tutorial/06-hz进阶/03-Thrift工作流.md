# Thrift 工作流
> 一句话定位：从一份 Thrift IDL 走完“建项目 → 改 handler → 增量更新 → model/client → 多服务拆分”的闭环。
## 本篇目标
- 写出可用的 Thrift IDL（namespace、struct、service、注解）
- 用 `hz new -idl` 建项目并跑通接口
- 掌握 `hz update` 的增量行为与安全姿势
- 用 `hz model`、`hz client` 生成模型与调用端
- 设计多服务拆分与字段兼容策略
## 前置知识
- 已读 [06-hz进阶/01-hz命令全景](01-hz命令全景.md) 与 [02-IDL注解说明](02-IDL注解说明.md)
- 已读 [00-IDL基础入门](00-IDL基础入门.md)，了解 Thrift 的 struct / service 语法
## 一、写 IDL
```thrift
// idl/user.thrift
namespace go api.user

struct GetUserReq {
    1: string ID (api.path="id", api.vd="len($)>0")
}

struct GetUserResp {
    1: i64    ID        (api.body="id")
    2: string Name      (api.body="name")
    3: i32    Age       (api.body="age")
    4: string CreatedAt (api.body="created_at")
}

struct CreateUserReq {
    1: string Name (api.body="name", api.vd="len($)>0 && len($)<=32")
    2: i32    Age  (api.body="age", api.vd="$>=0 && $<=150")
}

struct CreateUserResp {
    1: i64 ID (api.body="id")
}

service UserService {
    GetUserResp    GetUser(1: GetUserReq req)       (api.get="/user/:id");
    CreateUserResp CreateUser(1: CreateUserReq req) (api.post="/user");
}
```

| 约定 | 原因 |
| --- | --- |
| `namespace go <domain>.<service>` | 决定生成目录，避免撞名 |
| `XxxReq` / `XxxResp` 命名 | 生态通用风格 |
| 字段编号从 1 递增且不复用 | 兼容性底线 |
| 所有对外字段显式写注解 | 避免默认 tag 与预期不符 |
| IDL 放 `idl/` | 与生成代码分离，便于 review |
## 二、创建项目
```bash
hz new -module example.com/user -idl idl/user.thrift --sort_router
go mod tidy && go build ./... && go run .
```

```powershell
hz new -module example.com/user -idl idl/user.thrift --sort_router
go mod tidy; go build ./...
go run .
```

若 `go.mod` 中 Thrift 版本与官方示例不符：

```bash
go mod edit -replace github.com/apache/thrift=github.com/apache/thrift@v0.13.0
go mod tidy
```

测试：

```bash
curl "http://127.0.0.1:8888/user/42"
```

```powershell
curl.exe "http://127.0.0.1:8888/user/42"
```
## 三、生成后的目录
```text
user/
├── .hz
├── go.mod
├── main.go
├── router_gen.go
├── idl/user.thrift
└── biz/
    ├── handler/api/user/user_service.go
    ├── model/api/user/user.go
    └── router/
        ├── register.go
        └── api/user/
            ├── user.go        # 生成，不改
            └── middleware.go  # 可改，update 追加
```

| 路径 | 归属 | 能否手改 |
| --- | --- | --- |
| `biz/handler/**` | 业务 | 可以，update 只追加新方法 |
| `biz/model/**` | 生成 | 不改 |
| `biz/router/**/{idl}.go` | 生成 | 不改 |
| `biz/router/**/middleware.go` | 业务中间件 | 可以 |
| `router_gen.go` | 生成 | 不改 |
## 四、修改 handler
```go
// GetUser .
// @router /user/:id [GET]
func GetUser(ctx context.Context, c *app.RequestContext) {
	var err error
	var req api.GetUserReq
	if err = c.BindAndValidate(&req); err != nil {
		c.String(400, err.Error())
		return
	}

	resp := new(api.GetUserResp)
	resp.ID = 42
	resp.Name = "hertz-user-" + req.ID
	resp.Age = 18
	resp.CreatedAt = "2026-01-01T00:00:00Z"
	c.JSON(200, resp)
}
```

`BindAndValidate` 会同时应用 `query/path/json` tag 与 `vd` 校验。业务逻辑不要写进 model 或 router 目录。
## 五、增量更新
新增方法与结构：

```thrift
struct ListUserReq {
    1: i32 Page     (api.query="page", api.vd="$>=1")
    2: i32 PageSize (api.query="page_size", api.vd="$>=1 && $<=100")
}

struct ListUserResp {
    1: list<GetUserResp> Items (api.body="items")
    2: i64 Total               (api.body="total")
}

service UserService {
    GetUserResp    GetUser(1: GetUserReq req)       (api.get="/user/:id");
    CreateUserResp CreateUser(1: CreateUserReq req) (api.post="/user");
    ListUserResp   ListUser(1: ListUserReq req)     (api.get="/users");
}
```

```bash
hz update -idl idl/user.thrift
```

预期变化：

- `biz/handler/api/user/user_service.go` 末尾追加 `ListUser`
- `biz/model/api/user/user.go` 新增 `ListUserReq` / `ListUserResp`
- `biz/router/api/user/user.go` 新增 `GET /users`

已实现的 `GetUser` / `CreateUser` 函数体不会被改动。新增 service 会新增 handler 文件与路由，但同目录下 service 名必须唯一。

安全习惯：

```bash
git status --short
hz update -idl idl/user.thrift
git diff --stat && go build ./...
# 结果不对时
git checkout -- biz/model biz/router router_gen.go
```
## 六、只生成 model 与 client
```bash
hz model -idl idl/user.thrift -model_dir biz/model
hz client -idl idl/user.thrift -module example.com/user -client_dir biz/client
```

在 IDL 中指定客户端默认域名：

```thrift
service UserService {
    GetUserResp GetUser(1: GetUserReq req) (api.get="/user/:id");
}(
    api.base_domain="http://127.0.0.1:8888";
)
```

生成的 client 本质是 Hertz Client 封装，可注入中间件、服务发现与超时：

```go
cli := user.NewUserServiceClient(...) // 构造函数以生成代码为准
resp, err := cli.GetUser(ctx, &api.GetUserReq{ID: "42"})
```
## 七、多服务拆分
| 场景 | 推荐 |
| --- | --- |
| 小团队、接口耦合紧 | 单进程多 IDL：`hz new -idl a.thrift -idl b.thrift` |
| 独立扩缩容、独立发布 | 多进程，每个服务独立 `.hz` + 注册中心 |
| 网关 + BFF | Hertz 网关聚合多个下游 |

单进程多 IDL 时，`biz/router/register.go` 会注册多套路由；多进程时通过注册中心互相发现。
## 八、字段兼容性
| 操作 | 是否安全 | 说明 |
| --- | --- | --- |
| 新增 `optional` 字段 | 安全 | 老客户端不传即未设置 |
| 新增 `required` 字段 | 危险 | 老客户端请求直接 400 |
| 删除字段 | 危险 | 应保留编号并标注 deprecated |
| 修改字段类型 | 危险 | 序列化不兼容 |
| 复用字段编号 | 禁止 | 新老数据含义冲突 |
| 修改字段名 | 谨慎 | JSON tag 变化影响前端 |

```thrift
struct GetUserResp {
    1: i64    ID   (api.body="id")
    2: string Name (api.body="name")
    // 3 已废弃，勿复用
    // 3: string OldField
    4: string Avatar (api.body="avatar") // 新增可选字段
}
```

对外 API 只增不改；下线时先停用（不再返回/读取），观察一个发布周期后再移除编号。
## 常见坑
1. 改了 `namespace go`：生成目录整体变化，旧 handler 变孤儿、路由重复。
2. 新增 `required` 字段：线上老客户端全部 400，必须用 optional。
3. 复用字段编号：新老数据结构冲突，排查极难。
4. 手改 `biz/model` 或生成 router：下次 update 直接覆盖。
5. `update` 漏了 `-I`：有 include 的 IDL 解析失败或生成不完整。
6. Thrift 版本不匹配：编译报 `undefined: thrift.xxx`，按官方文档 replace。
7. 同目录 service 重名：handler 文件互相覆盖。
8. IDL 改了没提交：别人 update 得到不同结果，生成物与 IDL 漂移。
## 小结
- 工作流是“IDL → new → 改 handler → update → model/client”
- update 只追加 handler，不改函数体；生成目录与 router 会重写
- 字段兼容的核心：只加 optional、不复用编号、不随意改名
- 多服务可单进程多 IDL，也可多进程 + 注册中心
- 生成物与业务代码分区，是长期可维护的前提
## 练习
1. 用上面的 IDL 建项目，补全 `ListUser` 分页逻辑并测试。
2. 新增 `AdminService` 执行 update，确认只新增文件未覆盖旧 handler。
3. 用 `hz client` 生成客户端，调用 `GetUser` 并加 3 秒超时。
4. 设计一次“字段下线”：写出 IDL 变更、发布顺序与回滚方案。
## 官方文档
- hz 使用（thrift）：<https://www.cloudwego.io/zh/docs/hertz/tutorials/toolkit/usage-thrift/>
- IDL 注解说明：<https://www.cloudwego.io/zh/docs/hertz/tutorials/toolkit/annotation/>
- hz client：<https://www.cloudwego.io/zh/docs/hertz/tutorials/toolkit/more-feature/client/>
- 注意事项：<https://www.cloudwego.io/zh/docs/hertz/tutorials/toolkit/cautions/>
