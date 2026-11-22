# Kitex 快速入门与 IDL

> 一句话定位：从装工具、写 Thrift、生成代码到跑通第一个 RPC，把 Kitex 的最小闭环和“哪些生成代码能改”一次讲清。

## 本篇目标

- 安装 `kitex` 命令行工具并确认版本
- 掌握 Thrift IDL 的 namespace / struct / exception / service / 字段序号
- 用一条命令生成 user-service 的桩代码并读懂目录
- 实现 handler、启动服务端、写客户端调用
- 理解 Protobuf 模式与 Thrift 模式的差异，明确生成代码的修改边界

## 前置知识

- 已读 [01-Hertz加Kitex的微服务架构](01-Hertz加Kitex的微服务架构.md)
- 已读 [00-开始之前/03-Go-Module与国内代理](../00-开始之前/03-Go-Module与国内代理.md)
- 会写 Go 结构体、接口和 `main` 函数

## 一、安装与版本确认

Kitex 的代码生成器（CLI）与运行时库是两个东西，版本要匹配，生成后立刻 `go mod tidy`。

```bash
go install github.com/cloudwego/kitex/tool/cmd/kitex@latest
kitex -version
export PATH="$(go env GOPATH)/bin:$PATH"
```

```powershell
go install github.com/cloudwego/kitex/tool/cmd/kitex@latest
kitex -version
$env:Path = "$(go env GOPATH)\bin;$env:Path"
```

Windows 下建议把 `%GOPATH%\bin` 写进系统环境变量，而不是每次开终端都设置。`kitex -h` 可查看全部参数，本文只用 `-module`、`-service`、`-I`。

## 二、Thrift IDL 语法要点

```thrift
namespace go user                 // 生成代码的 Go 包名

include "common.thrift"           // 跨文件引用，路径相对 -I

enum Gender { UNKNOWN = 0, MALE = 1, FEMALE = 2 }

struct GetUserReq {
    1: required i64 user_id        // 必填，序号不可变
    2: optional string scene       // 可选，便于兼容
}

struct User {
    1: i64    user_id
    2: string nickname
    3: Gender gender = Gender.UNKNOWN
}

exception BizException {
    1: required i32 code
    2: required string message
}

service UserService {
    User GetUser(1: GetUserReq req) throws (1: BizException e)
    list<User> BatchGetUsers(1: list<i64> user_ids)
}
```

四条硬规则：

1. **字段序号一旦上线就不能改、不能复用**，删除字段只标记弃用，否则新旧版本反序列化会错位。
2. 新增字段优先用 `optional` 并给默认值，老客户端不传也能兼容。
3. `required` 只用在“缺了就不能工作”的字段上，滥用会让上游升级变成全链路强耦合。
4. `namespace go` 决定生成包名；多服务共享的 struct 放 `common.thrift`，用 `include` 引用。

## 三、生成代码

在模块根目录执行（`idl/user.thrift` 为相对路径）：

```bash
kitex -module mall-lite -service user idl/user.thrift
```

```powershell
kitex -module mall-lite -service user idl/user.thrift
```

```text
mall-lite/
├── go.mod                  # module mall-lite
├── idl/user.thrift
├── kitex_gen/              # 生成，禁止手改
│   └── user/
│       ├── user.go         # struct / enum
│       ├── k-user.go       # 编解码
│       └── userservice/    # 服务端/客户端桩
├── main.go                 # 入口，可改
├── handler.go              # 业务实现，可改
├── build.sh                # 构建脚本，可改
├── script/bootstrap.sh
└── kitex_info.yaml         # 生成元信息
```

`-module mall-lite` 让 import 路径变成 `mall-lite/kitex_gen/user`；`-service user` 决定服务名与默认端口 8888。多服务项目按“生成一个、把 main.go/handler.go/build.sh 移到 `user-service/`、再生成下一个”的顺序做，否则后一次会覆盖前一次。

## 四、实现 handler 与启动

```go
// handler.go
type UserServiceImpl struct{}

func (s *UserServiceImpl) GetUser(ctx context.Context, req *user.GetUserReq) (*user.User, error) {
    if req.UserId <= 0 {
        return nil, kerrors.NewBizStatusError(40001, "invalid user_id")
    }
    return &user.User{UserId: req.UserId, Nickname: "mall-user"}, nil
}
```

```go
// main.go
func main() {
    addr := utils.NewNetAddr("tcp", ":8888")
    svr := userservice.NewServer(
        new(UserServiceImpl),
        server.WithServiceAddr(addr),
        server.WithServerBasicInfo(&rpcinfo.EndpointBasicInfo{ServiceName: "user-service"}),
    )
    if err := svr.Run(); err != nil {
        klog.Fatal(err)
    }
}
```

