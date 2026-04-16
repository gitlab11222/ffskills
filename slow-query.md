# 慢查询优化规范

## 慢查询定义

### 慢查询阈值
| 环境 | 阈值 | 说明 |
|------|------|------|
| 开发环境 | > 200ms | 开发调试用 |
| 测试环境 | > 500ms | 功能测试用 |
| 生产环境 | > 500ms 记录，> 2s 告警 | 业务监控 |

### MySQL 配置
```sql
-- 启用慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
SET GLOBAL long_query_time = 0.5;  -- 500ms
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录无索引查询

-- 查看配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';
```

## 慢查询分析

### EXPLAIN 分析
```sql
-- 分析执行计划
EXPLAIN SELECT * FROM orders WHERE user_id = 123;

-- 详细分析（MySQL 8.0）
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;
```

### EXPLAIN 输出解读
| 列 | 说明 | 关注点 |
|-----|------|--------|
| id | 查询标识 | 子查询顺序 |
| select_type | 查询类型 | SIMPLE最好 |
| table | 表名 | 关注大表 |
| partitions | 分区 | 分区表关注 |
| type | 访问类型 | system/const/eq_ref/ref最好 |
| possible_keys | 可能索引 | 有值说明有索引可用 |
| key | 实际索引 | NULL说明无索引使用 |
| key_len | 索引长度 | 越短越好 |
| ref | 索引参考 | 关联字段 |
| rows | 扫描行数 | 越少越好 |
| filtered | 过滤率 | 越高越好 |
| Extra | 附加信息 | 关注 Using temporary/filesort |

### type 类型解读
| 类型 | 说明 | 性能 |
|------|------|------|
| system | 系统表单行 | 最优 ✅ |
| const | 单行常量 | 最优 ✅ |
| eq_ref | 唯一索引关联 | 很好 ✅ |
| ref | 非唯一索引 | 好 ✅ |
| ref_or_null | 包含NULL | 一般 |
| range | 范围扫描 | 一般 |
| index | 全索引扫描 | 较慢 |
| ALL | 全表扫描 | 最慢 ❌ |

### Extra 信息解读
| 信息 | 说明 | 问题 |
|------|------|------|
| Using index | 覆盖索引 | 好 ✅ |
| Using where | WHERE过滤 | 正常 |
| Using temporary | 使用临时表 | 警告 🟡 |
| Using filesort | 文件排序 | 警告 🟡 |
| Using join buffer | 关联缓冲 | 警告 🟡 |
| Impossible WHERE | WHERE无效 | 检查逻辑 |

## 慢查询优化方法

### 优化检查清单
```markdown
# 慢查询优化检查清单

## 1. 索引检查
- [ ] 是否使用索引？
- [ ] 索引是否覆盖查询字段？
- [ ] 是否符合最左前缀？
- [ ] 索引是否失效？

## 2. SQL检查
- [ ] 避免 SELECT *
- [ ] WHERE条件是否有效？
- [ ] 是否有大表关联？
- [ ] 是否有子查询？

## 3. 数据量检查
- [ ] 扫描行数是否过多？
- [ ] 结果集是否过大？
- [ ] 是否需要分页？

## 4. 结构检查
- [ ] 是否有临时表？
- [ ] 是否有文件排序？
- [ ] 关联表数量是否过多？
```

### 索引优化

#### 添加索引
```sql
-- 单列索引
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 复合索引（注意顺序：最左前缀）
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- 覆盖索引（包含查询字段）
CREATE INDEX idx_orders_user_cover ON orders(user_id, status, created_at);
```

#### 索引使用原则
```sql
-- ✓ 索引生效
SELECT * FROM orders WHERE user_id = 123;  -- 单列索引
SELECT * FROM orders WHERE user_id = 123 AND status = 1;  -- 复合索引（最左前缀）

-- ✗ 索引失效
SELECT * FROM orders WHERE status = 1;  -- 复合索引第二列，不满足最左前缀
SELECT * FROM orders WHERE user_id + 1 = 123;  -- 索引列参与计算
SELECT * FROM orders WHERE user_id LIKE '%123';  -- 前缀模糊
SELECT * FROM orders WHERE user_id != 123;  -- 不等于
SELECT * FROM orders WHERE user_id IS NULL;  -- NULL判断（部分失效）
```

### SQL 优化

#### 避免 SELECT *
```sql
-- ✗ 慢查询：查询所有字段
SELECT * FROM orders WHERE user_id = 123;

-- ✓ 优化：只查询需要的字段
SELECT id, user_id, status, total_amount FROM orders WHERE user_id = 123;
```

