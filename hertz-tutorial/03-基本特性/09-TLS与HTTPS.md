# 09-TLS与HTTPS
> 让 Hertz 服务安全地跑在 HTTPS 上：从自签证书、HTTP/2 协商到 mTLS 与证书热更新，一次讲透。

## 本篇目标

- 用 `server.WithTLS` 正确启动一个 HTTPS 服务，并理解它为何会自动切换网络库。
- 掌握 openssl 与 PowerShell 两种自签证书生成方式，能生成带 SAN 的可用证书。
- 理解 ALPN 与 h2c 的区别，知道 Hertz 开启 HTTP/2 的前置条件。
- 能配置 mTLS，让服务端强制校验客户端证书。
- 掌握证书热更新的思路与生产环境中的 TLS 落地方式。

## 前置知识

- 了解 `server.New` / `server.Default` / `h.Spin()` 的基本用法（见 02-Hertz入门/02）。
- 了解 TLS 握手、X.509 证书、CA、SAN（Subject Alternative Name）等概念。
- 会使用 Go 标准库 `crypto/tls`，知道 `tls.Config` 是 TLS 行为的核心配置。

## 1. Hertz 的 TLS 支持边界

Hertz 的 TLS 能力直接建立在 Go 标准库 `crypto/tls` 之上，入口是 `server.WithTLS(cfg *tls.Config)`。有一个关键限制必须先记住：

- **Netpoll 网络库不支持 TLS**，TLS 场景必须使用标准库网络库 `standard.NewTransporter`。
- `server.WithTLS` 在未显式设置 Transporter 时，会自动把网络库切到 `standard`。
- 如果后续又用 `server.WithTransport(netpoll.NewTransporter)` 覆盖，TLS 会失败——**选项顺序和组合要小心**。

```go
package main

import (
	"context"
	"crypto/tls"
	"log"

	"github.com/cloudwego/hertz/pkg/app"
	"github.com/cloudwego/hertz/pkg/app/server"
	"github.com/cloudwego/hertz/pkg/protocol/consts"
)

func main() {
	// 1. 加载证书链与私钥（PEM 格式）
	cert, err := tls.LoadX509KeyPair("certs/server.crt", "certs/server.key")
	if err != nil {
		log.Fatalf("load key pair: %v", err)
	}

	// 2. 组装 tls.Config
	cfg := &tls.Config{
		Certificates: []tls.Certificate{cert},
		MinVersion:   tls.VersionTLS12, // 生产建议至少 TLS 1.2
		NextProtos:   []string{"http/1.1"},
	}

	// 3. WithTLS 会自动选用 standard.NewTransporter
	h := server.Default(
		server.WithHostPorts(":8443"),
		server.WithTLS(cfg),
	)

	h.GET("/ping", func(ctx context.Context, c *app.RequestContext) {
		c.String(consts.StatusOK, "pong over tls")
	})
	h.Spin()
}
```

启动后访问 `https://localhost:8443/ping` 即可。注意 `WithTLS` 的注释明确写着：**一旦以 TLS 启动，就不会再接受非 TLS 请求**，HTTP 与 HTTPS 不能共用同一个 Hertz 实例的同一端口。

## 2. 自签证书：openssl 与 PowerShell

### 2.1 openssl（推荐，跨平台）

现代 Go 客户端校验证书时会检查 SAN，只写 CN 已经不够。先生成 CA，再签发带 SAN 的服务端证书：

```bash
# 1) 生成 CA 私钥与自签 CA 证书
openssl ecparam -genkey -name prime256v1 -out ca.key
openssl req -x509 -new -key ca.key -sha256 -days 3650 \
  -subj "/C=CN/O=HertzDemo/CN=Hertz Demo CA" -out ca.crt

# 2) 生成服务端私钥与 CSR
openssl ecparam -genkey -name prime256v1 -out server.key
openssl req -new -key server.key -subj "/CN=localhost" -out server.csr

# 3) SAN 扩展配置（关键）
cat > server.ext <<'EOF'
subjectAltName = DNS:localhost, IP:127.0.0.1
extendedKeyUsage = serverAuth
keyUsage = digitalSignature, keyEncipherment
EOF

# 4) 用 CA 签发服务端证书
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -days 825 -sha256 -extfile server.ext -out server.crt
```

`ca.crt` 交给客户端做 `RootCAs`，`server.crt` + `server.key` 给服务端。

