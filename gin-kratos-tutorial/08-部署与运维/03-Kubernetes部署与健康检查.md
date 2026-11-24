# Kubernetes部署与健康检查

> 一句话定位：探针语义分清、优雅退出链路打通、资源与扩缩容配齐，服务才能在 K8s 里滚动发布而不掉一单。

## 本篇目标

- 讲清 `/healthz` 与 `/readyz` 的语义差异，并给出 Gin 实现
- 写出完整可用的 Deployment / Service / Ingress / ConfigMap / Secret / HPA / PDB
- 设计三类探针与 requests/limits，打通 preStop → SIGTERM → `Shutdown`

## 前置知识

- [08-部署与运维/02-Docker镜像与多阶段构建](02-Docker镜像与多阶段构建.md)：非 root 镜像与 UID

## 一、健康检查的语义

| 接口 | 回答的问题 | 检查内容 | 失败后果 |
| --- | --- | --- | --- |
| `/healthz` | 进程是否活着、没卡死 | 只有自身，不碰外部依赖 | liveness 失败 → 重启 |
| `/readyz` | 现在能不能接流量 | MySQL/Redis/etcd + 预热完成 | readiness 失败 → 摘流量 |

```go
// app/gateway/internal/handler/health.go
package handler

import (
    "context"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/redis/go-redis/v9"
    "gorm.io/gorm"
)

type Health struct {
    db  *gorm.DB
    rdb *redis.Client
}

func (h *Health) Healthz(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"status": "ok"})
}

func (h *Health) Readyz(c *gin.Context) {
    ctx, cancel := context.WithTimeout(c.Request.Context(), 300*time.Millisecond)
    defer cancel()
    sqlDB, err := h.db.DB()
    if err != nil || sqlDB.PingContext(ctx) != nil {
        c.JSON(http.StatusServiceUnavailable, gin.H{"status": "mysql unavailable"})
        return
    }
    if h.rdb.Ping(ctx).Err() != nil {
        c.JSON(http.StatusServiceUnavailable, gin.H{"status": "redis unavailable"})
        return
    }
    c.JSON(http.StatusOK, gin.H{"status": "ready"})
}
```

Kratos gRPC 服务注册标准健康服务（`grpc_health_v1`），K8s 用 `grpc` 探针。探针不鉴权、不打业务日志、超时 ≤ `timeoutSeconds`；`/readyz` 只把**核心**依赖判 503，否则下游抖动会放大成全站摘流量。

## 二、Deployment 全量清单

```yaml
# deploy/k8s/gateway/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: gateway, labels: {app: gateway}}
spec:
  replicas: 3
  strategy: {type: RollingUpdate, rollingUpdate: {maxSurge: 1, maxUnavailable: 0}}
  selector: {matchLabels: {app: gateway}}
  template:
    metadata:
      labels: {app: gateway}
    spec:
      terminationGracePeriodSeconds: 40
      securityContext: {runAsNonRoot: true, runAsUser: 65532}
      containers:
        - name: gateway
          image: registry.example.com/mall/gateway:v1.4.0
          ports: [{name: http, containerPort: 8080}, {name: metrics, containerPort: 9101}]
          envFrom: [{configMapRef: {name: gateway-config}}, {secretRef: {name: gateway-secret}}]
          resources: {requests: {cpu: 200m, memory: 256Mi}, limits: {cpu: "1", memory: 512Mi}}
          securityContext: {allowPrivilegeEscalation: false, readOnlyRootFilesystem: true, capabilities: {drop: ["ALL"]}}
          startupProbe: {httpGet: {path: /healthz, port: http}, periodSeconds: 2, failureThreshold: 30}
          livenessProbe: {httpGet: {path: /healthz, port: http}, periodSeconds: 10, timeoutSeconds: 2, failureThreshold: 3}
          readinessProbe: {httpGet: {path: /readyz, port: http}, periodSeconds: 5, timeoutSeconds: 2, failureThreshold: 2}
          lifecycle: {preStop: {exec: {command: ["/bin/sh", "-c", "sleep 5"]}}}
```

`requests` 决定调度与 HPA 基线，`limits` 决定 OOM 与 CPU 节流；内存 limit 给观测峰值的 1.5 倍，`GOMEMLIMIT` 设为 limit 的 80%。开 `readOnlyRootFilesystem` 后要写临时文件就挂 `emptyDir` 到 `/tmp`。

优雅退出链路：摘 Endpoints → preStop sleep 5s（等 kube-proxy/Ingress/APISIX 收敛）→ SIGTERM → `Shutdown(15s)` → 关闭 DB/Redis、注销注册中心。`terminationGracePeriodSeconds`(40) 必须大于 preStop + shutdown 超时，否则进程被 SIGKILL，滚动发布期间成片 502。

## 三、Service、Ingress 与 APISIX

```yaml
# deploy/k8s/gateway/service.yaml
apiVersion: v1
kind: Service
metadata: {name: gateway}
spec:
  selector: {app: gateway}
  ports: [{name: http, port: 80, targetPort: http}]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: {name: gateway}
spec:
  ingressClassName: nginx
  rules:
    - host: api.mall.local
      http: {paths: [{path: /, pathType: Prefix, backend: {service: {name: gateway, port: {name: http}}}}]}
```

