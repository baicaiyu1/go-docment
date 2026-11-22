# Docker镜像与多阶段构建
> 一句话定位：用多阶段构建把“编译工具链”和“运行环境”分开，产出体积小、非 root、可健康检查的生产镜像。

## 本篇目标

- 写出 golang 构建 + alpine/distroless 运行的多阶段 Dockerfile
- 掌握非 root、时区、CA 证书、HEALTHCHECK 的正确配置
- 利用 go mod download 先行与 BuildKit cache 优化层缓存
- 用 docker compose 在本地联调完整依赖，并建立镜像安全与回滚习惯

## 前置知识

- 已读 [08-部署与运维/01-编译与交叉编译](01-编译与交叉编译.md)
- 本机安装 Docker / BuildKit，了解镜像层与 `.dockerignore`

## 一、为什么要多阶段

单阶段镜像会把 Go 工具链、模块缓存、源码一起带进运行环境，动辄 800 MB 以上；多阶段只把最终二进制复制到干净的基础镜像：

```text
builder（golang:1.25-bookworm，约 850 MB）── COPY 二进制 ──▶ runtime（alpine/distroless，约 12~20 MB）
```

构建阶段可有编译器和缓存，运行阶段没有 shell、没有包管理器，攻击面更小。

## 二、Dockerfile：alpine 运行

```dockerfile
# syntax=docker/dockerfile:1.7
FROM golang:1.25-bookworm AS build
WORKDIR /src
ENV CGO_ENABLED=0 GOOS=linux
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
ARG VERSION=dev
ARG COMMIT=none
ARG BUILD_TIME=unknown
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -trimpath -buildvcs=false \
    -ldflags "-s -w -X hertz_demo/internal/version.Version=${VERSION} -X hertz_demo/internal/version.Commit=${COMMIT} -X hertz_demo/internal/version.BuildTime=${BUILD_TIME}" \
    -o /out/userapi ./cmd/userapi

FROM alpine:3.21
RUN addgroup -S -g 10001 app && adduser -S -u 10001 -G app app \
 && apk add --no-cache ca-certificates tzdata wget \
 && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
 && echo "Asia/Shanghai" > /etc/timezone
WORKDIR /app
COPY --from=build /out/userapi /app/userapi
USER 10001:10001
EXPOSE 8888
ENV TZ=Asia/Shanghai
HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -q -O /dev/null http://127.0.0.1:8888/healthz || exit 1
ENTRYPOINT ["/app/userapi"]
```

注意 `AS build` 先给自己命名，运行阶段用 `COPY --from=build` 只拿产物；`ARG` 必须在构建阶段声明，`docker build --build-arg` 才能注入。

## 三、distroless 变体

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/userapi /app/userapi
USER nonroot:nonroot
EXPOSE 8888
ENV TZ=Asia/Shanghai
ENTRYPOINT ["/app/userapi"]
```

distroless 没有 shell、wget、curl，`HEALTHCHECK` 不能再用 wget；两种选择：在应用里加 `-healthcheck` 参数（探测本机 `/healthz` 后退出 0/1），或干脆交给 Kubernetes 的 `readinessProbe`（见下一篇）。distroless 已带 CA 证书，时区用 `TZ` 环境变量，若镜像内无 tzdata 可从构建阶段 `COPY --from=build /usr/share/zoneinfo /usr/share/zoneinfo`。

## 四、非 root、时区与证书

- 创建固定 UID/GID（如 10001），K8s 的 `runAsUser` 才能对齐；不要用 `USER app` 这种依赖镜像内解析的写法
- 必须安装 `ca-certificates`，否则访问 HTTPS 下游报 `x509: certificate signed by unknown authority`
- 设置 `TZ=Asia/Shanghai` 并写入 `/etc/localtime`，否则日志时间差 8 小时
- 只读根文件系统时，需要写临时文件要显式挂载 `/tmp`（`--tmpfs /tmp` 或 `emptyDir`）

## 五、层缓存与 .dockerignore

`COPY go.mod go.sum` 放在 `COPY . .` 之前：依赖不变时 `go mod download` 层命中缓存，改业务代码不会重新下载模块。

`.dockerignore`：

```text
.git
.github
bin
coverage.out
*.md
*.exe
.env
.vscode
.idea
```

把 `.git`、测试产物、本地二进制排除掉，既加快构建上下文上传，也避免密钥被误打进镜像。BuildKit 的 cache mount 默认开启，无需额外配置。

## 六、镜像体积对比

| 方案 | 基础镜像 | 运行镜像体积（示例） |
| --- | --- | --- |
| 单阶段 | golang:1.25 | 约 900 MB |
| alpine | alpine:3.21 | 约 20 MB |
| distroless static | distroless/static | 约 12 MB |

构建并查看（bash 与 PowerShell 命令相同）：

```bash
docker build --build-arg VERSION=v1.4.0 -t userapi:v1.4.0 .
docker images userapi
docker run --rm -p 8888:8888 --read-only --cap-drop ALL userapi:v1.4.0
```

## 七、docker compose 本地联调

```yaml
services:
  app:
    build:
      context: .
      args: {VERSION: dev}
    image: userapi:dev
    ports: ["8888:8888"]
    environment:
      APP_SERVER_ADDR: ":8888"
      APP_DB_DSN: "app:app@tcp(mysql:3306)/app?parseTime=true&loc=Local"
      APP_REDIS_ADDR: "redis:6379"
    depends_on:
      mysql: {condition: service_healthy}
    restart: unless-stopped
  mysql:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: app
      MYSQL_USER: app
      MYSQL_PASSWORD: app
    ports: ["3306:3306"]
    volumes: ["mysql-data:/var/lib/mysql"]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-uroot", "-proot"]
      interval: 5s
      timeout: 3s
      retries: 20
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
volumes:
  mysql-data:
