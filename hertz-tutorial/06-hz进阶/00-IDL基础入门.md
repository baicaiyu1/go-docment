# IDL 基础入门

> 一句话定位：用一份与语言无关的“接口契约”文件描述数据结构和服务方法，交给 hz 生成 Go 代码。先把 Thrift 和 Protobuf 怎么写看懂，后面的 hz 进阶才进得去。

## 本篇目标

- 说清 IDL 是什么，以及它和“手写 Go struct + 手写路由”相比解决了什么问题
- 掌握 Thrift 够用的核心语法：`namespace`、`struct`、`service`、字段编号、类型映射
- 掌握 Protobuf 够用的核心语法：`go_package`、`message`、`rpc`、字段编号、类型映射
- 知道 Thrift 和 Protobuf 该怎么选，为什么本教程两种都讲
- 能把一个熟悉的 FastAPI / TypeScript 接口翻译成 IDL
- 用一份 IDL 跑通 `hz new`，并且看得懂生成出来的文件

## 前置知识

- 已读 [02-Hertz入门/03-hz代码生成工具安装与使用](../02-Hertz入门/03-hz代码生成工具安装与使用.md) 与 [04-项目结构与代码生成布局](../02-Hertz入门/04-项目结构与代码生成布局.md)，能用 `hz new` 生成一个无 IDL 的项目并跑通
- 会写 Go 的结构体（见 [01-Go语言基础/04-结构体-方法-接口](../01-Go语言基础/04-结构体-方法-接口.md)）
- 有 TypeScript 或 FastAPI 经验会更快，本篇会给出对照，但**不强求**：完全没有也能看懂

> 为什么把这一篇放在 hz 进阶最前面：`hz new -idl` 的“原料”就是 IDL。前面 02 章生成的是**无 IDL** 的脚手架，所以一直没机会讲 IDL。不补这一课，直接看 [01-hz命令全景](01-hz命令全景.md) 和 [02-IDL注解说明](02-IDL注解说明.md) 会全是抽象符号。

## 一、IDL 到底是什么

**IDL = Interface Definition Language，接口定义语言。** 它是一份**独立的文本文件**，只描述两件事：

1. 有哪些**数据结构**（请求长什么样、响应长什么样）
2. 有哪些**服务方法**（方法名、用什么 HTTP 方法、走什么路径）

注意：它**不写业务逻辑**，也**不是任何一种编程语言的代码**。它只是一份“契约”。

### 三个视角的对照

| 你的背景 | 同样的东西通常怎么写 | 在 IDL 里怎么写 |
| --- | --- | --- |
| 前端 / TypeScript | `interface GetUserReq { id: string }` + 一张路由表 | `struct GetUserReq { 1: string ID }` + `service` 里声明路由 |
| Python / FastAPI | `class GetUserReq(BaseModel)` + `@app.get("/user/{id}")` 装饰器 | 同样两件事，写进一份独立契约文件 |
| Go | 手写 `type GetUserReq struct` + 手写 `h.GET(...)` | 由 hz 从契约**自动生成**这两样 |

所以最准确的一句话是：

> **IDL = 把“定义数据结构 + 声明路由”这两件事，从代码里抽出来，写进一份与语言无关的契约文件；hz 读契约，自动生成 Go 的结构体和路由注册。**

它和 FastAPI 的区别只在**时机**：FastAPI 是**运行时**用 Python 定义接口，IDL 是**编译期**用契约文件定义接口、再生成代码。两者想解决的是同一个问题。

### 为什么大厂要单独搞一份 IDL

手写 Go 结构体 + 手写路由，在只有一个服务、一个调用方时没问题。但一旦变成：

```
服务端 A  ── 我的请求结构体 ──  客户端 B
           ── 我的响应结构体 ──  客户端 C
```

