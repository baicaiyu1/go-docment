# Gin BFF 加 Kratos 的微服务架构

> 一句话定位：南北向流量用 APISIX + Gin BFF 接住浏览器与 App，东西向用 Kratos gRPC 传强类型契约；本篇把分层、边界、时序与代价一次钉死。

## 本篇目标

- 说清 `APISIX → Gin BFF → Kratos gRPC → MySQL/Redis/Kafka` 每层只做什么、不做什么
- 用「服务边界 + 数据所有权」判定一个功能该落在哪个服务
- 画出一次下单与一次查询的时序，给出部署拓扑与单体的代价对比
- 用技术决策表锁定选型，并明确本教程不做的三件事

## 前置知识

- [02-Gin入门/01-Gin是什么-架构与性能](../02-Gin入门/01-Gin是什么-架构与性能.md)
- [06-Gin进阶/03-Protobuf与gRPC工作流](../06-Gin进阶/03-Protobuf与gRPC工作流.md)

## 一、四层结构：入口、编排、领域、数据

南北向（客户端 → 系统）要兼容浏览器、App、第三方，东西向（服务 → 服务）只要快和稳，两者最优解不同。

| 层 | 组件 | 协议与端口 | 只做这些 |
| --- | --- | --- | --- |
| 接入层 | APISIX 3.11 | HTTP/HTTPS，9080 | 路由、JWT 初验、全局限流、CORS、灰度 |
| BFF 层 | Gin v1.10 | HTTP/JSON、SSE，8080 | 绑定校验、身份解析、多服务聚合、统一响应 |
| 领域层 | Kratos v2.8 | gRPC + Protobuf，9001/9002/9003 | 业务规则、事务、数据所有权、领域事件 |
| 数据层 | MySQL 8.4 / Redis 7.4 / Kafka 3.8 | 3306 / 6379 / 9092 | 持久化、缓存、异步解耦 |

三条硬规则：**接入层不放业务，BFF 不碰数据库，领域服务不直接面向公网。** BFF 的 Gin 装配与优雅退出见 [02-Gin入门/02-快速开始-第一个服务](../02-Gin入门/02-快速开始-第一个服务.md)、[03-基本特性/07-优雅退出](../03-基本特性/07-优雅退出.md)。

领域服务是纯 gRPC：没有 HTML、SSE、Cookie，只暴露契约里的方法。

```go
// app/user-service/cmd/main.go：领域服务只有 gRPC
package main

import (
    "time"

    "github.com/go-kratos/kratos/v2"
    kratosgrpc "github.com/go-kratos/kratos/v2/transport/grpc"
    v1 "mall-lite/api/user/v1"
)

func main() {
    gs := kratosgrpc.NewServer(
        kratosgrpc.Address(":9001"),
        kratosgrpc.Timeout(2*time.Second),
    )
    v1.RegisterUserServiceServer(gs, &v1.UnimplementedUserServiceServer{}) // 实现见 02
    app := kratos.New(kratos.Name("user-service"), kratos.Server(gs))
    if err := app.Run(); err != nil {
        panic(err)
    }
}
```

## 二、为什么 HTTP 入口用 Gin、内部 RPC 用 Kratos

| 方案 | 结果 | 代价 |
| --- | --- | --- |
| 全 Kratos（HTTP + gRPC） | 一套框架、契约统一 | HTTP 传输面向内部接口，SSE、静态资源、Swagger、CORS 都要自己补 |
| 全 Gin + 内部 HTTP/JSON | 简单、团队熟悉 | 契约靠注释，改字段无编译错误；JSON 序列化与带宽比 Protobuf 高 2~5 倍 |
| **Gin BFF + Kratos gRPC** | 边缘生态快、内部契约强 | 两套框架、两份心智模型，需统一目录、错误码、可观测性 |

结论：**边缘选 Gin，是它的中间件与渲染生态能最快交付 BFF；内部选 Kratos，是 Protobuf 契约、HTTP/2 多路复用和内置治理能兜住服务间调用。**

反例是把 BFF 做成第二个单体。判断标准只有一句——**换个小程序客户端还要这段逻辑吗？要，就下沉到领域服务。**

## 三、服务边界与数据所有权

