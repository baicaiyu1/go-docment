# TLS 与 HTTPS

> 一句话定位：Gin 的 TLS 就是 Go 标准库 `crypto/tls`；正确做法是自建 `http.Server` 配 `TLSConfig`（`MinVersion` 至少 1.2、`NextProtos` 含 `h2`），生产环境通常把 TLS 终止交给 Ingress / APISIX。

## 本篇目标

- 用 mkcert 或 openssl 生成带 SAN 的证书，理解 `server.crt` 与 `ca.crt` 的分工。
- 用 `http.Server` + `tls.Config` 启动 HTTPS，正确设置 `MinVersion` 与 HTTP/2。
- 配置 mTLS 双向认证，服务端与客户端都能验证证书。
- 用 `GetCertificate` + `atomic.Pointer` 实现证书热更新。
- 明确 Ingress / APISIX 卸载 TLS 的边界与常见坑（证书链、域名、时钟）。

## 前置知识

- [优雅退出](07-优雅退出.md)（`http.Server` 生命周期）
- [Kubernetes部署与健康检查](../08-部署与运维/03-Kubernetes部署与健康检查.md)

## 一、证书生成：mkcert 与 openssl

开发环境优先 mkcert：它把本地 CA 装进系统信任库，浏览器与 `curl` 都不会告警。

```bash
mkcert -install
mkcert -cert-file certs/server.crt -key-file certs/server.key localhost 127.0.0.1 ::1
```

没有 mkcert 时用 openssl 自签，关键是 `subjectAltName`（只写 CN 已不被 Go 接受）：

```bash
mkdir -p certs
openssl req -x509 -newkey rsa:2048 -nodes -days 825 -sha256 \
  -keyout certs/server.key -out certs/server.crt \
  -subj "/C=CN/O=mall-lite/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1" \
  -addext "keyUsage=digitalSignature,keyEncipherment" \
  -addext "extendedKeyUsage=serverAuth"
```

Windows 也可用 `New-SelfSignedCertificate -DnsName localhost`，但导出的是 PFX，Go 需转 PEM 且默认不被信任，仅适合本机调试。

## 二、http.Server 配置 TLS

不用 `r.RunTLS`，因为它无法设置 `MinVersion`、超时与 HTTP/2 细节；直接组装 `http.Server`：

```go
// app/gateway/cmd/main.go（片段，完整版见文末完整示例）
srv.TLSConfig = &tls.Config{
    Certificates: []tls.Certificate{cert},
    MinVersion:   tls.VersionTLS12,          // 生产至少 1.2，内部可上 1.3
    NextProtos:   []string{"h2", "http/1.1"}, // 声明 HTTP/2 优先
}
// 证书已在 TLSConfig 中，两个空字符串表示不再从文件读取
log.Fatal(srv.ListenAndServeTLS("", ""))
```

- 显式写 `h2` 更清晰（Go 也会自动为 HTTPS 配置 HTTP/2）；`curl -v` 看到 `ALPN: server accepted h2` 才算协商成功。
- 明文 h2c 需要 `golang.org/x/net/http2/h2c`：`h2c.NewHandler(r, &http2.Server{})`，只建议内网使用。

## 三、mTLS 双向认证

服务间调用可要求客户端也出示证书，服务端只需 `ClientAuth` + `ClientCAs`：

```go
// app/gateway/cmd/main.go（片段）
caPEM, err := os.ReadFile("certs/ca.crt")
if err != nil {
    log.Fatal(err)
}
pool := x509.NewCertPool()
if !pool.AppendCertsFromPEM(caPEM) {
    log.Fatal("invalid ca certificate")
}

srv.TLSConfig.ClientAuth = tls.RequireAndVerifyClientCert
srv.TLSConfig.ClientCAs = pool
```

客户端携带自己的证书，并信任服务端 CA，`ServerName` 必须与证书 SAN 一致：

```go
// app/gateway/internal/client/tls.go
clientCert, err := tls.LoadX509KeyPair("certs/client.crt", "certs/client.key")
if err != nil {
    log.Fatal(err)
}
httpClient := &http.Client{Transport: &http.Transport{TLSClientConfig: &tls.Config{
    Certificates: []tls.Certificate{clientCert},
    RootCAs:      pool,
    ServerName:   "localhost",
    MinVersion:   tls.VersionTLS12,
}}}
resp, err := httpClient.Get("https://localhost:8443/ping")
```

`RequireAndVerifyClientCert` 会校验证书链与有效期；只 `RequireAnyClientCert` 不校验 CA，等于没有安全收益。

## 四、证书热更新

`tls.Config` 被服务端使用后不能再改字段，正确做法是 `GetCertificate` 回调 + 原子指针，重载只替换指针：