每个调用方都手抄一遍结构体，字段名写错、类型对不上、改了一处忘了另一处，就是灾难。IDL 的价值在于：**一份定义，多端生成**。服务端代码、客户端 SDK、接口文档，都从同一份 IDL 长出来，天然一致。

这就是上一轮聊的“RPC 契约先行”在工程上的具体落点。

## 二、Thrift 语法（够用版）

Thrift 是 Facebook 开源的 IDL，也是 Hertz / hz 支持的两种 IDL 之一。**先只学写 HTTP 接口够用的那部分**，不用碰它全部的复杂特性。

### 2.1 最小骨架

```thrift
namespace go api.user

struct GetUserReq {
    1: string ID
}

struct GetUserResp {
    1: i64  ID   = 0
    2: string Name = ""
}

service UserService {
    GetUserResp GetUser(1: GetUserReq req) (api.get="/user/:id");
}
```

逐行拆：

| 行 | 作用 |
| --- | --- |
| `namespace go api.user` | 生成的 Go 代码放到 `api/user` 这个包路径下（决定目录） |
| `struct GetUserReq { ... }` | 定义一个数据结构，相当于 Go 的 `type GetUserReq struct` |
| `1: string ID` | 字段编号 `1` + 类型 `string` + 字段名 `ID` |
| `= 0` | 默认值（不写也有，零值） |
| `service UserService { ... }` | 声明一个服务，里面每个方法对应一个接口 |
| `(api.get="/user/:id")` | 这个方法的注解，表示用 GET 请求 `/user/:id`（注解细节在 [02-IDL注解说明](02-IDL注解说明.md) 展开） |

### 2.2 类型对照表（Thrift → Go）

| Thrift 类型 | Go 类型 | 说明 |
| --- | --- | --- |
| `bool` | `bool` | |
| `byte` / `i8` | `int8` | |
| `i16` | `int16` | |
| `i32` | `int32` | 最常用 |
| `i64` | `int64` | ID、时间戳常用 |
| `double` | `float64` | |
| `string` | `string` | |
| `binary` | `[]byte` | 文件、二进制 |
| `list<T>` | `[]T` | 列表 |
| `set<T>` | `[]T` | Go 没有 set，通常当切片用 |
| `map<K,V>` | `map[K]V` | 字典 |

看到没？**类型几乎和 Go 一一对应**，所以看懂 IDL 类型，等于看懂 Go 类型。

### 2.3 struct 和字段编号（最重要的一条规矩）

```thrift
struct CreateUserReq {
    1: string Name (api.body="name")
    2: i32    Age  (api.body="age")
}
```

字段前面的数字 `1: 2:` 是**字段编号**，不是顺序号，更不是数组下标。规则只有一条，但必须记牢：

> **编号一旦上线就永远不能复用、不能改。**

原因：IDL 的字段编号是跨服务通信时的“字段身份证”。你今天删了 `2: i32 Age`，明天又拿 `2:` 去存 `string Email`，老客户端发来的 `2` 字段会被误读成 email，直接把数据搞乱。

真实项目约定（后面几篇都会强调）：

- 从 1 开始递增
- 删字段时**只删代码、不删编号**，或把编号标记为保留
- 新增字段用**新的、更大的编号**

### 2.4 service：把方法变成接口

```thrift
service UserService {
    GetUserResp    GetUser(1: GetUserReq req)       (api.get="/user/:id");
    CreateUserResp CreateUser(1: CreateUserReq req) (api.post="/user");
}
```

- 每个方法一行，格式是 `返回类型 方法名(参数) (注解)`
- 一个方法 = 一个 HTTP 接口
- `(api.get=...)` / `(api.post=...)` 决定 HTTP 方法和路径

### 2.5 `namespace go` 决定生成目录

这是 Thrift 里最容易踩的坑：

```thrift
namespace go api.user
```

生成后，模型代码落在 `biz/model/api/user/` 下，handler 落在 `biz/handler/api/user/` 下。**`namespace go` 后面写什么，决定了包路径和目录**，所以命名要像 Go 包名一样：全小写、用点分层。