| 服务 | 拥有数据 | 对外能力 | 不允许别人做的事 |
| --- | --- | --- | --- |
| user-service | `mall_user` 库 | GetUser、BatchGetUsers、登录校验 | 读 user 表、校验密码 |
| product-service | `mall_product` 库 + Redis | GetProduct、DeductStock | 创建订单、判断订单状态 |
| order-service | `mall_order` 库 + outbox 表 | CreateOrder、GetOrder、CancelOrder | 直接改库存表 |

拆分依据是**业务能力 + 数据所有权**，不是技术分层：一个服务独占一份 schema，跨服务只走契约，禁止跨库 join、禁止共享表。

## 四、同步与异步的边界

按「失败要不要让用户重试」分类：必须立即回滚的走同步；积分、通知、搜索索引走异步；跨服务长事务用本地消息表 + Kafka 最终一致（见 [08-分布式事务与最终一致性](08-分布式事务与最终一致性.md)）。

事件体固定 `event_id`（消费端幂等键）、`order_id`、`user_id`、`sku_id`、`quantity`、`amount_cents`、`created_at`，再加 `trace_id` 便于回溯；它与 outbox 表同事务落库，表结构见第 08 篇。

## 五、一次下单与一次查询的时序

```text
下单（写路径，同步部分尽量短）
POST /api/v1/orders
  → APISIX：jwt-auth 校验 token、limit-req 限流、注入 X-Request-Id
  → Gin：ShouldBindJSON 校验、user_id 取自 token（不信任 body）
  → order-service：本地事务写 order + outbox，状态 CREATED
  → product-service.DeductStock(request_id, sku_id, quantity)：乐观锁扣减
      ├─ 库存不足 → 409 → 回滚本地事务 → 网关映射 409/400xx
      └─ 成功 → 提交事务 → 201 {order_id}
  → Kafka order.created → 积分、通知、搜索索引（异步，失败不阻塞用户）
```

```text
查询（读路径，允许降级与缓存）
GET /api/v1/orders/:id → APISIX → Gin：解析 id 与 user_id，设 800ms 预算
  → order-service.GetOrder（主数据，必须成功）
  → 并发 product.BatchGetProducts + user.GetUser（弱依赖，失败降级为占位）
  → 聚合裁剪后输出 {code,message,data,request_id}
```

## 六、部署拓扑与单体的代价

```text
Web / App ──HTTPS──▶ APISIX :9080（路由 · JWT · 限流 · 灰度）
                          │ upstream 固定指向 gateway
                          ▼
                gateway（Gin BFF）:8080
        ┌─────────────────┼─────────────────┐
        ▼ gRPC:9001       ▼ gRPC:9002       ▼ gRPC:9003
  user-service      product-service    order-service
        ▼                 ▼                 ▼
   MySQL(user)       MySQL(product)    MySQL(order) + Kafka
        └──── etcd:2379 / Consul:8500 · Prometheus:9090 · Jaeger:16686 ────┘
```

换微服务不是免费的，代价要提前认账：

| 维度 | 单体 | 本架构 | 多付的成本 |
| --- | --- | --- | --- |
| 运维 | 一个进程、一次发布 | 4 个服务 + 注册中心 + 网关 | 部署、探针、配置、证书 |
| 链路 | 函数调用，栈能看全 | 跨进程 RPC | 需要 trace_id / request_id 串联 |
| 一致性 | 一个本地事务 | 本地事务 + 消息 + 幂等 | 对账、补偿、重复消费 |
| 故障半径 | 一处崩全崩 | 可隔离 | 超时、重试、熔断必须配齐 |

只有当「独立伸缩、独立发布、团队边界」的收益大于上述成本时才拆。

## 七、技术决策表

| 决策点 | 选择 | 备选 | 理由与代价 |
| --- | --- | --- | --- |
| HTTP 入口 | Gin v1.10 | Kratos HTTP、net/http | 中间件生态成熟；多一套框架 |
| 内部 RPC | gRPC + Protobuf | HTTP/JSON | 契约可校验、性能好；要维护 proto 流程 |
| 注册中心 | etcd 3.5 / Consul 1.20 | K8s DNS | 支持元数据与健康检查；多一个强依赖 |
| 异步解耦 | Kafka 3.8 | RocketMQ、RabbitMQ | 吞吐与生态；运维复杂度高 |
| 入口网关 | APISIX 3.11 | 自研 Gin 网关 | 插件齐全、动态路由；学习成本高 |

