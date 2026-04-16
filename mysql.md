# MySQL 设计规范

## 数据库基础

### 版本与配置
- **MySQL 版本**: 8.x
- **ORM**: Entity Framework Core 8 + Pomelo MySQL Provider
- **字符集**: `utf8mb4`
- **排序规则**: `utf8mb4_unicode_ci`
- **存储引擎**: InnoDB

### 连接字符串
```
Server=localhost;Port=3306;Database=mydb;Uid=root;Pwd=password;Charset=utf8mb4;SslMode=None;
```

## 表设计规范

### 命名规则
- **表名**: 小写下划线，复数形式
  - ✓ `users`, `order_items`, `product_categories`
  - ✗ `User`, `orderItems`, `T_USER`

- **字段名**: 小写下划线
  - ✓ `user_name`, `created_at`, `is_deleted`
  - ✗ `userName`, `CreatedAt`, `IS_DELETED`

- **主键**: `id` 或 `{table}_id`
  - ✓ `id`, `user_id`

- **外键**: `{referenced_table}_id`
  - ✓ `user_id`, `order_id`

### 必备字段
每张表必须包含以下字段：

```sql
`id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
`created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
`updated_at` DATETIME(3) NULL ON UPDATE CURRENT_TIMESTAMP(3),
`is_deleted` TINYINT(1) NOT NULL DEFAULT 0,
`version` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '乐观锁版本号',
```

### EF Core 实体
```csharp
public class BaseEntity : IBaseEntity<long>
{
    [Key]
    [Column("id")]
    public long Id { get; set; }

    [Column("created_at")]
    public DateTime CreatedAt { get; set; }

    [Column("updated_at")]
    public DateTime? UpdatedAt { get; set; }

    [Column("is_deleted")]
    public bool IsDeleted { get; set; }

    [Column("version")]
    [ConcurrencyCheck]
    public int Version { get; set; }
}
```

## 字段类型映射

### C# → MySQL 类型映射表

| C# 类型 | MySQL 类型 | 说明 |
|---------|-----------|------|
| `long` | `BIGINT UNSIGNED` | 主键、外键 |
| `int` | `INT UNSIGNED` | 数量、状态 |
| `short` | `SMALLINT UNSIGNED` | 小范围状态 |
| `byte` | `TINYINT UNSIGNED` | 布尔、小状态 |
| `bool` | `TINYINT(1)` | 布尔 |
| `decimal` | `DECIMAL(18,4)` | 金额 |
| `double` | `DOUBLE` | 科学计算 |
| `float` | `FLOAT` | 一般精度 |
| `string` | `VARCHAR(N)` / `TEXT` | 字符串 |
| `DateTime` | `DATETIME(3)` | 时间 |
| `Guid` | `CHAR(36)` | UUID |
| `byte[]` | `LONGBLOB` | 二进制 |
| `JsonDocument` | `JSON` | JSON数据 |

### 字段长度建议

| 场景 | 类型 | 长度 |
|------|------|------|
| 用户名 | VARCHAR | 50 |
| 密码哈希 | VARCHAR | 255 |
| 邮箱 | VARCHAR | 100 |
| 手机号 | VARCHAR | 20 |
| 名称/标题 | VARCHAR | 200 |
| 描述/简介 | VARCHAR | 500 |
| 详细内容 | TEXT | - |
| URL | VARCHAR | 500 |
| IP地址 | VARCHAR | 45 |
| 金额 | DECIMAL | (18,4) |

## 索引设计

### 必须建索引的字段
- 主键（自动）
- 外键
- WHERE 条件常用字段
- ORDER BY 字段
- JOIN 关联字段
- 唯一约束字段

### 索引原则
1. **最左前缀原则**: 复合索引从左到右匹配
2. **单表索引数**: 不超过 5 个
3. **复合索引列**: 不超过 4 列
4. **索引命名**: `idx_{table}_{column1}_{column2}` 或 `uniq_{table}_{column}`

### EF Core 索引配置
```csharp
// 单列索引
modelBuilder.Entity<User>()
    .HasIndex(e => e.UserName)
    .IsUnique()
    .HasDatabaseName("uniq_user_user_name");

// 复合索引
modelBuilder.Entity<Order>()
    .HasIndex(e => new { e.UserId, e.Status })
    .HasDatabaseName("idx_order_user_id_status");

// 全文索引
modelBuilder.Entity<Product>()
    .HasIndex(e => e.Name)
    .HasDatabaseName("idx_product_name_fulltext")
    .HasMethod("FULLTEXT");
```

### SQL 索引创建
```sql
-- 普通索引
CREATE INDEX idx_order_user_id ON orders(user_id);

-- 唯一索引
CREATE UNIQUE INDEX uniq_user_email ON users(email);

-- 复合索引
CREATE INDEX idx_order_user_status ON orders(user_id, status);

-- 全文索引
CREATE FULLTEXT INDEX idx_product_name ON products(name);
```

## 数据库迁移

### EF Core Migrations 命令
```bash
# 添加迁移
dotnet ef migrations add InitialCreate

# 更新数据库
dotnet ef database update

# 生成 SQL 脚本
dotnet ef migrations script

# 回滚到指定迁移
dotnet ef database update PreviousMigration

# 删除最后一次迁移（未应用）
dotnet ef migrations remove
```

