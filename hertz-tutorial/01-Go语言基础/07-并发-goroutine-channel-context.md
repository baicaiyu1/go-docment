# 并发：goroutine、channel 与 context

> Go 的并发不是「多开线程」，而是用 goroutine + channel + context 组织任务；Hertz 处理器的 `ctx` 就是 `context.Context`，这一篇必须吃透。

## 本篇目标

- 理解 goroutine 的启动方式与生命周期，知道「main 退出会杀掉所有 goroutine」
- 掌握无缓冲/有缓冲 channel 的阻塞语义，会正确关闭与遍历 channel
- 用 `select` 实现多路等待、超时与非阻塞尝试
- 会用 `sync.WaitGroup` 与 `sync.Mutex` 解决等待与共享数据竞争
- 掌握 `context` 的取消、超时与传值，能写出「请求级超时」的代码

## 前置知识

- 已完成 [06-切片-map-字符串](06-切片-map-字符串.md)
- 理解函数、闭包与 `defer`
- 知道 `error` 的返回惯例

## 一、goroutine：用 go 关键字启动

```go
go func() {
	fmt.Println("我在另一个 goroutine 里")
}()
```

三点必须记住：

1. **goroutine 非常轻**：初始栈约 2 KB，可以轻松开十万个
2. **main 返回即进程退出**，所有未完成的 goroutine 被直接杀掉
3. **没有返回值**：要拿结果必须通过 channel 或共享变量 + 锁

最后一点最容易踩：`go fmt.Println("hello")` 可能来不及执行，main 就退出了。正确做法是等待：

```go
var wg sync.WaitGroup
wg.Add(1)
go func() {
	defer wg.Done()
	fmt.Println("hello")
}()
wg.Wait()
```

## 二、channel：goroutine 之间的管道

```go
ch := make(chan int)     // 无缓冲
buf := make(chan int, 3) // 有缓冲，容量 3
```

无缓冲 channel 的发送与接收必须同时就绪：

```go
ch := make(chan string)
go func() {
	ch <- "done" // 阻塞，直到有人接收
}()
msg := <-ch // 阻塞，直到有人发送
fmt.Println(msg)
```

它是**同步点**：双方在这一点「握手」后才继续。

有缓冲 channel 在缓冲未满时不阻塞，意义是**解耦生产与消费的速度**，但缓冲不是越大越好，过大会掩盖背压问题。

关闭与遍历：

```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
close(ch) // 由发送方关闭，表示不会再有数据

for v := range ch { // 关闭后自动结束
	fmt.Println(v)
}
```

规则：**只有发送方应该关闭 channel**；向已关闭的 channel 发送会 panic，接收会立即返回零值；关闭已关闭的 channel 也会 panic；只读/只写方向可以用类型约束 `func f(in <-chan int, out chan<- int)`。

## 三、select：多路等待

`select` 同时等待多个 channel 操作，哪个先就绪就执行哪个：

```go
select {
case v := <-ch1:
	fmt.Println("来自 ch1:", v)
case ch2 <- 42:
	fmt.Println("已写入 ch2")
case <-time.After(time.Second):
	fmt.Println("超时")
default:
	fmt.Println("没有任何 channel 就绪")
}
```

- 有 `default` 时是**非阻塞**尝试，常用于「快速取结果」
- 没有 `default` 且所有 case 都未就绪时会阻塞
- 多个 case 同时就绪时随机选一个，不要依赖顺序

更推荐用 `context` 的超时（见第五节），因为取消信号可以沿调用链传递。超时模式形如 `case <-time.After(time.Second)`。

## 四、sync.WaitGroup 与 sync.Mutex

`WaitGroup` 等待一组 goroutine 结束：

```go
var wg sync.WaitGroup
for i := 0; i < 3; i++ {
	wg.Add(1)
	go func(id int) {
		defer wg.Done()
		fmt.Println("worker", id)
	}(i)
}
wg.Wait()
```

**`Add` 必须在 `go` 之前调用**。如果写在 goroutine 内部，`Wait` 可能在 `Add` 之前执行并直接返回。

`Mutex` 保护共享数据：

```go
var (
	mu    sync.Mutex
	count int
)

for i := 0; i < 100; i++ {
	go func() {
		mu.Lock()
		count++ // 临界区
		mu.Unlock()
	}()
}
```

