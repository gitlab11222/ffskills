# 测试数据管理规范

## 数据分类

### 数据类型
| 类型 | 说明 | 来源 |
|------|------|------|
| 基础数据 | 用户、角色、权限 | 数据库初始化 |
| 业务数据 | 订单、商品、支付 | API/界面创建 |
| 测试数据 | 测试专用数据 | 测试脚本生成 |
| 边界数据 | 边界值测试数据 | 手工准备 |

### 数据生命周期
```
数据准备 → 数据使用 → 数据清理 → 数据归档
```

## 数据准备方式

### SQL脚本准备
```sql
-- 初始化基础数据
-- init_data.sql

-- 创建测试用户
INSERT INTO users (id, user_name, password_hash, status) VALUES
(1, 'testuser', 'hash123', 1),
(2, 'admin', 'hash456', 1);

-- 创建测试商品
INSERT INTO products (id, name, price, stock, status) VALUES
(1, '测试商品A', 100.00, 100, 1),
(2, '测试商品B', 200.00, 50, 1);

-- 创建测试订单
INSERT INTO orders (id, user_id, total_amount, status) VALUES
(1, 1, 100.00, 10),
(2, 1, 200.00, 20);
```

### API接口创建
```typescript
// 通过API创建测试数据
export class DataPreparation {
  private apiClient: ApiClient;

  async createTestUser(user: Partial<User>) {
    const response = await this.apiClient.post('/api/user/create', {
      userName: user.userName || `test_${Date.now()}`,
      password: user.password || 'password123',
      email: user.email || `test_${Date.now()}@example.com`,
    });
    return response.data.id;
  }

  async createTestProduct(product: Partial<Product>) {
    const response = await this.apiClient.post('/api/product/create', {
      name: product.name || `测试商品_${Date.now()}`,
      price: product.price || 100,
      stock: product.stock || 50,
    });
    return response.data.id;
  }

  async createTestOrder(order: Partial<Order>) {
    const response = await this.apiClient.post('/api/order/create', {
      userId: order.userId || 1,
      productId: order.productId || 1,
      quantity: order.quantity || 1,
    });
    return response.data.id;
  }
}
```

### 界面手工创建
```markdown
# 手工数据准备流程

## 前置条件
- 测试账号已创建
- 测试环境已部署
- 必要配置已完成

## 数据创建步骤
1. 登录测试账号
2. 进入功能页面
3. 手工创建数据
4. 记录数据ID

## 数据记录模板
| 数据ID | 数据类型 | 创建时间 | 创建人 | 说明 |
|--------|----------|----------|--------|------|
| 1001 | 用户 | 2024-01-01 | 测试人员 | 测试用户 |
| 2001 | 商品 | 2024-01-01 | 测试人员 | 测试商品 |
```

## 数据工厂模式

### 数据工厂类
```typescript
// TestDataFactory.ts
export class TestDataFactory {
  private counter = 0;

  // 创建用户数据
  createUser(overrides: Partial<User> = {}): User {
    this.counter++;
    return {
      id: this.counter,
      userName: `test_user_${Date.now()}_${this.counter}`,
      password: 'password123',
      email: `test_${this.counter}@example.com`,
      phone: `13800138000`,
      status: UserStatus.Active,
      createdAt: new Date(),
      ...overrides
    };
  }

  // 创建商品数据
  createProduct(overrides: Partial<Product> = {}): Product {
    this.counter++;
    return {
      id: this.counter,
      name: `测试商品_${this.counter}`,
      price: 100,
      stock: 50,
      status: ProductStatus.Active,
      ...overrides
    };
  }

  // 创建订单数据
  createOrder(overrides: Partial<Order> = {}): Order {
    this.counter++;
    return {
      id: this.counter,
      userId: 1,
      productId: 1,
      quantity: 1,
      totalAmount: 100,
      status: OrderStatus.Pending,
      createdAt: new Date(),
      ...overrides
    };
  }

  // 批量创建
  createUsers(count: number): User[] {
    return Array.from({ length: count }, () => this.createUser());
  }

  createProducts(count: number): Product[] {
    return Array.from({ length: count }, () => this.createProduct());
  }

  createOrders(count: number): Order[] {
    return Array.from({ length: count }, () => this.createOrder());
  }
}
```

### 测试中使用
```typescript
// 测试用例中使用数据工厂
describe('订单服务测试', () => {
  const factory = new TestDataFactory();

  test('创建订单', async () => {
    const user = factory.createUser();
    const product = factory.createProduct({ stock: 100 });
    const order = factory.createOrder({ userId: user.id, productId: product.id });

    const result = await orderService.create(order);
    expect(result).toBeDefined();
  });

  test('批量创建', async () => {
    const users = factory.createUsers(10);
    const products = factory.createProducts(10);
    // ...
  });
});
```