```

```bash
docker compose up -d --build
docker compose logs -f app
docker compose down -v
```

```powershell
docker compose up -d --build
docker compose logs -f app
docker compose down -v
```

compose 不要写 `version:` 字段（Compose Spec 已废弃）；`depends_on` 只保证启动顺序，真正的依赖就绪要靠 healthcheck。

## 八、安全与可回滚

- 镜像 tag 用 `v1.4.0` 与 `v1.4.0-<git短sha>` 双标签，生产禁止只推 `latest`
- 发布记录保留镜像 digest，回滚时按 digest 拉取，避免 tag 被覆盖
- 构建后跑漏洞扫描：`trivy image --severity HIGH,CRITICAL userapi:v1.4.0`
- 运行参数：`--read-only --cap-drop ALL --security-opt no-new-privileges`

## 常见坑

1. **忘记 `CGO_ENABLED=0`**：alpine 运行时报 `not found` 或 `libc` 缺失。
2. **先 `COPY . .` 再下载依赖**：任何源码改动都让模块缓存失效，构建慢数倍。
3. **用 root 运行**：容器逃逸风险高，K8s 的 `runAsNonRoot` 也会直接拒绝启动。
4. **缺 CA 证书**：调用 HTTPS 服务全部失败，且报错容易被误判为下游故障。
5. **时区未设置**：日志与业务时间全差 8 小时，跨团队排障互相甩锅。
6. **HEALTHCHECK 用 curl 但镜像没装**：容器一直 unhealthy；distroless 必须换方案。
7. **只推 `latest`**：无法定位线上版本，也无法快速回滚。

## 小结

- 多阶段构建把工具链留在 builder，运行镜像只保留二进制与必要证书
- alpine 适合要 shell/wget 排障的场景，distroless 更小更安全但要另做健康检查
- 缓存优化的关键是依赖文件先行 + `.dockerignore` + BuildKit cache
- 生产镜像必须非 root、固定 tag/digest、可扫描、可回滚

## 练习

1. 分别构建 alpine 与 distroless 两个镜像，用 `docker images` 对比体积。
2. 改动一个 handler 文件后重新构建，观察 `go mod download` 是否命中缓存。
3. 以 `--read-only --cap-drop ALL` 运行容器，验证 `/healthz` 与日志输出正常。

## 相关文档

- [08-部署与运维/01-编译与交叉编译](01-编译与交叉编译.md)
- [08-部署与运维/03-Kubernetes部署与健康检查](03-Kubernetes部署与健康检查.md)
- [06-hz进阶/05-常见坑与最佳实践](../06-hz进阶/05-常见坑与最佳实践.md)
- Docker 多阶段构建：<https://docs.docker.com/build/building/multi-stage/>