### 2.2 PowerShell（Windows 开发环境）

```powershell
# 1) 在当前用户证书存储中创建自签证书（带 SAN）
# 仅给 localhost 生成 DNS SAN；如需 127.0.0.1 请用上面的 openssl 方式加 IP SAN
$cert = New-SelfSignedCertificate `
  -DnsName "localhost" `
  -CertStoreLocation "Cert:\CurrentUser\My" `
  -NotAfter (Get-Date).AddYears(1) `
  -KeyAlgorithm RSA -KeyLength 2048

# 2) 导出为 PFX（含私钥）
$pwd = ConvertTo-SecureString -String "changeit" -Force -AsPlainText
New-Item -ItemType Directory -Force .\certs | Out-Null
Export-PfxCertificate -Cert $cert -FilePath .\certs\server.pfx -Password $pwd

# 3) 用 openssl 转成 PEM（Go 的 LoadX509KeyPair 需要 PEM）
openssl pkcs12 -in .\certs\server.pfx -nodes -passin pass:changeit -out .\certs\server.pem

# 4) 拆分证书与私钥（也演示提取 CA）
openssl pkcs12 -in .\certs\server.pfx -nokeys -passin pass:changeit -out .\certs\server.crt
openssl pkcs12 -in .\certs\server.pfx -nocerts -nodes -passin pass:changeit -out .\certs\server.key
```

> PowerShell 生成的是 PFX（PKCS#12），Go 不能直接读取；必须转 PEM。`New-SelfSignedCertificate` 生成的证书默认不受系统信任，如果希望浏览器不告警，需要把它导入"受信任的根证书颁发机构"（仅限本机开发）。

## 3. ALPN、HTTP/2 与 h2c

### 3.1 ALPN 协商

ALPN 是 TLS 握手阶段协商应用层协议的扩展。Hertz 通过 `server.WithALPN(true)` 打开协商逻辑，配合 `tls.Config.NextProtos` 声明支持的协议：

```go
cfg := &tls.Config{
	Certificates: []tls.Certificate{cert},
	NextProtos:   []string{"h2", "http/1.1"}, // 优先 h2
}
h := server.Default(server.WithHostPorts(":8443"),
	server.WithTLS(cfg), server.WithALPN(true))
```

需要说明的是：Hertz 核心内置的是 **ALPN / h2c 的协议嗅探入口**，HTTP/2 的具体协议实现由扩展提供（如 `hertz-contrib/http2`）。如果没有加载 HTTP/2 协议实现，Hertz 会打印警告并回退到 HTTP/1.1。是否支持、如何注册请以对应扩展仓库和官方文档为准，不要默认"开了 ALPN 就是 HTTP/2"。

### 3.2 h2c（明文 HTTP/2）

h2c 不需要 TLS，通过 HTTP/1.1 Upgrade 或直接发送 HTTP/2 连接前言（client preface）协商。Hertz 用 `server.WithH2C(true)` 打开嗅探，例如 `server.New(server.WithHostPorts(":8080"), server.WithH2C(true))`。

验证时可用 `curl --http2-prior-knowledge http://localhost:8080/ping`。同样地，h2c 也需要 HTTP/2 协议实现已注册，否则会回退到 HTTP/1 并输出 `HTTP2 server is not loaded` 的警告。

## 4. mTLS：双向证书校验

mTLS 要求客户端也出示证书，适合服务间内部调用。服务端只需配置 `ClientAuth` 与 `ClientCAs`：

```go
caPEM, err := os.ReadFile("certs/ca.crt")
if err != nil {
	log.Fatal(err)
}
pool := x509.NewCertPool()
if !pool.AppendCertsFromPEM(caPEM) {
	log.Fatal("invalid ca certificate")
}

cfg := &tls.Config{
	Certificates: []tls.Certificate{cert},
	ClientAuth:   tls.RequireAndVerifyClientCert, // 强制并校验客户端证书
	ClientCAs:    pool,
	MinVersion:   tls.VersionTLS12,
}
```

客户端则需要携带自己的证书，并信任服务端 CA：