读多写少用 `sync.RWMutex`：读操作 `RLock/RUnlock` 可并发，写操作 `Lock/Unlock` 互斥。用 `go test -race` 可以检测数据竞争，并发代码务必开启。

## 五、context：取消、超时与传值

`context.Context` 是 Go 并发控制的「信号总线」，也是 Hertz 处理器第一个参数的类型。

```go
ctx := context.Background() // 根 context，通常用于 main
ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
defer cancel() // 必须调用，否则泄漏计时器，go vet 会提示 lostcancel

ctx = context.WithValue(ctx, key, value) // 传值
```

监听取消：

```go
select {
case <-ctx.Done():
	return ctx.Err() // context.Canceled 或 context.DeadlineExceeded
case v := <-resultCh:
	return v
}
```

完整示例：并发调用两个下游并设置总超时，`order` 需要 200ms 而预算只有 100ms，所以必然超时：

```go
func fetch(ctx context.Context, name string, cost time.Duration) (string, error) {
	select {
	case <-time.After(cost):
		return name + " 返回成功", nil
	case <-ctx.Done():
		return "", fmt.Errorf("调用 %s 被取消: %w", name, ctx.Err())
	}
}

func aggregate(ctx context.Context) ([]string, error) {
	ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
	defer cancel()

	type result struct {
		val string
		err error
	}
	ch := make(chan result, 2)
	for _, job := range []struct {
		name string
		cost time.Duration
	}{{"user", 60 * time.Millisecond}, {"order", 200 * time.Millisecond}} {
		go func(name string, cost time.Duration) {
			v, err := fetch(ctx, name, cost)
			ch <- result{v, err}
		}(job.name, job.cost)
	}

	out := make([]string, 0, 2)
	for i := 0; i < 2; i++ {
		select {
		case r := <-ch:
			if r.err != nil {
				return nil, r.err
			}
			out = append(out, r.val)
		case <-ctx.Done():
			return nil, fmt.Errorf("聚合超时: %w", ctx.Err())
		}
	}
	return out, nil
}
```

这个模式在微服务里叫「请求级超时」，Hertz 的 `c.Request.Context()` 就是请求级 context。

`WithValue` 的正确用法：

```go
type ctxKey string

const requestIDKey ctxKey = "request_id"

ctx = context.WithValue(ctx, requestIDKey, "trace-123")
id, _ := ctx.Value(requestIDKey).(string)
```

规则：key 用**自定义类型**不要用 `string`，避免不同包撞 key；只放**请求级、不可变**的元数据（trace id、用户 id）；`WithValue` 是链式查找，不要滥用。**所有下游调用都要传递同一个 ctx**，一旦超时整条链路一起取消，不会留下僵尸请求。

## 常见坑

1. **goroutine 泄漏**：向无人接收的 channel 发送会永久阻塞。用带缓冲 channel 或 `select + ctx.Done()` 兜底。
2. **`WaitGroup.Add` 写在 goroutine 内**：`Wait` 可能先返回，导致漏等。
3. **忘记 `cancel()`**：定时器与子 context 泄漏，`go vet` 会报 `lostcancel`。
4. **用 `context.Background()` 替换请求 ctx**：取消信号传不下去，超时失效。

## 小结

goroutine 负责「并发执行」，channel 负责「通信与同步」，`select` 负责「多路选择」，`context` 负责「取消与超时」。记住一句话：**不要通过共享内存来通信，而要通过通信来共享内存**。下一篇 [08-包管理与项目结构](08-包管理与项目结构.md) 把这些代码组织成可维护的工程。

## 练习

1. 用 `WaitGroup` + `Mutex` 实现一个并发安全的计数器，启动 100 个 goroutine 各加 1，用 `go test -race` 验证没有数据竞争。
2. 实现 `func fanIn(ctx context.Context, chs ...<-chan int) <-chan int`，把所有输入 channel 合并到一个输出 channel，并在 ctx 取消时退出。
3. 改造第五节的 `aggregate`，让它在某个下游失败时立即返回错误（提示：用 `errgroup` 思想，或自己用 cancel + 收集首个错误）。

## 官方文档

- Go 并发模式：<https://go.dev/blog/pipelines>
- `context` 包：<https://pkg.go.dev/context>
- `sync` 包：<https://pkg.go.dev/sync>
- 数据竞争检测：<https://go.dev/doc/articles/race_detector>
