# Consul、Nacos、Etcd 等接入
> 一句话定位：把 Consul 与 Nacos 的“注册 + 发现 + 负载均衡”完整跑通，并补齐健康检查、上下线与灰度。
## 本篇目标
- 用 Docker 起 Consul / Nacos / Etcd
- 写出服务端注册 + 客户端发现的完整代码
- 理解注册参数、健康检查与权重
- 掌握上下线与灰度/权重的落地方式
- 知道三类注册中心的关键差异
## 前置知识
- 已读 [01-服务注册与发现总览](01-服务注册与发现总览.md)
- 会用 Docker Compose，了解 Hertz `server` / `client`
## 一、本地环境
```yaml
services:
  consul:
    image: hashicorp/consul:1.17
    command: ["agent", "-dev", "-client=0.0.0.0"]
    ports: ["8500:8500", "8600:8600/udp"]
  nacos:
    image: nacos/nacos-server:v2.3.0
    environment: ["MODE=standalone", "NACOS_AUTH_ENABLE=false"]
    ports: ["8848:8848", "9848:9848"]
  etcd:
    image: bitnami/etcd:3.5
    environment: ["ALLOW_NONE_AUTHENTICATION=yes"]
    ports: ["2379:2379"]
```
```bash
docker compose up -d
```
| 组件 | 控制台 | 默认账号 |
| --- | --- | --- |
| Consul | <http://localhost:8500> | 无 |
| Nacos | <http://localhost:8848/nacos> | nacos/nacos |
| Etcd | 无 UI，可用 `etcdctl` | 无 |
## 二、Consul 完整接入
```bash
go get github.com/hertz-contrib/registry/consul
go get github.com/hashicorp/consul/api
```
### 2.1 服务端注册
```go
package main

import (
	"context"
	"log"

	"github.com/cloudwego/hertz/pkg/app"
	"github.com/cloudwego/hertz/pkg/app/server"
	"github.com/cloudwego/hertz/pkg/app/server/registry"
	"github.com/cloudwego/hertz/pkg/common/utils"
	"github.com/cloudwego/hertz/pkg/protocol/consts"
	consulapi "github.com/hashicorp/consul/api"
	"github.com/hertz-contrib/registry/consul"
)

func main() {
	cfg := consulapi.DefaultConfig()
	cfg.Address = "127.0.0.1:8500"
	consulClient, err := consulapi.NewClient(cfg)
	if err != nil {
		log.Fatal(err)
	}

	check := &consulapi.AgentServiceCheck{
		HTTP: "http://10.0.0.12:8888/healthz", Timeout: "3s",
		Interval: "5s", DeregisterCriticalServiceAfter: "1m",
	}
	r := consul.NewConsulRegister(consulClient,
		consul.WithCheck(check),
		consul.WithAdditionInfo(&consul.AdditionInfo{
			Tags: []string{"v1", "http"},
			Meta: map[string]string{"version": "v1.2.0", "zone": "sh-a"},
		}),
	)

	addr := "10.0.0.12:8888"
	h := server.Default(
		server.WithHostPorts(addr),
		server.WithRegistry(r, &registry.Info{
			ServiceName: "order-service",
			Addr:        utils.NewNetAddr("tcp", addr),
			Weight:      10,
		}),
	)
	h.GET("/healthz", func(ctx context.Context, c *app.RequestContext) {
		c.String(consts.StatusOK, "ok")
	})
	h.Spin()
}
```
### 2.2 客户端发现与负载均衡
```go
import "github.com/cloudwego/hertz/pkg/app/client/loadbalance"

cli, err := client.NewClient()
if err != nil {
	panic(err)
}

r := consul.NewConsulResolver(consulClient)
cli.Use(sd.Discovery(r, sd.WithLoadBalanceOptions(
	loadbalance.NewWeightedBalancer(),
	loadbalance.Options{RefreshInterval: 5 * time.Second, ExpireInterval: 30 * time.Second},
)))

// host 会被当作服务名交给 Resolver
status, body, err := cli.Get(context.Background(), nil, "http://order-service/ping")
```
## 三、Nacos 完整接入
```bash
go get github.com/hertz-contrib/registry/nacos
```
> 社区另有 `registry/nacos/v2`（nacos-sdk-go v2），导入路径与构造函数不同，不要混用，以仓库 README 为准。
### 3.1 最快跑通
```go
r, err := nacos.NewDefaultNacosRegistry()
if err != nil {
	log.Fatal(err)
}

h := server.Default(
	server.WithHostPorts("127.0.0.1:8888"),
	server.WithRegistry(r, &registry.Info{
		ServiceName: "hertz.test.demo",
		Addr:        utils.NewNetAddr("tcp", "127.0.0.1:8888"),
		Weight:      10,
	}),
)
```
默认实现读取环境变量：
| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `serverAddr` | `127.0.0.1` | Nacos 地址 |
| `serverPort` | `8848` | Nacos 端口 |
| `namespace` | 无 | 命名空间 ID |
### 3.2 生产常用自定义配置
```go
import (
	"github.com/nacos-group/nacos-sdk-go/clients"
	"github.com/nacos-group/nacos-sdk-go/common/constant"
	"github.com/nacos-group/nacos-sdk-go/vo"
)

sc := []constant.ServerConfig{*constant.NewServerConfig("127.0.0.1", 8848)}
cc := constant.ClientConfig{
	NamespaceId: "public", TimeoutMs: 5000, NotLoadCacheAtStart: true,
	LogDir: "./nacos/log", CacheDir: "./nacos/cache", LogLevel: "info",
}
nacosCli, err := clients.NewNamingClient(vo.NacosClientParam{
	ClientConfig: &cc, ServerConfigs: sc,
})
if err != nil {
	panic(err)
}

// 服务端
r := nacos.NewNacosRegistry(nacosCli)
// 客户端
resolver := nacos.NewNacosResolver(nacosCli)
cli.Use(sd.Discovery(resolver))
```
Windows 下 `LogDir` / `CacheDir` 不要用 `/tmp/...`，改成 `./nacos/log` 或 `D:\logs\nacos`。
## 四、Etcd 简述
```bash
go get github.com/hertz-contrib/registry/etcd
go get go.etcd.io/etcd/client/v3
```
```go
etcdCli, err := clientv3.New(clientv3.Config{
	Endpoints: []string{"127.0.0.1:2379"}, DialTimeout: 5 * time.Second,
})
if err != nil {
	panic(err)
}

r := etcd.NewEtcdRegistry(etcdCli)        // 服务端
resolver := etcd.NewEtcdResolver(etcdCli) // 客户端
```
Etcd 依赖租约（Lease）维持存活：进程退出后租约到期，key 自动删除。它没有 Nacos 那样的配置中心与 UI，适合已有 Etcd 基础设施的团队。构造函数与选项以 `hertz-contrib/registry/etcd` README 为准。
## 五、配置项速查
| 能力 | Consul | Nacos | Etcd |
| --- | --- | --- | --- |
| 注册地址 | `AgentService.Address/Port` | `Info.Addr` | `Info.Addr` |
| 权重 | `Info.Weight` | 同左 | 同左 |
| 标签/元数据 | `Tags` + `Meta` | `Tags` | `Tags` |
| 健康检查 | HTTP/TCP/TTL/gRPC | 心跳（临时实例） | Lease TTL |
| 命名空间 | Namespace（企业版） | `NamespaceId` | key 前缀 |
| 变更感知 | Watch/阻塞查询 | 订阅推送 | Watch |
## 六、上下线与灰度
优雅下线：
```go
h := server.Default(
	server.WithExitWaitTime(10*time.Second),
	// ...
)
```
推荐顺序：收到 SIGTERM → 权重置 0 或 `Deregister` → 等待发现刷新（≥ `RefreshInterval`）→ 等存量请求结束 → 退出。registry 抽象只有 `Deregister(info)`，多数实现会在退出时处理，但发布脚本显式摘除更稳。
灰度三种低成本做法：
| 做法 | 实现 | 优点 | 局限 |
| --- | --- | --- | --- |
| 权重 | 新版本权重 1，旧版本 9 | 不改代码 | 按实例比例，粒度粗 |
| 标签路由 | 实例打 `version=v2`，自定义 LB 过滤 | 可按用户/租户 | 需要写代码 |
| 网关分流 | 网关按 header/uid 选目标 | 灵活 | 多一层网关 |
```go
func (b *VersionBalancer) Pick(result discovery.Result) discovery.Instance {
	for _, ins := range result.Instances {
		if ins.Tag("version") == b.version {
			return ins
		}
	}
	return result.Instances[0] // 兜底
}
```
Hertz v0.10.5+ 还提供 `LoadbalancerCtx`（`PickCtx`），可基于请求 header 做路由，具体接口以官方源码为准。
## 常见坑
1. 注册地址用了 `127.0.0.1`：本机自测通过，跨机器必然失败。
2. Nacos 命名空间不一致：注册在 `public`、消费端查 `dev`，实例在线却查不到。
3. Consul 健康检查 URL 端口写错：实例一直 critical 被摘除，调用方随机 503。
4. Nacos 缓存目录不可写：非 root 容器启动失败。
5. 客户端只加 Resolver 没加 `sd.Discovery`：仍按 DNS 解析，报 `no such host`。
6. 改了 `RefreshInterval` 却感觉不生效：发现结果有缓存，还要看 `ExpireInterval` 与推送延迟。
7. 灰度只改权重没做兜底：新版本异常时仍可能被选中，需准备回滚开关。
## 小结
- Consul = `NewConsulRegister/NewConsulResolver` + `WithCheck/WithAdditionInfo`
- Nacos = `NewNacosRegistry/NewNacosResolver`，重点看命名空间与缓存目录
- Etcd 依赖租约，适合已有 Etcd 的轻量场景
- 客户端必须同时配 Resolver 与 `sd.Discovery`
- 发布稳定性取决于“摘除 → 等待 → 退出”
## 练习
1. 用 Compose 起 Consul，注册两个实例，验证加权比例。
2. 把 Nacos `NamespaceId` 改成 `dev`，复现“查不到实例”。
3. 停掉进程，观察 Consul 从 passing 到 critical 的时间。
4. 实现按 `version` 标签选实例的 Loadbalancer。
## 官方文档
- 服务注册与发现扩展：<https://www.cloudwego.io/zh/docs/hertz/tutorials/framework-exten/service_discovery/>
- registry/consul：<https://github.com/hertz-contrib/registry/tree/main/consul>
- registry/nacos：<https://github.com/hertz-contrib/registry/tree/main/nacos>
- registry/etcd：<https://github.com/hertz-contrib/registry/tree/main/etcd>
