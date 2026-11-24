# 服务注册与发现：etcd 3.5 与 Consul 1.20 实战

> 一句话定位：Kratos 只依赖 `registry.Registrar` / `registry.Discovery` 两个接口，etcd 与 Consul 的差异全部收敛在 `pkg/discovery` 和配置里。

## 本篇目标

- 用 Docker Compose 起 etcd 3.5 与 Consul 1.20，说清 Key+Lease 与 Agent+Check 两种模型
- 用 `clientv3.New` + `etcdreg.New`、`consulapi.NewClient` + `consulreg.New` 注册，并以 `kratos.Registrar(reg)` 接入
- 客户端用 `grpc.DialInsecure` + `WithDiscovery` 单例建连，实例扩缩容自动生效，并会处理僵尸实例与注册中心故障

## 前置知识

- [05-治理与扩展/01-服务注册与发现总览](../05-治理与扩展/01-服务注册与发现总览.md)
- [04-Kratos客户端与Gin集成](04-Kratos客户端与Gin集成.md)

## 一、先对齐两种注册模型

| 维度 | etcd 3.5 | Consul 1.20 |
| --- | --- | --- |
| 注册单元 | Key + Lease，如 `/mall-lite/user-service/<实例ID>` | Agent 上的服务实例 + 健康检查 |
| 失效判定 | 租约 TTL（默认 15s）续约失败即删 Key | 检查失败标 critical，再等 deregister 时间摘除 |
| 变更通知 | Watch 前缀，事件驱动 | 阻塞查询（long polling）+ 本地缓存 |
| 额外能力 | 纯 KV，配置中心可复用同一集群 | 内置 DNS、UI、多数据中心、ACL |

`registry.ServiceInstance` 只有 `ID / Name / Version / Endpoints / Metadata`，没有 weight；灰度权重、zone、协议版本都塞进 `Metadata`，由消费端 `WithNodeFilter` 解释。

## 二、本地起 etcd 与 Consul

```yaml
# deploy/docker/docker-compose.registry.yml
services:
    etcd:
        image: bitnami/etcd:3.5
        environment:
            - ALLOW_NONE_AUTHENTICATION=yes
            - ETCD_ADVERTISE_CLIENT_URLS=http://etcd:2379
        ports:
            - "2379:2379"
    consul:
        image: hashicorp/consul:1.20
        command: ["agent", "-dev", "-client=0.0.0.0", "-ui"]
        ports:
            - "8500:8500"
```

```bash
docker compose -f deploy/docker/docker-compose.registry.yml up -d
docker exec -it etcd etcdctl endpoint health
```

```powershell
docker compose -f deploy/docker/docker-compose.registry.yml up -d
(Invoke-WebRequest http://127.0.0.1:8500/v1/status/leader).Content
```

## 三、服务端注册：构造函数只返回 `*Registry`

连接错误在 `clientv3.New` / `consulapi.NewClient` 阶段处理，contrib 注册器只返回 `*Registry`，不返回 error。

```go
// pkg/discovery/registry.go
package discovery

import (
    "time"
    consulreg "github.com/go-kratos/kratos/contrib/registry/consul/v2"
    etcdreg "github.com/go-kratos/kratos/contrib/registry/etcd/v2"
    "github.com/go-kratos/kratos/v2/registry"
    consulapi "github.com/hashicorp/consul/api"
    clientv3 "go.etcd.io/etcd/client/v3"
    "mall-lite/pkg/config"
)

// Registry 同时实现 Registrar 与 Discovery，服务端与网关共用一份连接。
type Registry struct {
    registry.Registrar
    registry.Discovery
    Close func()
}

func NewRegistry(cfg *config.Config) (*Registry, error) {
    if cfg.Registry.Type == "consul" {
        cli, err := consulapi.NewClient(&consulapi.Config{
            Address: cfg.Registry.Consul.Address, Token: cfg.Registry.Consul.Token,
        })
        if err != nil {
            return nil, err
        }
        reg := consulreg.New(cli,
            consulreg.WithHealthCheck(true),
            consulreg.WithHealthCheckInterval(10),            // 秒
            consulreg.WithDeregisterCriticalServiceAfter(60), // 秒
        )
        return &Registry{Registrar: reg, Discovery: reg, Close: func() {}}, nil
    }
    cli, err := clientv3.New(clientv3.Config{Endpoints: cfg.Registry.Etcd.Endpoints,
        DialTimeout: 3 * time.Second, Username: cfg.Registry.Etcd.Username,
        Password: cfg.Registry.Etcd.Password})
    if err != nil {
        return nil, err
    }
    reg := etcdreg.New(cli,
        etcdreg.Namespace(cfg.Registry.Etcd.Namespace), // 默认 /microservices
        etcdreg.RegisterTTL(15*time.Second),
    )
    return &Registry{Registrar: reg, Discovery: reg, Close: func() { _ = cli.Close() }}, nil
}
```

