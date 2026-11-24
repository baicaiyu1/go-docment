# 单测与 Mock

> 一句话定位：Gin handler 用 `httptest` + `gin.TestMode` 就能完整测试，无需监听端口；下游依赖用接口 + `go.uber.org/mock/gomock` 隔离，真实依赖交给 testcontainers 做集成测试。

## 本篇目标

- 理解 Gin 单测原理：`r.ServeHTTP(w, req)` 直接执行路由树，不经过 TCP。
- 会写表驱动的 handler 测试，用 testify 断言状态码、Header 与 JSON。
- 会用接口抽象 Repository，并用 gomock 生成与驱动 mock。
- 知道什么该放单元测试、什么该放 testcontainers 集成测试。
- 会上 `-race`、`-cover`，并读懂覆盖率报告。

## 前置知识

- [路由](01-路由.md) 与 [中间件](05-中间件.md)
- [Go 测试与基准测试](../01-Go语言基础/10-测试与基准测试.md)
- 了解构造函数注入。

## 一、测试分层

- **单元**（handler / service）：`httptest` + testify + gomock，无外部依赖，毫秒级。
- **集成**（DB / Redis / Kafka）：testcontainers 或 docker compose，秒级，验证真实 SQL 与事务。
- **压测 / 端到端**：k6、wrk、ghz，分钟级。能在接口层 mock 的绝不起容器。

## 二、用 httptest 测 Gin handler

三件套：`gin.SetMode(gin.TestMode)` 关调试日志、`httptest.NewRequest` 造请求、`httptest.NewRecorder` 接响应。

```go
// internal/handler/user_test.go
package handler_test

import (
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
    "github.com/stretchr/testify/assert"
)

func newRouter() *gin.Engine {
    gin.SetMode(gin.TestMode)
    r := gin.New()
    r.GET("/users/:id", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"code": 0, "data": gin.H{"id": c.Param("id")}})
    })
    return r
}

func TestRoutes(t *testing.T) { // 表驱动：一个循环覆盖正常与 404
    r := newRouter()
    cases := []struct {
        name, path string
        status     int
        body       string
    }{
        {"ok", "/users/42", http.StatusOK, `{"code":0,"data":{"id":"42"}}`},
        {"not found", "/nope", http.StatusNotFound, ""},
    }
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            w := httptest.NewRecorder()
            r.ServeHTTP(w, httptest.NewRequest(http.MethodGet, tc.path, nil))
            assert.Equal(t, tc.status, w.Code)
            assert.Contains(t, w.Header().Get("Content-Type"), "application/json")
            if tc.body != "" {
                assert.JSONEq(t, tc.body, w.Body.String()) // 解码后比较，别比原始字符串
            }
        })
    }
}
```

> `405` 需要 `r.HandleMethodNotAllowed = true`，否则默认 `404`。

## 三、接口抽象 + gomock

```go
// internal/repo/user.go：先把数据访问抽成接口
package repo

import "context"

//go:generate mockgen -source=user.go -destination=../mocks/user_mock.go -package=mocks
type UserRepo interface {
    GetByID(ctx context.Context, id int64) (*User, error)
}
```

安装与生成：`go install go.uber.org/mock/mockgen@latest && go generate ./...`。

```go
// internal/handler/user_mock_test.go
package handler_test

import (
    "errors"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
    "github.com/stretchr/testify/assert"
    "go.uber.org/mock/gomock"

    "mall-lite/internal/handler"
    "mall-lite/internal/mocks"
    "mall-lite/internal/repo"
)

func TestGetUser_NotFound(t *testing.T) {
    ctrl := gomock.NewController(t) // Go 1.14+ 自动 t.Cleanup，无需 defer Finish
    mr := mocks.NewMockUserRepo(ctrl)
    mr.EXPECT().GetByID(gomock.Any(), int64(99)).Return(nil, errors.New("not found")).Times(1)

    gin.SetMode(gin.TestMode)
    r := gin.New()
    r.GET("/users/:id", handler.GetUser(mr)) // 注入 mock

    w := httptest.NewRecorder()
    r.ServeHTTP(w, httptest.NewRequest(http.MethodGet, "/users/99", nil))
    assert.Equal(t, http.StatusNotFound, w.Code)
}

var _ repo.UserRepo = (*mocks.MockUserRepo)(nil) // 编译期确认接口实现
```