## 边界数据准备

### 边界值数据
| 字段 | 最小值 | 最大值 | 边界值-1 | 边界值+1 |
|------|--------|--------|----------|----------|
| 数量 | 1 | 100 | 0 | 101 |
| 价格 | 0.01 | 999999.99 | 0 | 1000000 |
| 名称长度 | 1 | 200 | 0 | 201 |
| 手机号 | 11位 | 11位 | 10位 | 12位 |

### 边界数据工厂
```typescript
export class BoundaryDataFactory {
  // 数量边界
  getQuantityBoundary() {
    return {
      min: 1,
      max: 100,
      belowMin: 0,
      aboveMax: 101,
    };
  }

  // 价格边界
  getPriceBoundary() {
    return {
      min: 0.01,
      max: 999999.99,
      belowMin: 0,
      aboveMax: 1000000,
    };
  }

  // 字符长度边界
  getNameBoundary() {
    return {
      minLength: 1,
      maxLength: 200,
      belowMin: '',
      aboveMax: 'x'.repeat(201),
    };
  }

  // 手机号边界
  getPhoneBoundary() {
    return {
      valid: '13800138000',
      short: '1380013800',  // 10位
      long: '138001380001', // 12位
      invalid: 'abc123',
    };
  }
}
```

## 数据隔离

### 环境数据隔离
| 环境 | 数据库 | Redis | 说明 |
|------|------|-------|------|
| 开发环境 | dev_db | dev_redis | 开发自测 |
| 测试环境 | test_db | test_redis | 功能测试 |
| 预发布环境 | pre_db | pre_redis | 验收测试 |
| 生产环境 | prod_db | prod_redis | 生产数据 |

### 数据隔离配置
```yaml
# 环境配置
environments:
  dev:
    database: dev_db
    redis: dev_redis
    api: http://dev.example.com
    
  test:
    database: test_db
    redis: test_redis
    api: http://test.example.com
    
  pre:
    database: pre_db
    redis: pre_redis
    api: http://pre.example.com
    
  prod:
    database: prod_db
    redis: prod_redis
    api: http://www.example.com
```

### 测试数据标记
```sql
-- 添加测试数据标记
ALTER TABLE users ADD COLUMN is_test_data TINYINT(1) DEFAULT 0;

-- 创建测试数据时标记
INSERT INTO users (user_name, is_test_data) VALUES ('test_user', 1);

-- 清理测试数据
DELETE FROM users WHERE is_test_data = 1;
```

## 数据清理

### 清理策略
| 策略 | 说明 | 适用场景 |
|------|------|----------|
| 立即清理 | 测试完成后立即清理 | 单元测试 |
| 定时清理 | 定时批量清理 | 定时测试 |
| 标记清理 | 根据标记批量清理 | 集成测试 |
| 手工清理 | 手工执行清理 | 复杂场景 |

### 自动清理脚本
```typescript
// 测试后自动清理
describe('订单测试', () => {
  let testDataIds: number[] = [];

  beforeEach(async () => {
    // 记录测试数据ID
    testDataIds = [];
  });

  afterEach(async () => {
    // 清理测试数据
    for (const id of testDataIds) {
      await apiClient.delete(`/api/order/delete?id=${id}`);
    }
  });

  test('创建订单', async () => {
    const order = await factory.createOrder();
    testDataIds.push(order.id);
    
    // 测试逻辑...
  });
});
```

### 批量清理脚本
```bash
#!/bin/bash
# cleanup_test_data.sh

DB_HOST="localhost"
DB_USER="root"
DB_PASS="xxx"
DB_NAME="test_db"

# 清理测试用户
mysql -h$DB_HOST -u$DB_USER -p$DB_PASS $DB_NAME -e "
DELETE FROM users WHERE user_name LIKE 'test_%';
"

# 清理测试商品
mysql -h$DB_HOST -u$DB_USER -p$DB_PASS $DB_NAME -e "
DELETE FROM products WHERE name LIKE '测试商品_%';
"

# 清理测试订单
mysql -h$DB_HOST -u$DB_USER -p$DB_PASS $DB_NAME -e "
DELETE FROM orders WHERE user_id IN (SELECT id FROM users WHERE user_name LIKE 'test_%');
"

# 清理Redis测试数据
redis-cli --scan --pattern 'test:*' | xargs redis-cli DEL

echo "测试数据清理完成"
```

### 定时清理任务
```yaml
# Cron定时清理
cleanup-test-data:
  schedule: "0 6 * * *"  # 每天凌晨6点
  script:
    - /scripts/cleanup_test_data.sh
```

