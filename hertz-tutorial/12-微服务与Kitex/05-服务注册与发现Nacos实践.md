# 服务注册与发现 Nacos 实践

> 一句话定位：把 mall-lite 的 Kitex 服务接上 Nacos——从本地单机到生产集群，注册、发现、权重、健康检查与多环境隔离一次跑通。

## 本篇目标

- 用 Docker Compose 起单机 Nacos，并了解集群部署要点
- 制定命名空间、分组、服务名的规范
- 完成 Kitex 服务端注册与客户端发现的完整代码
- 理解权重、元数据、临时实例与心跳机制
- 掌握 Hertz 侧的注册与负载均衡接入
- 设计上下线流程与多环境隔离，并与 Consul / Etcd 对照

## 前置知识

- 已读 [05-治理与扩展/01-服务注册与发现总览](../05-治理与扩展/01-服务注册与发现总览.md)
- 已读 [04-Kitex客户端与Hertz集成](04-Kitex客户端与Hertz集成.md)
- 会读 Docker Compose 文件，了解服务发现的“注册—发现—负载均衡”闭环

## 一、本地起一个 Nacos

```yaml
services:
  nacos:
    image: nacos/nacos-server:v2.3.0
    environment:
      - MODE=standalone
      - NACOS_AUTH_ENABLE=false
    ports:
      - "8848:8848"   # HTTP / 控制台
      - "9848:9848"   # 客户端 gRPC
      - "9849:9849"   # 服务端 gRPC
```

```bash
docker compose up -d
```

```powershell
docker compose up -d
```

控制台 <http://localhost:8848/nacos>，默认账号 `nacos/nacos`。生产集群至少 3 节点，外接 MySQL，前置 Nginx/SLB，关键环境变量：

| 变量 | 作用 |
| --- | --- |
| `MODE=cluster` | 集群模式 |
| `NACOS_SERVERS` | 集群节点列表 |
| `SPRING_DATASOURCE_PLATFORM=mysql` | 使用 MySQL 持久化 |
| `MYSQL_SERVICE_HOST/PORT/DB_NAME` | 数据库连接 |
| `NACOS_AUTH_ENABLE=true` + token | 开启鉴权 |

**9848/9849 不映射**是最常见的部署事故：控制台能打开，SDK 却一直连不上。

## 二、命名规范

| 维度 | 规范 | 示例 |
| --- | --- | --- |
| 命名空间 | 按环境隔离 | `mall-lite-dev` / `mall-lite-test` / `mall-lite-prod` |
| 分组 | 按业务域 | `mall-lite` |
| 服务名 | 小写中划线，与 IDL 服务语义一致 | `user-service` |
| 元数据 | 版本、可用区、负责人 | `version=v1.2.0`、`zone=sh-a` |

命名空间在 Nacos 里对应一个 **ID**，不是显示名。代码里填的是 ID，填错就会“注册成功但查不到”。

## 三、Kitex 服务端注册

```go
import (
    "github.com/cloudwego/kitex/pkg/klog"
    "github.com/cloudwego/kitex/pkg/registry"
    "github.com/cloudwego/kitex/pkg/utils"
    "github.com/cloudwego/kitex/server"
    nacosregistry "github.com/kitex-contrib/registry-nacos/registry"
    naming_client "github.com/nacos-group/nacos-sdk-go/clients/naming_client"
)

func newRegistry(namingCli naming_client.INamingClient) (registry.Registry, error) {
    return nacosregistry.NewNacosRegistry(namingCli, nacosregistry.WithGroup("mall-lite"))
}

func main() {
    addr := utils.NewNetAddr("tcp", ":8888")
    r, err := newRegistry(namingCli)
    if err != nil {
        klog.Fatal(err)
    }
    svr := userservice.NewServer(
        new(UserServiceImpl),
        server.WithServiceAddr(addr),
        server.WithRegistry(r),
        server.WithRegistryInfo(&registry.Info{
            ServiceName: "user-service",
            Addr:        addr,
            Weight:      10,
            Tags:        map[string]string{"version": "v1.2.0", "zone": "sh-a"},
        }),
    )
    if err := svr.Run(); err != nil {
        klog.Fatal(err)
    }
}
```

`registry.Info` 的 `ServiceName`、`Addr` 通常由 Kitex 自动补全，但显式写出更容易排查。注册发生在 `Run()` 启动监听之后，退出时由框架注销；需要控制顺序时用 `server.RegisterShutdownHook`（见 [03-Kitex服务端最佳实践](03-Kitex服务端最佳实践.md)）。

## 四、Kitex 客户端发现

```go
import (
    "github.com/cloudwego/kitex/client"
    nacosresolver "github.com/kitex-contrib/registry-nacos/resolver"
)

res, err := nacosresolver.NewNacosResolver(namingCli,
    nacosresolver.WithGroup("mall-lite"),
)
if err != nil {
    klog.Fatal(err)
}
cli, err := userservice.NewClient("user-service",
    client.WithResolver(res),
    client.WithRPCTimeout(300*time.Millisecond),
)
```

默认负载均衡是加权随机，实例的 `Weight` 越大被选中概率越高。需要一致性哈希（缓存类服务）或自定义标签路由时，用 `client.WithLoadBalancer` 注入：

