# Protobuf 工作流
> 一句话定位：从 proto3 语法与 `api.proto` 注解文件开始，跑通 Protobuf 版 hz 项目，并解决 protoc 环境问题。
## 本篇目标
- 掌握 hz 所需 proto3 语法与 `api.proto` 注解
- 正确设置 `go_package`，让代码生成到预期目录
- 用 `hz new -I ... -idl ...` 建项目并跑通
- 在 Windows/Linux 安装 protoc 并配置 include
- 区分 `google.api.http` 与 Hertz `api.*`
- 定位 `protoc not found`、`go_package` 等报错
## 前置知识
- 已读 [06-hz进阶/03-Thrift工作流](03-Thrift工作流.md)
- 已读 [00-IDL基础入门](00-IDL基础入门.md)，了解 proto3 的 message / service / option
- 能操作 PATH 环境变量
## 一、proto3 要点
```proto
syntax = "proto3";
package api.user;                                         // proto 包名
option go_package = "example.com/user/biz/model/api/user"; // Go 导入路径

message GetUserReq {
  string id = 1;
}

enum Status {
  STATUS_UNSPECIFIED = 0; // proto3 枚举必须有 0 值
  STATUS_ACTIVE = 1;
}

service UserService {
  rpc GetUser(GetUserReq) returns (GetUserResp);
}
```

| 维度 | Thrift | Protobuf |
| --- | --- | --- |
| 可选性 | required/optional | proto3 默认无 presence，除 `optional` |
| 枚举 0 值 | 无强制 | 必须有 |
| 包/路径 | `namespace go` | `package` + `go_package` |
| 注解文件 | 内置 | 需 `import "api.proto"` |
| update 依赖 | 自动 | 必须显式列出 |
## 二、`api.proto`
hz 的注解依赖一份 `api.proto`，放在 IDL 目录：

```proto
syntax = "proto2";
package api;
import "google/protobuf/descriptor.proto";
option go_package = "/api";

extend google.protobuf.FieldOptions {
  optional string raw_body = 50101;
  optional string query    = 50102;
  optional string header   = 50103;
  optional string cookie   = 50104;
  optional string body     = 50105;
  optional string path     = 50106;
  optional string vd       = 50107;
  optional string form     = 50108;
  optional string none     = 50111;
  optional string go_tag   = 51001;
}

extend google.protobuf.MethodOptions {
  optional string get     = 50201;
  optional string post    = 50202;
  optional string put     = 50203;
  optional string delete  = 50204;
  optional string patch   = 50205;
  optional string param   = 50307;
  optional string handler_path = 50309;
}

extend google.protobuf.ServiceOptions {
  optional string base_domain = 50402;
}
```

规则：业务 proto 必须 `import "api.proto";`；自定义扩展不要用 `5` 开头编号。完整定义以官方文档为准，升级 hz 后同步更新。
## 三、主 IDL
```proto
syntax = "proto3";
package api.user;
option go_package = "example.com/user/biz/model/api/user";
import "api.proto";

message GetUserReq {
  string id = 1 [(api.path) = "id", (api.vd) = "len($)>0"];
}

message GetUserResp {
  int64  id         = 1 [(api.body) = "id"];
  string name       = 2 [(api.body) = "name"];
}

message CreateUserReq {
  string name = 1 [(api.body) = "name", (api.vd) = "len($)>0 && len($)<=32"];
}

service UserService {
  rpc GetUser(GetUserReq) returns (GetUserResp) {
    option (api.get) = "/user/:id";
  }
  rpc CreateUser(CreateUserReq) returns (GetUserResp) {
    option (api.post) = "/user";
  }
}
```
## 四、`go_package` 与生成路径
假设 module 为 `example.com/user`，`model_dir` 默认 `biz/model`：

| `go_package` | model 路径 |
| --- | --- |
| `example.com/user/biz/model/api/user` | `biz/model/api/user` |
| `example.com/user/api/user` | `biz/model/api/user`（自动补） |
| `api/user` | `biz/model/api/user` |
| 不写 | 用 proto `package` 兜底，易冲突 |

推荐：`option go_package = "{MODULE}/biz/model/{PKG_PATH}";`

handler/router 取最后一级（上例为 `user`）：

```text
biz/handler/api/user/user_service.go
biz/router/api/user/user.proto.go
```
## 五、安装 protoc
hz 不内置 protoc，必须单独安装并放进 PATH。

Windows（官方 release，推荐）：下载 `protoc-*-win64.zip`，解压到 `C:\tools\protoc`，把 `C:\tools\protoc\bin` 加入 PATH。

```powershell
$env:Path += ";C:\tools\protoc\bin"
protoc --version
choco install protoc -y   # 或包管理器
```

Linux：

```bash
sudo apt update && sudo apt install -y protobuf-compiler
protoc --version
```

macOS：

```bash
brew install protobuf && protoc --version
```

