# 配置管理 Viper

> 一句话定位：Viper 把「默认值 + YAML 文件 + 环境变量 + flag」合并成一份配置并支持热更新；最值钱的结论是——`AutomaticEnv` 不会自动作用于 `Unmarshal`，关键 key 必须显式 `BindEnv`。

## 本篇目标

- 说清配置来源与优先级、`MALL_` 前缀与 `.` → `_` 的映射规则
- 用 `user-service-dev.yaml` / `user-service-prod.yaml` + flag 选择环境，把 YAML 解成 `mapstructure` 结构体
- 在启动期集中校验，并用 `WatchConfig` + `atomic.Value` 做并发安全的有限热更新
- 划清敏感信息、Viper 与 Kratos config 的边界

## 前置知识

- 已读 [etcd 与 Consul 接入](02-etcd与Consul接入.md)，本篇的 `registry.*` 就是那一篇的输入
- 了解 struct tag、`time.Duration` 文本格式（`3s`）与 12-Factor 的「配置与代码分离」

## 一、边界与优先级

手写 `os.ReadFile` + `yaml.Unmarshal` 只解决「读文件」，剩下四件事都得自己写：多格式、环境变量覆盖、默认值、热更新。Viper 把它们收进一个 API，代价是 `Get*` 与 `Unmarshal` 是两套语义——key 拼错时 `GetString` 返回空串、`Unmarshal` 保持零值，**都不会报错**，所以必须补一层启动期 `Validate()`。

| 优先级 | 来源 | 设置方式 |
| --- | --- | --- |
| 1 | 显式 Set | `v.Set("log.level", "debug")` |
| 2 | flag | `v.BindPFlag("app.env", fs.Lookup("env"))` |
| 3 | 环境变量 | `SetEnvPrefix` + `AutomaticEnv`/`BindEnv` |
| 4 | 配置文件 | `SetConfigFile` + `ReadInConfig` |
| 5 | 默认值 | `v.SetDefault("server.http.timeout", "3s")` |

约定环境变量前缀 `MALL_`，`mysql.dsn` 对应 `MALL_MYSQL_DSN`。密钥只从环境变量或 Secret 进入，YAML 里留空值，避免仓库出现生产密码。

## 二、配置文件与结构体

```yaml
# configs/user-service-dev.yaml
app: {name: user-service, env: dev}
server:
  http: {addr: 0.0.0.0:8080, timeout: 3s}
  grpc: {addr: 0.0.0.0:9001, timeout: 2s}
registry:
  type: etcd
  namespace: mall-lite
  etcd: {endpoints: ["127.0.0.1:2379"]}
  consul: {address: "127.0.0.1:8500"}
log: {level: info, format: json}
mysql:
  dsn: ""                 # 必须由 MALL_MYSQL_DSN 注入
  max_open_conns: 100
  max_idle_conns: 20
redis: {addr: "127.0.0.1:6379", password: "", db: 0}
kafka: {brokers: ["127.0.0.1:9092"]}
sentinel: {enabled: true, rule_file: configs/sentinel-rules.yaml}
```

```go
// pkg/config/config.go
type Config struct {
    App      AppConfig      `mapstructure:"app"`
    Server   ServerConfig   `mapstructure:"server"`
    Registry RegistryConfig `mapstructure:"registry"`
    Log      LogConfig      `mapstructure:"log"`
    MySQL    MySQLConfig    `mapstructure:"mysql"`
    Redis    RedisConfig    `mapstructure:"redis"`
}
type ServerConfig struct {
    HTTP ServerAddr `mapstructure:"http"`
    GRPC ServerAddr `mapstructure:"grpc"`
}
type ServerAddr struct {
    Addr    string        `mapstructure:"addr"`
    Timeout time.Duration `mapstructure:"timeout"`
}
```

tag 必须是 `mapstructure` 而不是 `yaml`：`Unmarshal` 走 mapstructure 解码器，写 `yaml` tag 只在字段名碰巧同名时生效。`3s` 能解成 `time.Duration`，靠 Viper 默认的 `StringToTimeDurationHookFunc`；自建 decoder 时要补上。其余 `AppConfig`/`LogConfig`/`RegistryConfig`/`MySQLConfig`/`RedisConfig` 与 YAML 一一对应，省略定义。

## 三、加载：默认值 → 文件 → 环境变量