```go
// 示例示意：具体构造函数以 kitex/pkg/loadbalance 文档为准
client.WithLoadBalancer(loadbalance.NewConsistentHashBalancer())
```

注意：客户端的服务名、Group、Namespace 必须与服务端完全一致，三者任一不同都会表现为“没有可用实例”。

## 五、权重、元数据与健康检查

| 能力 | 说明 | 用途 |
| --- | --- | --- |
| Weight | 加权随机/加权轮询的权重 | 灰度、异构机型 |
| Tags | 任意键值元数据 | 版本路由、可用区优先 |
| 临时实例 | 客户端心跳维持，默认 5s | 大多数业务服务 |

健康检查时间线（默认参数，可通过客户端配置调整）：

```text
心跳正常        → 健康
15s 未收到心跳  → 标记不健康（仍在实例列表）
30s 未收到心跳  → 从列表摘除
```

这意味着**进程被 kill 后，调用方最多还会拿到约 15s 的坏实例**。因此客户端必须有超时 + 重试（仅幂等）+ 熔断兜底，不能把可用性完全押在注册中心上。

## 六、Hertz 侧接入

Hertz 网关调用内部服务同样走注册中心：

```go
import (
    "time"

    "github.com/cloudwego/hertz/pkg/app/client"
    "github.com/cloudwego/hertz/pkg/app/client/discovery"
    "github.com/cloudwego/hertz/pkg/app/client/loadbalance"
    hertznacos "github.com/hertz-contrib/registry/nacos"
)

res, err := hertznacos.NewDefaultNacosResolver()
if err != nil {
    panic(err)
}
cli, _ := client.NewClient()
cli.Use(discovery.Discovery(res,
    discovery.WithLoadBalanceOptions(
        loadbalance.NewWeightedBalancer(),
        loadbalance.Options{RefreshInterval: 5 * time.Second, ExpireInterval: 30 * time.Second},
    ),
))
```

社区还提供 `github.com/hertz-contrib/loadbalance/round_robin` 等算法包，构造函数以仓库 README 为准（示例示意）。网关一般调用的是 Kitex 服务，更推荐在网关里直接用 Kitex 客户端；Hertz 客户端适合调用非 Kitex 的 HTTP 服务。

## 七、上下线流程

上线：

```text
注册实例 → 等待发现刷新（约 5s）→ 预热（可选）→ 放开流量
```

下线（K8s 场景）：

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
```

```text
SIGTERM → preStop 等待 → 注册中心摘除 → 再等在途请求 → 退出
```

只摘除不等待，或先停进程后摘除，都会让调用方在几秒内持续打到已停实例。发布脚本里显式 `Deregister` + sleep 是最稳的做法。

## 八、多环境隔离

- 每个环境独立命名空间，禁止跨环境复用
- 生产使用独立 Nacos 集群，至少 3 节点 + MySQL
- CI 注入 `NACOS_ADDR`、`NACOS_NAMESPACE`，镜像里不写死
- 灰度环境可用独立 Group，避免与正式流量混在一起

## 九、与 Consul / Etcd 对照

| 维度 | Nacos | Consul | Etcd |
| --- | --- | --- | --- |
| 注册模型 | 临时/持久实例 | Agent 注册 | Lease + key |
| 健康检查 | 心跳 | HTTP/TCP/TTL/gRPC | 租约到期 |
| 配置中心 | 内置 | 需 KV 自建 | KV 自建 |
| Kitex 支持 | registry-nacos | registry-consul | registry-etcd |

迁移注意：服务名与分组语义不同（Consul 用 Tag、Etcd 用 key 前缀），不要直接复制地址；切换前先双注册、灰度读新注册中心，再下线旧通道。

## 常见坑

1. **命名空间填显示名而非 ID**：注册在 `public`，消费端查 `mall-lite-dev`，实例在线却查不到。
2. **注册 IP 用 `127.0.0.1`**：本机自测通过，跨节点全失败；容器内要用 Pod IP 或宿主机可达地址。
3. **只映射 8848 没映射 9848/9849**：控制台正常，SDK 报连接超时。
4. **服务名 / Group 不一致**：客户端报“没有可用实例”，但控制台明明有。
5. **下线不等待**：摘除后立刻杀进程，发布期间出现 502/503 尖刺。

## 小结

- Nacos 用命名空间隔离环境、Group 隔离业务域、服务名标识服务
- Kitex 侧 = `NewNacosRegistry` + `WithRegistryInfo`，客户端 = `NewNacosResolver` + `WithResolver`
- 权重用于灰度，Tags 用于路由，临时实例靠心跳存活
- 上线等发现、下线先摘除，Nacos 的秒级延迟要用超时与熔断兜底

## 练习

1. 用 Compose 起 Nacos，注册两个 user-service 实例，把权重设为 9:1 观察流量比例。
2. 把客户端 Group 改成 `other`，复现“无可用实例”并修复。
3. 停掉一个实例，记录从停止到调用方不再命中的时间。

## 官方文档

- Nacos 快速开始：<https://nacos.io/zh-cn/docs/quick-start.html>
- Kitex 服务注册：<https://www.cloudwego.io/zh/docs/kitex/tutorials/basic-feature/service-discovery/>
- registry-nacos：<https://github.com/kitex-contrib/registry-nacos>
- hertz-contrib/registry：<https://github.com/hertz-contrib/registry>
