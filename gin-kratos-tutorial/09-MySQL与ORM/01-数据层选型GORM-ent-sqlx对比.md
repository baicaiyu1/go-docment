# 01-数据层选型GORM-ent-sqlx对比

> 一句话定位：Go 没有绝对最优的 ORM，只有匹配团队规模与领域复杂度的组合；mall-lite 以 GORM 为主数据层，复杂报表下沉原生 SQL。

## 本篇目标

- 用类型安全、生成代码、学习曲线、复杂 SQL、迁移、生态、性能七个维度给出对比结论
- 说清 mall-lite 选 GORM 的理由与边界，避免“ORM 万能”与“ORM 原罪”两种极端
- 在 Kratos 的 `internal/{biz,data}` 中完成 Repository 依赖倒置
- 写出可被 Wire 注入、可被单测替换的数据层骨架

## 前置知识

- 已读 [07-工程化与测试/01-项目分层与目录规范](../07-工程化与测试/01-项目分层与目录规范.md)
- 了解 SQL 的增删改查、索引与事务
- 能看懂 Go 的 struct、interface 与 `context.Context`

## 一、先给结论

- **业务系统（本教程 mall-lite）**：GORM 为主，交付快、资料多、关联与钩子够用
- **复杂领域模型、多人协作、要求编译期类型安全**：ent
- **报表/分析型查询、手写 SQL 为主**：sqlx
- **只有一两条 SQL 的小工具**：`database/sql` 就够
- 一个服务只保留**一套主数据层**，局部用原生 SQL 可以，但不要四套混用

选型的本质是：你愿意用多少**框架约束**换多少**开发效率**，以及团队能否承担它的**排查成本**。

## 二、四个抽象层次

| 层次 | 代表库 | 工作方式 | 典型使用者 |
| --- | --- | --- | --- |
| 标准库 | `database/sql` | 手写 SQL，手动 `Scan` | 追求零依赖、SQL 完全可控 |
| 轻量映射 | `github.com/jmoiron/sqlx` | 手写 SQL，结构体自动映射 | 报表多、SQL 复杂 |
| 全功能 ORM | `gorm.io/gorm` | 反射 + 链式 API 生成 SQL | 业务 CRUD、快速交付 |
| 代码生成 ORM | `entgo.io/ent` | 写 Schema，生成类型安全 Client | 复杂领域模型、大团队 |

它们不是四选一，而是**抽象层级从低到高**。层级越高开发越快、越难写错字段名，但越难精确控制 SQL。

## 三、七个维度对比

| 维度 | GORM | ent | sqlx |
| --- | --- | --- | --- |
| 类型安全 | 中，字段名是字符串，拼错运行时才报 | 强，字段/谓词均由代码生成，写错编译不过 | 中，列名靠 struct tag 映射 |
| 生成代码 | 无，纯运行时反射 | 有，`go generate` 产出 Client | 无 |
| 学习曲线 | 中，链式 API、Scope、Hook 概念多 | 高，要理解 Schema、Edge、Predicate | 低，会 SQL 即可 |
| 复杂 SQL | 中，窗口函数/CTE 需 `Raw` | 中，复杂聚合需 `Modify` 或原生 SQL | 最强，本身写的就是 SQL |
| 迁移 | `AutoMigrate` 仅适合开发 | 自带 Schema 迁移，仍需版本化 | 无，交给迁移工具 |
| 生态 | 最大，插件/文档/问答最全 | 活跃，工具链完善 | 稳定但更新慢 |
| 性能 | 反射有微秒级开销 | 接近 sqlx，生成代码开销小 | 很低 |

一句关键结论：**ORM 的反射开销是微秒级，一次 MySQL 往返是毫秒级**。线上慢绝大多数不是“ORM 慢”，而是 N+1、缺索引、大事务、连接池耗尽。选型不要拿空表微基准当唯一依据。

## 四、mall-lite 的选型结论