```go
clientCert, err := tls.LoadX509KeyPair("certs/client.crt", "certs/client.key")
if err != nil {
	log.Fatal(err)
}
tlsCfg := &tls.Config{
	Certificates: []tls.Certificate{clientCert},
	RootCAs:      pool,        // 复用上面的 CA 池
	ServerName:   "localhost", // 与证书 SAN 对应
}
c, err := client.NewClient(
	client.WithTLSConfig(tlsCfg),
	client.WithDialer(standard.NewDialer()), // TLS 必须用标准库拨号器
)
if err != nil {
	log.Fatal(err)
}
status, body, _ := c.Get(context.Background(), nil, "https://localhost:8443/ping")
```

## 5. 证书热更新思路

`tls.Config` 一旦被服务端使用，直接修改其字段是并发不安全的。正确做法是用 `GetCertificate` 回调，从原子变量里取当前证书：

```go
type certManager struct {
	current atomic.Pointer[tls.Certificate]
}

func (m *certManager) load(certFile, keyFile string) error {
	cert, err := tls.LoadX509KeyPair(certFile, keyFile)
	if err != nil {
		return err
	}
	m.current.Store(&cert)
	return nil
}

func (m *certManager) GetCertificate(*tls.ClientHelloInfo) (*tls.Certificate, error) {
	return m.current.Load(), nil
}

// 启动时 m.load(...) 一次；收到 SIGHUP 或 fsnotify 事件后重新 load 即可。
// 多域名场景可用 tls.Config.GetConfigForClient 按 SNI 返回不同配置。
```

热更新只影响**新建立的连接**，已建立的 TLS 连接仍使用旧证书。

## 6. 生产实践

- **TLS 终止位置**：绝大多数公司在负载均衡 / 网关（Nginx、ALB、Envoy、K8s Ingress）终止 TLS，后端 Hertz 走内网明文或 mTLS。这样证书轮换、密码套件策略集中管理。
- **证书轮换**：证书有效期越来越短（如 90 天），必须自动化续期与热加载；至少提前 30 天告警。
- **安全基线**：`MinVersion` 不低于 TLS 1.2；禁止 `InsecureSkipVerify: true` 进入生产；保持 keep-alive 并启用会话复用。

## 常见坑

1. **用了 netpoll 还开 TLS**：表现为握手失败或连接异常。TLS 必须用标准库网络库，且注意 `WithTransport` 不要覆盖 `WithTLS` 的自动选择。
2. **证书只有 CN 没有 SAN**：Go 1.15+ 校验证书时以 SAN 为准，`x509: certificate relies on legacy Common Name field` 就是这个问题，必须加 `subjectAltName`。
3. **客户端报 `not support tls`**：只设置了 `client.WithTLSConfig`，却忘了 `client.WithDialer(standard.NewDialer())`；netpoll 拨号器不支持 TLS。
4. **开了 ALPN 就以为支持 HTTP/2**：没有加载 HTTP/2 协议实现时会回退 HTTP/1.1，抓包或 `curl -v` 才能看出来。
5. **证书文件改了但服务没生效**：`tls.LoadX509KeyPair` 只在启动时读取一次，必须重启或实现 `GetCertificate` 热加载。

## 小结

- `server.WithTLS` 是 Hertz 唯一的 TLS 入口，本质是标准库 `tls.Config`；TLS 场景必须使用 standard 网络库。
- 自签证书要带 SAN；openssl 适合脚本化，PowerShell 适合 Windows 本地快速生成，但需要转 PEM。
- ALPN 与 h2c 是两条 HTTP/2 路径，Hertz 提供嗅探入口，协议实现需扩展支持，不能想当然。
- mTLS 由 `ClientAuth + ClientCAs` 完成；热更新用 `GetCertificate` + 原子变量，生产更推荐网关终止 TLS。

## 练习

1. 用 openssl 生成一套带 `DNS:localhost, IP:127.0.0.1` 的证书，启动 HTTPS 服务，并用 `curl --cacert ca.crt https://localhost:8443/ping` 验证成功。
2. 给服务端加上 `ClientAuth: tls.RequireAndVerifyClientCert`，分别用"不带客户端证书"和"带客户端证书"的 Hertz Client 请求，对比结果。
3. 实现一个基于 `SIGHUP` 的证书热更新示例：替换证书文件后发送信号，确认新连接使用新证书（可用 `openssl s_client` 查看序列号）。

## 官方文档

- TLS：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/tls/>
- 网络库（TLS 与网络库的关系）：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/network-lib/>
- Engine 选项（WithALPN / WithH2C）：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/engine/>
