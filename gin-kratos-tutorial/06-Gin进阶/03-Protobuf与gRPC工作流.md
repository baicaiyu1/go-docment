# Protobuf 与 gRPC 工作流

> 一句话定位：proto 是 Gin BFF 与 Kratos 服务之间唯一的契约——目录按 `api/<service>/v1` 版本化，`go_package` 决定 Go import，buf 负责 lint、生成与破坏性检查，最后用健康检查 + reflection + grpcurl 完成闭环验证。

## 本篇目标

- 写出可生成的 proto3 契约：`package`、`go_package`、`service`、`message`、`optional`、`oneof`、`enum`
- 用 `buf lint`、`buf generate`、`buf breaking` 替代手写一长串 protoc 参数
- 注册 gRPC 健康检查与 reflection，用 grpcurl 不写客户端也能调通接口
- 牢记字段编号兼容规则，知道什么改动必须开 v2

## 前置知识

- 已读 [工程工具链全景](01-工程工具链全景.md)，装有 `buf`、`protoc-gen-go`、`protoc-gen-go-grpc`
- 已读 [Gin 加 Kratos 的微服务架构](../12-微服务与Kratos/01-Gin加Kratos的微服务架构.md)，理解「网关只做 HTTP，服务间走 gRPC」

## 一、契约放在哪

| 路径 | 内容 | 维护方 |
| --- | --- | --- |
| `api/user/v1/user.proto` | 用户服务契约 | user-service owner |
| `api/product/v1/product.proto` | 商品服务契约 | product-service owner |

三条硬约定：版本目录 `v1` 必须存在，删字段、改语义、改方法签名一律新开 `v2`；proto 包名与目录同构（`package user.v1;`），gRPC 方法全名就是 `user.v1.UserService/GetUser`，注册中心 key、指标标签、Sentinel 资源名都依赖它；`option go_package = "mall-lite/api/user/v1;v1";`——分号前是 import 路径，分号后是 Go 包名。

## 二、proto3 语法要点

```text
// api/user/v1/user.proto
syntax = "proto3";

package user.v1;

option go_package = "mall-lite/api/user/v1;v1";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserReply);
}

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0; // 枚举 0 值必须 UNSPECIFIED，且不带业务语义
  USER_STATUS_ACTIVE      = 1;
  USER_STATUS_DISABLED    = 2;
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserReply {
  int64      id           = 1;
  string     name         = 2;
  string     email        = 3;
  int64      created_at   = 4;    // Unix 秒
  optional   string nickname = 5; // 区分「未设置」与「空字符串」
  UserStatus status       = 6;
  oneof contact {                 // 同一时刻只有一个字段有值
    string phone  = 7;
    string wechat = 8;
  }
}
```

三个实践约束：时间统一用 `int64` Unix 秒，别用字符串；批量场景直接给 `BatchGetUsers`，不留「循环单查」的口子；`optional` 用于 PATCH 语义，普通字段不能靠零值判断「有没有传」。

## 三、buf：lint、生成、兼容性

```yaml
# buf.yaml
version: v2
modules:
  - path: api
lint:
  use: [STANDARD]
breaking:
  use: [FILE]
---
# buf.gen.yaml
version: v2
plugins:
  - local: protoc-gen-go
    out: .
    opt: paths=source_relative
  - local: protoc-gen-go-grpc
    out: .
    opt: paths=source_relative
```

```bash
buf lint
buf generate
buf breaking --against '.git#branch=main'   # 与主干比对，拦住破坏性改动
buf format -w
```

`out: .` 是唯一正确写法：`paths=source_relative` 保留 proto 在 module 内的相对路径，产物落在 `api/user/v1/`。写成 `out: api` 会得到 `api/api/user/v1/`，import 全红。

## 四、protoc 直连对照

```bash
protoc -I api --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative api/user/v1/user.proto
```

buf 相比裸 protoc：搜索路径由 `modules.path` 决定、插件版本写在 `buf.gen.yaml`、还自带 `buf breaking`。Windows 上 protoc 需单独下载并加 PATH；报找不到 `google/protobuf/descriptor.proto` 时，把压缩包里的 `include` 目录与 `bin` 平级放置。

## 五、服务端：健康检查 + reflection

```go
// cmd/probe-server/main.go（仅验证契约；正式服务用 Kratos，见第 04 篇）
package main

import (
	"context"
	"log"
	"net"

	"google.golang.org/grpc"
	"google.golang.org/grpc/health"
	healthpb "google.golang.org/grpc/health/grpc_health_v1"
	"google.golang.org/grpc/reflection"

	v1 "mall-lite/api/user/v1"
)

type userServer struct {
	v1.UnimplementedUserServiceServer // 必须内嵌：新增方法时老实现仍可编译
}

func (s *userServer) GetUser(ctx context.Context, req *v1.GetUserRequest) (*v1.GetUserReply, error) {
	return &v1.GetUserReply{Id: req.GetId(), Name: "alice"}, nil
}

func main() {
	lis, err := net.Listen("tcp", "0.0.0.0:9001")
	if err != nil {
		log.Fatalf("listen: %v", err)
	}
	srv := grpc.NewServer()
	v1.RegisterUserServiceServer(srv, &userServer{})

	hs := health.NewServer()
	hs.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)
	healthpb.RegisterHealthServer(srv, hs)
	reflection.Register(srv) // 让 grpcurl 能自描述，不依赖本地 proto

	if err := srv.Serve(lis); err != nil {
		log.Fatalf("serve: %v", err)
	}
}
```