## 八、明确不做的事

1. **不引入服务网格**（Istio / Linkerd）：sidecar 会把 RPC 与治理细节藏起来，本教程要讲清这些机制；治理先由 APISIX + Sentinel 覆盖。
2. **不按技术层拆服务**：controller、service、dao 各起一个进程，一次请求跨三次 RPC，P99 直接翻倍。
3. **不做过度拆分**：一个 CRUD 也建服务只会提前引入复杂度。先单体，出现独立伸缩或团队边界再拆。
4. **不上 CQRS / Event Sourcing**：订单查询用「主库 + 缓存」足够，事件只用于解耦。

## 完整示例

```yaml
# deploy/docker/compose-infra.yaml：本地只起依赖，服务用 go run
services:
    etcd:
        image: bitnami/etcd:3.5
        environment: ["ALLOW_NONE_AUTHENTICATION=yes"]
        ports: ["2379:2379"]
    mysql:
        image: mysql:8.4
        environment: ["MYSQL_ROOT_PASSWORD=root", "MYSQL_DATABASE=mall"]
        ports: ["3306:3306"]
    redis:
        image: redis:7.4
        ports: ["6379:6379"]
    kafka:
        image: bitnami/kafka:3.8
        environment: ["KAFKA_CFG_NODE_ID=0", "KAFKA_CFG_PROCESS_ROLES=controller,broker", "KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093", "KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://127.0.0.1:9092", "KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093", "KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER"]
        ports: ["9092:9092"]
```

```bash
cd deploy/docker && docker compose -f compose-infra.yaml up -d
go run ./app/user-service/cmd &
go run ./app/gateway/cmd
curl -s http://127.0.0.1:8080/api/v1/healthz
```

```powershell
cd deploy\docker; docker compose -f compose-infra.yaml up -d
Start-Process go -ArgumentList "run ./app/user-service/cmd"
go run .\app\gateway\cmd
curl.exe -s http://127.0.0.1:8080/api/v1/healthz
```

预期输出 `{"code":0,"message":"ok"}`；接着按 [02-Kratos快速入门与Proto](02-Kratos快速入门与Proto.md) 补契约与首个服务。

## 常见坑

1. **BFF 越来越胖**：优惠、库存规则写进 Gin handler，等于把单体搬到网关。只做鉴权、校验、聚合、裁剪。
2. **服务间共享数据库**：图省事直连对方表，对方改字段时全线崩溃。跨服务只走契约。
3. **把弱依赖做成强依赖**：头像、昵称拉不到就让整页 500；弱依赖必须降级。
4. **gRPC 端口暴露公网**：9001~9003 只监听内网，公网入口只有 APISIX 的 9080。
5. **异步动作放进请求链路**：发短信、写索引同步执行，Kafka 一抖动用户就下单失败。
6. **超时倒挂**：网关 500ms、RPC 2s，网关超时后下游继续跑，重试再把下游打死。

## 小结

- 接入 APISIX、BFF 用 Gin、领域用 Kratos gRPC，数据层按服务切分；不用一套框架全包，是边缘生态与内部契约的诉求不同
- 同步只保留强一致步骤，通知类动作交给 Kafka 做最终一致
- 微服务的代价是运维、链路、一致性，必须用超时、重试、熔断与可观测性兜住

## 练习

1. 画出包含 APISIX、etcd、Kafka、Jaeger 的部署图，标出每个服务的端口与所属数据库。
2. 网关总预算 800ms，推算 order→product、product→MySQL 各设多少，写成配置项。
3. 找出你项目里「网关写了业务逻辑」的三处代码，说明应下沉到哪个服务、为什么。

## 官方文档

- Gin 官方文档：<https://gin-gonic.com/zh-cn/docs/>
- Kratos 官方文档：<https://go-kratos.dev/docs/>
- gRPC Go 快速开始：<https://grpc.io/docs/languages/go/quickstart/>
- Apache APISIX 文档：<https://apisix.apache.org/zh/docs/apisix/getting-started/>