流量链路固定为 `Ingress/APISIX → gateway Service → Pod`：APISIX 负责 JWT、限流、CORS，Ingress 只做入口与 TLS；**前端不得直连 Kratos gRPC 服务**，内部服务只暴露 ClusterIP。`/healthz`、`/readyz`、`/metrics` 必须在网关禁止外网路由。

## 四、ConfigMap、Secret 与外部依赖

```yaml
# deploy/k8s/gateway/config.yaml
apiVersion: v1
kind: ConfigMap
metadata: {name: gateway-config}
data:
  MALL_REGISTRY_ETCD_ENDPOINTS: '["etcd.mall.svc.cluster.local:2379"]'
  MALL_REDIS_ADDR: "redis.mall.svc.cluster.local:6379"
---
apiVersion: v1
kind: Secret
metadata: {name: gateway-secret}
stringData:
  MALL_MYSQL_DSN: "mall:change-me@tcp(mysql.mall.svc.cluster.local:3306)/mall?charset=utf8mb4&parseTime=True&loc=Local"
```

`envFrom` 注入的变量**不会热更新**，改配置要 `kubectl rollout restart`；需要热加载的走 Viper `WatchConfig`。Secret 只是 base64，生产用 External Secrets / KMS。MySQL/Redis/etcd/Consul/Kafka 的连接信息全部来自 ConfigMap/Secret：集群内用 Service DNS，集群外用域名，禁止硬编码。

## 五、HPA 与 PDB

```yaml
# deploy/k8s/gateway/hpa-pdb.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: gateway}
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: gateway}
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - {type: Resource, resource: {name: cpu, target: {type: Utilization, averageUtilization: 60}}}
    - {type: Pods, pods: {metric: {name: http_requests_per_second}, target: {type: AverageValue, averageValue: "500"}}}
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: gateway}
spec:
  minAvailable: 2
  selector: {matchLabels: {app: gateway}}
```

CPU 指标开箱即用；QPS 指标需 Prometheus Adapter 或 KEDA，把 `http_server_requests_total` 转成 `http_requests_per_second`。`minAvailable` 必须小于 `replicas`，否则节点排水会永久阻塞。

## 六、迁移 Job 与 NetworkPolicy

```yaml
# deploy/k8s/gateway/migrate-job.yaml
apiVersion: batch/v1
kind: Job
metadata: {name: mall-migrate-v140}
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: registry.example.com/mall/gateway:v1.4.0
          args: ["migrate", "--config", "/etc/mall/config.yaml"]
          envFrom: [{secretRef: {name: gateway-secret}}]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: gateway}
spec:
  podSelector: {matchLabels: {app: gateway}}
  ingress:
    - from: [{namespaceSelector: {matchLabels: {name: ingress-nginx}}}, {podSelector: {matchLabels: {app: apisix}}}]
      ports: [{port: 8080, protocol: TCP}]
  egress:
    - to: [{podSelector: {matchLabels: {app: user-service}}}]
      ports: [{port: 9001, protocol: TCP}]
    - to: [{namespaceSelector: {matchLabels: {name: data}}}]
      ports: [{port: 3306}, {port: 6379}, {port: 2379}]
```

迁移 Job 必须在 Deployment 滚动**之前**完成（CI 里 `Job → rollout` 顺序），且脚本向后兼容：先加字段/索引，再发代码，最后清理旧字段。

## 完整示例

```bash
kubectl apply -f deploy/k8s/gateway/config.yaml -f deploy/k8s/gateway/migrate-job.yaml && kubectl wait --for=condition=complete job/mall-migrate-v140 --timeout=180s
kubectl apply -f deploy/k8s/gateway/ && kubectl rollout status deploy/gateway --timeout=180s
```

## 常见坑

1. **liveness 检查数据库**：MySQL 抖动让全部 Pod 同时重启，小故障放大成全站不可用。
2. **没有 startupProbe**：冷启动慢的服务被 liveness 提前杀掉，进入 CrashLoopBackOff。
3. **缺 preStop**：Endpoints 未摘除就退出，滚动发布期间成片 502。
4. **grace period 小于 shutdown 超时**：进程被 SIGKILL，连接与日志来不及收尾。
5. **requests 不设或 limits 过低**：调度漂移、OOMKilled、CPU 节流导致 P99 飙升。

## 小结

- liveness 只看进程，readiness 才看依赖，且只对核心依赖返回 503
- Deployment 必须配 requests/limits、非 root、只读根文件系统、三类探针与 preStop
- 滚动更新靠 `maxUnavailable: 0` + readiness + PDB；HPA 优先 CPU，QPS 指标需额外设施

## 练习

1. 在 kind/minikube 部署本文清单，用 `kubectl get pods -w` 观察滚动发布行为。
2. 把 liveness 改成调 `/readyz`，断开 MySQL，观察是否全体重启并解释原因。
3. 给 HPA 增加内存指标，用 `kubectl describe hpa gateway` 查看扩缩容决策。

## 官方文档

- 探针配置：<https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/>
- Deployment 滚动更新：<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- HPA 算法：<https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/>