健康检查是 K8s readiness 与注册中心探活的标准接口，别自造 `Ping`；reflection 只在开发/内网开启，公网暴露等于公开接口清单。

## 六、grpcurl 调试与客户端超时

```go
// 客户端侧：连接在建连处复用，超时只能自己给（完整 Kratos 客户端见第 04 篇）
ctx, cancel := context.WithTimeout(c.Request.Context(), 800*time.Millisecond)
defer cancel()
reply, err := cli.GetUser(ctx, &v1.GetUserRequest{Id: 1})
```

```bash
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest
grpcurl -plaintext 127.0.0.1:9001 list                       # 依赖 reflection 列服务
grpcurl -plaintext -d '{"id":1}' 127.0.0.1:9001 user.v1.UserService/GetUser
grpcurl -plaintext 127.0.0.1:9001 grpc.health.v1.Health/Check
```

```powershell
grpcurl -plaintext -d '{\"id\":1}' 127.0.0.1:9001 user.v1.UserService/GetUser
```

gRPC 没有默认超时：不写 `context.WithTimeout`，下游卡住就会永久挂起。建连同理，客户端必须启动时创建一次并全局复用，禁止每请求 `grpc.NewClient`。

## 七、字段编号兼容规则

| 变更 | 是否兼容 | 正确做法 |
| --- | --- | --- |
| 新增字段（用新编号） | 兼容 | 老客户端忽略未知字段 |
| 删除字段 | 兼容但危险 | `reserved 5;` + `reserved "old_name";` |
| 复用已删除编号 | 不兼容 | 严禁，新老语义会串 |
| 修改字段类型 | 多数不兼容 | `string → int64` 直接错，要改就新字段 |
| 新增 `optional` | 兼容 | 用于区分「未设置」与零值 |
| 枚举新增值 | 兼容 | 老代码走 default 分支，必须处理 |

删字段的标准写法是 `reserved 9, 10;` 加 `reserved "legacy_mobile";`，编号与名字都占住，后来人无法误用。`buf breaking --against '.git#branch=main'` 把这套规则自动化，CI 跑一次能拦住绝大多数破坏性改动。

## 完整示例

```bash
buf format -w && buf lint && buf generate && go build ./...

go run ./cmd/probe-server          # 终端 A
grpcurl -plaintext -d '{"id":1}' 127.0.0.1:9001 user.v1.UserService/GetUser
```

## 常见坑

1. **`out` 与 `paths=source_relative` 叠加**：生成到 `api/api/user/v1`；`out` 必须是仓库根 `.`。
2. **`go_package` 少了分号包名**：多个 `v1` 目录编译冲突；写成 `.../v1;v1`。
3. **proto 包名与目录不同构**：gRPC 方法全名、指标标签、注册中心 key 全歪，排查成本极高。
4. **复用已删除的字段编号**：老数据用新语义解析，「看着正常」但值错乱；删字段必须 `reserved`。
5. **改了 proto 没重新生成**：编译能过、运行时字段为 nil；`make proto` 必须包含 `buf generate`。
6. **服务端不内嵌 `Unimplemented*Server`**：对端新增方法后本服务启动即崩。
7. **不设超时**：下游卡住时 goroutine 永久挂起；每个 RPC 都要 deadline 或继承上游 deadline。
8. **CI 只跑 `go build`**：破坏性改动合进主干，上线才发现老客户端全挂；加 `buf breaking`。

## 小结

- proto 是 BFF 与微服务之间唯一契约：`api/<service>/v1` + `package <service>.v1` + `go_package ...;v1`
- buf 三连：`lint` 管规范、`generate` 管产物、`breaking` 管兼容，参数写进 yaml 而不是命令行
- 服务端内嵌 `Unimplemented*`、注册健康检查与 reflection；客户端建连一次、每 RPC 带超时；字段编号只增不复用，删字段留 `reserved`，破坏性变更开 v2

## 练习

1. 加 `optional string nickname = 5;`，重新生成后区分「未设置」与「空字符串」。
2. 删字段 3 但不加 `reserved`，跑 `buf breaking` 看报错，再补 `reserved` 复测。
3. 用 grpcurl 调 `grpc.health.v1.Health/Check`，把状态改成 `NOT_SERVING` 再观察返回。

## 官方文档

- Protocol Buffers 语言指南（proto3）：<https://protobuf.dev/programming-guides/proto3/>
- buf CLI 文档：<https://buf.build/docs/cli/>
- gRPC 健康检查协议：<https://grpc.io/docs/guides/health-checking/>
- gRPC Go 基础教程：<https://grpc.io/docs/languages/go/basics/>