```go
// pkg/tlsutil/manager.go
type Manager struct {
    current          atomic.Pointer[tls.Certificate]
    certFile, keyFile string
}

func (m *Manager) Load() error {
    cert, err := tls.LoadX509KeyPair(m.certFile, m.keyFile)
    if err != nil {
        return err // 保留旧证书，避免热更新失败导致全站不可用
    }
    m.current.Store(&cert)
    return nil
}

func (m *Manager) GetCertificate(*tls.ClientHelloInfo) (*tls.Certificate, error) {
    if cert := m.current.Load(); cert != nil {
        return cert, nil
    }
    return nil, errors.New("certificate not loaded")
}
// TLSConfig 里设置 GetCertificate: mgr.GetCertificate，Certificates 留空
// 启动时 mgr.Load()；收到 SIGHUP 或 fsnotify 事件后再次 Load()
```

热更新只影响**新建立的连接**，已建立的 TLS 会话继续用旧证书；证书有效期不足 30 天要告警。

## 五、生产：把 TLS 卸载到 Ingress / APISIX

| 方案 | 优点 | 代价 |
| --- | --- | --- |
| APISIX / Ingress 终止 TLS | 证书、套件、OCSP 集中管理，后端简单 | 内网明文需 NetworkPolicy 隔离 |
| Gin 直接 TLS | 链路端到端加密，部署简单 | 每服务都要续期、配套件、做限速 |

主流做法是 APISIX 3.11 或 K8s Ingress 终止 TLS，上游指向 Gin 网关的 Service；内部服务间按需 mTLS（如订单调库存）。无论哪种，**证书私钥只从 Secret 挂载，不进镜像、不进仓库**。

## 完整示例

`app/gateway/cmd/main.go`：

```go
package main

import (
    "crypto/tls"
    "log"
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.New()
    r.Use(gin.Logger(), gin.Recovery())
    r.GET("/ping", func(c *gin.Context) { c.String(http.StatusOK, "pong over tls") })

    cert, err := tls.LoadX509KeyPair("certs/server.crt", "certs/server.key")
    if err != nil {
        log.Fatalf("load key pair: %v", err)
    }
    srv := &http.Server{
        Addr:              ":8443",
        Handler:           r,
        ReadHeaderTimeout: 5 * time.Second,
        TLSConfig: &tls.Config{
            Certificates: []tls.Certificate{cert},
            MinVersion:   tls.VersionTLS12,
            NextProtos:   []string{"h2", "http/1.1"},
        },
    }
    log.Fatal(srv.ListenAndServeTLS("", ""))
}
```

验证：mkcert 证书已被系统信任可去掉 `--cacert`；自签证书需带 `--cacert`，`-k` 只能验证连通性。

```powershell
go run ./app/gateway/cmd
curl.exe -v --cacert certs\server.crt https://localhost:8443/ping
curl.exe -v --http2 --cacert certs\server.crt https://localhost:8443/ping
```

```bash
go run ./app/gateway/cmd
curl -v --cacert certs/server.crt https://localhost:8443/ping
curl -v --http2 --cacert certs/server.crt https://localhost:8443/ping
```

## 常见坑

1. **证书只有 CN 没有 SAN**：Go 1.15+ 报 `x509: certificate relies on legacy Common Name field`；必须配 `subjectAltName`。
2. **证书链不完整**：服务端只发叶子证书，客户端报 `unknown authority`；`server.crt` 应为 fullchain，顺序是叶子 → 中间证书。
3. **域名不匹配**：证书 SAN 必须包含实际访问的域名与 IP；`curl 127.0.0.1` 与 `localhost` 是不同校验目标。
4. **服务器时钟偏差**：报 `certificate is not yet valid` 或 `expired`；容器与节点都要开 NTP。
5. **改了证书文件但没生效**：`LoadX509KeyPair` 只在启动时读一次；必须重启或用 `GetCertificate` 热更新。
6. **`InsecureSkipVerify: true` 进了生产**：等于关闭校验；只能出现在本地调试分支。
7. **mTLS 只开 `RequireAnyClientCert`**：任何自签证书都能通过；要 `RequireAndVerifyClientCert` + `ClientCAs`。

## 小结

- Gin 的 TLS 入口就是标准库 `crypto/tls`；用 `http.Server` + `TLSConfig` 才能控制 `MinVersion` 与超时。
- 证书必须带 SAN；开发用 mkcert，自签用 openssl，私钥只来自 Secret。
- HTTP/2 靠 ALPN 的 `h2`；mTLS 靠 `ClientAuth` + `ClientCAs`。
- 热更新用 `GetCertificate` + `atomic.Pointer`，只对新连接生效。
- 生产优先在 APISIX / Ingress 终止 TLS，内部按需 mTLS。

## 练习

1. 用 mkcert 生成 `localhost` 证书，验证 `curl --http2` 协商到 h2。
2. 开启 `RequireAndVerifyClientCert`，分别用带证书和不带证书的客户端请求，对比报错。
3. 实现基于 `SIGHUP` 的证书热更新，用 `openssl s_client` 查看新连接的证书序列号。

## 官方文档

- Go `crypto/tls`：<https://pkg.go.dev/crypto/tls>
- `http.Server.ListenAndServeTLS`：<https://pkg.go.dev/net/http#Server.ListenAndServeTLS>
- mkcert：<https://github.com/FiloSottile/mkcert>
- APISIX SSL：<https://apisix.apache.org/zh/docs/apisix/ssl/>
- K8s Ingress TLS：<https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/#tls>
