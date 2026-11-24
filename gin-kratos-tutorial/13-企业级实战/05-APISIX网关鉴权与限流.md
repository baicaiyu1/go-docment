# 05-APISIX 网关鉴权与限流

> 一句话定位：APISIX 3.11 在边缘做「验签初筛 + 限流 + 路由」，Gin 网关做「二次验签 + 身份下发 + 聚合」，Kratos 只认 metadata 里的身份——越权和雪崩都止步于入口。

## 本篇目标

- 用 Docker Compose 起 APISIX 3.11 + etcd，讲清 standalone 与 Admin API 两种管理方式的取舍
- 配好 `jwt-auth`、`limit-req`、`limit-count`、`cors`、`prometheus`、`traffic-split`，知道每个插件该挂在哪一层
- 在 Gin 里二次验签，用 gRPC metadata 把用户身份传给 Kratos
- 用 curl 验证 401 / 429 / 200，并能定位插件顺序、consumer 绑定、etcd 数据面三类故障

## 前置知识

- [04-订单服务与下单主链路](04-订单服务与下单主链路.md)：下单接口与 `Idempotency-Key`
- [12-微服务与Kratos/04-Kratos客户端与Gin集成](../12-微服务与Kratos/04-Kratos客户端与Gin集成.md)：gRPC 客户端与服务发现

## 一、三层职责边界

调用链固定为 `Client → APISIX:9080 → Gin:8080 → gRPC:9001~9003`。APISIX 只管边缘：TLS 卸载、JWT 验签初筛、IP/凭证级限流、CORS、入口指标；Gin 做二次验签、黑名单、身份注入、聚合与 SSE；Kratos 只认 metadata 里的身份，请求体里的 `user_id` 一律忽略并覆盖。

APISIX 的 upstream **只指向 Gin**：直连 gRPC 端口既没有 HTTP 语义，也绕过了聚合与限流，等于把内网接口挂到公网。两层都验签不是重复劳动——APISIX 挡掉大部分垃圾流量，Gin 保证「即使有人直连 8080，身份依然可信」。

## 二、Docker Compose：APISIX 3.11 + etcd

```yaml
# deploy/docker/docker-compose.apisix.yaml
services:
    etcd:
        image: bitnami/etcd:3.5
        environment: { ALLOW_NONE_AUTHENTICATION: "yes", ETCD_ADVERTISE_CLIENT_URLS: http://etcd:2379 }
    apisix:
        image: apache/apisix:3.11.0-debian
        depends_on: [etcd]
        ports: ["9080:9080", "9180:9180", "9091:9091"]
        volumes:
            - ../apisix/config.yaml:/usr/local/apisix/conf/config.yaml:ro
            - ../apisix/apisix.yaml:/usr/local/apisix/conf/apisix.yaml:ro
        environment: { APISIX_STAND_ALONE: "true" }
    gateway: { build: ../../app/gateway, ports: ["8080:8080"] }
```

`9080/9180/9091` 分别是数据面、Admin API、Prometheus 指标端口；`APISIX_STAND_ALONE=true` 让 APISIX 忽略 Admin API 与 etcd，去掉它就回到 etcd 模式（对比见下一节）。镜像版本为教程基线，以官方最新稳定版为准。

## 三、两种管理方式

standalone 适合本地与 GitOps：路由全在 `apisix.yaml`，改完重启容器生效，不存在「配置到底存哪」的疑问。

```yaml
# deploy/apisix/config.yaml（standalone）
apisix:
    node_listen: 9080
    enable_admin: false
    config_provider: yaml
```

生产用 Admin API + etcd：配置写入 etcd，多个数据面节点秒级同步，配合 CI 做版本化变更。注意 Admin 端口绝不能暴露公网。

```bash
# 生产由 CI 从 Secret 读取 key 后执行，禁止把 key 写进脚本
curl -X PUT http://127.0.0.1:9180/apisix/admin/consumers/mall-web \
    -H "X-API-KEY: $APISIX_ADMIN_KEY" -H "Content-Type: application/json" \
    -d '{"username":"mall-web","plugins":{"jwt-auth":{"key":"mall-web","secret":"from-secret","algorithm":"HS256"}}}'
```

Admin key 写在 `deployment.admin.admin_key`，由 K8s Secret 挂载整份 `config.yaml` 注入；9180 只对 CI 跳板或 VPN 网段开放。

## 四、standalone 路由与消费者

