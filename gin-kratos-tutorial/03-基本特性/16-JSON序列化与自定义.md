# JSON 序列化与自定义

> 一句话定位：`encoding/json` 有四个默认行为会咬人（HTML 转义、float64 精度、omitempty、时间格式）；搞懂 `c.JSON` 与 `c.PureJSON` 的差别，并给时间与金额写好自定义序列化，对外契约才稳定。

## 本篇目标

- 掌握 `c.JSON` / `c.PureJSON` / `c.IndentedJSON` 的差异与选择。
- 会用 build tag 或自定义 Render 替换 JSON 引擎（jsoniter / sonic）。
- 会为时间、金额等类型实现 `MarshalJSON` / `UnmarshalJSON`。
- 避开 `omitempty`、数字精度、HTML 转义三类高频坑。
- 会定义统一响应结构，并知道它与 protojson 的差异。

## 前置知识

- [响应与渲染](04-响应与渲染.md)
- [参数绑定与校验](03-参数绑定与校验.md)
- 了解 `encoding/json` 的反射实现与 `json.Marshaler` 接口。

## 一、encoding/json 的四个默认行为

| 行为 | 说明 | 代价 |
| --- | --- | --- |
| HTML 转义 | 默认把 `<` `>` `&` 转成 `\u003c` 等 | 前端拿到转义串，嵌入 JSON 时麻烦 |
| 数字统一 `float64` | 反序列化到 `interface{}` 时大整数丢精度 | `int64` ID 超过 2^53 会错 |
| `omitempty` 忽略零值 | `0`、`""`、`false`、`nil` 都被省略 | 前端分不清「没返回」与「值为 0」 |
| 时间用 RFC3339 | `time.Time` 输出 `2006-01-02T15:04:05Z07:00` | 与国内常用的 `yyyy-MM-dd HH:mm:ss` 不一致 |

## 二、c.JSON 与 c.PureJSON

同一份 `gin.H{"html": "<b>hi</b>"}`：`c.JSON` 输出 `{"html":"\u003cb\u003ehi\u003c/b\u003e"}`，`c.PureJSON` 输出 `{"html":"<b>hi</b>"}`。

- 普通业务 JSON 用 `c.JSON`，默认转义更安全，可防 XSS 注入页面脚本。
- 明确要原样输出（Markdown、HTML 片段、含 `<` 的配置）用 `c.PureJSON`。
- 调试期用 `c.IndentedJSON`，**线上不要用**，体积会翻倍。

## 三、替换 JSON 引擎

**方式 A：build tag（最省事）**：`go build -tags=jsoniter ./...` 或 `-tags=go_json`，代码零改动，但必须跑全量测试——它们对 `json.Number`、自定义 `Marshaler` 的错误返回、非法 JSON 容错度都与标准库略有不同。

**方式 B：自定义 Render（需要 sonic 这类库时）**：

```go
// pkg/render/sonic.go
package render

import (
    "net/http"

    "github.com/bytedance/sonic"
)

type SonicJSON struct{ Data any }

func (r SonicJSON) Render(w http.ResponseWriter) error {
    r.WriteContentType(w)
    b, err := sonic.ConfigDefault.Marshal(r.Data)
    if err != nil {
        return err
    }
    _, err = w.Write(b)
    return err
}

func (r SonicJSON) WriteContentType(w http.ResponseWriter) {
    if w.Header().Get("Content-Type") == "" {
        w.Header().Set("Content-Type", "application/json; charset=utf-8")
    }
}

// 使用：c.Render(http.StatusOK, render.SonicJSON{Data: payload})
```

## 四、自定义时间与金额

对外接口最稳的做法：**时间统一格式、金额用整数分**，并把规则固化到类型上。序列化用 `MarshalJSON`，反序列化用与之对称的 `UnmarshalJSON`（时间用 `time.ParseInLocation` 解析，金额把 `"12.34"` 拆成元与分再乘加）。