- `user-service`、`product-service`、`order-service` 统一用 **GORM v1.25.x**，数据访问封装在 Kratos 的 `internal/data`
- 商家 GMV、对账单等复杂统计用 **`db.Raw` 或 sqlx**，不硬套 `Preload` 拼出十几条 SQL
- 订单领域若演进为复杂状态机，可试点 ent，但**不与 GORM 操作同一张表**
- 连接池在 `data.NewDB` 创建一次，注入所有 Repository，**禁止每个包各自 `gorm.Open`**
- 迁移统一版本化（见 [06-数据库迁移与测试](06-数据库迁移与测试.md)），不依赖 `AutoMigrate` 上生产

Gin 网关**不直接连库**：handler 只依赖 `internal/client` 里的 gRPC 客户端。只有 BFF 聚合读模型（如首页商品卡片）需要跨服务拼装时，才允许直连只读库，且必须独立只读账号、加缓存、不复用业务服务写连接。

## 五、依赖倒置：biz 定义接口，data 实现

Kratos 官方 DDD 分层的核心约定：**接口定义在 `biz`，实现在 `data`**。`biz` 只依赖自己的领域模型，不知道 GORM 存在；换 ORM 或加缓存代理时只改 `data`。

```text
app/product-service/
├── cmd/main.go                 # Wire 注入 + kratos.New
├── internal/
│   ├── biz/
│   │   ├── product.go          # 领域模型，无 GORM 依赖
│   │   └── product_repo.go     # Repository 接口（biz 定义）
│   ├── data/
│   │   ├── data.go             # gorm.Open、连接池、Data 结构体
│   │   ├── product.go          # 接口实现，PO ↔ 领域模型
│   │   └── po/product_po.go    # 持久化对象，仅 data 层可见
│   ├── service/                # gRPC 业务编排
│   ├── server/
│   └── conf/
```

```go
// app/product-service/internal/biz/product_repo.go
package biz

import "context"

type Product struct {
    ID     uint64
    SKU    string
    Name   string
    Price  int64 // 单位：分
    Status int8
}

// ProductRepo 由 biz 定义，data 实现；biz 不感知 GORM。
type ProductRepo interface {
    Get(ctx context.Context, id uint64) (*Product, error)
    ListBySKU(ctx context.Context, skus []string) ([]*Product, error)
    Create(ctx context.Context, p *Product) (*Product, error)
    DeductStock(ctx context.Context, id uint64, qty int64) error
}
```

```go
// app/product-service/internal/data/data.go
package data

import (
    "time"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

type Data struct {
    db *gorm.DB
}

func NewDB(dsn string) (*gorm.DB, func(), error) {
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Warn), // 生产只打慢 SQL 与错误
    })
    if err != nil {
        return nil, nil, err
    }
    sqlDB, err := db.DB()
    if err != nil {
        return nil, nil, err
    }
    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetMaxIdleConns(20)
    sqlDB.SetConnMaxLifetime(30 * time.Minute) // 必须短于 MySQL/代理的空闲断连时间
    sqlDB.SetConnMaxIdleTime(5 * time.Minute)
    return db, func() { _ = sqlDB.Close() }, nil
}

func NewData(db *gorm.DB) *Data { return &Data{db: db} }
```

```go
// app/product-service/internal/data/product.go
package data

import (
    "context"
    "errors"

    "gorm.io/gorm"

    "mall-lite/app/product-service/internal/biz"
)

type productRepo struct{ data *Data }

func NewProductRepo(d *Data) biz.ProductRepo { return &productRepo{data: d} }

func (r *productRepo) Get(ctx context.Context, id uint64) (*biz.Product, error) {
    var po ProductPO
    err := r.data.db.WithContext(ctx).Where("id = ?", id).Take(&po).Error
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }
    return po.ToBiz(), nil
}

// 复杂报表：直接原生 SQL，比硬套 Preload 更可读、更可优化。
func (r *productRepo) SalesReport(ctx context.Context, from, to string) ([]*biz.SalesRow, error) {
    const q = `SELECT p.id, p.sku, SUM(o.amount) AS amount
FROM orders o JOIN products p ON p.id = o.product_id
WHERE o.created_at >= ? AND o.created_at < ?
GROUP BY p.id, p.sku`
    var rows []*biz.SalesRow
    err := r.data.db.WithContext(ctx).Raw(q, from, to).Scan(&rows).Error
    return rows, err
}
```

