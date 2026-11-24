# validator 速查

> 一句话定位：`binding` tag 是写给 go-playground/validator 看的“字段规则清单”，Gin 的 `ShouldBind`/`ShouldBindJSON` 内置就用它做校验，标签写得不对规则就不会生效。
>
> 本篇基于 `github.com/go-playground/validator/v10 v10.30.6` 的内置规则整理（Gin v1.10 内置的 validator 版本）。

## 先记住这 6 件事

1. **Gin 读取 `binding` tag；写 `validate` 标签不会生效**。  
   要用 `binding:"required,email"`，`ShouldBind*` 系列会自动调用 validator，无需自己注入，见 [03-参数绑定与校验](./03-参数绑定与校验.md)。
2. **多个规则用逗号分隔**：`binding:"required,min=2,max=20"`。
3. **规则参数用等号，多个参数用空格**：`oneof=admin user`、`required_if=Type card`。
4. **`omitempty` 是“有值才校验”**：字段为零值时，后面的规则全部跳过。
5. **`required` 是“不能为零值”**：空字符串、`0`、`false`、`nil` 都算零值。
6. **`dive` 是“进入元素继续校验”**：切片、数组、map 用得多。

---

## 使用前提

```go
import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	r.POST("/api/v1/users", func(c *gin.Context) {
		var req CreateUserReq
		if err := c.ShouldBindJSON(&req); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"code": 400, "message": err.Error()})
			return
		}
		c.JSON(http.StatusCreated, gin.H{"data": req})
	})

	r.Run(":8080")
}
```

`ShouldBindJSON` 内部已经用 `binding` tag 调用了 validator，不需要额外注册校验函数；把 tag 写成 `validate` 不会报错，但也不会生效。

---

## 特殊控制词

这些不是普通“格式规则”，而是控制校验行为。

| 写法 | 大白话 | 示例 |
| --- | --- | --- |
| `omitempty` | 字段是零值就跳过后面的规则 | `binding:"omitempty,email"` |
| `-` | 完全不校验这个字段 | `binding:"-"` |
| `dive` | 进入切片/数组/map 的元素继续校验 | `binding:"max=5,dive,required"` |
| `keys` / `endkeys` | 对 map 的 key 做校验，中间是 key 的规则 | `binding:"dive,keys,min=2,endkeys,required"` |
| `structonly` | 只执行结构体级校验，不递归校验字段 | `binding:"structonly"` |
| `nostructlevel` | 跳过结构体级校验，只校验字段 | `binding:"nostructlevel"` |

`dive` 可以出现多次，用来进入多层嵌套：

```go
Matrix [][]int `binding:"dive,dive,gte=0"`
```

---

## 必填与条件规则

| 规则 | 大白话 | 示例 |
| --- | --- | --- |
| `required` | 不能是零值 | `required` |
| `required_if` | 另一个字段等于某值时，当前字段必填 | `required_if=Type card` |
| `required_unless` | 另一个字段不等于某值时，当前字段必填 | `required_unless=Type free` |
| `required_with` | 另一个字段有值时，当前字段必填 | `required_with=Phone` |
| `required_with_all` | 列出的字段都有值时，当前字段必填 | `required_with_all=Phone Email` |
| `required_without` | 另一个字段没值时，当前字段必填 | `required_without=Phone` |
| `required_without_all` | 列出的字段都没值时，当前字段必填 | `required_without_all=Phone Email` |
| `excluded_if` | 另一个字段等于某值时，当前字段必须为空 | `excluded_if=Type free` |
| `excluded_unless` | 另一个字段不等于某值时，当前字段必须为空 | `excluded_unless=Type paid` |
| `excluded_with` | 另一个字段有值时，当前字段必须为空 | `excluded_with=Phone` |
| `excluded_with_all` | 列出的字段都有值时，当前字段必须为空 | `excluded_with_all=Phone Email` |
| `excluded_without` | 另一个字段没值时，当前字段必须为空 | `excluded_without=Phone` |
| `excluded_without_all` | 列出的字段都没值时，当前字段必须为空 | `excluded_without_all=Phone Email` |
| `skip_unless` | 条件不满足时，跳过当前字段后面的所有规则 | `skip_unless=Type card` |
| `isdefault` | 字段必须是零值/默认值 | `isdefault` |

注意：

- **`required` 对 `bool` 很坑**：`false` 是零值，`required` 会失败。要允许 `false`，用 `*bool` 或改用 `eq=true`。
- **`required` 对 `int` 也很坑**：`0` 是零值。如果 `0` 合法，用指针 `*int` 或 `omitempty` + `gte=0`。
- **`required` 用在结构体上**需要 `validator.WithRequiredStructEnabled()`（Gin 默认的 validator 没开，需要自己注册）。
- **条件里的字段名是 Go 结构体字段名**，不是 `json` 名。