handler 侧要点：`r.GetByID(c.Request.Context(), id)` 透传请求 context，出错时 `c.JSON(404, ...)` 后立即 `return`。

## 四、集成测试：testcontainers

testcontainers-go 在测试进程内拉起容器并自动回收，适合 CI；docker compose 适合跑全链路冒烟。

```go
//go:build integration

package repo_test

import (
    "context"
    "testing"
    "time"

    "github.com/stretchr/testify/require"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/wait"
)

func TestMySQL(t *testing.T) {
    ctx := context.Background()
    c, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "mysql:8.4",
            ExposedPorts: []string{"3306/tcp"},
            Env:          map[string]string{"MYSQL_ROOT_PASSWORD": "root", "MYSQL_DATABASE": "mall"},
            WaitingFor:   wait.ForLog("ready for connections").WithStartupTimeout(90 * time.Second),
        },
        Started: true,
    })
    require.NoError(t, err)
    defer testcontainers.CleanupContainer(t, c)
    port, err := c.MappedPort(ctx, "3306")
    require.NoError(t, err)
    require.NotEmpty(t, port.Port()) // 之后用 GORM 连接，验证事务提交与回滚
}
```

`go test ./...` 默认跳过它，`go test -tags=integration ./...` 才执行——单测永远不该依赖 Docker。

## 五、覆盖率与竞态

跑 `go test -race -coverprofile=cover.out -covermode=atomic ./...`，再 `go tool cover -html=cover.out` 看报告。

`-race` 必开：Gin 的 `Context` 是池化的，跨 goroutine 误用（没 `c.Copy()`）只有竞态检测能抓到。覆盖率盯关键错误分支，核心包建议 ≥ 70%、新代码 ≥ 80%。

## 完整示例

第二节的 `TestRoutes` 就是最小可运行测试：`gin.SetMode(gin.TestMode)` + `httptest.NewRecorder/NewRequest` + `r.ServeHTTP`，无需再写一份。整包运行 `go test -race -cover ./internal/handler/`。

## 常见坑

1. **测试里调用 `ListenAndServe()`**：会真的监听端口并阻塞；单测只用 `r.ServeHTTP`（运行方式：`go test -race -cover ./internal/handler/`）。
2. **忘记 `gin.SetMode(gin.TestMode)`**：每个用例打印大量调试日志，淹没 CI。
3. **比较 JSON 原始字符串**：字段顺序、空格、转义都可能不同，用 `assert.JSONEq`。
4. **mock 不写 `Times()`**：多余调用不会被发现，行为漂移无人知。
5. **单元测试连真实 MySQL**：CI 无容器就红；真实 SQL 放 `-tags=integration`。

## 小结

- Gin 单测 = `gin.TestMode` + `httptest` + `r.ServeHTTP`，不监听端口、毫秒级完成。
- 依赖用接口注入，gomock 生成 mock，`EXPECT()` 明确次数与参数；真实依赖用 testcontainers + build tag 隔离。
- CI 固定 `-race -cover`，覆盖率盯错误分支。

## 练习

1. 给 `POST /users` 写表驱动测试，覆盖 200 / 400（JSON 格式错）/ 422（校验失败）。
2. 用 gomock 给 `UserRepo` 生成 mock，写「找到用户」与「返回错误」两个用例。
3. 用 testcontainers 起 MySQL，验证 `db.Transaction` 出错时是否正确回滚。

## 官方文档

- `net/http/httptest`：<https://pkg.go.dev/net/http/httptest>
- testify：<https://github.com/stretchr/testify>
- go.uber.org/mock：<https://github.com/uber-go/mock>
- testcontainers-go：<https://golang.testcontainers.org/>
- Gin 测试：<https://gin-gonic.com/zh-cn/docs/testing/>
