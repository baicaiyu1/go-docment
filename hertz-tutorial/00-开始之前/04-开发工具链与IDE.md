# 开发工具链与 IDE

> 把 Go 自带命令和编辑器配成一条顺手的流水线：格式化、静态检查、构建、运行、测试、调试、版本管理。

## 本篇目标

- 掌握 `go fmt / vet / build / run / test` 的作用与典型用法
- 理解格式化不是风格偏好，而是团队协作的硬约束
- 在 VS Code 与 GoLand 中配好 gopls、dlv，能打断点调试
- 会用最基础的 Git 命令管理练习代码，写出正确的 `.gitignore`
- 知道项目代码该放在磁盘的什么位置，规避路径相关的坑

## 前置知识

- 已完成 [03-Go-Module与国内代理](03-Go-Module与国内代理.md)，能正常拉依赖
- 有一个可用的 `~/hertz-lab` 模块目录
- 已安装 VS Code 或 GoLand

## 一、工具链全景

| 命令 | 作用 | 频率 |
| --- | --- | --- |
| `go fmt ./...` | 按官方风格格式化 | 保存/提交前 |
| `go vet ./...` | 静态检查可疑代码 | 提交前必跑 |
| `go build ./...` | 编译检查全仓库 | 提交前必跑 |
| `go run .` | 编译并立即运行 | 开发时高频 |
| `go test ./...` | 运行测试 | 提交前必跑 |
| `go test -bench . -benchmem` | 基准与内存分配 | 性能改动时 |
| `go install` | 安装可执行文件到 GOBIN | 装工具时 |

关键认知：**`go run` 不是解释执行**，它先编译成临时二进制再运行，所以语法错误会在运行前暴露，速度与 `go build` 一致。

准备一个练习程序：

```go
// main.go
package main

import "fmt"

func main() {
	name := "Hertz"
	fmt.Printf("hello, %s\n", name)
}
```

## 二、格式化、检查、构建与测试

`gofmt` 是唯一格式标准，没有「几个空格」的争论：

```powershell
go fmt ./...
gofmt -l .        # 只列出需要格式化的文件
```

```bash
go fmt ./...
gofmt -l .        # 只列出需要格式化的文件
```

格式不统一时，评审里全是空白 diff，真正的逻辑问题被淹没；CI 常加 `gofmt -l .`，格式不对直接拒绝合并。

`go vet` 抓「能编译但很可能写错」的代码，例如 `fmt.Printf("%d\n", "hello")`（`%d` 收到 string）、拷贝含 `sync.Mutex` 的结构体、结构体标签错误：

```powershell
go vet ./...
go build ./...
```

```bash
go vet ./...
go build ./...
```

构建与运行：

```powershell
go build -o bin\app.exe .
.\bin\app.exe
```

```bash
go build -o bin/app .
./bin/app
```

- `go run .`：开发期验证，产物在临时目录，退出即丢
- `go build`：产出可分发的静态二进制，同架构机器无需装 Go 即可运行
- `go build ./...`：编译所有包但不产物，适合 CI 快速检查全仓库

交叉编译（为部署准备）：

```powershell
$env:GOOS = "linux"; $env:GOARCH = "amd64"
go build -o bin/app-linux .
```

```bash
GOOS=linux GOARCH=amd64 go build -o bin/app-linux .
```

测试命令：

```powershell
go test ./...
go test -v -run TestLogin ./...
go test -cover ./...
go test -race ./...
```

```bash
go test ./...
go test -v -run TestLogin ./...
go test -cover ./...
go test -race ./...
```

`-race` 明显变慢，但能抓真实数据竞争，并发代码必须开。写法见 [10-测试与基准测试](../01-Go语言基础/10-测试与基准测试.md)。

## 三、VS Code、GoLand 与 gopls

VS Code 安装官方 Go 插件后，确认三件事：`gopls` 正常运行、保存自动格式化、`dlv` 可调试（`Go: Install/Update Tools` 安装）。推荐设置：

```json
{
  "go.useLanguageServer": true,
  "go.formatTool": "gofmt",
  "go.lintTool": "staticcheck",
  "[go]": {
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": { "source.organizeImports": "explicit" }
  }
}
```

GoLand 只需检查：SDK 指向 `GOROOT`（Settings → Go → GOROOT）、启用 Go Modules、File Watchers 配 gofmt on save、Inspections 启用 go vet。

`gopls` 是官方语言服务器，负责跳转、补全、诊断。跳转失效时先 `Go: Restart Language Server`；import 标红但能编译，通常是没跑 `go mod tidy` 或缓存旧。

## 四、用 dlv 调试

调试比 `fmt.Println` 高效，尤其是并发与复杂分支。VS Code 的 `.vscode/launch.json`：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug main",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}",
      "args": [],
      "env": { "APP_ENV": "dev" }
    }
  ]
}
```

在 `fmt.Printf` 行左侧点断点，按 `F5` 停住后可看变量、单步（`F10` 跳过、`F11` 进入、`Shift+F5` 停止）。命令行方式：

```powershell
dlv debug .
# (dlv) break main.main
# (dlv) continue
# (dlv) print name
```

```bash
dlv debug .
# (dlv) break main.main
# (dlv) continue
# (dlv) print name
```

调试 Hertz 处理器时，dlv 能直接查看 `ctx` 里的值，这是日志做不到的。

## 五、Git、.gitignore 与项目位置

```powershell
git init
git add .
git commit -m "feat: 完成 Go 基础练习"
git log --oneline
```

```bash
git init
git add .
git commit -m "feat: 完成 Go 基础练习"
git log --oneline
```

`.gitignore` 示例：

```text
bin/
*.exe
*.test
coverage.out
*.prof
.idea/
.vscode/
.env
```

注意：**`go.mod` 与 `go.sum` 绝不能忽略**，它们是构建可复现的基础。

代码位置三原则：

1. 不放 `GOROOT`，升级 Go 会被覆盖
2. 尽量不放中文路径，规避工具链玄学问题
3. 不放 OneDrive/iCloud 同步目录，同步进程会在编译时锁文件

## 常见坑

1. **只在 IDE 点运行，从不敲命令**：CI 用的是 `go build ./...`，不熟命令行会在部署时抓瞎。
2. **忽略 `go vet`**：它抓的正是「能编译但会出错」的代码。
3. **把 `bin/`、`*.exe` 提交到 Git**：仓库膨胀且二进制与源码不一致。
4. **调试时改代码不重启**：断点对应编译时版本，改动后需重新启动。
5. **在同步盘里开发**：文件占用错误频发。
6. **两个 IDE 格式化配置不一致**：提交时产生无意义 diff。

## 小结

环境阶段到此结束：建模块 → 写代码 → 格式化 → vet → build → run → test → Git 提交。下一篇 [01-Go语言概览与第一个程序](../01-Go语言基础/01-Go语言概览与第一个程序.md) 正式开始学语言。

## 练习

1. 在 `00-toolchain` 子目录写一个打印当前时间的程序，依次跑 `go fmt`、`go vet`、`go build -o bin/clock`，再运行。
2. 在 `fmt.Println` 处打断点，用 VS Code 调试查看变量；再用 `dlv debug` 完成一次 `break / continue / print`。
3. 初始化 Git 仓库、写好 `.gitignore` 并提交，修改代码后看 `git diff`，确认 `bin/` 未被跟踪。

## 官方文档

- `go` 命令：<https://pkg.go.dev/cmd/go>
- `go vet`：<https://pkg.go.dev/cmd/vet>
- Delve：<https://github.com/go-delve/delve>
- VS Code Go 调试：<https://github.com/golang/vscode-go/wiki/Debugging>