`svr.Run()` 会阻塞并处理信号；生产环境还要加限流、注册中心、优雅退出，见 [03-Kitex服务端最佳实践](03-Kitex服务端最佳实践.md)。

## 五、客户端调用

```go
cli, err := userservice.NewClient("user-service",
    client.WithHostPorts("127.0.0.1:8888"), // 仅本地直连
)
if err != nil {
    panic(err)
}
u, err := cli.GetUser(context.Background(), &user.GetUserReq{UserId: 42})
```

生成的服务端包同时提供客户端构造函数。生产环境把 `WithHostPorts` 换成 `WithResolver`，客户端只建一次并复用，详见 [04-Kitex客户端与Hertz集成](04-Kitex客户端与Hertz集成.md)。

## 六、依赖与国内代理

```bash
export GOPROXY=https://goproxy.cn,direct
go mod tidy
go build ./...
```

```powershell
$env:GOPROXY = "https://goproxy.cn,direct"
go mod tidy
go build ./...
```

私有仓库用 `GOPRIVATE=git.example.com/*` 跳过代理。`go mod tidy` 报版本冲突时，先对比 `kitex -version` 与 `go.mod` 中的 Kitex 版本，再决定升级运行时还是重新生成代码。

## 七、Protobuf 模式

```proto
syntax = "proto3";
package user;
option go_package = "mall-lite/kitex_gen/user";

message GetUserReq { int64 user_id = 1; }
message User { int64 user_id = 1; string nickname = 2; }

service UserService {
  rpc GetUser(GetUserReq) returns (User);
}
```

```bash
kitex -module mall-lite -service user idl/user.proto
```

```powershell
kitex -module mall-lite -service user idl/user.proto
```

| 维度 | Thrift | Protobuf |
| --- | --- | --- |
| 默认协议 | TTHeader + Thrift 二进制 | gRPC 或 TTHeader + Protobuf |
| 字段默认值 | 需显式处理零值 | proto3 无 `required`，零值即默认 |
| 生态 | 字节系内部广泛使用 | 与 gRPC / 云原生更近 |

选型：已有 Thrift 资产或团队熟悉字节生态就用 Thrift；需要与 gRPC 网关、外部合作方互通就选 Protobuf。**不要在一个服务里混用两种 IDL**。

## 八、生成代码哪些能改

| 文件 / 目录 | 能否修改 | 说明 |
| --- | --- | --- |
| `kitex_gen/` | 禁止 | 重新生成会覆盖；改契约就改 IDL |
| `handler.go` | 可以 | 业务实现写在这里 |
| `main.go` | 可以 | 加选项、注册中心、初始化依赖 |
| `build.sh` | 可以 | 调整构建参数与输出路径 |
| `idl/` | 可以 | 契约的唯一来源 |

升级流程：改 IDL → 重新生成 → `git diff kitex_gen` 确认只有预期变更 → 实现新方法 → `go mod tidy`。生成代码进 Git 但不 review 内容，只 review IDL。

## 常见坑

1. **kitex CLI 与运行时版本差太多**：生成代码调用了运行时不存在的接口，编译报 `undefined`。
2. **手改 `kitex_gen`**：下次生成全部丢失，还会与 IDL 不一致。
3. **字段序号复用或改名**：新旧实例混部时反序列化错位。
4. **Protobuf 忘写 `go_package`**：生成失败或 import 路径变成相对路径。
5. **Windows 没配 PATH**：安装成功但命令找不到，误以为安装失败。
6. **多个服务生成到同一目录**：后生成的覆盖 `main.go`，启动的永远是最后一个服务。

## 小结

- Kitex = CLI 生成器 + 运行时库，两者版本要匹配
- Thrift 的核心是 `namespace`、`struct`、`exception`、`service` 和不可变的字段序号
- `kitex -module ... -service ...` 生成 `kitex_gen` 与可编辑的 `main.go` / `handler.go`
- 客户端用生成的 `NewClient`，生产环境接注册中心并复用单例
- Protobuf 与 Thrift 二选一；生成代码只读，契约只改 IDL

## 练习

1. 写一个 `product.thrift`，包含 `GetProduct`、`DeductStock`、`BizException`，生成并跑通。
2. 给 `GetUserReq` 增加 `optional string scene`，验证老客户端调用不报错。
3. 用 Protobuf 重写 user 服务，对比生成的目录与调用方式差异。
4. 故意把字段序号从 1 改成 3，观察新旧代码混调的结果。

## 官方文档

- Kitex 快速开始：<https://www.cloudwego.io/zh/docs/kitex/getting-started/>
- Kitex 代码生成：<https://www.cloudwego.io/zh/docs/kitex/tutorials/code-gen/>
- Thrift IDL：<https://thrift.apache.org/docs/idl>
- Kitex 仓库：<https://github.com/cloudwego/kitex>