使用 `google/protobuf/*.proto` 时，把压缩包 `include` 目录内容放到 protoc 同级，否则 import 解析失败。建议 protoc ≥ 3.20。
## 六、创建项目
```bash
hz new -module example.com/user -I idl -idl idl/user.proto --sort_router
go mod tidy && go build ./... && go run .
```

```powershell
hz new -module example.com/user -I idl -idl idl/user.proto --sort_router
go mod tidy; go build ./...
go run .
```

`-I idl` 等同于 protoc 的 `-I`，告诉解析器去哪里找 `api.proto`。主 IDL 与依赖不在同一目录时必须指定。

```bash
curl "http://127.0.0.1:8888/user/42"
```
## 七、增量更新
```bash
hz update -I idl -idl idl/user.proto
```

Protobuf 的依赖不会自动更新，必须列出所有被 import 的 proto：

```bash
hz update -I idl -idl idl/user.proto -idl idl/common/base.proto
```
## 八、生成结构对比
| 维度 | Thrift | Protobuf |
| --- | --- | --- |
| handler 目录 | `namespace go` | `go_package` 最后一级 |
| model 目录 | namespace 路径 | 去掉 module 后补 `biz/model` |
| router 文件 | `{idl}.go` | `{idl}.proto.go` |
| 依赖更新 | 自动 | 手动列出 |
| 注解文件 | 内置 | 需 `api.proto` |
## 九、`google.api.http` 与 Hertz 注解
已有 gRPC-Gateway 项目常见：

```proto
import "google/api/annotations.proto";

service UserService {
  rpc GetUser(GetUserReq) returns (GetUserResp) {
    option (google.api.http) = { get: "/v1/users/{id}" };
  }
}
```

| 维度 | `google.api.http` | Hertz `api.*` |
| --- | --- | --- |
| 维护方 | Google / grpc-gateway | CloudWeGo hz |
| 路径参数 | `{id}` | `:id` |
| 参数绑定 | gateway 规则 | `api.query/body/path` |
| 生成目标 | gRPC-Gateway 代理 | 原生 Hertz handler |
| 校验 | 无 `vd` | 支持 `api.vd` |

hz 不解析 `google.api.http`（以官方文档为准）。同一份 message 想同时服务 gRPC 与 HTTP，可分别维护注解，或 HTTP 用 hz、内部 RPC 用 Kitex。
## 十、常见报错
| 报错 | 原因 | 解决 |
| --- | --- | --- |
| `protoc not found` | PATH 未配置 | 安装并加入 PATH，重启终端 |
| `api.proto: File not found` | 缺 `-I` 或文件不存在 | 加 `-I idl` |
| `unknown option api.get` | 忘记 import | 补 `import "api.proto"` |
| `go_package is not defined` | 未声明 | 显式写 `go_package` |
| `duplicate route` | 两个 rpc 路径相同 | 检查 `api.get` |
| `cannot find package .../biz/model/...` | `go_package` 与实际目录不一致 | 统一 `{MODULE}/biz/model/...` |
| `descriptor.proto: not found` | include 未配置 | 复制 include 或加 `-I` |
## 常见坑
1. `go_package` 写相对路径：生成到 `biz/model/biz/model/...` 嵌套目录。
2. update 只传主 proto：依赖 message 没更新，编译报字段不存在。
3. Windows 没把 protoc 放进 PATH：IDE 内置终端与系统终端行为不一致。
4. 路径参数写 `{id}`：hz 要求 `:id`，路由匹配不到。
5. proto 包名与目录不一致：import 解析混乱，尽量同构。
6. 混用 `google.api.http` 与 `api.get`：只有一套生效，团队需统一。
7. 升级 hz 未更新 `api.proto`：新注解报 unknown option。
8. proto 文件带 UTF-8 BOM：部分解析器报语法错误。
## 小结
- Protobuf 的关键是 `api.proto`、`go_package`、`-I`、protoc 四件事
- update 必须显式列出依赖 proto
- 生成路径由 `go_package` 决定，推荐 `{MODULE}/biz/model/...`
- hz 不解析 `google.api.http`，两套注解不要混用
- 环境问题优先查 PATH 与 include
## 练习
1. 从零建 Protobuf 项目，验证缺 `-I idl` 时的报错。
2. 把 `go_package` 改成相对路径，观察生成目录并修复。
3. 新增 import 公共 message 的 rpc，执行 update 并补全依赖。
4. 对比同一接口在 Thrift 与 Protobuf 下的目录与 router 差异。
## 官方文档
- hz 使用（protobuf）：<https://www.cloudwego.io/zh/docs/hertz/tutorials/toolkit/usage-protobuf/>
- IDL 注解说明：<https://www.cloudwego.io/zh/docs/hertz/tutorials/toolkit/annotation/>
- 注意事项（go_package 规则）：<https://www.cloudwego.io/zh/docs/hertz/tutorials/toolkit/cautions/>
- protoc 下载：<https://github.com/protocolbuffers/protobuf/releases>