---

## 长度、大小与比较

| 规则 | 大白话 | 示例 |
| --- | --- | --- |
| `len` | 长度/数量必须等于 N | `len=6` |
| `min` | 最小长度/最小数量/最小值 | `min=2` |
| `max` | 最大长度/最大数量/最大值 | `max=20` |
| `eq` | 等于 | `eq=18` |
| `eq_ignore_case` | 忽略大小写等于 | `eq_ignore_case=admin` |
| `ne` | 不等于 | `ne=0` |
| `ne_ignore_case` | 忽略大小写不等于 | `ne_ignore_case=admin` |
| `gt` | 大于 | `gt=0` |
| `gte` | 大于等于 | `gte=0` |
| `lt` | 小于 | `lt=100` |
| `lte` | 小于等于 | `lte=100` |
| `eqfield` | 等于另一个字段 | `eqfield=Password` |
| `nefield` | 不等于另一个字段 | `nefield=Username` |
| `gtfield` | 大于另一个字段 | `gtfield=StartAt` |
| `gtefield` | 大于等于另一个字段 | `gtefield=StartAt` |
| `ltfield` | 小于另一个字段 | `ltfield=EndAt` |
| `ltefield` | 小于等于另一个字段 | `ltefield=EndAt` |
| `eqcsfield` | 等于另一个结构体里的字段 | `eqcsfield=Profile.Email` |
| `necsfield` | 不等于另一个结构体里的字段 | `necsfield=Profile.Email` |
| `gtcsfield` | 大于另一个结构体里的字段 | `gtcsfield=Range.Min` |
| `gtecsfield` | 大于等于另一个结构体里的字段 | `gtecsfield=Range.Min` |
| `ltcsfield` | 小于另一个结构体里的字段 | `ltcsfield=Range.Max` |
| `ltecsfield` | 小于等于另一个结构体里的字段 | `ltecsfield=Range.Max` |
| `fieldcontains` | 当前字段包含另一个字段的值 | `fieldcontains=Keyword` |
| `fieldexcludes` | 当前字段不包含另一个字段的值 | `fieldexcludes=Keyword` |

**`len/min/max` 对字符串数的是字符数（rune），不是字节数。**  
中文“张三”是 2 个字符，不是 6 个字节。

---

## 字符串与内容

| 规则 | 大白话 | 示例 |
| --- | --- | --- |
| `alpha` | 只能是英文字母 | `alpha` |
| `alphaspace` | 英文字母或空格 | `alphaspace` |
| `alphanum` | 英文字母或数字 | `alphanum` |
| `alphanumspace` | 英文字母、数字或空格 | `alphanumspace` |
| `alphaunicode` | 任意语言的字母 | `alphaunicode` |
| `alphanumunicode` | 任意语言的字母或数字 | `alphanumunicode` |
| `boolean` | 布尔值字符串，如 `true/false/1/0/t/f` | `boolean` |
| `numeric` | 数字，可带正负号和小数 | `numeric` |
| `number` | 纯数字，只允许 `0-9` | `number` |
| `hexadecimal` | 十六进制字符 | `hexadecimal` |
| `contains` | 必须包含某段文本 | `contains=abc` |
| `containsany` | 必须包含其中一个字符 | `containsany=abc` |
| `containsrune` | 必须包含某个字符 | `containsrune=@` |
| `excludes` | 不能包含某段文本 | `excludes=admin` |
| `excludesall` | 不能包含其中任何一个字符 | `excludesall=<>` |
| `excludesrune` | 不能包含某个字符 | `excludesrune=,` |
| `startswith` | 必须以某段文本开头 | `startswith=IMG_` |
| `endswith` | 必须以某段文本结尾 | `endswith=.jpg` |
| `startsnotwith` | 不能以某段文本开头 | `startsnotwith=_` |
| `endsnotwith` | 不能以某段文本结尾 | `endsnotwith=.exe` |
| `lowercase` | 必须全小写 | `lowercase` |
| `uppercase` | 必须全大写 | `uppercase` |
| `ascii` | 只能是 ASCII 字符 | `ascii` |
| `printascii` | 只能是可打印 ASCII | `printascii` |
| `multibyte` | 必须包含多字节字符 | `multibyte` |
| `unique` | 切片元素不能重复 | `unique` |
| `oneof` | 必须是列出的值之一 | `oneof=admin user guest` |
| `oneofci` | 同上，忽略大小写 | `oneofci=Admin User` |
| `noneof` | 不能是列出的任何值 | `noneof=root admin` |
| `noneofci` | 同上，忽略大小写 | `noneofci=Root Admin` |
| `html` | 必须是 HTML 片段 | `html` |
| `html_encoded` | 必须是 HTML 编码后的文本 | `html_encoded` |
| `url_encoded` | 必须是 URL 编码后的文本 | `url_encoded` |
| `datauri` | 必须是 data URI | `datauri` |

