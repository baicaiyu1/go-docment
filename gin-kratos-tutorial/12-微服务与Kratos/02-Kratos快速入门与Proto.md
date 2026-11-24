# Kratos 快速入门与 Proto

> 一句话定位：装好 kratos CLI 与 protoc 插件，写 user / product / order 三份契约，生成代码后跑通「服务端注册 + 客户端调用」的最小闭环。

## 本篇目标

- 安装并确认 kratos CLI、protoc、`protoc-gen-go`、`protoc-gen-go-grpc` 版本
- 写出 `api/{user,product,order}/v1/*.proto`，理解 `package`、`go_package` 与字段序号
- 用 `kratos proto client` 与 `protoc` 两种方式生成代码，知道哪些文件不能手改
- 实现 `GetUser`，用 grpcurl 与 Go 客户端调通，分清直连与注册中心两种模式

## 前置知识

- [01-Gin加Kratos的微服务架构](01-Gin加Kratos的微服务架构.md)

## 一、安装工具链

CLI 负责生成代码，运行时是 `go.mod` 里的 `github.com/go-kratos/kratos/v2`。教程基线 Kratos v2.8.x、Go 1.25，以官方最新稳定版为准。

```bash
go install github.com/go-kratos/kratos/cmd/kratos/v2@latest
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
go install github.com/go-kratos/kratos/cmd/protoc-gen-go-http/v2@latest
export PATH="$(go env GOPATH)/bin:$PATH" && kratos -v && protoc --version
```

```powershell
go install github.com/go-kratos/kratos/cmd/kratos/v2@latest
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
go install github.com/go-kratos/kratos/cmd/protoc-gen-go-http/v2@latest
$env:Path = "$(go env GOPATH)\bin;$env:Path"; kratos -v; protoc --version
```

Windows 还需 protoc 本体：`winget install protobuf`，或下载 release 后把 `bin` 加入 PATH。报 `--go-grpc_out` 失败基本都是插件没装或没进 PATH；`kratos: command not found` 则是 CLI 的 bin 目录没进 PATH。

## 二、三份契约

契约放 `api/`，它是唯一事实来源，生成代码永远重新生成。

```protobuf
// ---------- api/user/v1/user.proto ----------
syntax = "proto3";
package user.v1;
option go_package = "mall-lite/api/user/v1;v1";
service UserService {
    rpc GetUser(GetUserRequest) returns (User);
}
message GetUserRequest { int64 user_id = 1; }
message User { int64 user_id = 1; string nickname = 2; string avatar = 3; int32 status = 4; }

// ---------- api/product/v1/product.proto ----------
syntax = "proto3";
package product.v1;
option go_package = "mall-lite/api/product/v1;v1";
service ProductService {
    rpc GetProduct(GetProductRequest) returns (Product);
    rpc DeductStock(DeductStockRequest) returns (DeductStockReply);
}
message GetProductRequest { int64 sku_id = 1; }
message DeductStockRequest { string request_id = 1; int64 sku_id = 2; int32 quantity = 3; }
message DeductStockReply { int64 sku_id = 1; int32 stock = 2; }
message Product { int64 sku_id = 1; string title = 2; int64 price_cents = 3; int32 stock = 4; }

// ---------- api/order/v1/order.proto ----------
syntax = "proto3";
package order.v1;
option go_package = "mall-lite/api/order/v1;v1";
service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (Order);
    rpc GetOrder(GetOrderRequest) returns (Order);
}
message CreateOrderRequest { int64 user_id = 1; int64 sku_id = 2; int32 quantity = 3; string idempotency_key = 4; }
message GetOrderRequest { int64 order_id = 1; }
message Order { int64 order_id = 1; int64 user_id = 2; int64 amount_cents = 3; int32 status = 4; }
```

四条硬规则：**`package user.v1`** 决定全限定名 `user.v1.UserService/GetUser`，Sentinel 资源名、日志、追踪都依赖它；**`go_package` 必须带包名别名** `mall-lite/api/user/v1;v1`；**字段序号上线后不能改、不能复用**，删字段用 `reserved` 占位；proto3 零值是 `0`/`""`，要区分「没传」就加 `optional` 或标记字段。

## 三、两种代码生成方式

```bash
# 方式一：kratos CLI（推荐，同时生成 http 绑定与业务骨架）
kratos proto client api/user/v1/user.proto
kratos proto server api/user/v1/user.proto -t internal/service

# 方式二：直接用 protoc，插件必须已在 PATH 里
protoc -I . --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  api/user/v1/user.proto
```

生成物中 `user.pb.go`、`user_grpc.pb.go`、`user_http.pb.go` 禁止手改，要加方法先改 proto；`internal/service/user.go` 只生成一次，之后就是业务代码。

## 四、实现 UserService

