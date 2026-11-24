# 切片、map 与字符串

> 三个使用频率最高的数据结构，也是 bug 最集中的地方：切片共享底层数组、map 并发写崩溃、字符串按字节还是按字符。

## 本篇目标

- 分清数组与切片，理解 `len` / `cap` 与 `append` 的扩容行为
- 能解释并避开「切片共享底层数组」导致的隐蔽数据污染
- 正确使用 `copy`、删除、插入，知道三索引切片的用途
- 掌握 map 的增删改查、comma-ok、遍历无序与并发安全方案
- 熟练使用 `strings` / `strconv` 处理 Web 场景中的字符串与数字转换

## 前置知识

- 已完成 [05-错误处理与panic-recover](05-错误处理与panic-recover.md)
- 理解值传递与指针的基本概念
- 会写 `for range` 循环

## 一、数组 vs 切片

数组长度是类型的一部分，**赋值会整体拷贝**：

```go
var a [3]int = [3]int{1, 2, 3}
b := a // 拷贝整个数组
b[0] = 100
fmt.Println(a, b) // [1 2 3] [100 2 3]
```

切片是「指向底层数组的视图」，包含三个部分：指针、长度、容量。日常开发几乎只用切片，数组只在「长度固定且很小」时出现（如 `[32]byte` 做哈希）。

```go
s := []int{1, 2, 3}
fmt.Println(len(s), cap(s)) // 3 3
```

## 二、声明、初始化与 nil 切片

```go
var s1 []int            // nil 切片，len=0 cap=0
s2 := []int{}           // 空切片，非 nil
s3 := make([]int, 3)    // len=3 cap=3，元素全 0
s4 := make([]int, 0, 8) // len=0 cap=8，预留容量
```

`nil` 切片与空切片的 `len` 都是 0，都可以 `range`、都可以 `append`。区别在 JSON 序列化：

```go
var nilSlice []string
emptySlice := []string{}
b1, _ := json.Marshal(nilSlice)   // null
b2, _ := json.Marshal(emptySlice) // []
```

**接口返回列表时应初始化为空切片**，否则前端拿到 `null` 会报错。

## 三、append 与扩容

`append` 返回新切片，**必须接收返回值**：

```go
s := []int{1, 2}
s = append(s, 3)              // 正确
// append(s, 3)               // 错误：返回值被丢弃
s = append(s, 4, 5)           // 一次追加多个
s = append(s, []int{6, 7}...) // 追加另一个切片
```

扩容规则（实现细节，不要依赖精确值）：

- 容量足够时，直接在原数组追加，`cap` 不变
- 容量不足时，分配更大的数组并拷贝，通常是 2 倍（大切片约 1.25 倍）
- 扩容后新切片指向新数组，**与原切片彻底分离**

已知数据量时预分配能减少拷贝：`users := make([]User, 0, 1000)`。

## 四、最大的坑：共享底层数组

切片操作 `s[low:high]` **不拷贝数据**，新切片与原切片共享同一个底层数组：

```go
base := []int{1, 2, 3, 4, 5}
sub := base[1:3] // [2 3]，len=2，cap=4（从索引 1 到数组末尾）

sub[0] = 99
fmt.Println(base) // [1 99 3 4 5]，原切片被修改

sub = append(sub, 100)
fmt.Println(base) // [1 99 3 100 5]，append 写进了原数组！
```

原因：`sub` 的容量还有 2，`append` 不需要扩容，于是直接写 `base[3]`。三种解法：

```go
safe := make([]int, len(sub))
copy(safe, sub)          // 1. copy 独立副本

safe2 := base[1:3:3]     // 2. 三索引切片：len=2, cap=2，append 必然扩容

safe3 := slices.Clone(sub) // 3. Go 1.21+ 的 slices.Clone
```

同样的坑也出现在「函数返回切片的一部分」时。如果返回的切片会被长期持有或修改，务必 `Clone` 或 `copy`。

## 五、切片的删除与插入

```go
s := []int{1, 2, 3, 4, 5}

s = append(s[:2], s[3:]...) // 删除索引 2（值 3）-> [1 2 4 5]

s = append(s, 0)            // 在索引 2 前插入 99：先扩容一位
copy(s[3:], s[2:])          // 后移
s[2] = 99                   // [1 2 99 4 5]
```

注意：`append(s[:i], s[i+1:]...)` 会**修改底层数组**，如果还有其他切片共享它，会看到数据变化。需要安全删除时先 `Clone`。Go 1.21+ 也可以直接用 `slices.Delete`、`slices.Insert`，语义更清晰。

