# Go Module 与国内代理

> 让依赖「下得动、下得对、下得快」：`go.mod`/`go.sum` 是什么、GOPROXY 怎么配、私有仓库与 vendor 怎么处理。

## 本篇目标

- 理解 Go Modules 解决了什么问题，以及它与旧 GOPATH 模式的区别
- 读懂 `go.mod` / `go.sum` 的每一行，知道哪些必须提交、哪些不能手改
- 熟练使用 `go mod init / tidy / download`，能解释 `tidy` 做了什么
- 配好国内代理并理解 `GOPRIVATE`、`GOSUMDB` 的作用
- 独立排查 `dial tcp timeout`、`410 Gone`、`checksum mismatch` 三类报错

## 前置知识

- 已完成 [02-Go-安装与环境配置](02-Go-安装与环境配置.md)，`go version` 正常
- 知道「模块是一组包的集合」，细节见 [08-包管理与项目结构](../01-Go语言基础/08-包管理与项目结构.md)
- 能使用终端执行命令，并具备基本网络排查意识

## 一、为什么需要 Modules

Go 1.11 之前，代码必须放在 `GOPATH/src`，依赖版本不可控：项目 A 要 v1.2、项目 B 要 v2.0，只能人工切换。Go Modules 带来三件事：

1. 项目可放在任意目录，不再强制 `GOPATH/src`
2. 依赖版本显式写在 `go.mod`，构建可复现
3. `go.sum` 校验依赖内容，别人篡改会被发现

最小结构：

```text
go-lab/
├── go.mod
├── go.sum        # 有依赖后自动生成
└── main.go
```

## 二、初始化模块与两个文件

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\go-lab" | Out-Null
Set-Location "$HOME\go-lab"
go mod init go-lab
```

```bash
mkdir -p ~/go-lab && cd ~/go-lab
go mod init go-lab
```

`go mod init` 后跟**模块路径**，不是目录名：

- 会发布到 GitHub：`github.com/你的用户名/仓库名`
- 仅本地练习：`go-lab` 这类短名
- 模块路径就是别人 `import` 时写的字符串，**发布后不要随意改**

生成的 `go.mod` 只有两行：`module go-lab` 与 `go 1.25`。后者声明语言版本，决定编译器按哪个语义检查代码。

引入一个依赖：

```go
package main

import (
	"fmt"

	"github.com/google/uuid"
)

func main() {
	fmt.Println(uuid.NewString())
}
```

```text
go mod tidy
go run .
```

`go.mod` 追加 `require github.com/google/uuid v1.6.0`；`go.sum` 记录两行哈希：

```text
github.com/google/uuid v1.6.0 h1:NIvaJDMOsjHA8n1jAhLSgzrAzy1Hgr+hNrb57e+94F0=
github.com/google/uuid v1.6.0/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
```

- `require`：直接依赖；带 `// indirect` 的是间接依赖（你没直接 import）
- 不带 `/go.mod` 的行是模块内容哈希，带的是 `go.mod` 文件哈希
- **`go.mod` 与 `go.sum` 都必须提交**；`GOMODCACHE` 缓存目录不提交

## 三、三个最常用命令

| 命令 | 作用 | 时机 |
| --- | --- | --- |
| `go mod init <path>` | 创建 `go.mod` | 新项目第一步 |
| `go mod tidy` | 删除无用依赖、补齐缺失依赖 | 每次改完 import |
| `go mod download` | 只下载依赖到缓存 | CI 预热、离线准备 |

`go mod tidy` 实际做了：扫描所有 `.go`（含测试）的 import → 补齐缺失 require → 删除无引用依赖 → 更新 `go.sum`。

所以**不要手改 `go.mod` 的版本号**，改代码后跑 tidy 才是正确姿势。升级依赖用：

```text
go get github.com/google/uuid@v1.6.0
go get -u ./...
```

## 四、GOPROXY 与 GOSUMDB：国内加速

默认 `https://proxy.golang.org,direct` 在国内基本不可用。推荐：

```text
go env -w GOPROXY=https://goproxy.cn,direct
```

含义：