Consul 的健康检查是 TTL 类型：contrib 按 10s 心跳 `UpdateTTL`，不需要服务额外暴露 HTTP 端口；`WithDeregisterCriticalServiceAfter(60)` 决定 critical 后再保留多久摘除。

## 四、接入 `kratos.Registrar`

```go
// app/user-service/cmd/main.go（节选：注册相关）
reg, err := discovery.NewRegistry(cfg)
if err != nil {
    panic(err) // 注册中心不可用就别启动
}
defer reg.Close()

app := kratos.New(
    kratos.Name("user-service"),
    kratos.Metadata(map[string]string{"env": cfg.App.Env, "zone": cfg.Registry.Zone}),
    kratos.Logger(logger), kratos.Server(grpcSrv), kratos.Registrar(reg.Registrar),
)
if err := app.Run(); err != nil {
    panic(err)
}
```

`app.Run()` 启动 server 后写入实例，`app.Stop()` 先停 server 再 `Deregister`。Endpoint 由 `grpc.Server.Endpoint()` 从监听地址推导，`0.0.0.0` 会替换成本机可达 IP；若用 `kratos.Endpoint(...)` 手工拼地址，绝不能拼 `0.0.0.0`。

## 五、客户端发现与实例变更

```go
// app/gateway/internal/client/user.go
func NewUserClient(cfg *config.Config, reg *discovery.Registry) (userv1.UserServiceClient, func(), error) {
    conn, err := kgrpc.DialInsecure(context.Background(),
        kgrpc.WithEndpoint("discovery:///user-service"),
        kgrpc.WithDiscovery(reg.Discovery),
        kgrpc.WithTimeout(800*time.Millisecond),
        kgrpc.WithMiddleware(recovery.Recovery(), tracing.Client()),
    )
    if err != nil {
        return nil, func() {}, err
    }
    return userv1.NewUserServiceClient(conn), func() { _ = conn.Close() }, nil
}
```

协议头固定 `discovery`，后面才是服务名；写成 `etcd://user-service` 会落到 direct resolver，直接报找不到 resolver。客户端必须启动时建一次并注入 handler，每请求建连会重建 Watch 订阅。

- etcd：resolver Watch 前缀 `/mall-lite/user-service`，上线 PUT、下线 DELETE，事件驱动推新列表。
- Consul：每秒按 `index` 做阻塞查询，index 变化才拉全量，只取 passing 实例。
- gRPC 增删 subconn，**在途请求不受影响**；按元数据过滤用 `kgrpc.WithNodeFilter(...)`，同区无实例要退化为全量。

## 六、僵尸实例、网络分区与兜底

`kill -9` 不会执行 `Deregister`：etcd 要等 15s 租约过期，Consul 要等检查失败加 deregister 时间，调用方仍可能拿到坏实例。超时、仅对幂等接口重试、熔断三件套不能省（见 [05-治理与扩展/04-Sentinel限流与熔断](../05-治理与扩展/04-Sentinel限流与熔断.md)）。etcd 少数派失去 quorum 会续约失败，实例被摘除但进程还活着；Consul 分区一侧的 agent 仍响应本地查询，变更传不到另一侧。**注册中心只回答「谁在哪」，不回答「能不能服务」**。兜底：启动期不可用直接失败交给 K8s 重启，或读 `registry.fallback_endpoints` 静态地址先建连；运行期不可用靠客户端重连与最后一次地址列表顶着，扩缩容感知暂停。静态列表只能救急，要跟发布更新并配健康检查。

