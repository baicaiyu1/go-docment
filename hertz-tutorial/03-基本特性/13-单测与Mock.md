# 13-单测与Mock
> 不监听端口也能测完整路由：用 Hertz 的 `ut` 包做请求级单测，用接口替身隔离下游依赖。

## 本篇目标

- 理解 Hertz 单测工具的原理：只调用 `ServeHTTP`，不经过真实网络。
- 用 `ut.PerformRequest` 构造请求并断言状态码、Header、Body。
- 用 `ut.CreateUtRequestContext` 对单个 Handler 做更细粒度的测试。
- 通过接口注入 Mock 下游依赖，让单测稳定、快速、可并行。
- 写出有意义的 Benchmark，并避开测试中的常见陷阱。

## 前置知识

- 会写 Go 的 `testing` 测试与 table-driven 测试。
- 理解 `httptest` 的思路（构造请求 → 执行 Handler → 检查响应），Hertz 的 `ut` 与之类似。
- 熟悉 Hertz 路由注册与 `app.RequestContext` 的基本用法。

## 1. Hertz 单测工具在哪

Hertz 的单测能力在 `pkg/common/ut` 包（不是 `pkg/app/server/ut`，v0.10.6 没有这个包）：

```go
import (
	"github.com/cloudwego/hertz/pkg/common/ut"
	"github.com/cloudwego/hertz/pkg/common/test/assert"
)
```

核心 API：

| API | 作用 |
| --- | --- |
| `ut.PerformRequest(engine, method, url, body, headers...)` | 构造请求并直接交给 `route.Engine` 执行，返回 `*ut.ResponseRecorder` |
| `ut.CreateUtRequestContext(method, url, body, headers...)` | 只创建 `*app.RequestContext`，用于直接调用 Handler |
| `ut.Body{Body: io.Reader, Len: int}` | 请求体，`Len` 必须准确；`Len = -1` 表示流式 |
| `ut.Header{Key, Value}` | 请求头 |
| `recorder.Result()` | 拿到 `*protocol.Response`，可读状态码、Header、Body |

`PerformRequest` 内部会新建 Engine Context、调用 `engine.ServeHTTP(context.Background(), ctx)`，再把响应写进 `ResponseRecorder`。因此**不会监听端口，也不会走 TCP**。

## 2. 路由级测试：最常用的写法

```go
package handler

import (
	"encoding/json"
	"testing"

	"github.com/cloudwego/hertz/pkg/app/server"
	"github.com/cloudwego/hertz/pkg/common/test/assert"
	"github.com/cloudwego/hertz/pkg/common/ut"
)

type userResp struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

func newTestRouter() *server.Hertz {
	h := server.New()
	h.GET("/users/:id", func(ctx context.Context, c *app.RequestContext) {
		id := c.Param("id")
		c.JSON(200, userResp{ID: 1, Name: "user-" + id})
	})
	return h
}

func TestGetUser(t *testing.T) {
	h := newTestRouter()

	// 第四个参数为 nil 表示没有请求体
	w := ut.PerformRequest(h.Engine, "GET", "/users/42", nil)
	resp := w.Result()

	assert.DeepEqual(t, 200, resp.StatusCode())
	assert.DeepEqual(t, "application/json; charset=utf-8", string(resp.Header.ContentType()))

	// 不要直接比较 JSON 字符串，解码后比较结构体更稳
	var got userResp
	if err := json.Unmarshal(resp.Body(), &got); err != nil {
		t.Fatalf("unmarshal body: %v", err)
	}
	assert.DeepEqual(t, userResp{ID: 1, Name: "user-42"}, got)
}
```

带请求体与 Header 的 POST：

```go
body := `{"name":"hertz"}`
w := ut.PerformRequest(
	h.Engine,
	"POST",
	"/users?from=ut", // query 直接写在 URL 上
	&ut.Body{Body: strings.NewReader(body), Len: len(body)},
	ut.Header{Key: "Content-Type", Value: "application/json"},
	ut.Header{Key: "X-Trace-ID", Value: "trace-1"},
)
resp := w.Result()
```

table-driven 是更推荐的组织方式：

```go
func TestRoutes(t *testing.T) {
	h := newTestRouter()
	cases := []struct {
		name       string
		method     string
		path       string
		wantStatus int
	}{
		{"ok", "GET", "/users/1", 200},
		{"not found", "GET", "/nope", 404},
		{"method not allowed", "DELETE", "/users/1", 405},
	}
	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			w := ut.PerformRequest(h.Engine, tc.method, tc.path, nil)
			assert.DeepEqual(t, tc.wantStatus, w.Result().StatusCode())
		})
	}
}
```

> 404/405 的返回取决于 Engine 的 `WithHandleMethodNotAllowed` 等配置，测试前先确认路由配置，避免断言写死默认值。

## 3. Handler 级测试：直接构造 Context

如果只想测一个 Handler 函数，不需要注册路由：

