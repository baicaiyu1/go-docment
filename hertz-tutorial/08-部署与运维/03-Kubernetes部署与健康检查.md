# Kubernetes部署与健康检查
> 一句话定位：用正确的探针、优雅退出和资源约束，让 Hertz 服务在 K8s 里可滚动、可扩容、可回滚。

## 本篇目标

- 写出完整可用的 Deployment / Service / Ingress / HPA 清单
- 设计三类探针，配好资源、preStop 与优雅退出，避免“健康检查把服务打挂”
- 用 ConfigMap/Secret 注入配置，掌握发布、回滚与常见排障

## 前置知识

- 已读 [08-部署与运维/02-Docker镜像与多阶段构建](02-Docker镜像与多阶段构建.md)
- 了解 Pod、Deployment、Service、Ingress、探针的基本概念

## 一、应用侧健康检查

liveness 只反映“进程是否还能工作”，readiness 才检查外部依赖：

```go
func Healthz(ctx context.Context, c *app.RequestContext) {
	c.JSON(consts.StatusOK, utils.H{"status": "ok"})
}

func Readyz(db *sql.DB) app.HandlerFunc {
	return func(ctx context.Context, c *app.RequestContext) {
		pingCtx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
		defer cancel()
		if db.PingContext(pingCtx) != nil {
			c.JSON(consts.StatusServiceUnavailable, utils.H{"status": "not ready"})
			return
		}
		c.JSON(consts.StatusOK, utils.H{"status": "ready"})
	}
}
```

路由：`h.GET("/healthz", handler.Healthz)`、`h.GET("/readyz", handler.Readyz(db))`。健康检查不要鉴权、不要写日志、不要访问慢依赖。

## 二、Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: userapi
  labels: {app: userapi}
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: {maxSurge: 1, maxUnavailable: 0}
  selector:
    matchLabels: {app: userapi}
  template:
    metadata:
      labels: {app: userapi}
    spec:
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
      containers:
        - name: userapi
          image: registry.example.com/userapi:v1.4.0
          ports:
            - {name: http, containerPort: 8888}
          envFrom:
            - configMapRef: {name: userapi-config}
            - secretRef: {name: userapi-secret}
          resources:
            requests: {cpu: 200m, memory: 128Mi}
            limits: {cpu: "1", memory: 512Mi}
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: {drop: ["ALL"]}
          startupProbe:
            httpGet: {path: /healthz, port: http}
            periodSeconds: 2
            failureThreshold: 30
          livenessProbe:
            httpGet: {path: /healthz, port: http}
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          readinessProbe:
            httpGet: {path: /readyz, port: http}
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 2
          lifecycle:
            preStop:
              exec: {command: ["sleep", "5"]}
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels: {app: userapi}
```

`runAsUser` 必须与镜像内 UID 一致（alpine 示例是 10001，distroless nonroot 是 65532）。`readOnlyRootFilesystem` 打开后，应用不能往工作目录写文件，临时文件请挂 `emptyDir`。

## 三、Service

```yaml
apiVersion: v1
kind: Service
metadata: {name: userapi}
spec:
  type: ClusterIP
  selector: {app: userapi}
  ports:
    - {name: http, port: 80, targetPort: http, protocol: TCP}
```

## 四、Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: userapi
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "3"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: userapi-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: userapi
                port: {name: http}
```

## 五、ConfigMap 与 Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata: {name: userapi-config}
data:
  APP_SERVER_ADDR: ":8888"
---
apiVersion: v1
kind: Secret
metadata: {name: userapi-secret}
type: Opaque
stringData:
  APP_DB_DSN: "app:change-me@tcp(mysql:3306)/app?parseTime=true"