## 六、map 基础

```go
m := make(map[string]int) // 必须 make 才能写入
m["a"] = 1

scores := map[string]int{"alice": 90, "bob": 85} // 字面量

v, ok := m["c"] // comma-ok 区分「不存在」与「零值」
fmt.Println(v, ok) // 0 false
delete(m, "a")
fmt.Println(len(m))
```

`for range` 遍历 map 时**顺序是随机的**，Go 故意如此，防止你依赖顺序。需要有序输出时，先把 key 取出来排序：

```go
keys := make([]string, 0, len(scores))
for k := range scores {
	keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
	fmt.Println(k, scores[k])
}
```

## 七、map 的并发安全

**map 不支持并发读写**，多个 goroutine 同时写会直接触发 `fatal error: concurrent map writes`，这个错误无法 recover。方案一：`sync.RWMutex`（读多写少最常见）：

```go
type SafeCache struct {
	mu sync.RWMutex
	m  map[string]string
}

func (c *SafeCache) Get(key string) (string, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	v, ok := c.m[key]
	return v, ok
}

func (c *SafeCache) Set(key, val string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.m[key] = val
}
```

方案二：`sync.Map`，适合「键集合基本固定、读远多于写」的场景，不适合复杂业务。多数时候 `RWMutex` 更直观，也方便后期加过期与指标。

## 八、strings 与 strconv 常用函数

```go
s := "  Gin,Go  "

strings.TrimSpace(s)                  // "Gin,Go"
strings.ToLower(s)                    // "  gin,go  "
strings.Contains(s, "Gin")            // true
strings.Split("a,b,c", ",")           // [a b c]
strings.Join([]string{"a", "b"}, "-") // "a-b"
strings.ReplaceAll("a-b", "-", "_")   // "a_b"
k, v, ok := strings.Cut("k=v", "=")   // "k" "v" true
```

```go
n, err := strconv.Atoi("8080")       // string -> int
s := strconv.Itoa(8080)              // int -> string
f, _ := strconv.ParseFloat("3.14", 64)
b, _ := strconv.ParseBool("true")
```

大量拼接用 `strings.Builder`，避免 O(n²) 的字符串拷贝：

```go
var b strings.Builder
b.Grow(64) // 预分配
for i := 0; i < 3; i++ {
	fmt.Fprintf(&b, "item-%d,", i)
}
fmt.Println(b.String()) // item-0,item-1,item-2,
```

解析查询串这类工作，实际项目直接用标准库 `url.ParseQuery` 或 Gin 的参数绑定（[03-基本特性/03-参数绑定与校验](../03-基本特性/03-参数绑定与校验.md)）。

## 常见坑

1. **`append` 不接收返回值**：`append(s, x)` 后 `s` 没变，是新手最常见的「代码没生效」。
2. **切片截取后 append 污染原数组**：用三索引切片或 `slices.Clone` 隔离。
3. **nil map 写入 panic**：`var m map[string]int; m["a"]=1` 直接崩，必须 `make`。
4. **并发写 map**：`fatal error` 无法 recover，读多写少用 `RWMutex`。
5. **依赖 map 遍历顺序**：输出顺序随机，需要稳定顺序就先排序 key。
6. **JSON 输出 `null` 而不是 `[]`**：列表字段初始化为空切片。
7. **用 `+=` 在循环里拼字符串**：数据量大时性能极差，改用 `strings.Builder`。

## 小结

切片是「带长度和容量的视图」，共享底层数组是它最大的特性也是最大的坑；map 的并发安全必须自己保证；字符串与数字转换统一走 `strings` / `strconv`。下一篇 [07-并发-goroutine-channel-context](07-并发-goroutine-channel-context.md) 进入 Go 最核心的并发模型。

## 练习

1. 写 `func Unique(nums []int) []int`，返回去重后的新切片，要求不修改入参（用 map 记录 + 预分配）。
2. 复现「切片共享底层数组」的例子，分别用 `copy`、三索引切片、`slices.Clone` 三种方式修复，并打印 `cap` 验证差异。
3. 实现 `func TopN(scores map[string]int, n int) []string`，按分数降序返回前 n 个用户名；分数相同按名字升序，验证不依赖 map 遍历顺序。

## 官方文档

- 切片用法与内部结构：<https://go.dev/blog/slices-intro>
- `slices` 包：<https://pkg.go.dev/slices>
- map 类型：<https://go.dev/ref/spec#Map_types>
- `strings` 包：<https://pkg.go.dev/strings>
- `strconv` 包：<https://pkg.go.dev/strconv>