```go
// pkg/types/types.go
package types

import (
    "fmt"
    "strconv"
    "strings"
    "time"
)

const Layout = "2006-01-02 15:04:05"

type Time time.Time

func (t Time) MarshalJSON() ([]byte, error) {
    return []byte(fmt.Sprintf("%q", time.Time(t).Format(Layout))), nil
}

// UnmarshalJSON 与之对称：time.ParseInLocation(`"`+Layout+`"`, string(b), time.Local)

// Money 内部用「分」存储，对外输出 "12.34"
type Money int64

func (m Money) MarshalJSON() ([]byte, error) {
    v, sign := int64(m), ""
    if v < 0 {
        sign, v = "-", -v
    }
    return []byte(fmt.Sprintf(`"%s%d.%02d"`, sign, v/100, v%100)), nil
}

func (m *Money) UnmarshalJSON(b []byte) error {
    s := strings.Trim(string(b), `"`)
    if s == "" || s == "null" {
        return nil
    }
    neg := strings.HasPrefix(s, "-")
    yuanStr, centStr, _ := strings.Cut(strings.TrimPrefix(s, "-"), ".")
    yuan, err := strconv.ParseInt(yuanStr, 10, 64)
    if err != nil {
        return err
    }
    var cents int64
    if centStr != "" {
        if cents, err = strconv.ParseInt((centStr+"00")[:2], 10, 64); err != nil {
            return err
        }
    }
    *m = Money(yuan*100 + cents)
    if neg {
        *m = -*m
    }
    return nil
}
```

金额永远不要用 `float64`：`0.1+0.2 != 0.3`，一分钱误差在财务场景就是事故；更高精度（汇率、利率）用 `github.com/shopspring/decimal`。

## 五、数字精度与统一响应

前端 JS 的 `Number` 也是 float64，`int64` 主键超过 2^53 会静默丢精度：**ID 一律字符串化**。反序列化通用 JSON 时用 `json.NewDecoder(...).UseNumber()` 保留原始文本，再按需 `Int64()`。

统一响应结构（`Body` + `OK/Fail`）见 [响应与渲染](04-响应与渲染.md) 第二节，此处不重复；注意 `Data` 上的 `omitempty` 会让空对象整体消失，需要稳定字段时去掉该 tag。

用 protojson（`MarshalOptions{UseProtoNames: false}`）时有几处差异要写进契约：

- 字段名默认 lowerCamelCase（`UseProtoNames` 可切回 snake_case），`enum` 输出名字而非数字。
- `int64` / `uint64` 输出为**字符串**，前端按数字解析会失败；未知字段默认报错，`DiscardUnknown` 可关闭。

## 完整示例

```go
// cmd/json-demo/main.go
package main

import (
    "log"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"

    "mall-lite/pkg/types"
)

type Order struct {
    ID      int64       `json:"id"`
    Amount  types.Money `json:"amount"`
    Created types.Time  `json:"created_at"`
    Note    string      `json:"note,omitempty"`
    Coupon  int64       `json:"coupon_id,omitempty"` // 0 会被省略，需要区分就改 *int64
}

func main() {
    r := gin.New()
    r.GET("/order", func(c *gin.Context) {
        c.PureJSON(http.StatusOK, gin.H{"code": 0, "data": Order{
            ID:      1,
            Amount:  types.Money(1234), // 12.34 元
            Created: types.Time(time.Date(2026, 11, 25, 10, 0, 0, 0, time.Local)),
        }})
    })
    log.Fatal(http.ListenAndServe(":8080", r))
}
```

输出：`{"code":0,"data":{"id":1,"amount":"12.34","created_at":"2026-11-25 10:00:00"}}`。

## 常见坑

1. **用 `omitempty` 表达 `false`/`0`**：字段直接消失，前端按 `undefined` 处理导致逻辑错乱；关键值改用 `*bool`。
2. **金额用 `float64`**：累加后出现 `0.30000000000000004`，对账永远差几分。
3. **`int64` ID 直接给 JS**：超过 2^53 精度丢失，出现「查不到订单」；ID 一律字符串化。
4. **`c.PureJSON` 输出到 `<script>` 中**：关闭转义后 `</script>` 会提前闭合标签，产生 XSS。
5. **替换引擎后不跑测试**：jsoniter/sonic 对非法 JSON、`Marshaler` 错误、`json.Number` 行为不同。
6. **`time.Time` 直接输出 RFC3339**：前端时区转换出错；用自定义类型固定格式与时区。

## 小结

- 默认 `encoding/json` 会 HTML 转义、丢大整数精度、省略零值、时间用 RFC3339，设计 API 时逐条确认。
- `c.JSON` 安全、`c.PureJSON` 原样输出；格式化输出只在调试用。
- 换引擎优先 build tag，sonic 走自定义 Render 并承担平台风险。
- 金额用整数分 + 自定义序列化，时间用自定义类型，ID 字符串化。

## 练习

1. 写一个含 `int64` 大 ID（`9007199254740993`）的接口，用浏览器 `JSON.parse` 验证精度丢失，再改成字符串输出。
2. 给 `types.Money` 写单测，覆盖 `"0.01"`、`"-1.5"`、`"12"`、`"abc"` 四个输入。
3. 用 `go build -tags=jsoniter` 构建同一服务跑接口回归，记录 benchmark 前后差异。

## 官方文档

- `encoding/json`：<https://pkg.go.dev/encoding/json>
- Gin 渲染（JSON/PureJSON）：<https://gin-gonic.com/zh-cn/docs/rendering/>
- json-iterator：<https://github.com/json-iterator/go>
- sonic：<https://github.com/bytedance/sonic>
- protojson：<https://pkg.go.dev/google.golang.org/protobuf/encoding/protojson>
