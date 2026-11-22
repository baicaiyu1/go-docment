# 02-GORM核心用法与模型设计
> 一句话定位：把 DSN、日志、模型标签、CRUD 与关联一次配对，后面 90% 的数据库问题都能少踩一半。

## 本篇目标

- 正确配置 DSN、`gorm.Open`、连接池与慢 SQL 日志
- 用标签设计出可迁移、可读的 `users/products/stocks/orders` 模型
- 掌握 Create/First/Find/Where/Updates/Delete 的差异
- 理解四种关联，以及 Preload 与 Joins 的取舍
- 知道 `AutoMigrate` 的边界，避免在生产上用它改表

## 前置知识

- 已读 [09-MySQL与ORM/01-数据层选型GORM-ent-sqlx对比](01-数据层选型GORM-ent-sqlx对比.md)
- 熟悉 SQL 的建表、索引、JOIN 与事务
- 了解 Go 的 struct、指针接收者与 `context.Context`

## 一、DSN 与 gorm.Open

MySQL DSN 的必备参数：`user:pass@tcp(127.0.0.1:3306)/mall?charset=utf8mb4&parseTime=True&loc=Local&timeout=3s&readTimeout=3s&writeTimeout=3s`。

- `parseTime=True`：把 `DATETIME` 扫描成 `time.Time`，否则只能拿到 `[]byte`
- `loc=Local`：驱动按本地时区解析；跨时区部署建议统一 `loc=UTC`，展示层再转换
- `charset=utf8mb4`：完整支持 emoji；MySQL 8 默认排序规则是 `utf8mb4_0900_ai_ci`
- `timeout/readTimeout/writeTimeout`：连接、读、写超时，必须有，否则故障时无限等待

```go
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
	PrepareStmt:    true, // 复用预编译语句
	TranslateError: true, // 唯一键冲突翻译为 gorm.ErrDuplicatedKey
})
if err != nil {
	return nil, err
}
sqlDB, _ := db.DB()
sqlDB.SetMaxOpenConns(50)
sqlDB.SetMaxIdleConns(20)
sqlDB.SetConnMaxLifetime(30 * time.Minute) // 短于 MySQL wait_timeout
sqlDB.SetConnMaxIdleTime(5 * time.Minute)
```

连接池参数怎么算在第 05 篇展开，这里先给一组安全起点。

## 二、Logger 与慢 SQL

```go
gormLogger := logger.New(log.New(os.Stdout, "", log.LstdFlags), logger.Config{
	SlowThreshold:             200 * time.Millisecond,
	LogLevel:                  logger.Warn, // Info 会打印所有 SQL，生产慎用
	IgnoreRecordNotFoundError: true,
	ParameterizedQueries:      true, // 不打印参数，避免手机号/密码进日志
})
```

`Silent`/`Error`/`Warn`/`Info` 四级中，**生产用 `Warn`**：只打错误与慢 SQL。超过 `SlowThreshold` 会输出 `[201.3ms] [rows:1] SELECT ...`，接到日志平台做告警，比事后 `EXPLAIN` 更早发现问题。

## 三、命名策略与 TableName

默认策略把 `UserProfile` 映射为 `user_profiles`、`ID` 映射为 `id`。mall-lite 不靠前缀，统一显式声明表名（如 `func (User) TableName() string { return "users" }`）。一旦实现 `TableName()`，`NamingStrategy.TablePrefix` 对该模型**不再生效**，不要两套规则混用。

## 四、模型定义与标签