`oneof` 的值里有空格时要用引号包起来：

```go
Level string `binding:"oneof='very low' low high"`
```

---

## 格式、编码与文件

| 规则 | 大白话 | 示例 |
| --- | --- | --- |
| `email` | 邮箱格式（实用型，不是完整 RFC） | `email` |
| `e164` | E.164 手机号，带 `+` 和国家码 | `e164` |
| `url` | URL，要求带协议 | `url` |
| `http_url` | HTTP URL | `http_url` |
| `https_url` | HTTPS URL | `https_url` |
| `uri` | 通用 URI | `uri` |
| `origin` | 只有协议、域名、端口，不带路径 | `origin` |
| `urn_rfc8141` | RFC 8141 的 URN | `urn_rfc8141` |
| `urn_rfc2141` | RFC 2141 的 URN | `urn_rfc2141` |
| `base32` | Base32 编码 | `base32` |
| `base64` | Base64 编码 | `base64` |
| `base64url` | URL 安全的 Base64 | `base64url` |
| `base64rawurl` | 无填充的 URL 安全 Base64 | `base64rawurl` |
| `json` | 合法 JSON 字符串 | `json` |
| `jwt` | JWT 格式 | `jwt` |
| `file` | 路径存在且是文件 | `file` |
| `filepath` | 路径格式合法（不检查是否存在） | `filepath` |
| `dir` | 路径存在且是目录 | `dir` |
| `dirpath` | 目录路径格式合法 | `dirpath` |
| `image` | 文件是图片（按扩展名/类型） | `image` |
| `mimetype` | 文件的 MIME 类型匹配 | `mimetype=image/png` |
| `hostname_port` | 主机名 + 端口 | `hostname_port` |
| `port` | 合法端口号 | `port` |
| `datetime` | 按指定 Go 时间布局解析 | `datetime=2006-01-02` |
| `timezone` | 合法 IANA 时区 | `timezone` |
| `duration` | Go 的时长格式，如 `1h30m` | `duration` |

`datetime` 的参数是 **Go 的时间布局**，不是 `YYYY-MM-DD`：

```go
Birthday string `binding:"datetime=2006-01-02"`
Meeting  string `binding:"datetime=2006-01-02 15:04:05"`
```

---

## 网络与地址

| 规则 | 大白话 | 示例 |
| --- | --- | --- |
| `ip` | IPv4 或 IPv6 | `ip` |
| `ipv4` | IPv4 地址 | `ipv4` |
| `ipv6` | IPv6 地址 | `ipv6` |
| `cidr` | IPv4/IPv6 网段 | `cidr` |
| `cidrv4` | IPv4 网段 | `cidrv4` |
| `cidrv6` | IPv6 网段 | `cidrv6` |
| `ip_addr` | 可解析的 IP 地址（可带端口） | `ip_addr` |
| `ip4_addr` | 可解析的 IPv4 地址（可带端口） | `ip4_addr` |
| `ip6_addr` | 可解析的 IPv6 地址（可带端口） | `ip6_addr` |
| `tcp_addr` | 可解析的 TCP 地址 | `tcp_addr` |
| `tcp4_addr` | 可解析的 IPv4 TCP 地址 | `tcp4_addr` |
| `tcp6_addr` | 可解析的 IPv6 TCP 地址 | `tcp6_addr` |
| `udp_addr` | 可解析的 UDP 地址 | `udp_addr` |
| `udp4_addr` | 可解析的 IPv4 UDP 地址 | `udp4_addr` |
| `udp6_addr` | 可解析的 IPv6 UDP 地址 | `udp6_addr` |
| `unix_addr` | 可解析的 Unix socket 地址 | `unix_addr` |
| `uds_exists` | Unix domain socket 路径存在 | `uds_exists` |
| `mac` | MAC 地址 | `mac` |
| `hostname` | RFC 952 主机名 | `hostname` |
| `hostname_rfc1123` | RFC 1123 主机名 | `hostname_rfc1123` |
| `hostname_label` | 单个主机名标签 | `hostname_label` |
| `fqdn` | 完整域名 | `fqdn` |

