# Swagger 注解与 OpenAPI 文档

> 一句话定位：对外 HTTP 文档由 `swag` 从注释和 Go 类型生成 OpenAPI，`gin-swagger` 挂到 `/swagger` 供联调；对内服务契约由 proto 承担——两者分工明确，代码改了注解不改就是缺陷。

## 本篇目标

- 掌握 swag 的两级注解：入口文件的通用信息 + handler 上的接口注解
- 正确写出 `@Param`、`@Success`、`@Failure`、`@Router`、`@Security` 五件套
- 用「非泛型外壳 + data 覆盖」让统一响应在文档里如实展开
- 用 `swag init --parseInternal`、`swag fmt`、CI `git diff --exit-code` 保证文档不落后于代码
- 说清 OpenAPI（对外 HTTP）与 Proto（对内 gRPC）的边界

## 前置知识

- 已读 [工程工具链全景](01-工程工具链全景.md)，了解 swag 在工具链中的位置
- 已读 [参数绑定与校验](../03-基本特性/03-参数绑定与校验.md)，注解里的参数类型要与 `binding` tag 对齐

## 一、两份契约，各管一段

| 维度 | OpenAPI（swag） | Proto（buf） |
| --- | --- | --- |
| 消费方 | 前端、APISIX 网关、合作方 | Gin BFF、其它 Kratos 服务 |
| 协议 | HTTP/JSON | gRPC |
| 事实来源 | handler 注释 + Go 结构体 | `.proto` 文件 |
| 生成物 | `docs/docs.go`、`swagger.json` | `*.pb.go`、`*_grpc.pb.go` |
| 校验方式 | UI 人工点、CI diff | `buf breaking` 自动卡 |

结论：**不要试图用 OpenAPI 描述 gRPC**。内部方法签名一旦写进对外文档，前端就会开始直连内部服务，网关的鉴权和限流全部绕过。

## 二、入口文件：通用信息与 JWT 安全定义

```go
// app/gateway/cmd/main.go
package main

//	@title						Mall Lite Gateway API
//	@version					1.0
//	@description				Gin BFF 对外接口，统一响应 {"code":0,"message":"ok","data":{},"request_id":"..."}
//	@BasePath					/api/v1
//	@securityDefinitions.apikey	BearerAuth
//	@in							header
//	@name						Authorization
//	@description				填入 "Bearer {access_token}"，token 由 /auth/login 获取
func main() {
	// 启动逻辑见 Engine 详解与启动参数
}
```

三个必须对齐的点：

- `@BasePath` 与路由组前缀一致。代码里是 `r.Group("/api/v1")`，这里就写 `/api/v1`，否则 Try it out 全部 404。
- `@securityDefinitions.apikey` 后面的 `BearerAuth` 是自定义名字，handler 里 `@Security BearerAuth` 必须逐字相同，大小写错不会报错，只会不带 token。
- 通用注解必须紧贴 `package` 之后，中间插入其它注释块会导致整段失效。

## 三、handler 上的接口注解

```go
// app/gateway/internal/handler/user.go
package handler

// CreateUser 创建用户
//
//	@Summary		创建用户
//	@Description	校验通过后写入 MySQL，返回新用户 ID；邮箱重复返回 40900
//	@Tags			用户
//	@Accept			json
//	@Produce		json
//	@Param			req	body		CreateUserReq	true	"创建参数"
//	@Success		200	{object}	response.Body{data=response.UserResp}
//	@Failure		400	{object}	response.Body
//	@Failure		409	{object}	response.Body
//	@Security		BearerAuth
//	@Router			/users [post]
func (h *UserHandler) CreateUser(c *gin.Context) {
	var req CreateUserReq
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"code": 40001, "message": "invalid body"})
		return
	}
	c.JSON(http.StatusOK, gin.H{"code": 0, "message": "ok", "data": gin.H{"id": "10001"}})
}
```

`@Param` 格式是 `名字 位置 类型 是否必填 说明`，位置只接受 `path/query/header/body/form`。`@Router` 的路径**不含 BasePath**，只写 `/users [post]`；路径参数必须在 `@Router` 里出现（`/users/{id}`），否则 UI 无法填值。

## 四、请求与响应结构体：示例值写在 tag 里

```go
// pkg/response/body.go
package response

// Body 是所有接口的统一响应外壳，禁止在 handler 里另造结构。
type Body struct {
	Code      int    `json:"code" example:"0"`
	Message   string `json:"message" example:"ok"`
	Data      any    `json:"data" swaggertype:"object"`
	RequestID string `json:"request_id" example:"9f8c2a1b7d3e"`
}

// UserResp 只描述 data 里的业务字段。
type UserResp struct {
	ID        string `json:"id" example:"10001"` // int64 转 string，避免前端精度丢失
	Name      string `json:"name" example:"alice"`
	Email     string `json:"email" example:"alice@example.com"`
	CreatedAt int64  `json:"created_at" example:"1750000000"`
}
```

