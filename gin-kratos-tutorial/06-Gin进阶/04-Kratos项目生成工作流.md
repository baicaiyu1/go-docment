# Kratos 项目生成工作流

> 一句话定位：`kratos new` 生成标准骨架，`kratos proto add/client/server` 把契约变成可编译的桩，业务写进 `biz/data/service/server/conf`，wire 负责装配——Kratos 服务只以 gRPC 被 Gin BFF 调用，不自己对外暴露 HTTP。

## 本篇目标

- 用 `kratos new --nomod` 在 `mall-lite` 单仓单模块里生成服务，不产生嵌套 `go.mod`
- 分清 `cmd` 与 `internal/{biz,data,service,server,conf}` 的职责与禁止事项
- 跑通 `proto add → proto client → proto server → wire` 四步，知道哪些文件不能手改
- 写出 `kratos.New` + `grpc.NewServer` + etcd 注册的启动代码
- 给服务补齐 Makefile 目标，让构建、生成、运行命令一致

## 前置知识

- 已读 [Protobuf 与 gRPC 工作流](03-Protobuf与gRPC工作流.md)，会写 proto 并用 buf 生成
- 已读 [Gin 加 Kratos 的微服务架构](../12-微服务与Kratos/01-Gin加Kratos的微服务架构.md)，了解 BFF 与领域服务的分工

## 一、在 monorepo 里生成服务

```bash
# 在 mall-lite 根目录执行；--nomod = 不生成子 go.mod，复用根模块
go install github.com/go-kratos/kratos/cmd/kratos/v2@v2.8.4

kratos new app/user-service --nomod
go mod tidy && go build ./...
```

不加 `--nomod` 会在 `app/user-service` 下再生成一个 `go.mod`，根模块的 `go build ./...` 直接跳过该目录，import 全红。生成后先在**仓库根**构建一次，确认模块名是 `mall-lite` 而不是模板里的 `go-kratos/kratos-layout`。

## 二、生成的目录与手写边界

```text
app/user-service/
├── cmd/server/          # main.go（可改）、wire.go（可改）、wire_gen.go（生成）
├── internal/{biz,data,service,server,conf}
│   # biz 用例与 repo 接口；data 实现；service DTO；server 装配；conf 配置
└── configs/config.yaml  # 与 conf.proto 字段对齐
```

| 文件 | 来源 | 能否手改 |
| --- | --- | --- |
| `api/**/*.pb.go`、`*_grpc.pb.go` | `buf generate` / `kratos proto client` | 否，改 proto |
| `wire_gen.go` | `wire ./app/user-service/cmd/server` | 否，改 provider |
| `internal/conf/*.pb.go` | `kratos proto client internal/conf/conf.proto` | 否，改 conf.proto |
| `internal/service/*.go` | `kratos proto server` 首次生成 | 可改，再次生成会覆盖同名文件 |
| `main.go`、`wire.go`、`biz`、`data` | 手写 | 是 |

契约统一放仓库根 `api/`，服务内只 import `v1 "mall-lite/api/user/v1"`，不要每个服务复制一份 proto。

## 三、proto 三连

```bash
kratos proto add api/user/v1/user.proto        # 1) 新建契约模板
kratos proto client api/user/v1/user.proto     # 2) 生成 pb 与 gRPC 桩
kratos proto server api/user/v1/user.proto -t app/user-service/internal/service  # 3) 生成实现骨架
```

路径必须是**从仓库根出发**的相对路径，且与 `go_package` 的 import 路径一一对应；`option go_package = "mall-lite/api/user/v1;v1";` 决定 `import v1 "mall-lite/api/user/v1"`。

## 四、分层职责与依赖方向

```go
// app/user-service/internal/biz/user.go
package biz

type UserRepo interface { // biz 定义接口，data 实现，依赖方向永远指向 biz
	FindByID(ctx context.Context, id int64) (*User, error)
}

type UserUsecase struct{ repo UserRepo }

func NewUserUsecase(repo UserRepo) *UserUsecase { return &UserUsecase{repo: repo} }

func (uc *UserUsecase) Get(ctx context.Context, id int64) (*User, error) {
	return uc.repo.FindByID(ctx, id)
}
```

`data` 侧用 GORM 查 `UserPO`，`gorm.ErrRecordNotFound` 返回 `(nil, nil)` 交给 biz 决定语义；PO 与领域模型的转换只发生在 data，完整实现见 [项目分层与目录规范](../07-工程化与测试/01-项目分层与目录规范.md) 第三节。

禁止事项：service 不写 SQL、不开事务；biz 不 import `grpc`/`gorm`/`gin`；data 不写业务分支。自检方法很粗暴——把 `gorm` 从 biz 的 import 里删掉，删不掉说明逻辑放错了层。

## 五、server 装配与 wire

```go
// app/user-service/internal/server/grpc.go
func NewGRPCServer(c *conf.Server, svc *service.UserService, logger log.Logger) *grpc.Server {
	srv := grpc.NewServer(
		grpc.Address(c.Grpc.Addr), // 约定 0.0.0.0:9001
		grpc.Timeout(2*time.Second),
		grpc.Middleware(recovery.Recovery(), logging.Server(logger), tracing.Server()),
	)
	v1.RegisterUserServiceServer(srv, svc)
	return srv
}
```

