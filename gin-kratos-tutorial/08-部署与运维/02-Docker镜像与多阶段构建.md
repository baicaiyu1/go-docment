# Docker镜像与多阶段构建

> 一句话定位：把编译工具链留在 builder，运行镜像只带二进制、证书和时区，以非 root + 只读根文件系统运行，镜像从 900MB 降到 12MB，并且可扫描、可回滚。

## 本篇目标

- 写出 `golang:1.25` → `distroless/static:nonroot`（或 scratch）的多阶段 Dockerfile
- 正确处理非 root、CA 证书、时区、PID1 信号四件事
- 用 `buildx` 一次构建 amd64/arm64 双架构，并打全 OCI labels
- 用 `.dockerignore` + cache mount 把重复构建压到秒级
- 用 trivy 扫描并在 CI 卡住高危漏洞

## 前置知识

- [08-部署与运维/01-编译与交叉编译](01-编译与交叉编译.md)：`CGO_ENABLED=0` 与 `-ldflags` 注入
- [07-工程化与测试/04-CI流水线与代码质量](../07-工程化与测试/04-CI流水线与代码质量.md)：镜像扫描作为门禁
- 本机 Docker 24+，已启用 buildx

## 一、单阶段镜像为什么不能上生产

```text
单阶段：golang:1.25（约 900MB，含编译器、模块缓存、源码）
多阶段：builder（约 900MB）── COPY 二进制 ──▶ runtime（12~20MB）
```

单阶段镜像里带着 shell、包管理器、源码和完整工具链：体积大、拉取慢、CVE 数量多、被拿 shell 后能直接编译提权工具。多阶段构建让运行镜像只保留「二进制 + 证书 + 时区」。

## 二、distroless 多阶段 Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.7
# deploy/docker/gateway.Dockerfile
FROM --platform=$BUILDPLATFORM golang:1.25-bookworm AS build
ARG TARGETOS TARGETARCH
WORKDIR /src
ENV CGO_ENABLED=0 GOFLAGS=-mod=readonly

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .

ARG VERSION=dev COMMIT=none BUILD_TIME=unknown
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -trimpath -buildvcs=false \
    -ldflags "-s -w -X mall-lite/pkg/version.Version=${VERSION} -X mall-lite/pkg/version.Commit=${COMMIT} -X mall-lite/pkg/version.BuildTime=${BUILD_TIME}" \
    -o /out/gateway ./app/gateway/cmd

FROM gcr.io/distroless/static-debian12:nonroot
ARG VERSION=dev COMMIT=none
LABEL org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.revision="${COMMIT}"
WORKDIR /app
COPY --from=build /out/gateway /app/gateway
ENV TZ=Asia/Shanghai GOMEMLIMIT=400MiB
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/app/gateway"]
```

要点：`ARG` 必须在**每个**使用它的阶段重新声明；`FROM --platform=$BUILDPLATFORM` 让构建在宿主架构上编译目标架构产物，配合 `TARGETOS/TARGETARCH` 完成交叉编译；distroless 自带 CA 证书与 `/etc/passwd`，`nonroot` 是 UID 65532。

## 三、scratch 变体：证书和时区自己带

```dockerfile
FROM alpine:3.21 AS certs
RUN apk add --no-cache ca-certificates tzdata

FROM scratch
COPY --from=certs /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=certs /usr/share/zoneinfo/Asia/Shanghai /usr/share/zoneinfo/Asia/Shanghai
COPY --from=build /out/gateway /app/gateway
ENV TZ=Asia/Shanghai
USER 65532:65532
ENTRYPOINT ["/app/gateway"]
```

scratch 里没有 `/tmp`、没有 `/etc/passwd`、没有 `/etc/resolv.conf`。因此：`USER` 必须写数字 UID（`nonroot` 无法解析）；开了 `readOnlyRootFilesystem` 后要写文件必须挂 `emptyDir` 到 `/tmp`；DNS 依赖 Go 的纯 Go resolver + kubelet 注入的 resolv.conf，不要自带一份。

## 四、非 root、PID1 与信号

- 非 root：`USER 65532:65532`，与 K8s 的 `runAsUser` 保持一致，否则 `runAsNonRoot` 校验会拒绝启动
- 只用 exec 形式 `ENTRYPOINT ["/app/gateway"]`；写成 `ENTRYPOINT /app/gateway` 会引入 `/bin/sh -c`，SIGTERM 发给 shell，Go 进程收不到，只能等 30s 后被 SIGKILL
- Go 程序自己是 PID1 时没有僵尸进程问题，不需要额外塞 tini；但必须实现 `srv.Shutdown(ctx)`，见 [03-基本特性/07-优雅退出](../03-基本特性/07-优雅退出.md)

## 五、HEALTHCHECK：distroless 没有 curl

alpine 版本可以用 `wget`/`curl` 加 `HEALTHCHECK CMD`；distroless/scratch 没有 shell 和 wget，改用应用自带子命令：

```go
// app/gateway/cmd/main.go 片段
healthcheck := flag.Bool("healthcheck", false, "probe local /healthz and exit")
flag.Parse()
if *healthcheck {
    resp, err := http.Get("http://127.0.0.1:8080/healthz")
    if err != nil || resp.StatusCode != http.StatusOK {
        os.Exit(1)
    }
    os.Exit(0)
}
```

```dockerfile
HEALTHCHECK CMD ["/app/gateway", "-healthcheck"]
```

Docker 的 HEALTHCHECK 只影响 `docker ps` 状态，K8s 不读它；上 K8s 后探针交给 `readinessProbe`，见下一篇。

## 六、.dockerignore 与层缓存

```text
.git
.github
bin
coverage.out
*.md
*.exe
.env
.vscode
```

```bash
docker build -f deploy/docker/gateway.Dockerfile -t mall/gateway:dev .
```

层缓存的关键是顺序：`COPY go.mod go.sum` → `go mod download` → `COPY . .`。只改业务代码时模块层命中缓存；`RUN --mount=type=cache` 让重复构建的编译产物也复用，典型二次构建从 90s 降到 10s 内。`.dockerignore` 同时避免 `.git`、`.env` 被打进构建上下文甚至镜像。

## 七、buildx 多架构与推送

```bash
docker buildx create --name mall --use --bootstrap
VERSION=$(git describe --tags --always) COMMIT=$(git rev-parse --short HEAD)
docker buildx build --platform linux/amd64,linux/arm64 \
  --build-arg VERSION="$VERSION" --build-arg COMMIT="$COMMIT" \
  --build-arg BUILD_TIME="$(git show -s --format=%cI HEAD)" \
  -f deploy/docker/gateway.Dockerfile \
  -t registry.example.com/mall/gateway:"$VERSION" --push .
