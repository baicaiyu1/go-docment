# 02-GORM核心用法与模型设计

> 一句话定位：把 DSN、日志、模型标签、CRUD 与关联一次配对，后面 90% 的数据库问题都能少踩一半。

## 本篇目标

- 正确配置 DSN、`gorm.Open`、连接池与慢 SQL 日志
- 用标签设计可迁移、可读的 `users/orders/order_items` PO
- 掌握 Create/First/Take/Find/Updates 的差异与零值语义
- 会写 PO ↔ 领域模型转换、Scope 复用与 Hook
- 知道 `AutoMigrate` 的边界，生产改表走版本化迁移

## 前置知识

- 已读 [09-MySQL与ORM/01-数据层选型GORM-ent-sqlx对比](01-数据层选型GORM-ent-sqlx对比.md)
- 熟悉 SQL 建表、索引、JOIN 与事务
- 了解 Go 的 struct、指针接收者与 `context.Context`

## 一、DSN 与 gorm.Open

MySQL DSN 必备参数：`user:pass@tcp(127.0.0.1:3306)/mall?charset=utf8mb4&parseTime=True&loc=Local&timeout=3s&readTimeout=3s&writeTimeout=3s`。

- `parseTime=True`：把 `DATETIME` 扫描成 `time.Time`，否则只能拿到 `[]byte`
- `loc=Local`：驱动按本地时区解析；跨时区部署统一 `loc=UTC`，展示层再转换
- `charset=utf8mb4`：完整支持 emoji
- 三类超时必须有，否则故障时会无限等待

```go
// app/user-service/internal/data/data.go
package data

import (
    "log"
    "os"
    "time"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

func NewDB(dsn string) (*gorm.DB, func(), error) {
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
        PrepareStmt:    true, // 复用预编译语句，降低解析开销
        TranslateError: true, // 唯一键冲突 -> gorm.ErrDuplicatedKey
        Logger: logger.New(log.New(os.Stdout, "", log.LstdFlags), logger.Config{
            SlowThreshold:             200 * time.Millisecond,
            LogLevel:                  logger.Warn, // 生产只打错误与慢 SQL
            IgnoreRecordNotFoundError: true,
            ParameterizedQueries:      true, // 不打印参数，避免手机号/密码进日志
        }),
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
    sqlDB.SetConnMaxLifetime(30 * time.Minute) // 短于 MySQL wait_timeout
    sqlDB.SetConnMaxIdleTime(5 * time.Minute)
    return db, func() { _ = sqlDB.Close() }, nil
}
```

连接池参数的计算方式见 [05-连接池与数据库性能调优](05-连接池与数据库性能调优.md)。

## 二、命名策略与表名

默认策略把 `UserProfile` 映射为 `user_profiles`、`ID` 映射为 `id`。mall-lite 不靠前缀，统一显式声明表名（`func (UserPO) TableName() string { return "users" }`）。一旦实现 `TableName()`，`NamingStrategy.TablePrefix` 对该模型**不再生效**，不要两套规则混用。PO 统一放在 `internal/data/po`，字段用 `column:` 显式绑定，避免以后改字段名导致线上列名漂移。

## 三、PO 模型与标签