```go
func TestPingHandler(t *testing.T) {
	c := ut.CreateUtRequestContext("GET", "/ping", nil,
		ut.Header{Key: "X-Env", Value: "test"})

	Ping(context.Background(), c) // 直接调用业务 Handler

	assert.DeepEqual(t, 200, c.Response.StatusCode())
	assert.DeepEqual(t, `{"message":"pong"}`, string(c.Response.Body()))
}
```

适合场景：Handler 逻辑复杂、依赖通过 `c` 上的值注入、不想被路由匹配规则干扰。但如果要验证参数绑定、中间件、404 等，仍应使用 `PerformRequest`。

## 4. Mock 下游依赖

单测要避免真实数据库、Redis、HTTP 下游。推荐用接口 + 构造函数注入：

```go
// service.go
type UserRepo interface {
	GetByID(ctx context.Context, id int64) (*User, error)
}

type UserService struct {
	repo UserRepo
}

func NewUserService(repo UserRepo) *UserService {
	return &UserService{repo: repo}
}

func (s *UserService) Get(ctx context.Context, id int64) (*User, error) {
	return s.repo.GetByID(ctx, id)
}
```

```go
// service_test.go
type fakeRepo struct {
	user *User
	err  error
}

func (f *fakeRepo) GetByID(context.Context, int64) (*User, error) {
	return f.user, f.err
}

func TestUserService_Get(t *testing.T) {
	svc := NewUserService(&fakeRepo{user: &User{ID: 1, Name: "hertz"}})
	got, err := svc.Get(context.Background(), 1)
	if err != nil {
		t.Fatal(err)
	}
	assert.DeepEqual(t, "hertz", got.Name)
}
```

HTTP Handler 中同理：在 `newTestRouter(repo)` 里注入 fake，路由测试就能完全离线运行。

如果确实需要模拟 HTTP 下游（例如验证 Hertz Client 的重试），可以用 `httptest.NewServer`，但它会监听本地端口，更适合放在集成测试（`//go:build integration`）而不是纯单元测试。能在接口层 Mock 的，就不要起服务。

## 5. Benchmark

```go
func BenchmarkGetUser(b *testing.B) {
	h := newTestRouter()
	b.ReportAllocs()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		w := ut.PerformRequest(h.Engine, "GET", "/users/1", nil)
		if w.Result().StatusCode() != 200 {
			b.Fatal("unexpected status")
		}
	}
}
```

运行 `go test -bench=BenchmarkGetUser -benchmem -count=5`。注意：

- `PerformRequest` 包含 Context 创建与响应记录开销，适合做**相对对比**（改动前后）；
- 不要用它冒充真实网络性能，真实压测请用 `hey` / `wrk`。

## 6. 测试组织建议

- 路由构造抽成 `newTestRouter()`，避免每个用例重复注册。
- 断言优先用标准库 `testing` 或 `hertz/pkg/common/test/assert`，团队内统一一种。
- CI 中加 `-race`，Hertz 的 Context 池化在并发测试下更容易暴露问题。

## 常见坑

1. **测试里调用 `h.Spin()`**：会真的监听端口并阻塞；单测只用 `h.Engine` + `ut.PerformRequest`。
2. **`ut.Body.Len` 与实际长度不一致**：内容会被截断或读取阻塞，务必用 `len(body)`，流式才用 `-1`。
3. **直接比较 JSON 字符串**：字段顺序、空格、转义都可能不同，解码后比较结构体。
4. **`assert.DeepEqual` 参数顺序写反**：签名是 `(t, expected, actual)`，反了会让失败信息难以理解。
5. **全局 JSON Marshal 被其他测试改掉**：`render.ResetJSONMarshal` 是全局状态，测试结束要恢复，否则用例互相污染。
6. **中间件依赖真实外部服务**：路由测试会连带执行全局中间件，鉴权、DB 初始化要做成可注入或可跳过。
7. **`c.GetResponse()` 与 `c.Response` 混用**：前者返回拷贝，断言请使用 `c.Response` 或 `recorder.Result()`。

## 小结

- Hertz 单测 = 无网络执行 `ServeHTTP`，核心 API 在 `pkg/common/ut`；`PerformRequest` 测路由，`CreateUtRequestContext` 测 Handler。
- 断言状态码、Header、Body；JSON 建议解码后比较。
- 下游依赖用接口注入 fake；Benchmark 只做相对对比。

## 练习

1. 给一个带路径参数和 query 的接口写 table-driven 测试，覆盖 200 / 400 / 404。
2. 用 `CreateUtRequestContext` 直接测试一个读取请求头的 Handler。
3. 把某个直接依赖 MySQL 的 Handler 改造成接口注入，并用 fake 实现完成离线测试。
4. 为你的核心接口写 Benchmark，记录优化前后的 `ns/op` 与 `allocs/op`。
5. 故意把 `ut.Body.Len` 写错，观察测试表现并解释原因。

## 官方文档

- 单测：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/unit-test/>
- 路由：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/route/>