### 2.6 完整可跑的例子

```thrift
namespace go api.user

struct GetUserReq {
    1: string ID (api.path="id")
}

struct GetUserResp {
    1: i64    ID
    2: string Name
    3: i32    Age
}

struct CreateUserReq {
    1: string Name (api.body="name")
    2: i32    Age  (api.body="age")
}

struct CreateUserResp {
    1: i64 ID
}

service UserService {
    GetUserResp    GetUser(1: GetUserReq req)       (api.get="/user/:id");
    CreateUserResp CreateUser(1: CreateUserReq req) (api.post="/user");
}
```

字段上的 `api.path` / `api.body` 注解决定“参数从哪来”，这一篇先混个眼熟，下一篇 [02-IDL注解说明](02-IDL注解说明.md) 会完整讲。

## 三、Protobuf 语法（够用版）

Protobuf 是 Google 开源的 IDL，是 gRPC 的官方搭配，也是 hz 支持的另一种 IDL。语法比 Thrift 更规整一点。

### 3.1 最小骨架

```proto
syntax = "proto3";

package api.user;

option go_package = "example.com/user/api/user";

message GetUserReq {
  string id = 1;
}

message GetUserResp {
  int64  id   = 1;
  string name = 2;
}

service UserService {
  rpc GetUser(GetUserReq) returns (GetUserResp) {
    option (api.get) = "/user/:id";
  }
}
```

逐行拆：

| 行 | 作用 |
| --- | --- |
| `syntax = "proto3";` | 固定写死在第一行，表示用 proto3 语法 |
| `package api.user;` | Protobuf 自己的包名 |
| `option go_package = "...";` | **决定 Go 代码的 import 路径**（相当于 Thrift 的 namespace go） |
| `message Xxx {}` | 数据结构，相当于 Go 的 struct |
| `string id = 1;` | 类型 + 字段名 + **字段编号 = 1**（注意编号在等号后面） |
| `service` + `rpc` | 服务与方法声明 |
| `option (api.get) = ...` | 方法注解，和 Thrift 的 `(api.get=...)` 等价 |

**和 Thrift 最大的写法差异**：Protobuf 的字段编号写在**等号后面**（`string id = 1;`），Thrift 写在**最前面**（`1: string ID`）。很多人第一次就卡在这。

### 3.2 类型对照表（Protobuf → Go）

| Protobuf 类型 | Go 类型 | 说明 |
| --- | --- | --- |
| `bool` | `bool` | |
| `int32` | `int32` | |
| `int64` | `int64` | ID、时间戳常用 |
| `uint32` / `uint64` | `uint32` / `uint64` | |
| `float` | `float32` | |
| `double` | `float64` | |
| `string` | `string` | |
| `bytes` | `[]byte` | |
| `repeated T` | `[]T` | 列表 |
| `map<K,V>` | `map[K]V` | 字典 |
| 嵌套 `message` | 嵌套 `struct` | |

### 3.3 message、repeated、map

```proto
message Order {
  int64              id       = 1;
  repeated string    tags     = 2;              // 列表 → []string
  map<string, string> attrs   = 3;              // 字典 → map[string]string
  Address            address  = 4;              // 嵌套 message → 嵌套 struct
}

message Address {
  string city = 1;
  string road = 2;
}
```

`repeated` 就是“数组/列表”，`map<K,V>` 就是字典，嵌套 `message` 就是嵌套结构体。全部一一对应 Go。

### 3.4 service 和 rpc

```proto
service UserService {
  rpc GetUser(GetUserReq) returns (GetUserResp) {
    option (api.get) = "/user/:id";
  }
}
```

- `rpc 方法名(请求类型) returns (响应类型)`，这是 Protobuf 声明方法的固定句式
- 注解写成 `option (api.get) = "/path";` 的形式

### 3.5 `go_package` 决定生成路径