## 七、ACL / TLS 与 CLI 排查

- etcd：ACL 用 `clientv3.Config{Username, Password}`；TLS 用 `TLS: &tls.Config{RootCAs: pool, MinVersion: tls.VersionTLS12}`，证书由 Secret 挂载。
- Consul：`Scheme: "https"` + `Token` + `TLSConfig{CAFile: ...}`，token 只从环境变量注入。
- 认证失败常报 permission denied 而非超时；本地 compose 关认证只是省事，共享环境必须开。

```bash
docker exec -it etcd etcdctl get --prefix /mall-lite/user-service/ -w json
docker exec -it etcd etcdctl endpoint status --cluster -w table
docker exec -it consul consul catalog services
docker exec -it consul consul health service user-service
```

读不到键先看命名空间：Kratos etcd 的 key 是 `<namespace>/<服务名>/<实例ID>`，默认 `/microservices`，mall-lite 统一配 `/mall-lite`；两端不一致的表现就是「注册成功但列表为空」。

## 八、etcd 还是 Consul

- 已有 etcd / K8s 运维经验、配置中心也想复用：选 etcd，Watch 事件驱动、延迟低、职责单一。
- 多数据中心、混合部署、要内置 DNS 与 UI：选 Consul，健康检查、ACL、多 DC 更完整；纯 K8s 小规模可只用 Service + `round_robin`。
- mall-lite 默认 etcd：注册与配置中心共用一个集群，运维面最小。

## 完整示例

```yaml
# configs/user-service.yaml（注册相关部分）
registry:
    type: etcd
    etcd:
        endpoints: ["127.0.0.1:2379"]
        namespace: /mall-lite
    fallback_endpoints: ["10.0.0.11:9001"]
```

切 Consul 只改 `MALL_REGISTRY_TYPE=consul`（PowerShell 用 `$env:MALL_REGISTRY_TYPE = "consul"`），服务名与端口不动。

## 常见坑

1. **把 `0.0.0.0` 注册出去**：同机自测正常、跨节点全超时；注册地址必须是调用方可达的 Pod IP。
2. **两端 namespace / 集群不一致**：注册成功但列表为空；先用 `etcdctl get --prefix` 对齐前缀。
3. **每请求 `DialInsecure`**：Watch 与连接泄漏、fd 耗尽；客户端必须单例。
4. **以为摘除是实时的**：`kill -9` 后仍有 15~60s 窗口，超时与熔断不能省。
5. **Consul 健康检查配错类型**：自己注册 HTTP 检查但服务没实现，实例永远 critical；用 contrib 的 TTL 心跳。

## 小结

- Kratos 只认 `Registrar` / `Discovery`，换注册中心只改 `pkg/discovery` 与配置
- `etcdreg.New` / `consulreg.New` 只返回 `*Registry`，连接错误在 client 构造阶段处理
- 注册的关键是「注册出去的地址从调用方是否可达」；etcd 靠租约、Consul 靠健康检查，摘除都有延迟，超时与兜底必须有

## 练习

1. 只改 `MALL_REGISTRY_TYPE`，把 user-service 从 etcd 切到 Consul，用 CLI 对比注册键与健康状态。
2. 起两个实例并给其中一个加 `zone=sh-b`，用 `WithNodeFilter` 过滤观察流量分布，再 `kill -9` 记录摘除延迟。

## 官方文档

- etcd v3.5：<https://etcd.io/docs/v3.5/>
- Consul 服务注册与健康检查：<https://developer.hashicorp.com/consul/docs/services/usage/register-services-checks>
- Kratos 服务注册：<https://go-kratos.dev/docs/component/registry>
- Kratos contrib registry：<https://github.com/go-kratos/kratos/tree/main/contrib/registry>