- `https://goproxy.cn`：七牛公共代理，缓存绝大多数公开模块
- `direct`：代理返回 404/410 时回源到原始地址
- 逗号表示按序尝试；网络错误会直接终止，只有 404/410 才继续

再加校验库镜像，避免代理通了但校验超时：

```text
go env -w GOSUMDB=sum.golang.google.cn
```

改完用 `go env GOPROXY` 确认，再 `go clean -modcache` 后重新下载，几十秒内完成即正常。

## 五、GOPRIVATE 与 vendor

公司内部模块不能走公共代理，否则 404 且泄露路径。声明私有域名：

```text
go env -w GOPRIVATE=git.company.com,github.com/your-org/*
```

它同时隐式设置 `GONOPROXY`（不走代理）与 `GONOSUMDB`（不做公共校验）。私有仓库需要认证时，让 Git 走 SSH：`git config --global url."git@github.com:".insteadOf "https://github.com/"`。

`vendor` 把依赖复制进项目，构建不再依赖网络：

```text
go mod vendor
go build -mod=vendor ./...
```

适合 CI 求稳、合规审计；代价是仓库变大、升级依赖后必须重新 vendor。若项目存在 `vendor/` 且 `go.mod` 的 go 版本 ≥ 1.14，`go build` 默认自动使用 vendor，忘了重新 vendor 就会出现「本地能跑、CI 挂掉」。

## 六、三类网络报错速查

**1. `dial tcp ... i/o timeout`** — 八成是代理没生效：

```text
go env -w GOPROXY=https://goproxy.cn,direct
go clean -modcache
go mod tidy
```

仍失败则检查公司 HTTP 代理、DNS 污染（`nslookup goproxy.cn`）。

**2. `410 Gone` / `404 Not Found`** — 私有模块被公共代理拦截就加 `GOPRIVATE`；模块路径拼错或仓库改名就到 pkg.go.dev 确认；版本不存在用 `go list -m -versions <module>` 查看。

**3. `checksum mismatch` / `SECURITY ERROR`** — 先清缓存与 `go.sum` 再重建：

```powershell
go clean -modcache
Remove-Item go.sum -ErrorAction SilentlyContinue
go mod tidy
```

```bash
go clean -modcache
rm -f go.sum
go mod tidy
```

生产环境不要长期用 `GONOSUMDB` 关闭校验。

## 常见坑

1. **只设 `GOPROXY` 忘了 `GOSUMDB`**：代理通了但校验库连不上，依然卡住。
2. **手改 `go.mod` 版本**：容易漏改 `go.sum`，导致校验失败。
3. **把 `go.sum` 加入 `.gitignore`**：构建不可复现，团队其他人的校验失效。
4. **`GOPRIVATE` 写 `*.company.com`**：Go 不支持这种通配，写 `company.com` 或具体域名。
5. **vendor 与 `go.mod` 不同步**：改了依赖却没重新 vendor，CI 必挂。
6. **在 `GOPATH/src` 下用 module**：与旧模式混淆，项目放任意非 GOPATH 目录。

## 小结

记住链路：`go mod init` 建模块 → 写代码 import → `go mod tidy` 补依赖 → GOPROXY 下载 → go.sum 校验。Gin/Kratos 项目的依赖管理完全一致。下一篇 [04-开发工具链与IDE](04-开发工具链与IDE.md) 把这些命令串成日常流程。

## 练习

1. 新建 `mod-demo`，`go mod init example.com/mod-demo`，引入 `github.com/google/uuid` 并打印 UUID，观察两个文件变化。
2. 清空 `GOPROXY` 后执行 `go clean -modcache && go mod download`，记录报错原文，再恢复代理对比耗时。
3. 执行 `go mod vendor`，观察目录结构并用 `go build -mod=vendor` 构建，思考何时会选择 vendor。

## 官方文档

- Go Modules 参考：<https://go.dev/ref/mod>
- 模块教程：<https://go.dev/doc/tutorial/create-module>
- goproxy.cn：<https://goproxy.cn/>
- 校验数据库：<https://sum.golang.org/>