带 `_addr` 的规则可能会做地址解析，不只是正则匹配。

---

## 标识、业务与编码

| 规则 | 大白话 | 示例 |
| --- | --- | --- |
| `uuid` | 任意版本 UUID | `uuid` |
| `uuid3` / `uuid4` / `uuid5` / `uuid7` | 指定版本 UUID | `uuid4` |
| `uuid_rfc4122` | RFC 4122 UUID | `uuid_rfc4122` |
| `uuid3_rfc4122` / `uuid4_rfc4122` / `uuid5_rfc4122` | 指定版本且符合 RFC 4122 | `uuid4_rfc4122` |
| `ulid` | ULID | `ulid` |
| `isbn` / `isbn10` / `isbn13` | ISBN 书号 | `isbn13` |
| `issn` | ISSN 期刊号 | `issn` |
| `ssn` | 美国社会安全号 | `ssn` |
| `ein` | 美国雇主识别号 | `ein` |
| `credit_card` | 信用卡号 | `credit_card` |
| `luhn_checksum` | 通过 Luhn 校验 | `luhn_checksum` |
| `cve` | CVE 编号 | `cve` |
| `latitude` | 纬度 | `latitude` |
| `longitude` | 经度 | `longitude` |
| `mongodb` | MongoDB ObjectId | `mongodb` |
| `mongodb_connection_string` | MongoDB 连接串 | `mongodb_connection_string` |
| `cron` | Cron 表达式 | `cron` |
| `spicedb` | SpiceDB 关系字符串 | `spicedb` |
| `semver` | 语义化版本 | `semver` |
| `dns_rfc1035_label` | RFC 1035 DNS 标签 | `dns_rfc1035_label` |
| `bcp47_language_tag` | BCP 47 语言标签 | `bcp47_language_tag` |
| `bcp47_strict_language_tag` | 严格 BCP 47 标签 | `bcp47_strict_language_tag` |
| `iso3166_1_alpha2` | 两位国家代码，如 `CN` | `iso3166_1_alpha2` |
| `iso3166_1_alpha3` | 三位国家代码，如 `CHN` | `iso3166_1_alpha3` |
| `iso3166_1_alpha_numeric` | 数字国家代码 | `iso3166_1_alpha_numeric` |
| `iso3166_2` | 国家行政区划代码 | `iso3166_2` |
| `iso4217` | 货币代码，如 `CNY` | `iso4217` |
| `iso4217_numeric` | 数字货币代码 | `iso4217_numeric` |
| `postcode_iso3166_alpha2` | 指定国家的邮编 | `postcode_iso3166_alpha2=CN` |
| `postcode_iso3166_alpha2_field` | 用另一个字段的国家代码校验邮编 | `postcode_iso3166_alpha2_field=Country` |
| `bic` / `bic_iso_9362_2014` | 银行 BIC/SWIFT 代码 | `bic` |
| `eth_addr` | 以太坊地址 | `eth_addr` |
| `eth_addr_checksum` | 带校验和的以太坊地址 | `eth_addr_checksum` |
| `btc_addr` | 比特币地址 | `btc_addr` |
| `btc_addr_bech32` | Bech32 比特币地址 | `btc_addr_bech32` |
| `validateFn` | 字段是合法的 validator 函数 | `validateFn` |

哈希规则：

| 规则 | 大白话 |
| --- | --- |
| `md4` | 32 位十六进制 MD4 |
| `md5` | 32 位十六进制 MD5 |
| `sha256` | 64 位十六进制 SHA-256 |
| `sha384` | 96 位十六进制 SHA-384 |
| `sha512` | 128 位十六进制 SHA-512 |
| `ripemd128` | 32 位十六进制 RIPEMD-128 |
| `ripemd160` | 40 位十六进制 RIPEMD-160 |
| `tiger128` | 32 位十六进制 Tiger-128 |
| `tiger160` | 40 位十六进制 Tiger-160 |
| `tiger192` | 48 位十六进制 Tiger-192 |

---

## 颜色

| 规则 | 大白话 | 示例 |
| --- | --- | --- |
| `hexcolor` | `#fff` / `#ffffff` | `hexcolor` |
| `rgb` | `rgb(0,0,0)` | `rgb` |
| `rgba` | `rgba(0,0,0,0.5)` | `rgba` |
| `hsl` | `hsl(0,0%,0%)` | `hsl` |
| `hsla` | `hsla(0,0%,0%,0.5)` | `hsla` |
| `cmyk` | `cmyk(0,0,0,0)` | `cmyk` |
| `iscolor` | 上面任意一种颜色格式 | `iscolor` |

