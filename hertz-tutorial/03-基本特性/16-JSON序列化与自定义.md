# 16-JSON序列化与自定义
> Hertz 默认用 Sonic 做 JSON 序列化，但默认值不一定适合你的 API：本文讲清切换、定制与精度陷阱。

## 本篇目标

- 理解 Hertz 默认 JSON 实现（Sonic）与 `stdjson` 条件编译。
- 会用 `c.JSON`、`c.PureJSON`、`render.JSON` 输出 JSON，并知道三者差异。
- 会用 `render.ResetJSONMarshal` 自定义序列化，用 `BindConfig` 自定义反序列化。
- 掌握大整数精度、时间格式、`omitempty` 等业务高频问题。
- 避开全局配置、PureJSON 绕过自定义、HTML 转义等坑。

## 前置知识

- 熟悉 Go 的 `encoding/json`、struct tag、`omitempty`。
- 了解 Hertz 的 `app.RequestContext` 与渲染器（见 03-基本特性/04）。
- 知道 JSON number 在 JS 中会被解析为 IEEE 754 双精度浮点。

## 1. Hertz 的默认 JSON 实现

Hertz 把 JSON 能力收敛在 `pkg/common/json`：

- 在 amd64/arm64 且未使用 `stdjson` 构建标签时，使用 [Sonic](https://github.com/bytedance/sonic)；
- 其他平台或 `go build -tags stdjson` 时，回退到 `encoding/json`。

`pkg/common/json` 暴露了 `Marshal`、`Unmarshal`、`MarshalIndent`、`NewDecoder`、`NewEncoder`，以及标识当前实现的常量 `Name`（`"sonic"` 或 `"std"`）。

业务代码通常不需要直接引用它，因为：

- `c.JSON` 走 `render.JSONRender`，内部调用 `render` 包的 `jsonMarshalFunc`，默认就是 `hjson.Marshal`；
- `c.Bind` / `c.BindAndValidate` 的请求体反序列化由 `binding` 包负责，默认也使用 Sonic。

### 1.1 用构建标签切回标准库

```bash
go build -tags stdjson -o app .
go test -tags stdjson ./...
```

适合排查 Sonic 在特定平台的兼容性问题。注意生成物与不带 tag 的构建行为不同，部署时要保持一致。

## 2. 三种输出 JSON 的方式

```go
import (
	"github.com/cloudwego/hertz/pkg/common/utils"
	"github.com/cloudwego/hertz/pkg/protocol/consts"
)

func handler(ctx context.Context, c *app.RequestContext) {
	// 1) 最常用：使用当前配置的 Marshal（默认 Sonic），会转义 HTML 字符
	c.JSON(consts.StatusOK, utils.H{
		"message": "hello <hertz>",
	})

	// 2) 不转义 HTML 字符，内部固定使用 encoding/json
	c.PureJSON(consts.StatusOK, utils.H{
		"html": "<div>hello</div>",
	})

	// 3) 手动指定渲染器
	c.Render(consts.StatusOK, render.JSONRender{Data: utils.H{"ok": true}})
	c.Render(consts.StatusOK, render.IndentedJSON{Data: utils.H{"ok": true}})
}
```

关键差异：

| 方式 | 使用的库 | HTML 转义 | 受自定义 Marshal 影响 |
| --- | --- | --- | --- |
| `c.JSON` | 当前 `render` 配置（默认 Sonic） | 是 | 是 |
| `c.PureJSON` | 固定 `encoding/json` | 否 | **否** |
| `c.Render(render.JSONRender{})` | 当前配置 | 是 | 是 |

`utils.H` 只是 `map[string]interface{}` 的别名，写起来方便，但缺乏类型约束，正式 API 建议用 struct。

## 3. 自定义序列化

### 3.1 替换全局 Marshal

`render.ResetJSONMarshal` 可以替换 `c.JSON` 使用的序列化函数。**不存在 `WithJSONMarshal` 这样的服务端选项**，正确 API 是：

```go
import (
	"encoding/json"

	"github.com/cloudwego/hertz/pkg/app/server/render"
)

func init() {
	// 切回标准库
	render.ResetStdJSONMarshal()

	// 或者接入自定义实现，例如带统一时间格式的包装
	render.ResetJSONMarshal(func(v interface{}) ([]byte, error) {
		return json.Marshal(v)
	})
}
```

必须在服务启动前（如 `init` 或 `main` 开头）设置。运行期并发修改会导致数据竞争，且不同请求可能拿到不同实现。

### 3.2 自定义反序列化（绑定）

请求体解析由 `binding` 独立控制：

```go
func newServer() *server.Hertz {
	bc := binding.NewBindConfig()
	bc.UseStdJSONUnmarshaler() // 或 UseThirdPartyJSONUnmarshaler(func([]byte, interface{}) error)

	// 也可以配置校验相关的 ValidateConfig
	vc := binding.NewValidateConfig()

	return server.New(
		server.WithHostPorts(":8080"),
		server.WithBindConfig(bc),
		server.WithValidateConfig(vc),
	)
}
```

注意 `binding` 与 `render` 是**两套独立配置**：前者管入参，后者管出参，改一个不会影响另一个。

## 4. 大数字精度

JSON number 在 JavaScript 中最多安全表示 2^53-1。雪花 ID、订单号常见 19 位，直接以 number 返回会被前端截断。

```go
// 错误：前端可能拿到 1234567890123456800
type Order struct {
	ID int64 `json:"id"`
}

// 方案一：序列化为字符串
type OrderSafe struct {
	ID int64 `json:"id,string"` // 输出 "1234567890123456789"
}

// 方案二：字段本身就用 string
type OrderString struct {
	ID string `json:"id"`
}
```

解析未知 JSON 时，若要保留原始数字精度：

```go
dec := json.NewDecoder(bytes.NewReader(data))
dec.UseNumber() // 数字解析为 json.Number 而非 float64
var v map[string]interface{}
if err := dec.Decode(&v); err != nil {
	return err
}
```

绑定到结构体时优先用 `int64` / `string`，不要用 `float64` 接 ID。

## 5. 时间格式

`time.Time` 默认按 RFC3339（带纳秒）输出，例如 `2026-11-23T10:00:00.123456789+08:00`。移动端或旧系统常要求 `2006-01-02 15:04:05`：

```go
type Timestamp time.Time

func (t Timestamp) MarshalJSON() ([]byte, error) {
	s := time.Time(t).Format("2006-01-02 15:04:05")
	return []byte(`"` + s + `"`), nil
}

func (t *Timestamp) UnmarshalJSON(b []byte) error {
	s := strings.Trim(string(b), `"`)
	parsed, err := time.ParseInLocation("2006-01-02 15:04:05", s, time.Local)
	if err != nil {
		return err
	}
	*t = Timestamp(parsed)
	return nil
}
```

团队内也可以统一封装 `response` 层，把 `time.Time` 转成字符串后再输出，避免每个 DTO 都写自定义类型。

## 6. omitempty 的正确理解

```go
type User struct {
	Name  string   `json:"name,omitempty"`  // 空字符串被省略
	Age   int      `json:"age,omitempty"`   // 0 被省略
	Score *int     `json:"score,omitempty"` // nil 被省略，0 会保留
	Tags  []string `json:"tags,omitempty"`  // nil 与空切片都被省略
}
```

注意：

- `omitempty` 对 `false`、`0`、`""`、`nil`、空 slice/map 生效；
- 对**零值 struct 不生效**，需要改为指针或 `*struct{}`；
- 如果业务上"0 是合法值"，不要用 `omitempty`，否则前端无法区分"未传"和"值为 0"。

## 7. 性能对比怎么看

Sonic 通过 JIT、SIMD 等手段在 amd64/arm64 上通常快于 `encoding/json`，但差距取决于数据结构、字段数量与字符串占比。建议：

```go
func BenchmarkMarshalJSON(b *testing.B) {
	v := User{Name: "hertz", Age: 3, Tags: []string{"a", "b"}}
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		if _, err := hjson.Marshal(v); err != nil {
			b.Fatal(err)
		}
	}
}
```

分别用 `-tags stdjson` 与默认构建跑 `go test -bench`，用自己真实的数据结构比较。不要直接引用网上的固定倍数。

## 常见坑

1. **找 `server.WithJSONMarshal`**：不存在；正确做法是 `render.ResetJSONMarshal` / `render.ResetStdJSONMarshal`。
2. **以为 `c.PureJSON` 会走自定义 Marshal**：不会，它固定用 `encoding/json` 且关闭 HTML 转义。
3. **运行期修改全局 Marshal**：数据竞争，必须在启动前设置一次。
4. **大整数用 number 返回**：前端精度丢失，用 `,string` 或直接返回字符串。
5. **`omitempty` 把合法的 0 / false 吞掉**：根据业务语义选择是否使用，必要时用指针。
6. **只改了 render 却以为绑定也变了**：入参和出参是两套配置，分别设置 `BindConfig` 与 `render`。
7. **依赖 map 的 key 顺序**：JSON 对象顺序不保证，测试请解码后比较，不要比较原始字符串。
8. **Sonic 在非常规平台报错**：确认架构支持情况，必要时用 `stdjson` 构建标签回退。

## 小结

- Hertz 默认用 Sonic，`stdjson` 可回退标准库；`render` 管输出，`binding` 管输入。
- `c.JSON` 受自定义 Marshal 影响，`c.PureJSON` 不受且不转义 HTML。
- 自定义入口是 `render.ResetJSONMarshal` 与 `binding.BindConfig.UseStdJSONUnmarshaler`。
- 大整数必须用字符串或 `json.Number`，避免前端精度丢失。
- 时间格式、`omitempty` 要在 DTO 层明确约定并写测试。

## 练习

1. 用 `render.ResetStdJSONMarshal` 切换实现，验证 `c.JSON` 输出变化，并观察 HTML 字符是否被转义。
2. 给一个包含 19 位 ID 的接口分别用 `int64` 与 `json:"id,string"` 输出，在前端/JS 环境验证精度差异。
3. 自定义一个 `Timestamp` 类型，实现 `MarshalJSON` 与 `UnmarshalJSON`，完成请求-响应闭环。
4. 用 `binding.NewBindConfig().UseStdJSONUnmarshaler()` 替换入参解析，验证非法 JSON 的错误信息变化。
5. 写 Benchmark 对比 Sonic 与 `encoding/json` 在你真实 DTO 上的 `ns/op` 与 `allocs/op`。

## 官方文档

- JSON Marshal 库：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/json/>
- 渲染：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/render/>
- 绑定与校验：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/binding-and-validate/>