```yaml
# deploy/apisix/apisix.yaml
upstreams:
    - id: gateway
      type: roundrobin
      nodes:
          "gateway:8080": 1
      retries: 2
      timeout: { connect: 1, send: 2, read: 5 }
      checks:
          active: { type: http, http_path: /healthz, healthy: { interval: 2, successes: 2 }, unhealthy: { interval: 2, http_failures: 3, tcp_failures: 3 } }
routes:
    - id: auth
      uri: /api/v1/auth/*
      upstream_id: gateway
      plugins:
          limit-req: { rate: 5, burst: 3, key: remote_addr, key_type: var, rejected_code: 429 }
          cors: {}
    - id: api
      uri: /api/v1/*
      upstream_id: gateway
      plugins:
          jwt-auth: {}
          limit-count: { count: 600, time_window: 60, key_type: var, key: consumer_name, policy: local, rejected_code: 429 }
          proxy-rewrite:
              headers: { set: { X-User-Id: "$jwt_claims_user_id" } }
          prometheus: {}
consumers:
    - username: mall-web
      plugins:
          jwt-auth: { key: mall-web, secret: "dev-only-change-me", algorithm: HS256, exp: 7200 }
#END
```

末尾 `#END` 是 standalone 的结束标记，缺了整份配置被忽略且不报明显错误，是「改了没反应」的头号原因。`limit-count` 用 `consumer_name` 而不是客户端 header：后者可伪造，且在验签插件执行前根本不存在。`proxy-rewrite` 的 `set` 会覆盖客户端自带的同名 header，但没挂 `jwt-auth` 的路由仍可能收到伪造的 `X-User-Id`，所以 Gin 必须自己解 token。

## 五、jwt-auth：签发在 Gin，验签在两层

APISIX 的 `jwt-auth` 只验签、不签发。登录接口由 Gin 用同一个 secret 签发 HS256 token，客户端以 `Authorization: Bearer <token>` 访问。

```bash
TOKEN=$(curl -s -X POST http://127.0.0.1:8080/api/v1/auth/login -H "Content-Type: application/json" \
    -d '{"username":"demo","password":"demo"}' | jq -r .data.access_token)
```

```go
// app/gateway/internal/middleware/auth.go
func Auth(secret []byte) gin.HandlerFunc {
    return func(c *gin.Context) {
        raw := strings.TrimPrefix(c.GetHeader("Authorization"), "Bearer ")
        claims, err := jwtx.Parse(raw, secret) // HS256，校验 exp/iss/aud
        if err != nil || claims.Type != "access" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"code": 40101, "message": "登录状态无效"})
            return
        }
        if h := c.GetHeader("X-User-Id"); h != "" && h != strconv.FormatInt(claims.UserID, 10) {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"code": 40102, "message": "身份信息不一致"})
            return
        }
        c.Set("uid", claims.UserID) // 请求级数据，后续 c.Get("uid") 取
    }
}
```

Gin 引擎最外层已调用 `c.Next()`，中间件不写它，后续 handler 也会在其返回后执行；失败分支只要 `AbortWithStatusJSON` + `return`。退出登录时把 `jti` 写进 Redis 直到过期，就能做到「立即下线」。

身份下行用 metadata：`metadata.AppendToOutgoingContext(ctx, "x-user-id", strconv.FormatInt(uid, 10), "x-request-id", requestid.FromContext(ctx))`。key 必须全小写（gRPC 规范，大写直接拒收），`ctx` 原样透传；服务端只从 metadata 取身份，取不到就返回 `Unauthorized`，请求体里的 `user_id` 忽略并覆盖。

## 六、限流：limit-req 与 limit-count

| 插件 | 算法 | 适用场景 | 关键参数 |
| --- | --- | --- | --- |
| `limit-req` | 漏桶 | 登录、验证码等防突发 | `rate` / `burst`，按 `remote_addr` |
| `limit-count` | 固定窗口 | 按消费者/租户配额 | `count` / `time_window`，`key=consumer_name` |

`limit-req` 的 `rejected_code` 默认是 503，不改会被客户端当故障重试，反而放大压力。`policy: local` 是单节点计数，多副本时实际额度是「配额 × 节点数」；要精确配额必须换 `policy: redis` 并配 `redis_host`。两者返回 429 时都应带 `Retry-After` 语义，客户端按退避重试。

## 七、插件执行顺序与优先级

APISIX 按插件 `priority` 从大到小执行，同一阶段内顺序错了，语义就变了。

| 插件 | priority | 阶段 | 注意事项 |
| --- | --- | --- | --- |
| `opentelemetry` | 12009 | rewrite | 最先生成 trace，后续插件都能打标 |
| `cors` | 4000 | rewrite | 预检请求要在鉴权前返回 |
| `ip-restriction` | 3000 | access | 放最前挡黑名单，省掉验签开销 |
| `jwt-auth` | 2510 | rewrite | 产出 `consumer_name` / `jwt_claims_*` |
| `proxy-rewrite` | 1008 | rewrite | 依赖 jwt 变量，必须排在其后 |
| `limit-req` / `limit-count` | 1001 / 1002 | access | 用 `consumer_name` 时必须晚于 jwt-auth |
| `traffic-split` | 966 | rewrite | 决定最终 upstream |
| `prometheus` | 500 | log | 只做埋点，放最后 |