```go
// app/gateway/internal/handler/dto.go
package handler

// CreateUserReq 与 binding tag 一一对应，注解里的类型直接引用它。
type CreateUserReq struct {
	Name     string `json:"name" binding:"required,min=2,max=32" example:"alice"`
	Email    string `json:"email" binding:"required,email" example:"alice@example.com"`
	Password string `json:"password" binding:"required,min=8" example:"P@ssw0rd123"`
}
```

要点：swag 对 Go 泛型支持有限，所以外壳用普通 struct + `{data=response.UserResp}` 覆盖，前端能看到真实字段；`example` tag 直接变成文档示例值，比在 `@Description` 里写一坨 JSON 好维护。

## 五、生成、格式化与挂载

```bash
go install github.com/swaggo/swag/cmd/swag@v1.16.4

swag fmt                                    # 统一注解缩进，配合 make fmt
swag init -g app/gateway/cmd/main.go -o docs --parseDependency --parseInternal
```

| 参数 | 作用 | 不加的后果 |
| --- | --- | --- |
| `-g` | 指定含通用注解的入口文件 | 找不到 `@title`，spec 残缺 |
| `--parseDependency` | 解析依赖包里的类型 | 外部 struct 变成空对象 |
| `--parseInternal` | 解析 `internal/` 包 | 团队主流 handler 类型全部缺失 |

```go
// app/gateway/cmd/docs.go
package main

import (
	"github.com/gin-gonic/gin"
	swaggerFiles "github.com/swaggo/files"
	ginSwagger "github.com/swaggo/gin-swagger"

	_ "mall-lite/docs" // 必须导入，触发 docs.go 里的 init 注册 spec
)

func mountDocs(r *gin.Engine) {
	if gin.Mode() == gin.ReleaseMode {
		return // 生产不暴露；要给合作方就走 APISIX + 鉴权
	}
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
}
```

`_ "mall-lite/docs"` 这一行最容易在重构时被删，删了之后编译照样通过，但 `/swagger/doc.json` 404、页面白屏。

## 六、CI 校验：文档不能落后于代码

```yaml
# .github/workflows/docs-check.yml（片段）
on: [pull_request]
jobs:
  swag:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: {go-version: "1.25"}
      - run: go install github.com/swaggo/swag/cmd/swag@v1.16.4
      - run: swag init -g app/gateway/cmd/main.go -o docs --parseDependency --parseInternal
      - run: git diff --exit-code docs
```

`git diff --exit-code docs` 一次拦住两类问题：改了接口没重新生成、有人手改了生成文件。版本号与 `tools.go` 保持一致，否则本地生成的注释顺序可能不同。

## 完整示例

```bash
# 改完注解 -> 格式化 -> 生成 -> 编译 -> 浏览器验证
swag fmt && swag init -g app/gateway/cmd/main.go -o docs --parseDependency --parseInternal && go build ./...
curl -s http://127.0.0.1:8080/swagger/doc.json | head -c 200
```

```powershell
curl.exe -s http://127.0.0.1:8080/swagger/doc.json | Select-Object -First 1
```

## 常见坑

1. **`@BasePath` 与路由组不一致**：文档能打开、Try it out 全 404；两处前缀必须同步。
2. **忘了 `--parseInternal`**：`internal/handler` 的类型解析不到，`data` 变成 `{}`，前端照着写成错误 SDK。
3. **`@Success` 直接写业务结构**：丢了 `code/message/request_id` 外壳，前端无法统一拦截错误；用 `Body{data=UserResp}`。
4. **安全定义名大小写不一致**：定义 `BearerAuth`、引用 `bearerAuth`，UI 能点 Authorize 但请求不带 token，表现为莫名 401。
5. **`@Param` 位置写成 `json`**：合法值只有 `body/query/path/header/form`，写错该参数被静默忽略。
6. **把 `/swagger` 带到生产**：接口全貌公开，还被扫描器当攻击面；`gin.ReleaseMode` 下不挂载。
7. **只生成不提交**：他人 clone 后编译缺少 `docs` 包；生成物入库并加 CI diff 校验。

## 小结

- 通用注解放 main 的 package 注释块，接口注解放 handler 上方，两者都靠 `swag fmt` 维持格式
- `@BasePath` 对齐路由组，`@Router` 只写相对路径，路径参数写成 `{id}`
- 统一响应用「普通外壳 + `{data=具体类型}`」，泛型结构不要直接交给 swag
- OpenAPI 只描述对外 HTTP；对内契约用 proto，由 `buf breaking` 守门

## 练习

1. 给 `/users/{id}` 写完整注解，删掉 `@Param id path string true`，观察 UI 里无法填参的表现。
2. 把 `{object} response.Body{data=response.UserResp}` 改成 `{object} response.UserResp`，对比生成 JSON 的差异。
3. 本地改一个字段但不重新生成，跑一遍 CI 里的 `git diff --exit-code docs`，确认能被拦住。

## 官方文档

- swaggo/swag：<https://github.com/swaggo/swag>
- gin-swagger：<https://github.com/swaggo/gin-swagger>
- OpenAPI Specification 3.0：<https://spec.openapis.org/oas/v3.0.3>
- Gin 渲染与响应：<https://gin-gonic.com/docs/examples/rendering/>