Wire 只需把实现绑定到接口，`biz` 的 Usecase 构造参数写 `biz.ProductRepo` 即可：

```go
// app/product-service/cmd/main.go
var ProviderSet = wire.NewSet(
    data.NewDB,
    data.NewData,
    data.NewProductRepo, // 返回 biz.ProductRepo，供 Usecase 注入
    biz.NewProductUsecase,
    service.NewProductService,
)
```

## 六、什么时候该换

- 同一 Repository 里 `Raw`/`Exec` 越来越多，说明 ORM 表达力到顶，可局部换 sqlx
- 团队频繁因字段名、表名拼错出事故，ent 的代码生成收益最明显
- 需要声明式迁移与多方言，考虑 atlas + sqlc/ent
- 换框架要重写 Repository、重建测试与培训，**不要为了“新”而换**

## 完整示例

把上面的三层串起来：`cmd` 只做装配，`biz` 只依赖接口，`data` 持有 `*gorm.DB`，Gin 网关通过 gRPC 客户端访问，不碰数据库。

```go
// app/product-service/cmd/main.go
func main() {
    logger, _ := zap.NewProduction()
    defer logger.Sync()

    db, cleanup, err := data.NewDB(os.Getenv("MALL_MYSQL_DSN"))
    if err != nil {
        panic(err)
    }
    defer cleanup()

    d := data.NewData(db)
    repo := data.NewProductRepo(d) // 返回 biz.ProductRepo
    uc := biz.NewProductUsecase(repo, logger)
    svc := service.NewProductService(uc)

    srv := grpc.NewServer(
        grpc.Address(":9002"),
        grpc.Middleware(recovery.Recovery(), logging.Server(logger)),
    )
    v1.RegisterProductServiceServer(srv, svc)

    app := kratos.New(kratos.Name("product-service"), kratos.Logger(logger), kratos.Server(srv))
    if err := app.Run(); err != nil {
        panic(err)
    }
}
```

## 常见坑

1. **把 GORM 当银弹**：复杂报表用 `Preload` 拼出十几条 SQL，比手写 SQL 慢一个数量级。
2. **以为 ent 类型安全就没有慢 SQL**：类型安全只管字段名，缺索引、N+1 一样打爆数据库。
3. **GORM v1/v2 混用**：误导入 `github.com/jinzhu/gorm`，与 `gorm.io/gorm` 类型不兼容，报错难找。
4. **多套 ORM 各自建池**：GORM 一个池、sqlx 一个池，总连接翻倍直接 `Too many connections`。
5. **在 biz 里 import GORM**：领域层被框架污染，单测必须起数据库；接口应在 biz，实现留 data。
6. **网关直连业务库**：BFF 图省事查库，读放大和慢 SQL 直接压到核心库，必须走 gRPC 或只读副本。

## 小结

- 没有绝对最优的 ORM，只有与团队和场景匹配的选择
- `database/sql` < sqlx < GORM/ent 是抽象层级，不是优劣顺序
- mall-lite 用 GORM 做主数据层，复杂查询局部下沉原生 SQL
- 性能差异主要来自 SQL 与索引设计，而非 ORM 的反射开销
- 依赖倒置：biz 定义 Repository 接口，data 实现，连接池统一由 data 创建注入

## 练习

1. 把本文的 `ProductRepo` 换成 ent 实现，保持接口不变，观察 biz 层是否需要改动。
2. 为你的项目列出所有数据访问点，标注哪些适合 GORM、哪些应下沉原生 SQL，并说明理由。
3. 用 `go test -bench` 对比 GORM 与 `database/sql` 查同一张表的开销，解释占比最大的部分。

## 官方文档

- GORM 中文文档：<https://gorm.io/zh_CN/docs/>
- ent 官方文档：<https://entgo.io/docs/getting-started>
- sqlx 官方文档：<https://jmoiron.github.io/sqlx/>
- Kratos 分层与依赖注入：<https://go-kratos.dev/docs/component/di>
- Go `database/sql`：<https://pkg.go.dev/database/sql>
