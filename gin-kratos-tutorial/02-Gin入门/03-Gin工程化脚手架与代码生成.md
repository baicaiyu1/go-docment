# Gin 工程化脚手架与代码生成

> 一句话定位：Gin 没有官方代码生成器，工程一致性只能靠「分层模板 + 依赖注入 + lint 门禁 + Makefile」四件套保证。

## 本篇目标

- 搭出 air + swag + wire + golangci-lint + Makefile 的开发环境。
- 理解为什么 Gin 不需要「从 IDL 生成 handler」，一致性该由谁保证。
- 会写可被 wire 生成的 provider 与 injector，让 `main` 只做装配。
- 会用 swag 生成 OpenAPI，并把 lint 与生成动作固化进 Makefile。

## 前置知识

- 已跑通第一个服务，见[快速开始：第一个服务](./02-快速开始-第一个服务.md)；依赖注入概念可先读 [Wire 与配置分层 Viper](../07-工程化与测试/02-依赖注入Wire与配置分层Viper.md)。

## 一、工具链全景

| 工具 | 解决的问题 | 是否 Gin 官方 | 本教程用法 |
| --- | --- | --- | --- |
| air | 改代码自动重编译重启 | 否 | `.air.toml` + `make dev` |
| swag | 从 Go 注解生成 OpenAPI | 否 | `swag init` + `gin-swagger` |
| wire | 编译期依赖注入 | 否（Google） | provider 集合 + injector |
| golangci-lint | 聚合多 linter 的静态检查 | 否 | `.golangci.yml` + CI 门禁 |
| Makefile | 统一任务入口 | 否 | `make dev/test/lint` |

**Gin 生态没有「代码生成器」这一层**：BFF 的接口面向浏览器，字段常变，从 IDL 生成 handler 的收益远小于约束成本。一致性靠四件事：目录分层模板（第 04 篇）、wire 装配、lint 门禁、Makefile 统一入口；proto 那侧才需要生成，由 `buf`/`kratos proto` 负责。

## 二、air：开发期热重载

```powershell
go install github.com/air-verse/air@latest; air -v
```

```bash
go install github.com/air-verse/air@latest && air -v
```

```text
# .air.toml
root = "."
tmp_dir = "tmp"
[build]
  cmd = "go build -o ./tmp/main ./app/gateway/cmd"
  bin = "./tmp/main"
  include_ext = ["go", "yaml"]
  exclude_dir = ["tmp", "vendor", "docs", "deploy", "bin"]
  delay = 500
```

`tmp/`、`bin/` 必须进 `.gitignore`；air 只用于开发，镜像里不装——生产靠重新构建镜像。

## 三、swag：注解生成 OpenAPI

```go
// app/gateway/internal/router/router.go
package router

import (
    "github.com/gin-gonic/gin"
    swaggerFiles "github.com/swaggo/files"
    ginSwagger "github.com/swaggo/gin-swagger"

    _ "mall-lite/app/gateway/docs" // swag init 生成的包，必须 blank import
)

func New() *gin.Engine {
    r := gin.New()
    r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
    return r
}
```

```text
swag init -g app/gateway/cmd/main.go -o app/gateway/docs --parseDependency --parseInternal
```

不加 `--parseInternal` 就扫不到 `internal/` 里的 handler 注解，这是最高频的坑；注解写法见 [Swagger 注解与 OpenAPI 文档](../06-Gin进阶/02-Swagger注解与OpenAPI文档.md)。

## 四、wire：把装配从 main 里赶出去

```go
// app/gateway/internal/server/providers.go
package server

import "github.com/google/wire"

// ProviderSet 声明构造顺序，wire 在编译期展开成顺序调用
var ProviderSet = wire.NewSet(
    NewConfig, NewRedis, NewUserClient, NewUserHandler, NewRouter, NewHTTPServer,
)
```

```go
//go:build wireinject

// app/gateway/internal/server/wire.go
package server

import "github.com/google/wire"

// InitApp 的实现由 wire 生成到 wire_gen.go
func InitApp() (*App, func(), error) { panic(wire.Build(ProviderSet)) }
```

```bash
go install github.com/google/wire/cmd/wire@latest && wire ./app/gateway/internal/server
```

`wire_gen.go` 可以提交、可以单步调试，但**不要手改**——下次生成会覆盖；报 “no provider found” 时先确认类型是否加进了 `ProviderSet`。

