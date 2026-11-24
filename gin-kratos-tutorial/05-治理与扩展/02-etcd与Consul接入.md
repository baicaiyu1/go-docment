# etcd 与 Consul 接入

> 一句话定位：用 Docker 起 etcd 3.5 与 Consul 1.20，通过 Kratos contrib 的 `etcdreg` / `consulreg` 完成注册与发现，再把健康检查、优雅上下线与灰度元数据补齐——两条链路只差一个工厂函数。

## 本篇目标

- 用 Docker Compose 起 etcd 3.5 与 Consul 1.20，并用 contrib 的 `etcdreg` / `consulreg` 写出服务端注册的完整代码。
- 用 Viper 配置驱动「换注册中心不改业务代码」的工厂，客户端统一走 `discovery:///`。
- 掌握两种健康检查的差异，用 `Metadata` 做灰度并落地优雅上下线，知道 ACL/TLS 与网络分区如何兜底。

## 前置知识

- 已读 [服务注册与发现总览](01-服务注册与发现总览.md) 与 [配置管理 Viper](03-配置管理Viper.md)，理解 `Registrar` / `Discovery` / `ServiceInstance`，注册中心配置同样由 Viper 加载。
- 会用 Docker，能读懂 Kratos `grpc.NewServer` 与 `kratos.New` 的装配。

## 一、本地环境

```yaml
# deploy/docker/docker-compose.registry.yaml
services:
  etcd: {image: bitnami/etcd:3.5, container_name: mall-etcd, environment: [ALLOW_NONE_AUTHENTICATION=yes], ports: ["2379:2379"]}
  consul: {image: hashicorp/consul:1.20, container_name: mall-consul, command: ["agent", "-dev", "-client=0.0.0.0", "-ui"], ports: ["8500:8500"]}
```

```bash
docker compose -f deploy/docker/docker-compose.registry.yaml up -d
docker exec -it mall-etcd etcdctl endpoint health
curl -s http://127.0.0.1:8500/v1/status/leader   # PowerShell 用 curl.exe，其余命令一致
```

Consul UI 在 <http://localhost:8500> 能看实例与检查状态；etcd 用 `etcdctl` 看 key。`-dev` 是内存存储，重启即丢，别用于共享环境。

## 二、配置：一个 type 切换两套实现

```yaml
# configs/user-service.yaml
registry:
  type: etcd                    # etcd | consul
  namespace: mall-lite          # etcd key 前缀
  etcd: {endpoints: ["127.0.0.1:2379"], dial_timeout: 5s, register_ttl: 15s}
  consul: {address: "127.0.0.1:8500", health_check: true, check_interval: 5s, tags: ["grpc", "v1"]}
```

`register_ttl` 决定「进程被杀后多久从列表消失」，建议 10~20 秒；`check_interval` 是 Consul 探测周期，直接决定故障发现时间。

## 三、etcd 注册：client + 租约

```go
// app/user-service/cmd/main.go
func newApp(cfg *conf.Config, logger log.Logger, svc *server.UserService) (*kratos.App, error) {
    cli, err := clientv3.New(clientv3.Config{
        Endpoints:   cfg.Registry.Etcd.Endpoints, // 来自 Viper，禁止硬编码
        DialTimeout: cfg.Registry.Etcd.DialTimeout,
    })
    if err != nil {
        return nil, err
    }
    // etcdreg.New 只返回 *Registry，不返回 error；连接问题在 clientv3.New 阶段暴露
    reg := etcdreg.New(cli,
        etcdreg.Namespace(cfg.Registry.Namespace),
        etcdreg.RegisterTTL(cfg.Registry.Etcd.RegisterTTL),
    )
    gs := grpc.NewServer(grpc.Address(cfg.Server.Grpc.Addr), grpc.Timeout(cfg.Server.Grpc.Timeout))
    v1.RegisterUserServiceServer(gs, svc)
    return kratos.New(
        kratos.Name("user-service"),
        kratos.Metadata(map[string]string{"version": "v1.0.0", "zone": "sh-a", "weight": "10"}),
        kratos.Logger(logger),
        kratos.Server(gs),
        kratos.Registrar(reg),
    ), nil
}
```

三个细节决定成败：

- **Namespace 是 key 前缀**。etcd 里的 key 形如 `/mall-lite/user-service/10.0.0.12:9001`，多环境共集群时必须带环境前缀（`mall-lite-dev`），否则测试环境会捞到生产实例。
- **Endpoint 必须可达**。Kratos 从监听地址推导 `Endpoints`，绑定 `0.0.0.0` 时只能尽力猜本机 IP；容器多网卡、NAT 环境务必在启动日志打一行实际注册地址。
- **续约由 contrib 负责**。注册时拿 lease 并后台 `KeepAlive`；进程崩溃 → 续约停止 → TTL 到期 → key 删除，这就是 etcd 的健康检查。