```go
type BaseModel struct {
	ID        uint64         `gorm:"primaryKey;autoIncrement;comment:主键" json:"id"`
	CreatedAt time.Time      `gorm:"type:datetime(3);not null;default:CURRENT_TIMESTAMP(3);comment:创建时间" json:"created_at"`
	UpdatedAt time.Time      `gorm:"type:datetime(3);not null;default:CURRENT_TIMESTAMP(3);comment:更新时间" json:"updated_at"`
	DeletedAt gorm.DeletedAt `gorm:"index;comment:软删除时间" json:"-"`
}

type User struct {
	BaseModel
	Email    string  `gorm:"type:varchar(128);not null;uniqueIndex:uk_users_email;comment:邮箱" json:"email"`
	Password string  `gorm:"type:char(60);not null;comment:bcrypt 哈希" json:"-"`
	Nickname string  `gorm:"type:varchar(64);not null;default:'';comment:昵称" json:"nickname"`
	Status   int8    `gorm:"type:tinyint;not null;default:1;index:idx_users_status;comment:1正常 0禁用" json:"status"`
	Orders   []Order `gorm:"foreignKey:UserID" json:"orders,omitempty"`
}

type Product struct {
	BaseModel
	SKU    string `gorm:"type:varchar(64);not null;uniqueIndex:uk_products_sku;comment:SKU" json:"sku"`
	Name   string `gorm:"type:varchar(128);not null;comment:商品名" json:"name"`
	Price  int64  `gorm:"type:bigint;not null;comment:单价(分)" json:"price"`
	Status int8   `gorm:"type:tinyint;not null;default:1;index:idx_products_status;comment:状态" json:"status"`
}

type Stock struct {
	ID        uint64 `gorm:"primaryKey"`
	ProductID uint64 `gorm:"not null;uniqueIndex:uk_stocks_product;comment:商品ID"`
	Available int64  `gorm:"not null;default:0;comment:可售库存"`
	Version   uint64 `gorm:"not null;default:0;comment:乐观锁版本"`
}

type Order struct {
	BaseModel
	UserID  uint64      `gorm:"not null;index:idx_orders_user;comment:用户ID" json:"user_id"`
	OrderNo string      `gorm:"type:varchar(32);not null;uniqueIndex:uk_orders_no;comment:订单号" json:"order_no"`
	Amount  int64       `gorm:"type:bigint;not null;comment:总额(分)" json:"amount"`
	Status  int8        `gorm:"type:tinyint;not null;default:0;index:idx_orders_status;comment:0待支付 1已支付 2已取消" json:"status"`
	Items   []OrderItem `gorm:"foreignKey:OrderID" json:"items,omitempty"`
}

type OrderItem struct {
	ID        uint64 `gorm:"primaryKey"`
	OrderID   uint64 `gorm:"not null;index:idx_order_items_order;comment:订单ID"`
	ProductID uint64 `gorm:"not null;comment:商品ID"`
	Quantity  int    `gorm:"not null;comment:数量"`
	Price     int64  `gorm:"not null;comment:下单单价(分)"`
}
```

标签速查：`primaryKey;autoIncrement` 主键自增；`type:bigint unsigned` 显式统一类型；`not null;default:0` 表达非空与零值语义；`uniqueIndex:name` 与 `index:name,priority:2` 让同名字段组成联合索引；`comment:` 写入表结构。金额统一用**分**存 `BIGINT`，不要用 `FLOAT/DOUBLE`。

## 五、时间字段与软删除

- GORM 自动填充 `CreatedAt`/`UpdatedAt`；自定义字段用 `autoCreateTime`/`autoUpdateTime`，`int64` 存毫秒时加 `autoCreateTime:milli`
- 数据库存 UTC、接口按需转时区最省心；至少保证所有服务 DSN 的 `loc` 一致
- `gorm.DeletedAt` 会让查询、更新、删除自动追加 `deleted_at IS NULL`；查已删除数据用 `db.Unscoped()`
- 软删除与唯一索引的冲突在第 03 篇给出完整方案

## 六、AutoMigrate 为什么生产要慎用

仅在本地开发与自动化测试中调用 `db.AutoMigrate(&User{}, &Product{}, &Stock{}, &Order{}, &OrderItem{})`。它只做“加法”：建表、补列、补索引；**不会删除多余列、不会安全地改类型、不会回填数据、没有 down 脚本**，大表上还可能锁表并造成主从延迟。生产一律使用版本化迁移（第 06 篇）。

## 七、CRUD 核心用法

**批量创建**（自动开启事务）：

```go
users := []User{{Email: "alice@mall.dev", Password: "plain"}, {Email: "bob@mall.dev", Password: "plain"}}
if err := db.WithContext(ctx).CreateInBatches(&users, 100).Error; err != nil {
	return err
}
```

**查询**：`First` 按主键升序取一条，`Take` 不排序，`Find` 取多条且**不返回** `ErrRecordNotFound`。

```go
var u User
err := db.WithContext(ctx).Where("email = ?", "alice@mall.dev").First(&u).Error
if errors.Is(err, gorm.ErrRecordNotFound) {
	return nil, ErrUserNotFound
}
var list []Product
db.WithContext(ctx).Where("status = ?", 1).Order("id DESC").Limit(20).Find(&list)
db.WithContext(ctx).Find(&list, []uint64{1, 2, 3})                // 主键 IN
db.WithContext(ctx).Select("id", "name", "price").Find(&list) // 避免 SELECT *
```

**条件组合**：多个 `Where` 之间是 AND；需要 OR 时用条件组，避免 OR 逃出预期范围：

```go
db.WithContext(ctx).Model(&Product{}).
	Where("status = ?", 1).
	Where("price BETWEEN ? AND ?", 1000, 5000).
	Where(db.Where("name LIKE ?", "耳机%"). // 前缀模糊才能走索引
		Or("sku = ?", "SKU-001")).
	Find(&products)
```

**更新：Update vs Updates vs Save**：