```go
// app/user-service/internal/service/user.go
package service

import (
    "context"

    "github.com/go-kratos/kratos/v2/errors"
    v1 "mall-lite/api/user/v1"
)

type UserService struct {
    v1.UnimplementedUserServiceServer
}

func NewUserService() *UserService { return &UserService{} }

func (s *UserService) GetUser(ctx context.Context, req *v1.GetUserRequest) (*v1.User, error) {
    if req.GetUserId() <= 0 {
        return nil, errors.BadRequest("USER_ID_INVALID", "user id must be positive")
    }
    return &v1.User{UserId: req.GetUserId(), Nickname: "mall-user", Status: 1}, nil
}
```

用 `req.GetUserId()` 而不是 `req.UserId`，nil 请求也安全；错误一律用 Kratos 错误类型，网关才能稳定映射成 HTTP 状态码。

## 五、启动服务

```go
// app/user-service/cmd/main.go
package main

import (
    "time"

    "github.com/go-kratos/kratos/v2"
    "github.com/go-kratos/kratos/v2/transport/grpc"
    "google.golang.org/grpc/reflection"
    v1 "mall-lite/api/user/v1"
    "mall-lite/app/user-service/internal/service"
)

func main() {
    gs := grpc.NewServer(grpc.Address(":9001"), grpc.Timeout(2*time.Second))
    v1.RegisterUserServiceServer(gs, service.NewUserService())
    reflection.Register(gs.Server) // 仅本地开发：让 grpcurl 能反射出接口

    app := kratos.New(kratos.Name("user-service"), kratos.Version("v1.0.0"), kratos.Server(gs))
    if err := app.Run(); err != nil {
        panic(err)
    }
}
```

## 六、调用验证：grpcurl 与 Go 客户端

```bash
grpcurl -plaintext -d '{"user_id":42}' 127.0.0.1:9001 user.v1.UserService/GetUser
# PowerShell 会吞双引号，改用：-d '{\"user_id\":42}'
```

```go
// cmd/usercli/main.go：本地验证
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/go-kratos/kratos/v2/transport/grpc"
    v1 "mall-lite/api/user/v1"
)

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    // 模式一：直连单个实例，本地调试用
    conn, err := grpc.DialInsecure(ctx, grpc.WithEndpoint("127.0.0.1:9001"))
    // 模式二：注册中心，生产用；reg 由 etcdreg.New(clientv3.New(...)) 构造
    // conn, err = grpc.DialInsecure(ctx,
    //     grpc.WithEndpoint("discovery:///user-service"),
    //     grpc.WithDiscovery(reg))
    if err != nil {
        panic(err)
    }
    defer conn.Close()

    u, err := v1.NewUserServiceClient(conn).GetUser(ctx, &v1.GetUserRequest{UserId: 42})
    if err != nil {
        panic(err)
    }
    fmt.Printf("user_id=%d nickname=%s\n", u.GetUserId(), u.GetNickname())
}
```

两种模式只差两行：直连写 IP；注册中心写 `discovery:///user-service` 并注入 `registry.Discovery`（构造见 [03-Kratos服务端最佳实践](03-Kratos服务端最佳实践.md)）。进程内必须建连一次并复用，见 [04-Kratos客户端与Gin集成](04-Kratos客户端与Gin集成.md)。

## 完整示例

```bash
kratos proto client api/user/v1/user.proto && kratos proto server api/user/v1/user.proto -t internal/service
go mod tidy && go build ./... && go run ./app/user-service/cmd   # 终端 A
go run ./cmd/usercli                                            # 终端 B：user_id=42 nickname=mall-user
```

## 常见坑

1. **`kratos` 命令找不到**：装好了但 PATH 没生效，把 `$(go env GOPATH)/bin` 加进去后重开终端。
2. **`go_package` 只写路径不写别名**：生成代码包名与 import 冲突，必须写 `;v1`。
3. **字段序号复用**：删除后把序号让给新字段，新旧实例混部时反序列化错位。
4. **`package` 与服务名不一致**：契约写 `user.v1`、注册中心写 `user`，治理资源名对不上，限流规则失效。
5. **用 `context.Background()` 调客户端**：超时、取消、trace 全部断链。

## 小结

- Kratos = CLI + 运行时库，另需 protoc 与生成插件，版本与 `go.mod` 对齐
- 契约放 `api/`，`package` 用 `user.v1`，`go_package` 用 `mall-lite/api/user/v1;v1`
- `kratos proto client` 生成桩代码，`proto server` 生成可改骨架；proto 只做兼容演进

## 练习

1. 给 `user.proto` 增加 `UpdateNickname` 并重新生成，验证未实现时客户端收到 `Unimplemented`。
2. 按 product / order 契约生成代码，实现 `DeductStock` 的库存不足错误（返回 409）。
3. 把 `go_package` 改成不带别名的形式，观察编译错误并修正。

## 官方文档

- Kratos 快速开始：<https://go-kratos.dev/docs/getting-started/start>
- Protobuf 语言指南：<https://protobuf.dev/programming-guides/proto3/>
- gRPC Go 快速开始：<https://grpc.io/docs/languages/go/quickstart/>
- grpcurl 使用说明：<https://github.com/fullstorydev/grpcurl>