```go
// app/user-service/cmd/server/wire.go（import 省略）
//go:build wireinject

package main

func wireApp(*conf.Server, *conf.Data, *conf.Registry, log.Logger) (*kratos.App, func(), error) {
	panic(wire.Build(server.ProviderSet, data.ProviderSet, biz.ProviderSet, service.ProviderSet, newRegistry, newApp))
}
```

生成命令是 `wire ./app/user-service/cmd/server`，报错即依赖图缺 provider 或存在多绑定；产物 `wire_gen.go` 不要手改。ProviderSet 如何按包组织见 [依赖注入 Wire 与配置分层 Viper](../07-工程化与测试/02-依赖注入Wire与配置分层Viper.md)。

## 六、启动与注册

```go
// app/user-service/cmd/server/main.go
app := kratos.New(
	kratos.Name("user-service"),
	kratos.Version("v1.0.0"),
	kratos.Logger(logger),
	kratos.Server(grpcSrv),
	kratos.Registrar(reg), // 来自配置，禁止硬编码
)
if err := app.Run(); err != nil {
	panic(err)
}
```

```go
// app/user-service/cmd/server/registry.go
func newRegistry(c *conf.Registry) (*etcdreg.Registry, error) {
	cli, err := clientv3.New(clientv3.Config{Endpoints: c.Etcd.Endpoints})
	if err != nil {
		return nil, err
	}
	// 注册中心构造函数只返回 *Registry，不返回 error
	return etcdreg.New(cli, etcdreg.RegisterTTL(15*time.Second)), nil
}
```

`app.Run()` 内部完成注册、启动 server、阻塞等信号；收到 SIGTERM 后反注册并优雅关闭，不用自己写 `signal.NotifyContext`。改了 `internal/conf/conf.proto` 必须重新生成并 `go build ./...`，因为 `Scan` 对未知字段不报错。

## 七、Makefile 目标

```text
# app/user-service/Makefile（命令前必须是 Tab）
.PHONY: api build run
api:
	kratos proto client api/user/v1/user.proto
build:
	go build -trimpath -o bin/user-service ./app/user-service/cmd/server
run:
	go run ./app/user-service/cmd/server -conf ./app/user-service/configs
```

## 完整示例

```bash
kratos new app/user-service --nomod
kratos proto client api/user/v1/user.proto
kratos proto server api/user/v1/user.proto -t app/user-service/internal/service
wire ./app/user-service/cmd/server
go run ./app/user-service/cmd/server -conf ./app/user-service/configs
```

## 常见坑

1. **`kratos new` 忘了 `--nomod`**：子目录多出 `go.mod`，根模块构建跳过该服务，import 全红。
2. **proto 路径写错**：`kratos proto client` 只接受仓库根相对路径，绝对路径或相对 `app/` 的路径都会失败。
3. **`go_package` 与模块名不一致**：模板残留 `example.com/...`，只能靠 `replace` 硬撑；生成后第一件事是全局搜一遍。
4. **手改 `wire_gen.go` / `*.pb.go`**：下次生成全部覆盖；要改就改 provider 和 proto。
5. **`kratos proto server` 覆盖已有实现**：同名文件会被重写，业务代码先进 Git 或改名后再生成。
6. **conf.proto 改了没重新生成**：拼错的 key 只得到零值，表现为「超时配置不生效」，且 `Scan` 不报错。
7. **模板依赖版本与仓库不一致**：`go mod tidy` 后 API 编译失败；统一 `go get github.com/go-kratos/kratos/v2@v2.8.4`，并核对模板默认端口与项目端口表（9001/9002/9003）。

## 小结

- `kratos new --nomod` 是单仓单模块的正确姿势，生成后先在仓库根构建
- 生成物与手写物界线清晰：`*.pb.go`、`wire_gen.go`、`conf.pb.go` 只读，业务只写 biz/data/service/server
- proto 四步：`proto add` 建契约、`proto client` 生成桩、`proto server` 生成实现、`wire` 装配
- 启动用 `kratos.New` + `kratos.Server(grpcSrv)` + `kratos.Registrar(reg)`，注册地址来自配置

## 练习

1. 用 `kratos new app/product-service --nomod` 生成第二个服务，接 `api/product/v1` 跑通 `GetProduct`。
2. 在 `conf.proto` 增加 `log.level` 字段，同步 yaml，验证 `Scan` 后能读到值。
3. 把 `NewUserRepo` 的返回类型改成 `*userRepo`，观察 wire 报错，再用 `wire.Bind` 修正。

## 官方文档

- Kratos 快速开始：<https://go-kratos.dev/docs/getting-started/start>
- Kratos 官方布局：<https://github.com/go-kratos/kratos-layout>
- Kratos 配置组件：<https://go-kratos.dev/docs/component/config>
- Kratos gRPC 传输层：<https://go-kratos.dev/docs/component/transport/grpc>
- Kratos CLI：<https://github.com/go-kratos/kratos/tree/main/cmd/kratos>