```

Secret 默认只是 base64，不是加密；生产用 SealedSecrets、External Secrets 或云 KMS。`envFrom` 注入的变量不会热更新，改配置后需触发滚动重启。

## 六、探针设计

| 探针 | 检查内容 | 参数建议 |
| --- | --- | --- |
| startup | 进程是否完成启动 | `periodSeconds: 2`、`failureThreshold: 30`，覆盖最慢启动 |
| liveness | 进程是否卡死 | `periodSeconds: 10`、`timeoutSeconds: 2`、失败 3 次重启 |
| readiness | 是否可接流量 | `periodSeconds: 5`、失败 2 次摘流量 |

关键原则：**liveness 绝不检查 DB/Redis**，否则下游抖动会触发全量重启，形成雪崩。启动慢的服务用 startupProbe 保护，避免 liveness 提前杀进程。

## 七、优雅退出链路

```text
Pod 进入 Terminating → Endpoints 摘除（并行）→ preStop sleep 5s（等 kube-proxy/Ingress 收敛）
→ SIGTERM → Hertz Engine.Shutdown(15s) → 进程退出
```

- `terminationGracePeriodSeconds` 必须大于 `preStop` + 应用 shutdown 超时（示例 30 > 5 + 15）
- 应用侧必须处理 SIGTERM 并 `Shutdown`；`maxUnavailable: 0` 要求预留资源，否则新 Pod 一直 Pending

## 八、HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: userapi}
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: userapi}
  minReplicas: 3
  maxReplicas: 30
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies: [{type: Percent, value: 100, periodSeconds: 30}]
    scaleDown:
      stabilizationWindowSeconds: 300
      policies: [{type: Percent, value: 50, periodSeconds: 60}]
  metrics:
    - type: Resource
      resource: {name: cpu, target: {type: Utilization, averageUtilization: 60}}
    - type: Pods
      pods:
        metric: {name: http_requests_per_second}
        target: {type: AverageValue, averageValue: "500"}
```

CPU 指标开箱即用；QPS 这类自定义指标需要 Prometheus Adapter + Hertz 的 Prometheus 中间件暴露 `http_requests_per_second`。没有这套设施时先只用 CPU，按 [04-性能调优与压测](04-性能调优与压测.md) 估副本数。

## 九、滚动更新与回滚

```bash
kubectl rollout status deploy/userapi --timeout=120s
kubectl rollout history deploy/userapi
kubectl rollout undo deploy/userapi --to-revision=2
```

PowerShell 命令完全相同。

发布前确认 readiness 能真实反映依赖；否则新 Pod 未就绪就被判成功，流量打进来直接 5xx。

## 常见坑

1. **liveness 检查 DB**：数据库抖动导致所有 Pod 同时重启，故障放大数倍。
2. **没配 preStop**：Endpoints 尚未摘除就退出，滚动发布期间出现 502。
3. **grace period 小于 shutdown 超时**：进程被 SIGKILL，连接和日志来不及收尾。
4. **requests 不设或 limits 过低**：调度失衡、OOMKilled、CPU 被限流导致 P99 飙升。
5. **image 用 `latest` 或 HPA 与固定 replicas 混用**：前者无法回滚，后者每次 apply 都改回副本数；探针路径写错则 Pod 永远 NotReady。

## 小结

- liveness 与 readiness 职责分离，健康检查不碰慢依赖
- Deployment 要配资源、非 root、只读根文件系统、preStop 与合理的 grace period
- 滚动更新依赖 readiness 与 `maxUnavailable: 0`，回滚按 revision 或镜像 digest
- HPA 优先 CPU；QPS 指标需额外监控设施，容量仍要压测确认

## 练习

1. 部署本文清单到本地 kind/minikube，观察滚动更新时 `kubectl get pods -w` 的变化。
2. 把 liveness 改成检查 DB，断开数据库后观察 Pod 是否被批量重启。
3. 给 HPA 增加内存指标，用 `kubectl describe hpa` 查看扩缩容决策。

## 相关文档

- [08-部署与运维/02-Docker镜像与多阶段构建](02-Docker镜像与多阶段构建.md)
- [08-部署与运维/04-性能调优与压测](04-性能调优与压测.md)
- [03-基本特性/07-优雅退出](../03-基本特性/07-优雅退出.md)
- Kubernetes 探针文档：<https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/>