```go
// app/user-service/internal/data/po/user_po.go
package po

import (
    "time"

    "gorm.io/gorm"
)

type UserPO struct {
    ID        uint64         `gorm:"column:id;primaryKey;autoIncrement;comment:主键"`
    Email     string         `gorm:"column:email;type:varchar(128);not null;uniqueIndex:uk_users_email;comment:邮箱"`
    Password  string         `gorm:"column:password;type:char(60);not null;comment:bcrypt 哈希"`
    Nickname  string         `gorm:"column:nickname;type:varchar(64);not null;default:'';comment:昵称"`
    Status    int8           `gorm:"column:status;type:tinyint;not null;default:1;index:idx_users_status;comment:1正常 0禁用"`
    CreatedAt time.Time      `gorm:"column:created_at;type:datetime(3);not null;default:CURRENT_TIMESTAMP(3);comment:创建时间"`
    UpdatedAt time.Time      `gorm:"column:updated_at;type:datetime(3);not null;default:CURRENT_TIMESTAMP(3);comment:更新时间"`
    DeletedAt gorm.DeletedAt `gorm:"column:deleted_at;index;comment:软删除时间"`
}

func (UserPO) TableName() string { return "users" }

type OrderItemPO struct {
    ID        uint64 `gorm:"column:id;primaryKey;autoIncrement"`
    OrderID   uint64 `gorm:"column:order_id;not null;index:idx_order_items_order;comment:订单ID"`
    ProductID uint64 `gorm:"column:product_id;not null;comment:商品ID"`
    Quantity  int    `gorm:"column:quantity;not null;comment:数量"`
    Price     int64  `gorm:"column:price;not null;comment:下单单价(分)"`
}

func (OrderItemPO) TableName() string { return "order_items" }

type OrderPO struct {
    ID        uint64        `gorm:"column:id;primaryKey;autoIncrement"`
    UserID    uint64        `gorm:"column:user_id;not null;index:idx_orders_user_status_created,priority:1"`
    OrderNo   string        `gorm:"column:order_no;type:varchar(32);not null;uniqueIndex:uk_orders_no"`
    Amount    int64         `gorm:"column:amount;type:bigint;not null;comment:总额(分)"`
    Status    int8          `gorm:"column:status;type:tinyint;not null;default:0;index:idx_orders_user_status_created,priority:2"`
    CreatedAt time.Time     `gorm:"column:created_at;type:datetime(3);not null;index:idx_orders_user_status_created,priority:3"`
    Items     []OrderItemPO `gorm:"foreignKey:OrderID"`
}

func (OrderPO) TableName() string { return "orders" }
```

标签速查：`primaryKey;autoIncrement` 主键；`type:`/`size:` 显式统一类型与长度；`not null;default:0` 表达非空语义；`uniqueIndex:name` 与 `index:name,priority:2` 让同名字段组成联合索引；`comment:` 写入表结构。金额统一用**分**存 `BIGINT`。

## 四、PO / 领域模型 / DTO 转换

三层各司其职：`PO` 只表达表结构，`biz` 领域模型表达业务，HTTP 响应 DTO 只暴露需要的字段（永远不要直接返回带 `Password` 的 PO）。转换写成显式函数，比反射拷贝更可控。

```go
// app/user-service/internal/data/user.go（片段）
func (p *UserPO) ToBiz() *biz.User {
    if p == nil {
        return nil
    }
    return &biz.User{ID: p.ID, Email: p.Email, Nickname: p.Nickname, Status: p.Status}
}

func FromBiz(u *biz.User) *UserPO {
    return &UserPO{ID: u.ID, Email: u.Email, Nickname: u.Nickname, Status: u.Status}
}
```

## 五、CRUD 与查询语义

`Create` 批量插入会自动开事务；`First` 按主键升序取一条并返回 `ErrRecordNotFound`；`Take` 不排序；`Find` 取多条且**不返回**该错误。`Updates(struct)` 默认忽略零值，要把 `status` 改成 0 必须用 map 或 `Select`。

```go
// app/user-service/internal/data/user.go
func (r *userRepo) BatchCreate(ctx context.Context, us []*biz.User) error {
    pos := make([]UserPO, 0, len(us))
    for _, u := range us {
        pos = append(pos, *FromBiz(u))
    }
    return r.data.db.WithContext(ctx).CreateInBatches(&pos, 100).Error
}

func (r *userRepo) UpdateStatus(ctx context.Context, id uint64, status int8) error {
    return r.data.db.WithContext(ctx).Model(&UserPO{}).
        Where("id = ?", id).
        Updates(map[string]any{"status": status}).Error // map 不忽略零值
}
```

## 六、Preload、Joins 与 Scope 复用

`Preload` 发 1+N 张关联表的 SQL、无重复行，适合取聚合根与子集合；`Joins` 只发一条 SQL，适合用子表条件筛父表，但一对多会产生重复行，需要 `Distinct` 或 `Group`。`Scopes` 把常用条件（软删除之外的过滤、分页）组合成可复用函数。

```go
func Active(db *gorm.DB) *gorm.DB { return db.Where("status = ?", 1) }

func Paginate(page, size int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB { return db.Offset((page - 1) * size).Limit(size) }
}

func (r *orderRepo) ListByUser(ctx context.Context, uid uint64, page, size int) ([]OrderPO, error) {
    var pos []OrderPO
    err := r.data.db.WithContext(ctx).
        Scopes(Active, Paginate(page, size)).
        Preload("Items"). // 订单 + 明细，共 2 条 SQL
        Where("user_id = ?", uid).
        Order("created_at DESC").Find(&pos).Error
    return pos, err
}

func (r *orderRepo) ListWithUser(ctx context.Context) ([]OrderRow, error) {
    var rows []OrderRow
    err := r.data.db.WithContext(ctx).Table("orders").
        Select("orders.order_no, users.nickname").
        Joins("JOIN users ON users.id = orders.user_id").
        Where("orders.status = ?", 1).Scan(&rows).Error
    return rows, err
}
```