## 数据恢复

### 备份恢复
```bash
# 测试前数据备份
mysqldump test_db > backup_before_test.sql

# 测试后数据恢复
mysql test_db < backup_before_test.sql
```

### 快照恢复
```typescript
// 数据快照
export class DataSnapshot {
  private snapshot: Map<string, any> = new Map();

  async take(table: string) {
    const data = await db.query(`SELECT * FROM ${table}`);
    this.snapshot.set(table, data);
  }

  async restore(table: string) {
    const data = this.snapshot.get(table);
    if (data) {
      await db.query(`DELETE FROM ${table}`);
      for (const row of data) {
        await db.query(`INSERT INTO ${table} SET ?`, row);
      }
    }
  }
}

// 使用
describe('订单测试', () => {
  const snapshot = new DataSnapshot();

  beforeAll(async () => {
    await snapshot.take('orders');
    await snapshot.take('order_items');
  });

  afterAll(async () => {
    await snapshot.restore('orders');
    await snapshot.restore('order_items');
  });
});
```

## 数据文档

### 数据字典
```markdown
# 测试数据字典

## 用户数据
| 字段 | 类型 | 说明 | 测试值示例 |
|------|------|------|------------|
| userName | string | 用户名 | test_user_001 |
| password | string | 密码 | password123 |
| email | string | 邮箱 | test@example.com |
| phone | string | 手机号 | 13800138000 |
| status | int | 状态 | 1(正常) |

## 商品数据
| 字段 | 类型 | 说明 | 测试值示例 |
|------|------|------|------------|
| name | string | 名称 | 测试商品_A |
| price | decimal | 价格 | 100.00 |
| stock | int | 库存 | 50 |
| status | int | 状态 | 1(正常) |

## 订单数据
| 字段 | 类型 | 说明 | 测试值示例 |
|------|------|------|------------|
| userId | long | 用户ID | 1 |
| productId | long | 商品ID | 1 |
| quantity | int | 数量 | 1 |
| totalAmount | decimal | 总金额 | 100.00 |
| status | int | 状态 | 10(待支付) |
```

### 数据关系图
```
用户表(users)
  ↓
订单表(orders)
  ↓
订单明细表(order_items)
  ↓
商品表(products)
```

## DO - 推荐做法

```typescript
// ✓ 使用数据工厂创建
const user = TestDataFactory.createUser();

// ✓ 测试后清理数据
afterEach(async () => await cleanupData(testDataIds));

// ✓ 数据环境隔离
// 使用独立测试数据库

// ✓ 边界数据完整
const boundary = BoundaryDataFactory.getQuantityBoundary();

// ✓ 测试数据标记
INSERT INTO users (user_name, is_test_data) VALUES ('test_user', 1);

// ✓ 定时清理任务
schedule: "0 6 * * *"

// ✓ 数据文档化
// 维护数据字典
```

## DON'T - 禁止做法

```typescript
// ✗ 硬编码测试数据
const user = { userName: 'testuser', password: 'password123' };  // 禁止

// ✓ 正确做法
const user = TestDataFactory.createUser();

// ✗ 无数据清理
// 无 afterEach/afterAll  // 禁止

// ✓ 正确做法
afterEach(async () => await cleanupData(testDataIds));

// ✗ 使用生产数据测试
// 在生产环境测试  // 禁止

// ✓ 正确做法
// 使用测试环境数据

// ✗ 边界数据不完整
// 只测试正常值  // 禁止

// ✓ 正确做法
const boundary = BoundaryDataFactory.getQuantityBoundary();
test('数量边界', () => {
  testQuantity(boundary.min);
  testQuantity(boundary.max);
  testQuantity(boundary.belowMin);
  testQuantity(boundary.aboveMax);
});

// ✗ 无数据标记
INSERT INTO users (user_name) VALUES ('test_user');  // 禁止，无标记

// ✓ 正确做法
INSERT INTO users (user_name, is_test_data) VALUES ('test_user', 1);

// ✗ 无定时清理
// 不定期清理测试数据  // 禁止

// ✓ 正确做法
schedule: "0 6 * * *"

// ✗ 无数据文档
// 数据结构无说明  // 禁止

// ✓ 正确做法
// 维护数据字典和数据关系图
```

## 最佳实践

### 数据准备
- 数据工厂统一生成
- 边界数据完整覆盖
- 环境数据严格隔离

### 数据管理
- 测试数据标记管理
- 定时清理过期数据
- 数据字典文档维护

### 数据清理
- 测试后立即清理
- 定时批量清理
- 快照恢复机制