```go
// pkg/config/load.go
func Load(path string) (*Config, *viper.Viper, error) {
    v := viper.New()
    v.SetConfigFile(path) // 显式文件，避免容器工作目录不同找不到
    v.SetEnvPrefix("MALL")                             // 只认 MALL_，避免误吃系统变量
    v.SetEnvKeyReplacer(strings.NewReplacer(".", "_")) // mysql.dsn → MYSQL_DSN
    v.AutomaticEnv()
    v.SetDefault("app.env", "dev")

    if err := v.ReadInConfig(); err != nil {
        return nil, nil, fmt.Errorf("read config %s: %w", path, err)
    }
    for _, key := range v.AllKeys() { // Unmarshal 前必须逐 key 绑定
        _ = v.BindEnv(key)
    }
    cfg := &Config{}
    if err := v.Unmarshal(cfg); err != nil {
        return nil, nil, fmt.Errorf("unmarshal config: %w", err)
    }
    if err := cfg.Validate(); err != nil {
        return nil, nil, err
    }
    return cfg, v, nil
}
```

`SetDefault` 不是可选项：`AllKeys()` 只包含「默认值 + 文件里出现过」的 key，漏掉默认值的 key，循环 `BindEnv` 也覆盖不到。返回 `*viper.Viper` 是为了热更新复用同一个实例，新建实例只会在启动时读一次文件。

## 四、为什么 AutomaticEnv 对 Unmarshal 不生效

`AutomaticEnv()` 的实现是在 `Get` 系列方法里查一次环境变量；而 `Unmarshal` 走「遍历结构体字段 → 从 `AllSettings()` 取值」，不经过 `Get`，环境变量也无法反推 `MALL_MYSQL_DSN` 属于哪个 key。于是 `v.GetString("mysql.dsn")` 能读到，`v.Unmarshal(&cfg)` 却是空串。两种修法：

1. **默认值 + 循环 `BindEnv`**（上面的写法）。`BindEnv` 把环境变量登记进 `AllSettings`，`Unmarshal` 才看得见；可测试、可审计，推荐。
2. 对 3~5 个密钥显式绑定：`v.BindEnv("mysql.dsn", "MALL_MYSQL_DSN")`；或密钥字段完全不走 Viper，直接 `os.LookupEnv` 赋值。

## 五、多环境与 flag

```go
fs := pflag.NewFlagSet("user-service", pflag.ContinueOnError)
fs.String("env", "dev", "dev|test|prod")
_ = fs.Parse(os.Args[1:])
path := fmt.Sprintf("configs/user-service-%s.yaml", fs.Lookup("env").Value.String())
cfg, v, err := config.Load(path)
_, _, _ = cfg, v, err
```

`-env prod` 只负责挑文件；不要让 `MALL_ENV` 同时决定文件名和配置内容，选文件发生在 `ReadInConfig` 之前，环境变量要等 `BindEnv` 才生效，顺序混在一起很难排查。需要 flag 覆盖具体配置时，用 `v.BindPFlag("app.env", fs.Lookup("env"))`，优先级高于文件。

## 六、启动期校验

```go
func (c *Config) Validate() error {
    switch {
    case c.App.Name == "":
        return errors.New("app.name is required")
    case c.MySQL.DSN == "": return errors.New("mysql.dsn is empty, inject MALL_MYSQL_DSN")
    case c.MySQL.MaxIdleConns > c.MySQL.MaxOpenConns: return errors.New("max_idle_conns must <= max_open_conns")
    }
    return nil
}
```

校验只做三件事：必填项、跨字段一致性、生产禁默认口令（`prod` 下出现 `root:root` 直接拒绝启动）。实现成 `Validate() error`，在 `Load` 里调用，失败就让进程退出——「配置没生效」绝不能带病上线。

## 七、热更新与并发安全

只热更「限流阈值、降级开关、日志级别、灰度名单」；DSN、监听地址、`registry.type`、日志路径改了也只生效一半，必须重启。热更必须原子替换整个结构体，校验失败保留旧值：

```go
type Manager struct {
    cur atomic.Value // 存 *Config
    v   *viper.Viper
}

func NewManager(v *viper.Viper, cfg *Config) *Manager {
    m := &Manager{v: v}; m.cur.Store(cfg); return m
}

func (m *Manager) Watch(lg *zap.Logger) error {
    m.v.OnConfigChange(func(e fsnotify.Event) {
        next := &Config{}
        if err := m.v.Unmarshal(next); err != nil || next.Validate() != nil {
            lg.Error("reload failed, keep old", zap.String("file", e.Name))
            return
        }
        m.cur.Store(next) // 整体替换，读侧永远看到完整快照
        lg.Info("config reloaded", zap.String("file", e.Name))
    })
    return m.v.WatchConfig()
}
func (m *Manager) Get() *Config { return m.cur.Load().(*Config) }
```