## 四、Consul 注册：显式健康检查

```go
// app/product-service/cmd/main.go（片段）
cli, err := consulapi.NewClient(&consulapi.Config{Address: cfg.Registry.Consul.Address})
if err != nil {
    return nil, err
}
// consulreg.New 同样只返回 *Registry；健康检查间隔的单位是秒
reg := consulreg.New(cli,
    consulreg.WithHealthCheck(cfg.Registry.Consul.HealthCheck),
    consulreg.WithHealthCheckInterval(int(cfg.Registry.Consul.CheckInterval.Seconds())),
    consulreg.WithDeregisterCriticalServiceAfter(60),
    consulreg.WithTags(cfg.Registry.Consul.Tags), // ["grpc","v1"]
)
```

`WithHealthCheck(true)` 让 Consul 对注册端点做 TCP 探测，语义是「端口能连上」，**不是**「业务就绪」。要探 `/readyz` 得用 `WithServiceCheck(&consulapi.AgentServiceCheck{HTTP: "http://.../readyz"})`，或把就绪判断交给 K8s readinessProbe。检查失败后实例是 `critical`，要等 `WithDeregisterCriticalServiceAfter(60)` 到点才摘除，期间列表里的 critical 实例容易被误判成「服务还活着」。

## 五、客户端：一个工厂，两种实现

```go
// pkg/discovery/registry.go
func New(cfg *config.Config) (registry.Registrar, registry.Discovery, error) {
    switch cfg.Registry.Type {
    case "etcd":
        cli, err := clientv3.New(clientv3.Config{
            Endpoints: cfg.Registry.Etcd.Endpoints, DialTimeout: cfg.Registry.Etcd.DialTimeout,
        })
        if err != nil {
            return nil, nil, err
        }
        reg := etcdreg.New(cli, etcdreg.Namespace(cfg.Registry.Namespace)); return reg, reg, nil // 同时实现两个接口
    case "consul":
        cli, err := consulapi.NewClient(&consulapi.Config{Address: cfg.Registry.Consul.Address})
        if err != nil {
            return nil, nil, err
        }
        reg := consulreg.New(cli, consulreg.WithHealthCheck(cfg.Registry.Consul.HealthCheck)); return reg, reg, nil
    default:
        return nil, nil, fmt.Errorf("unsupported registry type %q", cfg.Registry.Type)
    }
}
```

网关只用 `registry.Discovery`，把 `discovery.New(cfg)` 拿到的 `disc` 传给 `client.NewUserClient(ctx, disc)`，启动时建一次连接注入 handler。

## 六、上下线与灰度

`kratos.App.Run()` 内部监听 `SIGINT`/`SIGTERM`：先 `Registrar.Deregister`，再关停 Server，四步顺序见 [总览](01-服务注册与发现总览.md) 第五节；`kill -9` 会跳过摘除与等待，只能等 TTL 到期。

灰度用 `Metadata` + 客户端过滤，而不是改注册中心：v2 实例注册 `{"version": "v2", "zone": "sh-a", "weight": "1"}`，网关给 `WithNodeFilter` 传 `func(_ context.Context, ins *registry.ServiceInstance) bool { return ins.Metadata["version"] == "v2" }` 即可（完整 Dial 写法见 [服务注册与发现总览](01-服务注册与发现总览.md) 第三节）。

负载均衡由 selector 决定，与注册中心无关：Kratos 默认 `wrr`（权重缺省为 1，可按实例 `Metadata` 调权），也可进程级 `selector.SetGlobalSelector(p2c.NewBuilder())` 换成 p2c 或 random。`weight` 只是元数据，灰度规则本身仍要发网关版本。

## 七、ACL 与 TLS

`-dev` 的 Consul 和 `ALLOW_NONE_AUTHENTICATION=yes` 的 etcd 只适合本机。共享环境至少要补上认证与传输加密：

```go
// etcd：账号密码 + CA；ACL 需给该账号 /<namespace>/ 前缀 readwrite
cli, _ := clientv3.New(clientv3.Config{
    Endpoints: cfg.Registry.Etcd.Endpoints, Username: cfg.Registry.Etcd.Username,
    Password: cfg.Registry.Etcd.Password,
    TLS:      &tls.Config{RootCAs: pool, MinVersion: tls.VersionTLS12}, // pool 由 CA 文件构造
})
reg := etcdreg.New(cli, etcdreg.Namespace(cfg.Registry.Namespace))

// Consul：ACL Token + HTTPS；Token 需绑定 service:write / node:read
cli2, _ := consulapi.NewClient(&consulapi.Config{
    Address: cfg.Registry.Consul.Address, Scheme: "https", Token: cfg.Registry.Consul.Token,
    TLSConfig: consulapi.TLSConfig{CAFile: cfg.Registry.Consul.CAFile},
})
```