---

## 别名

| 别名 | 等价于 |
| --- | --- |
| `iscolor` | `hexcolor\|rgb\|rgba\|hsl\|hsla\|cmyk` |
| `country_code` | `iso3166_1_alpha2\|iso3166_1_alpha3\|iso3166_1_alpha_numeric` |
| `eu_country_code` | 欧盟国家代码的三种格式 |

---

## 最容易混的几组

### `required` vs `omitempty`

```go
Name  string `binding:"required"`        // 必须有值
Email string `binding:"omitempty,email"` // 可以不传；传了就必须是邮箱
```

### `min/max` vs `len`

- `len=6`：必须正好 6；
- `min=6`：至少 6；
- `max=6`：最多 6。

对字符串数的是字符数，对切片数的是元素个数。

### `number` vs `numeric`

- `number`：只能是 `0-9`；
- `numeric`：可以带正负号和小数，如 `-12.5`。

### `eq` vs `eqfield`

- `eq=1`：和固定值比；
- `eqfield=Password`：和另一个字段比。

### `email` vs `url`

- `email`：实用型邮箱校验，不追求完整 RFC；
- `url`：要求带协议；
- `http_url` / `https_url`：进一步限定协议。

---

## 完整示例

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type Address struct {
	City string `json:"city" binding:"required"`
	Zip  string `json:"zip" binding:"required,len=6,number"`
}

type RegisterReq struct {
	Username string   `json:"username" binding:"required,min=3,max=16,alphanum"`
	Password string   `json:"password" binding:"required,min=8,max=32"`
	Confirm  string   `json:"confirm" binding:"required,eqfield=Password"`
	Email    string   `json:"email" binding:"omitempty,email"`
	Age      int      `json:"age" binding:"gte=0,lte=130"`
	Role     string   `json:"role" binding:"oneof=user owner admin"`
	Tags     []string `json:"tags" binding:"max=5,dive,min=1,max=12"`
	Address  Address  `json:"address"`
}

func main() {
	r := gin.Default()

	r.POST("/api/v1/register", func(c *gin.Context) {
		var req RegisterReq
		if err := c.ShouldBindJSON(&req); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{
				"code":    400,
				"message": err.Error(),
			})
			return
		}
		c.JSON(http.StatusCreated, gin.H{"data": req})
	})

	r.Run(":8080")
}
```

请求示例：

```json
{
  "username": "tom2026",
  "password": "12345678",
  "confirm": "12345678",
  "email": "tom@example.com",
  "age": 18,
  "role": "user",
  "tags": ["new", "vip"],
  "address": {
    "city": "Shanghai",
    "zip": "200000"
  }
}
```

### 条件必填示例

```go
type PayReq struct {
	Method string `json:"method" binding:"required,oneof=card alipay wechat"`
	CardNo string `json:"card_no" binding:"required_if=Method card,omitempty,min=12,max=19,number"`
}
```

- `Method=card`：`CardNo` 必填；
- `Method=alipay`：`CardNo` 可以不传；
- 传了 `CardNo`：必须是 12–19 位数字。

### map 校验示例

```go
type ScoreReq struct {
	Scores map[string]int `json:"scores" binding:"required,min=1,dive,keys,min=2,max=10,endkeys,gte=0,lte=100"`
}
```

- `keys` 到 `endkeys` 之间：校验 key；
- `endkeys` 之后：校验 value。

---

## 常见坑

1. **tag 写成 `validate`**：Gin 只认 `binding`，规则完全不生效。
2. **把 `required` 用在 `bool` 上**：`false` 会被判为空。
3. **把 `required` 用在 `int` 上**：`0` 会被判为空。
4. **`omitempty` 把 `0` 跳过了**：想校验 `0` 是否合法，用指针 `*int`。
5. **`min/max` 以为是字节数**：字符串按字符数算。
6. **`dive` 位置写错**：`dive` 前面针对切片，后面针对元素。
7. **`oneof` 值里有空格没加引号**：会被拆成多个参数。
8. **跨字段写了 `json` 名**：`eqfield` 要写 Go 字段名。
9. **`datetime` 写了 `YYYY-MM-DD`**：要写 Go 布局 `2006-01-02`。
10. **`required` 用在结构体上**：需要 `WithRequiredStructEnabled()`。

---

## 官方资料

- go-playground/validator：<https://github.com/go-playground/validator>
- Gin 绑定与校验：<https://gin-gonic.com/docs/examples/binding-and-validation/>
- 内置规则源码：`baked_in.go` 的 `bakedInValidators`
- 中文错误翻译：`validator/translations/zh`