#### 分页优化
```sql
-- ✗ 慢查询：大偏移量
SELECT * FROM orders LIMIT 10000, 20;

-- ✓ 优化：基于索引分页
SELECT * FROM orders WHERE id > 10000 LIMIT 20;

-- ✓ 优化：延迟关联
SELECT o.* FROM orders o
INNER JOIN (SELECT id FROM orders LIMIT 10000, 20) t
ON o.id = t.id;
```

#### 关联优化
```sql
-- ✗ 慢查询：大表关联
SELECT * FROM orders o
LEFT JOIN users u ON o.user_id = u.id
LEFT JOIN products p ON o.product_id = p.id;

-- ✓ 优化：小表驱动大表
SELECT o.id, u.name, p.name
FROM (SELECT id, user_id, product_id FROM orders WHERE status = 1 LIMIT 100) o
LEFT JOIN users u ON o.user_id = u.id
LEFT JOIN products p ON o.product_id = p.id;
```

#### 子查询优化
```sql
-- ✗ 慢查询：相关子查询
SELECT * FROM orders o
WHERE o.user_id = (SELECT id FROM users WHERE name = '张三');

-- ✓ 优化：改为JOIN
SELECT o.* FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE u.name = '张三';
```

#### ORDER BY 优化
```sql
-- ✗ 慢查询：无索引排序
SELECT * FROM orders ORDER BY created_at DESC LIMIT 100;

-- ✓ 优化：添加排序索引
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- ✓ 优化：覆盖索引排序
SELECT id, created_at FROM orders ORDER BY created_at DESC LIMIT 100;
```

#### GROUP BY 优化
```sql
-- ✗ 慢查询：无索引分组
SELECT status, COUNT(*) FROM orders GROUP BY status;

-- ✓ 优化：添加分组索引
CREATE INDEX idx_orders_status ON orders(status);

-- ✓ 优化：使用覆盖索引
SELECT status, COUNT(*) FROM orders 
WHERE status IN (1, 2, 3)
GROUP BY status;
```

### 数据量优化

#### 分区表
```sql
-- 按时间分区
CREATE TABLE orders (
    id BIGINT,
    user_id BIGINT,
    created_at DATETIME,
    ...
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);

-- 查询指定分区
SELECT * FROM orders PARTITION(p2024) WHERE user_id = 123;
```

#### 分库分表
```sql
-- 原单表 orders 拆分为 orders_0 ~ orders_15
-- 分片键: user_id % 16

-- 查询路由
-- user_id = 123 → orders_123 % 16 = orders_11
SELECT * FROM orders_11 WHERE user_id = 123;
```

## 慢查询监控

### 慢查询日志分析
```bash
# 使用 mysqldumpslow 分析
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# 参数说明
-s t: 按查询时间排序
-s c: 按查询次数排序
-s l: 按锁定时间排序
-s r: 按返回记录数排序
-t 10: 显示前10条

# 使用 pt-query-digest 分析
pt-query-digest /var/log/mysql/slow.log > slow_report.txt
```

### 实时慢查询监控
```sql
-- 查看当前执行查询
SELECT * FROM information_schema.processlist 
WHERE time > 5 AND command != 'Sleep';

-- 查看锁等待
SELECT * FROM information_schema.innodb_lock_waits;

-- 查看事务状态
SELECT * FROM information_schema.innodb_trx;
```

### Prometheus 监控
```yaml
# MySQL慢查询指标
- alert: MySQLSlowQuery
  expr: mysql_global_status_slow_queries > 100
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "MySQL慢查询数量过多"
    description: "慢查询数量: {{ $value }}"
```

## 慢查询记录

### 慢查询记录模板
```markdown
## SQ-{编号}: {慢查询摘要}

### 基本信息
- **发现时间**: {YYYY-MM-DD HH:mm}
- **发现人**: {姓名}
- **环境**: 生产/测试
- **数据库**: {库名}

### SQL信息
- **SQL摘要**: {SQL}
- **执行时间**: {ms}
- **扫描行数**: {rows}
- **返回行数**: {rows}

### EXPLAIN分析
| id | type | key | rows | Extra |
|----|------|-----|------|-------|
| {值} | {值} | {值} | {值} | {值} |

### 问题诊断
- **问题类型**: 无索引/索引失效/全表扫描/临时表/文件排序
- **问题原因**: {分析}

### 优化方案
- **方案**: {优化方案}
- **SQL优化**: {优化后SQL}
- **索引优化**: {新增索引}

### 优化效果
| 指标 | 优化前 | 优化后 | 提升 |
|------|--------|--------|------|
| 执行时间 | {ms} | {ms} | {倍} |
| 扫描行数 | {rows} | {rows} | {倍} |

### 验证结果
- **验证人**: {姓名}
- **验证时间**: {YYYY-MM-DD HH:mm}
- **验证结果**: ✅ 通过
```