- 证书、Token、密码一律从环境变量或 Vault 注入，禁止写进提交仓库的 yaml；不要开 `InsecureSkipVerify`。
- etcd ACL 默认拒绝：权限漏配时注册不报错但 key 写不进去，表现为「服务正常启动、调用方永远发现不了」，启动日志要打印一次注册结果。

## 八、对照与排障

| 操作 | etcd | Consul |
| --- | --- | --- |
| 看全部实例 | `etcdctl get --prefix "/mall-lite/"` | `consul catalog services` |
| 看单个服务 | `etcdctl get --prefix "/mall-lite/user-service/"` | `consul catalog nodes -service=user-service` |
| 故障发现 | 租约到期（TTL） | 检查失败转 critical |
| 手动摘除 | `etcdctl del <key>` | `consul services deregister -id=<id>` |

## 完整示例

注册后停进程，分别观察两种注册中心的行为：

```go
// cmd/probe/main.go
func main() {
    cfg, _, err := config.Load("configs/user-service.yaml") // 第三个返回值是 *viper.Viper
    if err != nil {
        panic(err)
    }
    _, disc, _ := discovery.New(cfg) // 示例省略错误分支
    instances, err := disc.GetService(context.Background(), "user-service")
    if err != nil {
        panic(err)
    }
    for _, ins := range instances {
        fmt.Printf("id=%s version=%s endpoints=%v\n", ins.ID, ins.Version, ins.Endpoints)
    }
}
```

先 `go run ./app/user-service`，再 `go run ./cmd/probe`，应看到 `grpc://<ip>:9001`；`Ctrl+C` 后观察实例消失的方式。

## 常见坑

1. **etcd 里出现两个环境的同名服务**：Namespace 没带环境前缀。用 `mall-lite-dev` / `mall-lite-prod` 隔离并在启动日志打印。
2. **注册地址是容器内 IP**：跨节点调用超时。检查实际注册的 `Endpoints`，必要时用环境变量指定对外可达地址。
3. **Consul 检查一直 critical**：探测端口写错（把 metrics 端口当 gRPC 端口）或安全组未放通。先用 `consul members` 与本地连通性验证。
4. **`GetService` 返回空却没人报错**：客户端没注入 `WithDiscovery`，仍在按 DNS 解析，报 `no such host`。
5. **网络分区 / 实例异常下线**：TTL 过短时 Leader 切换或分区期间实例被成批误摘，恢复后流量瞬间打回；`kill -9` 又不会走 `Deregister`，只能等 TTL 或 critical 超时。TTL 10~20 秒起步，调用方必须配超时、重试、熔断，并保留本地缓存兜底。
6. **改完配置没重启**：`registry.type` 不支持热切换，变更必须重启，别指望 WatchConfig 生效。

## 小结

- `etcdreg.New(*clientv3.Client, ...)` 与 `consulreg.New(*consulapi.Client, ...)` 返回的对象同时实现 `Registrar` 与 `Discovery`，两者都不返回 error。
- etcd 靠租约 TTL 判活，Consul 靠可配置探测；后者的「就绪」语义仍要自己补。
- 用 `discovery.New(cfg)` 按 `registry.type` 切换，业务代码只依赖接口；灰度靠 `Metadata` + `WithNodeFilter`，权重交给 `wrr` / `p2c` 选择器。
- 发布稳定性 = `Deregister` + 发现刷新等待 + 优雅停机。

## 练习

1. 用 Compose 起两套注册中心，把 `registry.type` 从 `etcd` 切到 `consul`，确认业务代码零改动。
2. 把 `register_ttl` 设为 5s 和 30s，`kill -9` 后计时实例从列表消失的时间。
3. 给 `pkg/discovery` 加 `Watch` 订阅，打印实例增减事件，验证发布期间摘除的感知延迟。

## 官方文档

- Kratos registry 组件：<https://go-kratos.dev/docs/component/registry/>
- Kratos contrib 仓库：<https://github.com/go-kratos/kratos/tree/main/contrib/registry>
- etcd 3.5 API（Lease）：<https://etcd.io/docs/v3.5/learning/api/>
- Consul 服务与健康检查：<https://developer.hashicorp.com/consul/docs/services/usage/checks>
