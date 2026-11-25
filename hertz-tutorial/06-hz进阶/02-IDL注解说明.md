# IDL 注解说明
> 一句话定位：把 Thrift 与 Protobuf 的 hz 注解、生成的 tag、参数绑定与校验规则一一对应起来。
## 本篇目标
- 掌握 Field 注解：`query/header/cookie/body/form/path/raw_body/none/go_tag/vd`
- 掌握 Method 注解：`get/post/put/delete/patch/options/head/any`
- 理解 `base_domain`、`handler_path`、`param` 等进阶注解的实际作用
- 看懂注解如何映射为 Go struct tag 与路由注册
- 用 `api.vd` 写出可用的参数校验
- 避开注解名、路径参数、tag 转义等常见错误
## 前置知识
- 已读 [06-hz进阶/01-hz命令全景](01-hz命令全景.md)
- 已读 [03-基本特性/03-参数绑定与校验](../03-基本特性/03-参数绑定与校验.md)
- 熟悉 Thrift 或 Protobuf 的基本语法
## 一、注解的两类作用
| 类别 | 作用位置 | 影响 |
| --- | --- | --- |
| Field 注解 | struct 字段 / message 字段 | 生成 Go struct tag，决定参数从哪取、怎么校验 |
| Method 注解 | service 方法 / rpc | 决定 HTTP 方法与路由，并生成路由注册代码 |

一句话：**Field 注解决定“参数从哪来”，Method 注解决定“接口怎么暴露”。**
## 二、Field 注解对照表
| 注解（Thrift / Protobuf） | 生成的 tag | 参数来源 |
| --- | --- | --- |
| `api.query` | `query:"xxx"` | URL Query |
| `api.header` | `header:"xxx"` | 请求头 |
| `api.cookie` | `cookie:"xxx"` | Cookie |
| `api.body` | `json:"xxx"` | JSON Body |
| `api.form` | `form:"xxx"` | 表单 / multipart |
| `api.path` | `path:"xxx"` | 路径参数 |
| `api.raw_body` | `raw_body:"xxx"` | 原始 Body |
| `api.none` | `-` | 不参与绑定 |
| `go.tag`（Thrift） / `api.go_tag`（Protobuf） | 原样透传 | 自定义 tag |
| `api.vd` | `vd:"..."` | 校验表达式 |

写法示例。

Thrift：

```thrift
namespace go api.user

struct GetUserReq {
    1: string ID     (api.path="id", api.vd="len($)>0")
    2: string Source (api.query="source", api.vd="$=='app'||$=='web'")
    3: string Token  (api.header="X-Token")
}

struct UpdateUserReq {
    1: string ID   (api.path="id")
    2: string Name (api.body="name", api.vd="len($)>0 && len($)<=32")
    3: i32    Age  (api.body="age", api.vd="$>=0 && $<=150")
}

struct UploadReq {
    1: binary File (api.form="file")
    2: string Note (api.form="note")
}
```

Protobuf：

```proto
message GetUserReq {
  string id     = 1 [(api.path) = "id", (api.vd) = "len($)>0"];
  string source = 2 [(api.query) = "source"];
  string token  = 3 [(api.header) = "X-Token"];
}

message UpdateUserReq {
  string id   = 1 [(api.path) = "id"];
  string name = 2 [(api.body) = "name", (api.vd) = "len($)>0 && len($)<=32"];
  int32  age  = 3 [(api.body) = "age", (api.vd) = "$>=0 && $<=150"];
}
```
## 三、Method 注解对照表
| 注解 | HTTP 方法 | 示例 |
| --- | --- | --- |
| `api.get` | GET | `(api.get="/user/:id")` |
| `api.post` | POST | `(api.post="/user")` |
| `api.put` | PUT | `(api.put="/user/:id")` |
| `api.delete` | DELETE | `(api.delete="/user/:id")` |
| `api.patch` | PATCH | `(api.patch="/user/:id")` |
| `api.options` | OPTIONS | 预检 |
| `api.head` | HEAD | 探活/元信息 |
| `api.any` | 任意方法 | 网关透传 |

Thrift：

```thrift
service UserService {
    GetUserResp GetUser(1: GetUserReq req) (api.get="/user/:id");
    UpdateUserResp UpdateUser(1: UpdateUserReq req) (api.put="/user/:id");
    CreateUserResp CreateUser(1: CreateUserReq req) (api.post="/user");
}
```

Protobuf：

```proto
service UserService {
  rpc GetUser(GetUserReq) returns (GetUserResp) {
    option (api.get) = "/user/:id";
  }
  rpc UpdateUser(UpdateUserReq) returns (UpdateUserResp) {
    option (api.put) = "/user/:id";
  }
}
```

路径参数必须与 `api.path` 的名字一致：路由里写 `:id`，字段就必须 `api.path="id"`，否则绑定为空。
## 四、进阶注解
| 注解 | 位置 | 作用 | 备注 |
| --- | --- | --- | --- |
| `api.base_domain` | service | 客户端默认域名 | `hz client` 生成时使用 |
| `api.param` | rpc | 标识公共参数 | 主要影响 client 生成，具体语义以官方文档为准 |
| `api.handler_path` | rpc（Protobuf） | 指定 handler 生成路径 | Thrift 侧能力以 hz 版本为准 |
| `api.gen_path` | rpc（Protobuf） | client 生成路径 | 优先级高于 `api_version` |
| `api.api_version` | rpc（Protobuf） | 替换路径中的 `:version` | client 场景 |
| `api.serializer` | rpc（Protobuf） | 指定序列化方式 | client 场景 |
| `api.none` | field | 不生成绑定 tag | 用于内部字段 |