### 慢查询统计表
| 慢查询ID | SQL摘要 | 执行时间 | 扫描行数 | 问题 | 优化方案 | 效果 |
|----------|----------|----------|----------|------|----------|------|
| SQ-001 | SELECT * FROM orders WHERE user_id=? | 500ms | 10000 | 无索引 | 添加索引 | 500ms→50ms |
| SQ-002 | SELECT * FROM orders ORDER BY created_at | 2000ms | 500000 | 文件排序 | 添加排序索引 | 2000ms→100ms |

## 慢查询预防

### 开发规范
- 所有查询必须有索引覆盖
- 避免 SELECT *
- 大数据量必须分页
- 复杂查询必须先 EXPLAIN

### 索引设计规范
- WHERE/ORDER BY/GROUP BY/JOIN 字段建索引
- 复合索引遵循最左前缀
- 覆盖索引减少回表

### 代码审查
```markdown
# SQL代码审查检查清单

## 索引检查
- [ ] WHERE条件是否有索引？
- [ ] ORDER BY字段是否有索引？
- [ ] GROUP BY字段是否有索引？
- [ ] JOIN字段是否有索引？

## SQL检查
- [ ] 是否使用 SELECT *？
- [ ] 是否有大偏移分页？
- [ ] 是否有大表关联？
- [ ] 是否有相关子查询？

## 性能检查
- [ ] 是否执行 EXPLAIN 分析？
- [ ] 扫描行数是否 < 1000？
- [ ] 是否有 Using temporary？
- [ ] 是否有 Using filesort？
```

## DO - 推荐做法

```sql
-- ✓ 查询前执行 EXPLAIN
EXPLAIN SELECT * FROM orders WHERE user_id = 123;

-- ✓ 只查询需要的字段
SELECT id, user_id, status FROM orders WHERE user_id = 123;

-- ✓ 使用索引分页
SELECT * FROM orders WHERE id > 10000 LIMIT 20;

-- ✓ 小表驱动大表
FROM (SELECT id FROM orders LIMIT 100) o JOIN users u

-- ✓ 使用覆盖索引
SELECT id, user_id FROM orders WHERE user_id = 123;  -- 索引包含 id, user_id

-- ✓ 复合索引遵循最左前缀
CREATE INDEX idx_user_status ON orders(user_id, status);
WHERE user_id = 123 AND status = 1  -- 使用索引
```

## DON'T - 禁止做法

```sql
-- ✗ 不执行 EXPLAIN
SELECT * FROM orders WHERE user_id = 123;  // 禁止，未分析

-- ✓ 正确做法
EXPLAIN SELECT * FROM orders WHERE user_id = 123;
-- 分析后再执行

-- ✗ 使用 SELECT *
SELECT * FROM orders;  // 禁止，查询所有字段

-- ✓ 正确做法
SELECT id, user_id, status FROM orders;

-- ✗ 大偏移分页
LIMIT 10000, 20;  // 禁止，扫描前10020行

-- ✓ 正确做法
WHERE id > 10000 LIMIT 20;

-- ✗ 无索引排序
ORDER BY created_at  // 禁止，文件排序

-- ✓ 正确做法
CREATE INDEX idx_created_at ON orders(created_at);
ORDER BY created_at

-- ✗ 索引列参与计算
WHERE user_id + 1 = 123  // 禁止，索引失效

-- ✓ 正确做法
WHERE user_id = 122

-- ✗ 前缀模糊
WHERE user_id LIKE '%123'  // 禁止，索引失效

-- ✓ 正确做法
WHERE user_id LIKE '123%'
```

## 最佳实践

### 慢查询治理流程
```
发现慢查询 → EXPLAIN分析 → 问题诊断 → 制定方案 → 实施优化 → 效果验证 → 记录归档
```

### 慢查询预防
- 开发阶段执行 EXPLAIN
- 索引设计遵循规范
- 代码审查检查 SQL
- 定期分析慢查询日志

### 持续监控
- 慢查询日志持续收集
- 慢查询数量告警
- 慢查询优化记录
- 定期慢查询评审