`OnConfigChange` 在独立 goroutine 执行，业务侧可能同时读配置；不要原地改 map/slice，也不要把 `*Config` 字段长期缓存，每次 `Get()` 取当前快照即可。`Store` 成功后调用 `sentinelx.LoadFile(next.Sentinel.RuleFile)`，限流阈值才能跟着热更。

## 八、敏感信息与 Kratos 边界

- DSN、密码、JWT 密钥只从环境变量或 K8s Secret 注入，`user-service-prod.yaml` 不进仓库；日志只打 `app.name/env`、`log.level`，禁止 `zap.Any("cfg", cfg)`。
- Gin 网关与共享 `pkg/` 用 Viper；Kratos 服务用官方 `config`：`c := config.New(config.WithSource(src))` 后 `c.Load()` 再 `c.Scan(&bc)`，`src` 来自 etcd / file 等 contrib 配置源。
- 同一个 key 只允许一个来源生效，Viper 与 Kratos config 不要双写，否则两套缓存、两套热更新，出问题无法判断谁生效。

## 完整示例

```go
func main() {
    cfg, v, err := config.Load("configs/user-service-dev.yaml")
    if err != nil { panic(err) }
    zl, _ := logger.New(logger.Options{Service: cfg.App.Name, Env: cfg.App.Env, Level: cfg.Log.Level})
    m := config.NewManager(v, cfg)
    _ = m.Watch(zl) // 失败则降级为静态配置，不影响主流程
    r := gin.New()
    r.Use(otelgin.Middleware(cfg.App.Name), middleware.AccessLog(zl))
    _ = http.ListenAndServe(cfg.Server.HTTP.Addr, r)
}
```

```bash
MALL_MYSQL_DSN='app:secret@tcp(127.0.0.1:3306)/mall?charset=utf8mb4&parseTime=True&loc=Local' MALL_LOG_LEVEL=debug go run ./app/user-service -env dev
```
```powershell
$env:MALL_MYSQL_DSN = "app:secret@tcp(127.0.0.1:3306)/mall?charset=utf8mb4&parseTime=True&loc=Local"; $env:MALL_LOG_LEVEL = "debug"
go run ./app/user-service -env dev
```

## 常见坑

1. **只开 `AutomaticEnv` 就 `Unmarshal`**：环境变量完全不生效，结构体全是零值；必须默认值齐全 + 逐 key `BindEnv`。
2. **`SetEnvKeyReplacer` 写反**：`strings.NewReplacer("_", ".")` 会打乱路径，方向必须是 `"." → "_"`。
3. **`SetConfigFile` 用相对路径**：容器工作目录不是项目根，读不到文件；固定 `WORKDIR` 或传绝对路径。
4. **`timeout: 3`**：被解成 3ns，超时立即触发；一律写 `3s`/`500ms`。
5. **热更新新建了另一个 Viper，或 `Store` 前不校验**：前者监听与加载不是同一实例，改文件毫无反应；后者一个错字段让全实例配置归零。

## 小结

- 优先级：显式 Set > flag > 环境变量 > 文件 > 默认值，排查先确认哪一层生效
- `AutomaticEnv` 只影响 `Get*`；要让 `Unmarshal` 吃到环境变量，必须默认值齐全 + 逐 key `BindEnv`
- `mapstructure` tag 与启动期 `Validate()` 是配置正确性的唯一保障；热更新用 `atomic.Value` 整体替换 `*Config`，失败保留旧值
- 密钥只走 Secret/环境变量；Viper 管网关，Kratos 服务用官方 config

## 练习

1. 用 `user-service-dev.yaml` 启动网关，再用 `MALL_SERVER_HTTP_ADDR=:18080` 覆盖端口，验证生效。
2. 删掉 `mysql.max_open_conns` 的默认值，观察 `BindEnv` 循环为何覆盖不到，并解释 `AllKeys()` 的行为。
3. 改 YAML 的 `log.level`，验证 3 秒内热更生效；写入非法级别时确认保留旧值并打告警。

## 官方文档

- Viper 仓库与示例：<https://github.com/spf13/viper>
- Viper Unmarshal 与 DecodeHook：<https://github.com/spf13/viper#unmarshaling>
- mapstructure v2（Viper v1.20 起）：<https://github.com/go-viper/mapstructure>
- Kratos 配置组件：<https://github.com/go-kratos/kratos/tree/main/config>
- 12-Factor Config：<https://12factor.net/config>