## 七、Hook：BeforeCreate / BeforeSave

钩子在**同一个事务**内执行，返回 error 会回滚整条链路。只做本地、快速、可回滚的事，禁止在钩子里调 gRPC、发 Kafka。

```go
// app/user-service/internal/data/po/user_po.go
func (u *UserPO) BeforeSave(tx *gorm.DB) error {
    if u.Password == "" {
        return errors.New("password required")
    }
    if !strings.HasPrefix(u.Password, "$2") { // 已哈希则跳过，避免重复加密
        hash, err := bcrypt.GenerateFromPassword([]byte(u.Password), bcrypt.DefaultCost)
        if err != nil {
            return err
        }
        u.Password = string(hash)
    }
    return nil
}
```

批量 `Create` 会逐行触发钩子，10 万行时开销明显。`BeforeCreate` 只做必填校验，`BeforeSave` 适合哈希/时间戳等统一赋值；跳钩子用 `db.WithContext(ctx).Session(&gorm.Session{SkipHooks: true})`；`AfterCreate` 适合写本地 outbox 表，再由独立消费者投递 Kafka。

## 完整示例

`Get` 与第 01 篇 `ProductRepo.Get` 同构，这里只保留差异方法 `GetByEmail`：

```go
// app/user-service/internal/data/user.go
package data

import (
    "context"
    "errors"

    "gorm.io/gorm"

    "mall-lite/app/user-service/internal/biz"
)

type userRepo struct{ data *Data }

func NewUserRepo(d *Data) biz.UserRepo { return &userRepo{data: d} }

func (r *userRepo) GetByEmail(ctx context.Context, email string) (*biz.User, error) {
    var po UserPO
    err := r.data.db.WithContext(ctx).Select("id", "email", "nickname", "status").
        Where("email = ?", email).First(&po).Error
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }
    return po.ToBiz(), nil
}
```

## 常见坑

1. **`Updates(UserPO{...})` 更新不了 `status=0`**：struct 零值被忽略，改用 map、`Update` 或 `Select`。
2. **`Save` 全字段覆盖**：把并发请求刚改的字段冲回旧值，表现为更新丢失。
3. **循环里单条查关联**：100 个订单查 100 次明细，QPS 一高就打满数据库（见第 04 篇）。
4. **字符串拼接 `Where`**：`fmt.Sprintf` 拼输入会 SQL 注入，永远用占位符。
5. **`First` 与 `Find` 混用后没判错**：`First` 查不到返回 `ErrRecordNotFound`，`Find` 返回空切片。
6. **把 `AutoMigrate` 放启动流程**：多副本同时改表会互相阻塞，生产必须版本化迁移。

## 小结

- DSN 必须带 `parseTime`、`loc` 与三类超时；日志生产用 `Warn` + 慢 SQL 阈值
- 模型标签显式表达类型、非空、默认值、索引与注释，金额用分存 `BIGINT`
- CRUD 的核心差异在 `Updates` 的零值语义与 `Save` 的全字段覆盖
- 关联优先 `Preload`，用子表条件筛父表时用 `Joins + Select`，分页用 `Scopes`
- 钩子只做本地快速操作；生产改表走版本化迁移

## 练习

1. 为 `orders` 设计“用户 + 状态 + 创建时间”的联合索引，写出 GORM 标签并解释字段顺序。
2. 实现 `ListOrdersByUser`：返回订单及明细，要求只发 2 条 SQL，并打印日志确认。
3. 写一个 `BeforeSave` 校验库存不能为负，并说明为什么不在钩子里调用 product-service。

## 官方文档

- GORM 模型与标签：<https://gorm.io/zh_CN/docs/models.html>
- GORM 查询：<https://gorm.io/zh_CN/docs/query.html>
- GORM 关联 Preload：<https://gorm.io/zh_CN/docs/preload.html>
- GORM Hooks：<https://gorm.io/zh_CN/docs/hooks.html>
- MySQL 8.4 数据类型：<https://dev.mysql.com/doc/refman/8.4/en/data-types.html>
