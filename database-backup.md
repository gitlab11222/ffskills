# 数据库备份规范

## 备份策略

### 备份类型
| 类型 | 说明 | 频率 | 保留周期 |
|------|------|------|----------|
| 全量备份 | 完整数据库备份 | 每日凌晨2点 | 7天 |
| 增量备份 | Binlog增量备份 | 每6小时 | 3天 |
| 实时备份 | Binlog实时同步 | 实时 | 24小时 |
| 结构备份 | 表结构备份 | 每周 | 30天 |

### 备份保留策略
| 环境 | 全量保留 | 增量保留 | Binlog保留 |
|------|----------|----------|------------|
| 开发环境 | 3天 | 1天 | 不保留 |
| 测试环境 | 7天 | 3天 | 24小时 |
| 生产环境 | 30天 | 7天 | 7天 |

## 全量备份

### mysqldump 全量备份
```bash
# 全库备份
mysqldump -h$HOST -u$USER -p$PASSWORD \
  --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --master-data=2 \
  --flush-logs \
  | gzip > /backup/full_$(date +%Y%m%d).sql.gz

# 单库备份
mysqldump -h$HOST -u$USER -p$PASSWORD \
  --single-transaction \
  --routines \
  --triggers \
  mydb | gzip > /backup/mydb_$(date +%Y%m%d).sql.gz

# 单表备份
mysqldump -h$HOST -u$USER -p$PASSWORD \
  --single-transaction \
  mydb orders | gzip > /backup/orders_$(date +%Y%m%d).sql.gz

# 仅结构备份
mysqldump -h$HOST -u$USER -p$PASSWORD \
  --no-data \
  mydb > /backup/mydb_structure_$(date +%Y%m%d).sql
```

### 参数说明
| 参数 | 说明 |
|------|------|
| --single-transaction | InnoDB一致性备份，不锁表 |
| --routines | 包含存储过程、函数 |
| --triggers | 包含触发器 |
| --events | 包含事件 |
| --master-data=2 | 记录Binlog位置 |
| --flush-logs | 刷新日志，便于增量备份 |

### 自动化备份脚本
```bash
#!/bin/bash
# full_backup.sh

# 配置
HOST="localhost"
USER="backup"
PASSWORD="xxx"
BACKUP_DIR="/backup/mysql"
RETENTION_DAYS=7

# 创建备份目录
mkdir -p $BACKUP_DIR

# 执行备份
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/full_$DATE.sql.gz"

mysqldump -h$HOST -u$USER -p$PASSWORD \
  --all-databases \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --master-data=2 \
  | gzip > $BACKUP_FILE

# 检查备份结果
if [ $? -eq 0 ]; then
  echo "[$DATE] 备份成功: $BACKUP_FILE"
  
  # 发送通知
  curl -X POST "https://oapi.dingtalk.com/robot/send?access_token=xxx" \
    -H "Content-Type: application/json" \
    -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"MySQL全量备份成功: $BACKUP_FILE\"}}"
else
  echo "[$DATE] 备份失败"
  
  # 发送告警
  curl -X POST "https://oapi.dingtalk.com/robot/send?access_token=xxx" \
    -H "Content-Type: application/json" \
    -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"🔴 MySQL全量备份失败\"}}"
fi

# 清理过期备份
find $BACKUP_DIR -name "full_*.sql.gz" -mtime +$RETENTION_DAYS -delete
echo "[$DATE] 清理过期备份完成"

# 记录日志
echo "[$DATE] 备份大小: $(ls -lh $BACKUP_FILE | awk '{print $5}')" >> $BACKUP_DIR/backup.log
```

### Cron 定时任务
```bash
# 每日凌晨2点执行全量备份
0 2 * * * /scripts/full_backup.sh

# 每6小时执行增量备份
0 */6 * * * /scripts/incremental_backup.sh
```

## 增量备份

### Binlog 增量备份
```bash
# 查看Binlog列表
SHOW BINARY LOGS;

# 备份Binlog
mysqlbinlog --read-from-remote-server \
  --host=$HOST \
  --user=$USER \
  --password=$PASSWORD \
  --raw \
  --stop-never \
  mysql-bin.000001 mysql-bin.000002 \
  --result-file=/backup/binlog/

# 从指定位置备份
mysqlbinlog --start-position=1000 \
  --stop-position=2000 \
  mysql-bin.000001 > /backup/incremental.sql
```