```proto
option go_package = "example.com/user/api/user";
```

这一行决定 hz 生成的 model 放在哪。和 Thrift 的 `namespace go` 一样重要，**写错了 import 全红**。

### 3.6 完整可跑的例子

```proto
syntax = "proto3";

package api.user;

option go_package = "example.com/user/api/user";

message GetUserReq {
  string id = 1;
}

message GetUserResp {
  int64  id   = 1;
  string name = 2;
  int32  age  = 3;
}

message CreateUserReq {
  string name = 1;
  int32  age  = 2;
}

message CreateUserResp {
  int64 id = 1;
}

service UserService {
  rpc GetUser(GetUserReq) returns (GetUserResp) {
    option (api.get) = "/user/:id";
  }
  rpc CreateUser(CreateUserReq) returns (CreateUserResp) {
    option (api.post) = "/user";
  }
}
```

## 四、Thrift 还是 Protobuf，选哪个

| 维度 | Thrift | Protobuf |
| --- | --- | --- |
| 生态 | 字节内部 + Hertz / Kitex 大量使用 | gRPC 官方搭配，云原生通用 |
| 语法 | 放开写，`1: 类型 名字` | 更规整，`类型 名字 = 1` |
| 依赖处理 | `hz update` 时只给主 IDL 即可，依赖自动生成 | `hz update` 时**所有依赖 proto 都要列出来或用 `-I` 解析到** |
| 上手难度 | 略低 | 略高（`go_package`、依赖路径容易配错） |

**给你的建议**：

- 按你所在团队/公司的技术栈选，**别两种混用**
- 本教程后面 [03-Thrift工作流](03-Thrift工作流.md) 和 [04-Protobuf工作流](04-Protobuf工作流.md) 都会讲，**先精读你实际会用的那一种**，另一种浏览即可
- 如果暂时没有偏好：**Hertz 场景下先学 Thrift**（字节生态、`hz` 支持更顺），之后学 Kitex 再补 Protobuf 也不迟

## 五、把熟悉的接口翻译成 IDL

拿你写过无数次的“按 ID 查用户”做例子。

**FastAPI 版：**

```python
from fastapi import FastAPI, Path, Query
from pydantic import BaseModel

app = FastAPI()

class GetUserResp(BaseModel):
    id: int
    name: str
    age: int

@app.get("/user/{id}")
def get_user(
    id: str = Path(...),
    source: str = Query("app"),
) -> GetUserResp:
    return GetUserResp(id=1, name="张三", age=20)
```

**翻译成 Thrift IDL：**

```thrift
namespace go api.user

struct GetUserReq {
    1: string ID     (api.path="id")
    2: string Source (api.query="source")
}

struct GetUserResp {
    1: i64    ID
    2: string Name
    3: i32    Age
}

service UserService {
    GetUserResp GetUser(1: GetUserReq req) (api.get="/user/:id");
}
```

一一对应关系：

| FastAPI | Thrift IDL |
| --- | --- |
| `class GetUserResp(BaseModel)` | `struct GetUserResp` |
| `id: int` | `1: i64 ID` |
| `@app.get("/user/{id}")` | `(api.get="/user/:id")` |
| `id: str = Path(...)` | `1: string ID (api.path="id")` |
| `source: str = Query("app")` | `2: string Source (api.query="source")` |

发现没有？**它就是把 Python 里用装饰器和 Pydantic 写的东西，翻译成了一份纯文本契约。** 你已经会这个概念，只是换了个写法。

## 六、跑通：IDL → hz new → 看产物

把上面的 Thrift 内容存成 `idl/user.thrift`，然后：

Windows PowerShell：

```powershell
hz new -module example.com/user -idl idl/user.thrift --sort_router
go mod tidy; go build ./...
go run .
```

Linux / macOS：

```bash
hz new -module example.com/user -idl idl/user.thrift --sort_router
go mod tidy && go build ./...
go run .
```

生成后重点看这几个文件：