```go
// 1) Updates(map)：显式更新，零值也生效（推荐）
db.Model(&User{}).Where("id = ?", id).
	Updates(map[string]interface{}{"nickname": "bob", "status": 0})
// 2) Update：只更新单个字段
db.Model(&User{}).Where("id = ?", id).Update("status", 0)
// 3) Updates(struct)：默认忽略零值，status=0 不会更新
db.Model(&User{}).Where("id = ?", id).Updates(User{Nickname: "bob", Status: 0})
// 4) Select 指定字段，让 struct 的零值也生效
db.Model(&User{}).Where("id = ?", id).Select("Nickname", "Status").
	Updates(User{Nickname: "bob", Status: 0})
// 5) Save：按主键更新全部字段，容易覆盖并发修改
db.Save(&u)
```

**删除**：`db.Delete(&User{}, id)` 软删除，`db.Unscoped().Delete(&User{}, id)` 物理删除，`db.Where("created_at < ?", cutoff).Delete(&Order{})` 条件批量软删除。GORM 拒绝无条件的全局删除（`ErrMissingWhereClause`），这是保护机制，不要用 `AllowGlobalUpdate` 绕过。

## 八、关联关系与 Preload

| 关系 | 场景 | 标签写法 |
| --- | --- | --- |
| BelongsTo | 订单属于用户 | 可选 `User` 字段 + `foreignKey:UserID;references:ID` |
| HasOne | 商品有一份库存 | 可选 `Stock` 字段 + `foreignKey:ProductID` |
| HasMany | 订单有多个明细 | `Items []OrderItem` + `foreignKey:OrderID` |
| Many2Many | 用户收藏商品 | 可选 `Favs []Product` + `many2many:user_favorites` |

`Preload` 发 1+N 条 SQL、无重复行，适合取聚合根与多个子集合；`Joins` 只发一条 SQL，适合用子表条件筛父表，但一对多会产生重复行，需要 `Distinct` 或 `Group`。

```go
var orders []Order
db.WithContext(ctx).Preload("Items").Preload("Items.Product").
	Where("user_id = ?", uid).Find(&orders) // 订单 + 明细 + 商品，共 3 条 SQL

var rows []OrderRow
db.WithContext(ctx).Table("orders").Select("orders.order_no, users.nickname").
	Joins("JOIN users ON users.id = orders.user_id").
	Where("orders.status = ?", 1).Scan(&rows)
```

`Preload` 支持条件 `Preload("Items", "status = ?", 1)`，嵌套用 `Preload("Items.Product")`。**不要在循环里逐条查关联**，那就是 N+1（第 04 篇）。

## 九、钩子：BeforeCreate/AfterCreate

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
	if u.Email == "" {
		return errors.New("email required")
	}
	hash, err := bcrypt.GenerateFromPassword([]byte(u.Password), bcrypt.DefaultCost)
	if err != nil {
		return err
	}
	u.Password = string(hash) // 写入前哈希，业务层永远拿不到明文
	return nil
}
```

钩子在**同一个事务**内执行，返回 error 会回滚整条链路；批量 `Create` 会逐行触发，10 万行时开销明显。钩子里不要做 RPC/HTTP/MQ 等跨网络调用，否则事务被拉长、锁持有时间不可控。跳过钩子用 `db.Session(&gorm.Session{SkipHooks: true})`；`AfterCreate` 适合写本地 outbox 表，由独立消费者投递 Kafka。

## 常见坑

1. **`Updates(User{...})` 更新不了 `status=0`**：struct 零值被忽略，改用 map、`Update` 或 `Select`。
2. **`Save` 全字段覆盖**：把并发请求刚改的字段冲回旧值，表现为“更新丢失”。
3. **循环里单条查询关联**：100 个订单查 100 次明细，QPS 一高就打满数据库。
4. **字符串拼接 `Where`**：`fmt.Sprintf("name='%s'", input)` 直接 SQL 注入，永远用占位符。

## 小结

- DSN 必须带 `parseTime`、`loc` 与三类超时；日志生产用 `Warn` + 慢 SQL 阈值
- 模型标签要显式表达类型、非空、默认值、索引与注释，金额用分存 `BIGINT`
- CRUD 的核心差异在 `Updates` 的零值语义与 `Save` 的全字段覆盖
- 关联优先 `Preload`，用子表条件筛父表时再用 `Joins`
- 钩子只做本地、快速、可回滚的事；生产迁移不要依赖 `AutoMigrate`

## 练习

1. 为 `orders` 设计“用户 + 状态 + 创建时间”的联合索引，写出 GORM 标签并解释字段顺序。
2. 实现 `ListOrdersByUser`：返回订单及明细、明细商品，要求只发 3 条 SQL。
3. 写一个 `BeforeCreate` 钩子校验库存不能为负，并说明为什么不在钩子里调用 product-service。

## 官方文档

- GORM 模型与标签：<https://gorm.io/zh_CN/docs/models.html>
- GORM 查询：<https://gorm.io/zh_CN/docs/query.html>
- GORM 关联 Preload：<https://gorm.io/zh_CN/docs/preload.html>
- GORM Hooks：<https://gorm.io/zh_CN/docs/hooks.html>
- MySQL 8.4 数据类型：<https://dev.mysql.com/doc/refman/8.4/en/data-types.html>