### 迁移注意事项
1. **生产环境禁止自动迁移**: 使用 SQL 脚本手动执行
2. **大表修改**: 在低峰期执行，可能需要临时禁用外键检查
3. **添加列**: 为 NOT NULL 列设置默认值
4. **删除列**: 确认无代码引用后再删除
5. **重命名**: 使用独立迁移，分步执行

### 大表迁移脚本模板
```sql
-- 1. 禁用外键检查（仅必要时）
SET FOREIGN_KEY_CHECKS = 0;

-- 2. 添加列（带默认值）
ALTER TABLE `orders`
ADD COLUMN `remark` VARCHAR(500) NULL COMMENT '备注';

-- 3. 创建索引（ONLINE 方式）
ALTER TABLE `orders`
ADD INDEX `idx_orders_created_at` (`created_at`) ALGORITHM=INPLACE, LOCK=NONE;

-- 4. 重新启用外键检查
SET FOREIGN_KEY_CHECKS = 1;
```

## 查询优化

### DO - 推荐做法
```csharp
// ✓ 使用 AsNoTracking 查询只读数据
var users = await _context.Users
    .AsNoTracking()
    .Where(e => e.Status == UserStatus.Active)
    .ToListAsync();

// ✓ 分页查询
var orders = await _context.Orders
    .Where(e => e.UserId == userId)
    .OrderByDescending(e => e.CreatedAt)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();

// ✓ 批量操作
await _context.Users
    .Where(e => e.Status == UserStatus.Inactive)
    .ExecuteUpdateAsync(e => e.SetProperty(u => u.IsDeleted, true));

// ✓ 只查询需要的字段
var userNames = await _context.Users
    .AsNoTracking()
    .Where(e => e.Status == UserStatus.Active)
    .Select(e => new { e.Id, e.UserName })
    .ToListAsync();
```

### DON'T - 禁止做法
```csharp
// ✗ 禁止 N+1 查询
var orders = await _context.Orders.ToListAsync();
foreach (var order in orders)
{
    var user = await _context.Users.FindAsync(order.UserId); // N+1
}

// ✓ 使用 Include 或 Join
var orders = await _context.Orders
    .Include(e => e.User)
    .ToListAsync();

// ✗ 禁止循环中执行数据库操作
foreach (var userId in userIds)
{
    var user = await _context.Users.FindAsync(userId); // N次查询
}

// ✓ 批量查询
var users = await _context.Users
    .Where(e => userIds.Contains(e.Id))
    .ToDictionaryAsync(e => e.Id);

// ✗ 禁止 SELECT *
var allData = await _context.Users.ToListAsync();

// ✓ 只查询需要的字段
var data = await _context.Users
    .Select(e => new { e.Id, e.UserName })
    .ToListAsync();

// ✗ 禁止大事务
using var transaction = await _context.Database.BeginTransactionAsync();
// 大量操作...
await transaction.CommitAsync();

// ✓ 小事务 + 批处理
foreach (var batch in items.Chunk(100))
{
    using var transaction = await _context.Database.BeginTransactionAsync();
    _context.Items.AddRange(batch);
    await _context.SaveChangesAsync();
    await transaction.CommitAsync();
}
```

## 事务管理

### 标准事务模式
```csharp
public class OrderService
{
    private readonly AppDbContext _context;

    public async Task<bool> CreateOrderAsync(CreateOrderRequest request)
    {
        await using var transaction = await _context.Database.BeginTransactionAsync();
        try
        {
            // 业务操作
            var order = new Order { ... };
            _context.Orders.Add(order);
            await _context.SaveChangesAsync();

            // 扣减库存
            var inventory = await _context.Inventories
                .FromSqlRaw("SELECT * FROM inventories WHERE product_id = {0} FOR UPDATE", request.ProductId)
                .FirstOrDefaultAsync();

            if (inventory == null || inventory.Quantity < request.Quantity)
                throw new BusinessException("库存不足");

            inventory.Quantity -= request.Quantity;
            await _context.SaveChangesAsync();

            await transaction.CommitAsync();
            return true;
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

### 分布式事务
```csharp
// 使用 System.Transaction
using var scope = new TransactionScope(
    TransactionScopeOption.Required,
    new TransactionOptions { IsolationLevel = IsolationLevel.ReadCommitted },
    TransactionScopeAsyncFlowOption.Enabled);

// 跨 DbContext 操作
await _orderContext.Orders.AddAsync(order);
await _orderContext.SaveChangesAsync();

await _inventoryContext.Inventories.UpdateAsync(inventory);
await _inventoryContext.SaveChangesAsync();

scope.Complete();
```

## 分表策略

### 分表时机
- 单表超过 500 万行
- 单表超过 10GB
- 单表 QPS 超过 2000

### 分表方案

#### 垂直分表
```
users          → users_base + users_profile
orders         → orders + order_details
products       → products + product_specs
```

#### 水平分表
```
orders         → orders_0, orders_1, ..., orders_15
order_items    → order_items_0, order_items_1, ..., order_items_15

分片键: user_id % 16
```

### EF Core 分表实现
```csharp
// 动态表名
public class Order
{
    public long Id { get; set; }
    public long UserId { get; set; }
    // ...
}

// 分表路由
public static string GetOrderTableName(long userId)
{
    var suffix = userId % 16;
    return $"orders_{suffix}";
}

// 查询时动态设置表名
var orders = await _context.Orders
    .FromSqlRaw($"SELECT * FROM {GetOrderTableName(userId)} WHERE user_id = {{0}}", userId)
    .ToListAsync();
```