```text
user/
├── .hz
├── idl/user.thrift                       # 你写的契约，是“原料”
├── biz/
│   ├── model/api/user/user.go            # IDL 里的 struct → Go struct（产物，别手改）
│   ├── handler/api/user/user_service.go  # 每个方法生成一个 handler 函数（业务写这里）
│   └── router/api/user/user.go           # 路由注册（产物，别手改）
└── router_gen.go                         # 路由总入口（产物，别手改）
```

打开 `biz/model/api/user/user.go`，你会看到 `GetUserReq` 被翻译成了带 tag 的 Go 结构体；打开 handler，你会看到每个方法一个空函数，里面是 `BindAndValidate` 的模板代码。**这就是“IDL 是原料、这些是产物”的实感**。有了这个实感，再回去看 [01-hz命令全景](01-hz命令全景.md) 的参数表，就不会飘在空中了。

## 七、这一篇之后看什么

- 字段上那些 `api.path` / `api.body` / `api.query` / `api.vd` 到底怎么用 → [02-IDL注解说明](02-IDL注解说明.md)
- `hz new / update / model / client` 全套命令与目录约束 → [01-hz命令全景](01-hz命令全景.md)
- Thrift 从建项目到多服务拆分的完整链路 → [03-Thrift工作流](03-Thrift工作流.md)
- Protobuf 同上 → [04-Protobuf工作流](04-Protobuf工作流.md)

## 常见坑

1. **字段编号随手改、随手复用**：跨版本通信直接串数据。编号只增不改。
2. **Thrift 的 `namespace go` / Protobuf 的 `go_package` 写错**：生成到错误目录，import 全红。
3. **Protobuf 字段编号位置写反**：写成 `1: string id`（这是 Thrift 写法），proto 直接报语法错。
4. **以为 IDL 里能写业务逻辑**：IDL 只描述结构和方法，逻辑写在生成的 handler 里。
5. **Protobuf `hz update` 只列主 proto**：依赖的 message 不更新，编译报字段不存在。
6. **一个字段既标 `api.query` 又标 `api.body`**：绑定行为混乱，一个字段只选一个来源。
7. **上手就两种 IDL 混学**：先把一种用熟，另一种是“换皮”，别同时开两条线。

## 小结

- IDL 是**语言无关的接口契约**，只描述数据结构和服务方法，不写业务逻辑
- Thrift 字段编号在**前面**（`1: string ID`），Protobuf 在**等号后**（`string id = 1;`）
- 字段编号**只增不改不复用**，这是跨服务兼容的底线
- Thrift 的 `namespace go` 和 Protobuf 的 `go_package` 决定生成目录，写错必翻车
- IDL 本质和你熟的 TS interface / FastAPI 模型是一回事，区别只在“编译期生成代码”
- **先跑一次 `hz new -idl`，看到产物，再读后面四篇，就不抽象了**

## 练习

1. 把本篇第五节的用户查询接口，分别用 Thrift 和 Protobuf 各写一遍，对比两种写法差异。
2. 用 Thrift 版跑一次 `hz new`，打开 `biz/model` 和 `biz/handler`，对照 IDL 说出每个字段、每个方法生成到了哪。
3. 故意把 Thrift 的 `namespace go` 改成 `api.order`，重新生成，观察目录变化后改回来。
4. 把你以前写过的一个 FastAPI 或 TS 接口，按第五节的对照表翻译成 Thrift IDL，说说哪一步最容易写错。

## 官方文档

- hz 快速开始（含 IDL）：<https://www.cloudwego.io/zh/docs/hertz/tutorials/quick-start/>
- Thrift IDL 官方语法：<https://thrift.apache.org/docs/idl>
- Protobuf 语言指南（proto3）：<https://protobuf.dev/programming-guides/proto3/>
- hz 注解说明：<https://www.cloudwego.io/zh/docs/hertz/tutorials/toolkit/annotation/>