## 五、golangci-lint：把规范变成门禁

```yaml
# .golangci.yml
run:
  timeout: 5m
linters:
  enable: [govet, staticcheck, errcheck, revive, bodyclose, gosec, unused, gofmt, goimports]
issues:
  exclude-rules:
    - path: _test\.go
      linters: [gosec]
```

```bash
golangci-lint run ./...       # CI 必须绿灯；本地自动修用 --fix
```

它替代口头约定：`errcheck` 抓漏判错误，`bodyclose` 抓响应体没关，`revive` 抓导出符号没注释。

## 六、Makefile：五个目标覆盖日常

```text
.PHONY: dev build test lint swag wire
dev:
	air -c .air.toml
build:
	CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o bin/gateway ./app/gateway/cmd
test:
	go test -race -cover ./...
lint:
	golangci-lint run ./...
swag:
	swag init -g app/gateway/cmd/main.go -o app/gateway/docs --parseDependency --parseInternal
wire:
	wire ./app/gateway/internal/server
```

recipe 行**必须**用 Tab 缩进，空格会报 `missing separator`；Windows 需先装 make（如 `choco install make`），没有就直接执行对应命令。

## 七、目录初始化

```powershell
New-Item -ItemType Directory -Force api/user/v1, app/gateway/cmd, app/gateway/internal/router,
  app/gateway/internal/{handler,middleware,client,server}, pkg/config, configs, deploy/apisix
```

```bash
mkdir -p api/{user,product,order}/v1 app/gateway/{cmd,internal/{router,handler,middleware,client,server}} \
  pkg/{config,logger,errs} configs deploy/{apisix,docker,k8s}
```

## 完整示例

入口只做三件事：初始化、调用 wire 生成的装配函数、优雅退出（空目录靠 `doc.go` 占位）：

```go
// app/gateway/cmd/main.go
package main

import (
    "log/slog"

    "mall-lite/app/gateway/internal/server"
)

// @title        Mall Gateway API
// @version      1.0
// @description  mall-lite BFF 接口
// @host         localhost:8080
// @BasePath     /api/v1
func main() {
    app, cleanup, err := server.InitApp()
    if err != nil {
        slog.Error("init app failed", "err", err)
        return
    }
    defer cleanup() // 关闭 DB、Redis、gRPC 连接
    if err := app.Run(); err != nil {
        slog.Error("gateway exited", "err", err)
    }
}
```

```powershell
make swag; make wire; make dev
curl.exe -i http://127.0.0.1:8080/swagger/index.html
```

## 常见坑

1. **air 的 `bin` 指向仓库根目录**：产物混进业务代码，固定放 `tmp/` 并加入 `.gitignore`。
2. **swag 扫不到 `internal/` 注解**：漏了 `--parseInternal`；`@BasePath` 与路由前缀不一致也会让文档 URL 全错。
3. **忘记 blank import `docs` 包**：编译通过但 `/swagger/index.html` 404。
4. **手改 `wire_gen.go`**：下次生成被覆盖，要改行为就改 provider 或 injector 签名。
5. **golangci-lint 版本与 Go 版本不匹配**：升级 lint，而不是关规则。
6. **Makefile 用空格缩进**：`missing separator`，换成 Tab。

## 小结

- Gin 没有官方生成器，工程一致性由「分层 + wire + lint + Makefile」共同保证。
- air 管开发体验，swag 管文档，wire 管装配，lint 管底线；`--parseInternal`、`ProviderSet`、Makefile 的 Tab 是最常踩的细节。

## 练习

1. 用 `make swag` 生成 docs，打开 `/swagger/index.html`，说明 `@Param` 注解对应哪一行代码。
2. 给 `ProviderSet` 新增 `NewLogger() *slog.Logger` 并让 handler 依赖它，重新 `wire` 观察变化。
3. 在 `.golangci.yml` 打开 `gocyclo`，看现有代码是否超标。

## 官方文档

- air：<https://github.com/air-verse/air>
- swag：<https://github.com/swaggo/swag>
- gin-swagger：<https://github.com/swaggo/gin-swagger>
- wire 指南：<https://github.com/google/wire/blob/main/docs/guide.md>
- golangci-lint：<https://golangci-lint.run/>