把 `limit-count` 的 key 设成 `consumer_name` 却排在 `jwt-auth` 前面，计数会全部落到空值上；把 `ip-restriction` 放到最后，攻击流量已经打完了。具体数值以官方 plugin priority 表为准，3.x 小版本会微调。

## 八、traffic-split 金丝雀与上游健康检查

```yaml
# deploy/apisix/apisix.yaml（routes 片段，gateway-canary 为 v2 上游）
    - id: api-canary
      uri: /api/v1/*
      plugins:
          jwt-auth: {}
          traffic-split:
              rules:
                  - match:
                        - vars: [["arg_debug", "==", "v2"]]
                    weighted_upstreams:
                        - upstream_id: gateway-canary
                          weight: 100
                  - weighted_upstreams:
                        - upstream_id: gateway
                          weight: 90
                        - upstream_id: gateway-canary
                          weight: 10
```

按权重分流是随机桶，同一用户可能来回跳版本；要求稳定分桶就改用 cookie/header 匹配（`vars` 只能用请求期已存在的变量，别引用 `$jwt_claims_user_id`）。上游 `checks.active` 连续失败 3 次会摘节点，`retries: 2` 只对幂等方法安全，别对 `POST /orders` 开重试。权重不会因为 canary 全挂自动归零，必须由指标门禁把比例打回 0。

## 完整示例：验证 401 / 429 / 200

```bash
curl -i http://127.0.0.1:9080/api/v1/users/me                                    # 401，APISIX 直接返回
curl -i -H "Authorization: Bearer $TOKEN" http://127.0.0.1:9080/api/v1/users/me  # 200，响应带 request_id
for i in $(seq 1 700); do curl -s -o /dev/null -w "%{http_code}\n" \
    -H "Authorization: Bearer $TOKEN" http://127.0.0.1:9080/api/v1/users/me; done | sort | uniq -c
```

```powershell
curl.exe -i http://127.0.0.1:9080/api/v1/users/me
curl.exe -i -H "Authorization: Bearer $env:TOKEN" http://127.0.0.1:9080/api/v1/users/me
```

401 一定来自 APISIX（响应体是 APISIX 的错误结构）；如果 401 里出现业务码，说明请求已进 Gin，是二次校验拒绝的。429 同样要分清来源：APISIX 会带限流头，Sentinel 返回的是业务错误体。

## 常见坑

1. **`apisix.yaml` 少了 `#END`**：整份配置被忽略、全部 404 且日志无明显报错；提交前 grep 一下。
2. **consumer 绑定漏一半**：`jwt-auth` 必须同时挂在 route 与 consumer 上，只挂 route 会一直 401。
3. **`limit-req` 用默认 `rejected_code`**：返回 503 被当成故障重试；显式写 429。
4. **standalone 下改 Admin API**：数据面根本不读 etcd，改了没反应；确认 `config_provider` 与部署模式一致。
5. **etcd 挂掉后全站 404**：Admin API 模式下游节点只缓存很短时间，etcd 不可用会丢路由；etcd 要三节点且独立于业务集群。
6. **超时预算不一致**：APISIX 默认 `read` 60s，Gin `server.http.timeout` 3s，客户端等到 60s 才收到 504；把网关超时压到略大于下游预算。
7. **信任客户端 `X-User-Id`**：伪造 header 即越权；身份只从 token 解出，并与 header 比对；灰度权重也不要指望上游健康检查自动归零。

## 小结

- APISIX 只管边缘，upstream 永远指向 Gin；Gin 必须自己验签，Kratos 只认 metadata
- standalone 靠 `apisix.yaml` + `#END`，生产靠 Admin API + etcd，密钥全部来自 Secret
- 插件顺序即语义：`jwt-auth` 先于依赖 `consumer_name` 的限流，`ip-restriction` 最先；`traffic-split` 做金丝雀，重试只对幂等流量放开

## 练习

1. 给 `/api/v1/auth/login` 单独加 5 次/分钟/IP 的 `limit-count`，用 700 次请求验证 429。
2. 把 `limit-count` 的 `policy` 改成 `redis`，起两个 APISIX 节点验证配额是否共享。
3. 用 `traffic-split` 的 header 匹配做按用户灰度，并写一条门禁：5xx > 1% 时自动把权重调回 0。

## 官方文档

- APISIX standalone 部署：https://apisix.apache.org/docs/apisix/deployment-modes/
- APISIX jwt-auth：https://apisix.apache.org/docs/apisix/plugins/jwt-auth/
- APISIX limit-count：https://apisix.apache.org/docs/apisix/plugins/limit-count/
- APISIX traffic-split：https://apisix.apache.org/docs/apisix/plugins/traffic-split/
