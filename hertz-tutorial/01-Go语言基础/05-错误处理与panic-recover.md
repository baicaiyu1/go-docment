# 错误处理与 panic/recover

> Go 用返回值而不是异常表达「可预期的失败」；这一篇讲清 error 的包装与判定，以及 panic/recover 的正确使用边界。

## 本篇目标

- 理解 `error` 是一个普通接口，掌握「最后一个返回值是 error」的惯例
- 会用 `errors.New`、`fmt.Errorf` 创建错误，用 `%w` 包装并保留错误链
- 掌握 `errors.Is` / `errors.As` 的判定方式，能区分哨兵错误与自定义错误类型
- 知道 `panic` / `recover` 只该用在「程序无法继续」的场景，并能写出安全的 recover 中间件
- 能为 Web 服务设计一套「错误 → HTTP 状态码」的映射

## 前置知识

- 已完成 [04-结构体-方法-接口](04-结构体-方法-接口.md)
- 理解接口的隐式实现与类型断言
- 会用 `fmt.Printf` 与 `fmt.Sprintf`

## 一、error 是什么

`error` 是标准库里的一个接口，定义简单到只有一行：

```go
type error interface {
	Error() string
}
```

任何实现了 `Error() string` 的类型都是错误。Go 没有异常机制，函数出错时把错误作为**最后一个返回值**返回。

**为什么用返回值而不是异常？**

1. 错误是函数签名的一部分，调用方无法「假装看不见」
2. 控制流线性，读代码不需要在脑海里维护「哪里可能抛异常」
3. 错误可以被当作普通值传递、包装、聚合

代价是代码里会有大量 `if err != nil`，这是 Go 的取舍，习惯后反而觉得清晰。

## 二、创建错误：errors.New 与 fmt.Errorf

```go
import "errors"

var ErrUserNotFound = errors.New("user not found")

func find(id int64) error {
	if id <= 0 {
		return fmt.Errorf("非法用户 ID: %d", id)
	}
	return nil
}
```

- `errors.New`：创建**不带动态信息**的固定错误，通常定义为包级变量（哨兵错误）
- `fmt.Errorf`：需要拼接上下文时使用
- **不要用 `errors.New(fmt.Sprintf(...))`**，直接 `fmt.Errorf` 即可

哨兵错误的命名惯例是 `ErrXxx`，并且**导出**，让调用方可以判定，例如 `ErrUserNotFound`、`ErrUserExists`。

## 三、包装：%w 与错误链

直接返回底层错误会丢失上下文，调用方只看到 `connection refused`，不知道是查用户还是查订单。用 `%w` 包装：

```go
func (s *UserService) Get(ctx context.Context, id int64) (*User, error) {
	u, err := s.repo.FindByID(ctx, id)
	if err != nil {
		return nil, fmt.Errorf("查询用户 %d: %w", id, err)
	}
	return u, nil
}
```

`%w` 与 `%v` 的区别：

| 写法 | 输出 | 是否保留错误链 |
| --- | --- | --- |
| `fmt.Errorf("...: %v", err)` | 字符串被拼接 | 否，`errors.Is` 失效 |
| `fmt.Errorf("...: %w", err)` | 字符串被拼接 | 是，可 `Unwrap` |

错误链可以多层，例如「查询用户 42: 读取数据库: connection refused」。每一层都保留了「我在做什么」，最终错误仍能被 `errors.Is(err, ErrUserNotFound)` 命中。一般只包装**根因**，不要重复包装。

## 四、判定：errors.Is 与 errors.As

`errors.Is` 判断错误链中是否有某个哨兵错误：

```go
u, err := svc.Get(ctx, 42)
if errors.Is(err, ErrUserNotFound) {
	// 返回 404
}
```

**不要写 `err == ErrUserNotFound`**：一旦错误被 `%w` 包装，直接比较就失败了。

`errors.As` 从错误链中取出特定类型。自定义错误类型可以携带字段：

```go
type ValidationError struct {
	Field string
	Msg   string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("字段 %s 校验失败: %s", e.Field, e.Msg)
}

var ve *ValidationError
if errors.As(err, &ve) {
	fmt.Printf("字段 %s 不合法: %s\n", ve.Field, ve.Msg)
}
```

`errors.As` 的第二个参数必须是**指向错误类型的指针**（这里是 `**ValidationError`），这样才能把值写回去。