### 增量备份脚本
```bash
#!/bin/bash
# incremental_backup.sh

HOST="localhost"
USER="backup"
PASSWORD="xxx"
BACKUP_DIR="/backup/mysql/binlog"
RETENTION_HOURS=72

# 获取当前Binlog位置
CURRENT_BINLOG=$(mysql -h$HOST -u$USER -p$PASSWORD -e "SHOW MASTER STATUS" | awk 'NR==2 {print $1}')
CURRENT_POSITION=$(mysql -h$HOST -u$USER -p$PASSWORD -e "SHOW MASTER STATUS" | awk 'NR==2 {print $2}')

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/binlog_$DATE.tar.gz"

# 复制Binlog文件
tar -czf $BACKUP_FILE /var/lib/mysql/mysql-bin.*

if [ $? -eq 0 ]; then
  echo "[$DATE] Binlog备份成功: $BACKUP_FILE"
  
  # 记录位置信息
  echo "Binlog: $CURRENT_BINLOG, Position: $CURRENT_POSITION" > $BACKUP_DIR/position_$DATE.txt
else
  echo "[$DATE] Binlog备份失败"
fi

# 清理过期备份
find $BACKUP_DIR -name "binlog_*.tar.gz" -mtime +3 -delete
find $BACKUP_DIR -name "position_*.txt" -mtime +3 -delete
```

## 数据恢复

### 全量恢复
```bash
# 解压备份文件
gunzip full_20240101.sql.gz

# 恢复数据库
mysql -h$HOST -u$USER -p$PASSWORD < full_20240101.sql

# 或指定库恢复
mysql -h$HOST -u$USER -p$PASSWORD mydb < mydb_20240101.sql
```

### 增量恢复
```bash
# 1. 先恢复全量备份
mysql -h$HOST -u$USER -p$PASSWORD < full_20240101.sql

# 2. 应用Binlog增量
mysqlbinlog --start-position=1000 \
  --stop-position=2000 \
  mysql-bin.000001 | mysql -h$HOST -u$USER -p$PASSWORD

# 3. 应用后续Binlog
mysqlbinlog mysql-bin.000002 | mysql -h$HOST -u$USER -p$PASSWORD
```

### 时间点恢复
```bash
# 恢复到指定时间点
mysqlbinlog --start-datetime="2024-01-01 10:00:00" \
  --stop-datetime="2024-01-01 12:00:00" \
  mysql-bin.000001 | mysql -h$HOST -u$USER -p$PASSWORD
```

### 单表恢复
```bash
# 从全量备份提取单表
# 1. 找到表结构位置
grep -n "CREATE TABLE \`orders\`" full_20240101.sql

# 2. 提取表结构和数据
sed -n '/CREATE TABLE `orders`/,/CREATE TABLE `.*`/p' full_20240101.sql > orders.sql

# 3. 恢复单表
mysql -h$HOST -u$USER -p$PASSWORD mydb < orders.sql
```

## 恢复流程

### 标准恢复流程
```markdown
## 恢复流程模板

### 1. 恢复准备
- **恢复原因**: {原因}
- **恢复时间点**: {时间点}
- **影响范围**: {影响评估}
- **恢复方案**: 全量恢复 + 增量恢复

### 2. 恢复执行

#### Step 1: 停止应用服务
```bash
kubectl scale deployment order-service --replicas=0
```

#### Step 2: 恢复全量备份
```bash
gunzip full_20240101.sql.gz
mysql -h$HOST -u$USER -p$PASSWORD < full_20240101.sql
```

#### Step 3: 应用增量备份
```bash
mysqlbinlog --start-position=1000 mysql-bin.000001 | mysql -h$HOST -u$USER -p$PASSWORD
mysqlbinlog mysql-bin.000002 | mysql -h$HOST -u$USER -p$PASSWORD
```

#### Step 4: 数据验证
```bash
# 检查表数量
mysql -e "SELECT COUNT(*) FROM orders"

# 检查关键数据
mysql -e "SELECT * FROM orders WHERE id = 1"
```

#### Step 5: 启动服务
```bash
kubectl scale deployment order-service --replicas=3
```

### 3. 恢复验证
- [ ] 服务启动正常
- [ ] 数据完整性验证
- [ ] 功能验证通过
- [ ] 监控指标正常

### 4. 恢复记录
| 时间 | 操作 | 结果 |
|------|------|------|
| {时间} | {操作} | {结果} |
```

## 备份验证

### 备份完整性验证
```bash
# 检查备份文件大小
ls -lh /backup/full_20240101.sql.gz

# 解压检查
gunzip -t full_20240101.sql.gz

# 检查备份内容
zcat full_20240101.sql.gz | head -100

# 统计表数量
zcat full_20240101.sql.gz | grep "CREATE TABLE" | wc -l

# 统计数据行数
zcat full_20240101.sql.gz | grep "INSERT INTO" | wc -l
```

### 定期恢复演练
```markdown
# 恢复演练计划

## 演练频率
| 类型 | 频率 | 说明 |
|------|------|------|
| 全量恢复演练 | 每月 | 恢复到测试环境验证 |
| 增量恢复演练 | 每季度 | 时间点恢复演练 |
| 单表恢复演练 | 每月 | 单表恢复演练 |

## 演练流程
1. 在测试环境准备空数据库
2. 恢复最近全量备份
3. 应用增量备份
4. 验证数据完整性
5. 记录演练结果

## 演练记录模板
| 演练日期 | 备份文件 | 恢复耗时 | 数据验证 | 结论 |
|----------|----------|----------|----------|------|
| 2024-01-01 | full_20240101.sql.gz | 30分钟 | 通过 | ✅ 成功 |
```