关于 `handler` 与 `middleware`，需要特别说明：

- `handler_path` 只影响“生成到哪个文件/路径”，不改变 handler 的调用逻辑
- **hz 没有 `api.middleware` 这样的路由级中间件注解**。路由中间件应在 `biz/router/{namespace}/middleware.go` 中通过 `h.Group(...).Use(...)` 或路由组注册；这是官方推荐的扩展点
- 网上出现的 `api.middleware` 写法多来自其它框架或自定义插件，不要直接套用

关于 `api.param`：它用于标记“客户端请求是否携带公共参数”，属于 client 生成侧的元信息。不同 hz 版本对它的处理可能变化，使用前以官方注解文档与生成结果为准。
## 五、注解到生成代码的映射
假设 Thrift：

```thrift
namespace go api.user

struct GetUserReq {
    1: string ID (api.path="id", api.vd="len($)>0")
}

service UserService {
    GetUserResp GetUser(1: GetUserReq req) (api.get="/user/:id");
}
```

生成的 model 近似为：

```go
type GetUserReq struct {
	ID string `path:"id" vd:"len($)>0" json:"ID,omitempty"`
}
```

生成的 handler：

```go
// GetUser .
// @router /user/:id [GET]
func GetUser(ctx context.Context, c *app.RequestContext) {
	var err error
	var req api.GetUserReq
	err = c.BindAndValidate(&req)
	if err != nil {
		c.String(400, err.Error())
		return
	}
	resp := new(api.GetUserResp)
	c.JSON(200, resp)
}
```

生成的 router（简化）：

```go
func Register(r *server.Hertz) {
	root := r.Group("/", rootMw()...)
	root.GET("/user/:id", append(_getUserMw(), GetUser)...)
}
```

可以看到：注解只影响 tag 与路由，不会替你写业务逻辑。
## 六、校验注解 `api.vd`
`vd` 使用一套简洁的表达式语法，`$` 代表当前字段值：

| 需求 | 表达式示例 |
| --- | --- |
| 非空 | `len($)>0` |
| 长度范围 | `len($)>=2 && len($)<=32` |
| 枚举值 | `$=='app'||$=='web'` |
| 数值范围 | `$>=0 && $<=150` |
| 正则 | `regexp('^1[3-9]\\d{9}$',$)` |
| 非默认值 | `$!='your string'` |

正则里的反斜杠要转义：Thrift/Protobuf 字符串里写 `\\d`，最终传给 Go 的才是 `\d`。

`vd` 的能力由绑定与校验组件提供，复杂规则（跨字段、依赖外部数据）建议放到业务层：

```go
if req.StartTime.After(req.EndTime) {
	c.String(400, "start_time must be before end_time")
	return
}
```
## 七、tag 相关的生成开关
| 需求 | 参数 |
| --- | --- |
| 去掉 `omitempty` | `--unset_omitempty` |
| tag 用 snake_case | `--snake_tag` |
| Protobuf json tag 驼峰 | `--pb_camel_json_tag` |
| 移除默认 json/query/form tag | `--rm_tag` |
| Thrift 枚举用字符串 | `--json_enumstr` |

这些参数会改变已生成 model 的 tag，**中途修改会造成大面积 diff**，应在项目初期一次性确定。
## 常见坑
1. **路由 `:id` 与 `api.path="ID"` 大小写/拼写不一致**：能启动，但参数永远空。
2. **同一个字段同时标 `api.query` 和 `api.body`**：生成两个 tag，绑定行为取决于请求类型，极易混淆；一个字段只选一个来源。
3. **Protobuf 忘记 `import "api.proto"`**：注解无法解析，报 unknown option。
4. **`api.vd` 里的引号/反斜杠没转义**：IDL 解析失败或正则永远不匹配。
5. **想用 `api.middleware` 给单路由加中间件**：hz 无此注解，应改 `middleware.go` 或路由组。
6. **`go.tag` 手写 JSON tag 与 `api.body` 冲突**：两者都生成 json tag，后者可能覆盖前者。
7. **中途加 `--unset_omitempty`**：所有 model 的 tag 全变，review 成本极高。
8. **路径参数用了中文或特殊字符**：路由匹配与 URL 解码都会出问题，统一用英文 + 下划线/短横线。
## 小结
- Field 注解 → tag → 参数来源；Method 注解 → HTTP 方法 + 路由
- `api.path` 必须与路由 `:param` 完全一致
- `api.vd` 覆盖大部分单字段校验，复杂规则放业务层
- `handler_path` 只影响生成位置，hz 没有 `api.middleware` 注解
- tag 生成开关要一次定好，避免全量 diff
## 练习
1. 写一个同时包含 path、query、header、body 的 IDL，生成后检查 struct tag。
2. 给手机号字段加正则校验，用非法值请求验证返回 400。
3. 把 `--unset_omitempty` 打开后再生成，观察 diff 并评估是否值得。
4. 在 `middleware.go` 中给 `/user` 路由组加一个鉴权中间件，验证不影响其它路由。
## 官方文档
- IDL 注解说明：<https://www.cloudwego.io/zh/docs/hertz/tutorials/toolkit/annotation/>
- 参数绑定与校验：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/binding-and-validate/>
- api.none 注解：<https://www.cloudwego.io/zh/docs/hertz/tutorials/toolkit/more-feature/api_none/>