什么时候用哪个：错误是「一个固定的标记」→ 哨兵错误 + `errors.Is`；错误需要「携带结构化信息」→ 自定义类型 + `errors.As`；两者可以叠加。

## 五、为 Web 服务设计错误映射

真实项目里，Service 层返回领域错误，Handler 层负责翻译成 HTTP 状态码：

```go
func writeError(w http.ResponseWriter, err error) {
	var ve *ValidationError

	switch {
	case errors.Is(err, ErrUserNotFound):
		w.WriteHeader(http.StatusNotFound)
		_, _ = w.Write([]byte(`{"code":404,"msg":"用户不存在"}`))
	case errors.As(err, &ve):
		w.WriteHeader(http.StatusBadRequest)
		fmt.Fprintf(w, `{"code":400,"msg":%q}`, ve.Error())
	default:
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte(`{"code":500,"msg":"内部错误"}`))
	}
}
```

要点：

1. **Handler 不关心错误细节**，只做映射，业务错误定义留在 Service 层
2. **对外不暴露内部错误**，500 只返回通用文案，详细信息写日志
3. **顺序重要**：`errors.Is` / `errors.As` 从具体到通用排列

Hertz 的错误处理（[03-基本特性/06-错误处理](../03-基本特性/06-错误处理.md)）会用到完全相同的分层思路。

## 六、panic 与 recover

`panic` 表示「程序遇到了无法继续的状态」，例如数组越界、空指针解引用、显式 `panic(...)`。它会终止当前函数的正常执行，逐层执行 `defer`，然后让程序崩溃。

**不要用 panic 表达业务错误**，例如「用户不存在」是可预期情况。正确用途：初始化阶段配置缺失、编程错误（不可能到达的分支）、依赖库的断言失败。

`recover` 只有在 **defer 函数中** 调用才有效，且只能捕获**同一个 goroutine** 的 panic：

```go
func safeCall(fn func()) (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("panic recovered: %v", r)
		}
	}()
	fn()
	return nil
}
```

HTTP 服务通常会在最外层中间件里 recover，避免单个请求的 panic 拖垮整个进程：

```go
func Recovery(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				log.Printf("panic: %v\n%s", rec, debug.Stack())
				http.Error(w, `{"code":500,"msg":"内部错误"}`, http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

注意：**recover 之后不要静默吞掉**，必须记录日志与堆栈；**不要 recover 后继续执行业务逻辑**，这个请求已处于不一致状态；子 goroutine 里的 panic 不会被外层 recover 捕获，需要子 goroutine 自己 defer recover。

## 常见坑

1. **`err == ErrXxx` 判定**：包装后必然失败，统一用 `errors.Is`。
2. **用 `%v` 代替 `%w`**：错误链断了，上层无法判定根因。
3. **recover 写在普通函数里**：不在 defer 中调用 `recover()` 永远返回 nil。
4. **以为 recover 能救子 goroutine**：每个 goroutine 的 panic 必须自己 recover。
5. **panic 当作 return 用**：业务错误用 panic 会导致调用方无法预期，且性能差。
6. **把内部错误直接返回给用户**：泄露表名、SQL、内网地址，属于安全问题。

## 小结

`error` 是值，`%w` 保留链，`errors.Is/As` 做判定，panic 只用于不可恢复场景，recover 只在 defer 中有效。这套规则会在 Hertz 的错误处理、中间件、客户端调用中反复出现。下一篇 [06-切片-map-字符串](06-切片-map-字符串.md) 回到数据结构，讲清最容易出 bug 的切片共享问题。

## 练习

1. 定义 `ErrOrderNotFound` 与 `ValidationError`，写一个 `OrderService.Get`，内部包装仓储错误，外部用 `errors.Is` 与 `errors.As` 分别判定并打印不同信息。
2. 实现 `Recovery` 中间件，捕获 panic 后输出堆栈并返回 500；再写一个测试函数手动触发 panic，验证进程没有崩溃。
3. 写 `classify(err error) int`，把「未找到 → 404」「校验失败 → 400」「其他 → 500」映射为状态码，并用表驱动方式验证三类输入。

## 官方文档

- `errors` 包：<https://pkg.go.dev/errors>
- 错误处理与 Go：<https://go.dev/blog/error-handling-and-go>
- 错误包装提案说明：<https://go.dev/blog/go1.13-errors>
- `panic` 与 `recover`：<https://go.dev/ref/spec#Handling_panics>