### 自动化验证脚本
```bash
#!/bin/bash
# backup_verify.sh

BACKUP_FILE="/backup/full_$(date +%Y%m%d).sql.gz"
TEST_DB="backup_verify"
HOST="localhost"
USER="root"
PASSWORD="xxx"

echo "开始验证备份文件: $BACKUP_FILE"

# 1. 解压检查
if ! gunzip -t $BACKUP_FILE; then
  echo "❌ 备份文件损坏"
  exit 1
fi
echo "✅ 备份文件完整"

# 2. 创建测试库
mysql -h$HOST -u$USER -p$PASSWORD -e "CREATE DATABASE IF NOT EXISTS $TEST_DB"

# 3. 恢复到测试库
zcat $BACKUP_FILE | mysql -h$HOST -u$USER -p$PASSWORD $TEST_DB

# 4. 检查表数量
TABLE_COUNT=$(mysql -h$HOST -u$USER -p$PASSWORD -N -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='$TEST_DB'")
echo "✅ 恢复表数量: $TABLE_COUNT"

# 5. 清理测试库
mysql -h$HOST -u$USER -p$PASSWORD -e "DROP DATABASE $TEST_DB"

echo "✅ 备份验证通过"
```

## 备份存储

### 存储位置
| 环境 | 存储位置 | 说明 |
|------|----------|------|
| 本地存储 | /backup/mysql | 本地磁盘 |
| 远程存储 | OSS/S3 | 云存储异地备份 |
| 冷存储 | 归档存储 | 长期保存 |

### OSS异地备份
```bash
# 上传到阿里云OSS
aliyun oss cp /backup/full_20240101.sql.gz oss://backup/mysql/full_20240101.sql.gz

# 定期同步
aliyun oss sync /backup/mysql oss://backup/mysql/ --delete

# 获取备份列表
aliyun oss ls oss://backup/mysql/
```

### 备份加密
```bash
# 加密备份文件
openssl enc -aes-256-cbc -salt -in full.sql -out full.sql.enc -pass pass:xxx

# 解密备份文件
openssl enc -aes-256-cbc -d -in full.sql.enc -out full.sql -pass pass:xxx
```

## 备份监控

### 监控指标
| 指标 | 告警阈值 | 说明 |
|------|----------|------|
| 备份失败 | 任何失败 | 立即告警 |
| 备份延迟 | > 1小时 | 告警 |
| 备份大小变化 | 骤降50% | 告警 |
| 备份文件损坏 | 检查失败 | 告警 |
| 存储空间不足 | > 85% | 告警 |

### 备份状态检查
```bash
# 检查今天是否有备份
ls -lh /backup/full_$(date +%Y%m%d).sql.gz

# 检查备份时间
stat /backup/full_$(date +%Y%m%d).sql.gz | grep Modify

# 检查备份大小
du -sh /backup/mysql/
```

## DO - 推荐做法

```bash
# ✓ 使用 --single-transaction 不锁表
mysqldump --single-transaction mydb > backup.sql

# ✓ 备份压缩存储
mysqldump mydb | gzip > backup.sql.gz

# ✓ 备份异地存储
aliyun oss cp backup.sql.gz oss://backup/

# ✓ 定期验证备份
gunzip -t backup.sql.gz

# ✓ 定期恢复演练
每月执行恢复演练

# ✓ 备份有通知
curl -X POST dingtalk_webhook -d "备份成功"

# ✓ 备份加密存储
openssl enc -aes-256-cbc -in backup.sql -out backup.sql.enc
```

## DON'T - 禁止做法

```bash
# ✗ 备份无压缩存储
mysqldump mydb > backup.sql  // 禁止，占用空间大

# ✓ 正确做法
mysqldump mydb | gzip > backup.sql.gz

# ✗ 备份无验证
mysqldump mydb > backup.sql
# 不验证  // 禁止

# ✓ 正确做法
gunzip -t backup.sql.gz

# ✗ 备份无异地存储
备份仅本地  // 禁止，单点故障

# ✓ 正确做法
本地 + OSS异地备份

# ✗ 备份无通知
备份失败无告警  // 禁止

# ✓ 正确做法
备份成功/失败都发送通知

# ✗ 备份保留过短
保留1天  // 禁止，无法回溯

# ✓ 正确做法
生产环境保留30天

# ✗ 无恢复演练
从不演练恢复  // 禁止

# ✓ 正确做法
每月执行恢复演练
```

## 最佳实践

### 备份自动化
- Cron定时执行
- 备份结果通知
- 自动清理过期备份
- 自动验证备份完整性

### 备份安全
- 加密存储敏感数据
- 异地备份防灾难
- 保留足够周期
- 定期验证可用性

### 恢复准备
- 恢复文档准备
- 恢复演练定期执行
- 恢复工具准备
- 恢复预案制定