```

```powershell
$env:VERSION = git describe --tags --always
$env:COMMIT  = git rev-parse --short HEAD
docker buildx build --platform linux/amd64,linux/arm64 `
  --build-arg VERSION=$env:VERSION --build-arg COMMIT=$env:COMMIT `
  -f deploy/docker/gateway.Dockerfile -t registry.example.com/mall/gateway:$env:VERSION --push .
```

多架构镜像在仓库里是一个 manifest list，节点按自身架构拉取。`--push` 是必须的：`--load` 无法装多架构镜像。tag 同时打 `$VERSION` 与 `$VERSION-$Commit`，生产只用后者或直接锁 digest。

## 八、体积对比与漏洞扫描

| 方案 | 基础镜像 | 体积 | 适用 |
| --- | --- | --- | --- |
| 单阶段 | golang:1.25 | 约 900 MB | 本地调试 |
| alpine | alpine:3.21 | 约 20 MB | 需要 shell 排障 |
| distroless static | distroless/static | 约 12 MB | 生产默认 |
| scratch | 无 | 约 10 MB | 极致精简、无 CGO |

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 \
  --ignore-unfixed registry.example.com/mall/gateway:v1.4.0
docker run --rm -p 8080:8080 --read-only --cap-drop ALL \
  --security-opt no-new-privileges mall/gateway:v1.4.0
```

distroless/scratch 的 CVE 基本只来自那个二进制本身，所以扫描重点变成：基础镜像 tag 是否浮动、Go 依赖版本是否有已知漏洞（`govulncheck ./...` 一起跑）。

## 九、本地联调 compose

```yaml
# deploy/docker/compose.yaml
services:
  gateway:
    build:
      context: ../..
      dockerfile: deploy/docker/gateway.Dockerfile
    image: mall/gateway:dev
    ports: ["8080:8080"]
    environment:
      MALL_APP_ENV: dev
      MALL_MYSQL_DSN: "root:root@tcp(mysql:3306)/mall?charset=utf8mb4&parseTime=True&loc=Local"
      MALL_REDIS_ADDR: "redis:6379"
    depends_on:
      mysql: {condition: service_healthy}
      redis: {condition: service_started}
    read_only: true
    tmpfs: ["/tmp"]
  mysql:
    image: mysql:8.4
    environment: {MYSQL_ROOT_PASSWORD: root, MYSQL_DATABASE: mall}
    ports: ["3306:3306"]
    healthcheck: {test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-uroot", "-proot"], interval: 5s, retries: 20}
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

```bash
docker compose -f deploy/docker/compose.yaml up -d --build
docker compose -f deploy/docker/compose.yaml logs -f gateway
```

## 常见坑

1. **构建时开了 CGO**：运行镜像里没有 libc，启动即 `no such file or directory`；builder 阶段必须 `CGO_ENABLED=0`。
2. **DNS 异常**：scratch 自带 `/etc/resolv.conf` 会覆盖 kubelet 注入的，导致找不到 Service；不要 COPY resolv.conf。
3. **时区差 8 小时**：只设 `TZ` 但没有 zoneinfo 文件，Go 会回退 UTC；scratch 必须显式 COPY 时区文件。
4. **信号收不到**：`ENTRYPOINT gateway`（shell 形式）导致 SIGTERM 打到 sh，进程被 SIGKILL，连接与日志全部中断。
5. **只推 `latest`**：无法定位版本也无法快速回滚；至少双 tag，回滚按 digest。
6. **HEALTHCHECK 用 curl/wget 但镜像里没有**：容器永远 unhealthy，distroless 用应用自带子命令。

## 小结

- 多阶段构建把工具链留在 builder，运行镜像只保留二进制与必要证书、时区
- distroless 更小更安全，但没有 shell，健康检查必须由应用自己提供
- 非 root + 只读根文件系统 + exec 形式 ENTRYPOINT 是信号与安全的基本盘
- buildx 一次产出多架构镜像，trivy/go vulncheck 作为流水线门禁

## 练习

1. 分别用 alpine、distroless、scratch 构建同一服务，用 `docker images` 对比体积与 `trivy` 结果。
2. 把 `ENTRYPOINT` 改成 shell 形式，`docker stop` 观察退出耗时与日志是否有「优雅关闭」记录。
3. 用 buildx 构建双架构镜像推到本地 registry，在 arm64 环境（或 qemu）验证可运行。

## 官方文档

- Docker 多阶段构建：<https://docs.docker.com/build/building/multi-stage/>
- buildx 多平台构建：<https://docs.docker.com/build/building/multi-platform/>
- distroless 镜像说明：<https://github.com/GoogleContainerTools/distroless>
- Trivy 镜像扫描：<https://trivy.dev/latest/docs/target/container_image/>
