# 自动化测试规范

## 自动化测试框架

### 框架选择
| 类型 | 工具 | 说明 |
|------|------|------|
| 接口自动化 | Newman / REST Client | Postman CLI执行 |
| UI自动化 | Playwright / Cypress | Web端自动化 |
| 移动端自动化 | Appium | iOS/Android自动化 |
| 单元测试 | xUnit / Jest | 代码级自动化 |

### 项目结构
```
tests/
├── api/                    # 接口自动化
│   ├── collections/        # Postman Collection
│   │   └── user-api.json
│   ├── environments/       # 环境配置
│   │   ├── dev.json
│   │   └── test.json
│   │   └── prod.json
│   └── scripts/            # 执行脚本
│   │   └── run-tests.sh
│   └── reports/            # 测试报告
├── ui/                     # UI自动化
│   ├── playwright.config.ts
│   ├── tests/
│   │   ├── login.spec.ts
│   │   ├── order.spec.ts
│   │   └── user.spec.ts
│   ├── pages/              # Page Object
│   │   ├── LoginPage.ts
│   │   └── OrderPage.ts
│   └── utils/
│   │   └── helper.ts
```

## 接口自动化

### Postman Collection
```json
{
  "info": {
    "name": "User API Tests",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "用户登录",
      "request": {
        "method": "POST",
        "header": [
          { "key": "Content-Type", "value": "application/json" }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"userName\": \"test\",\n  \"password\": \"password123\"\n}"
        },
        "url": {
          "raw": "{{baseUrl}}/api/user/login",
          "host": ["{{baseUrl}}"],
          "path": ["api", "user", "login"]
        }
      },
      "response": []
    },
    {
      "name": "获取用户列表",
      "request": {
        "method": "POST",
        "header": [
          { "key": "Authorization", "value": "{{token}}" }
        ],
        "url": {
          "raw": "{{baseUrl}}/api/user/list"
        }
      },
      "response": []
    }
  ],
  "variable": [
    { "key": "baseUrl", "value": "http://localhost:8080" },
    { "key": "token", "value": "" }
  ]
}
```

### Pre-request Script
```javascript
// 登录获取Token
pm.sendRequest({
    url: pm.environment.get("baseUrl") + "/api/user/login",
    method: "POST",
    header: { "Content-Type": "application/json" },
    body: {
        mode: "raw",
        raw: JSON.stringify({
            userName: "test",
            password: "password123"
        })
    }
}, function(err, res) {
    pm.environment.set("token", res.json().data.token);
});
```

### Tests Script
```javascript
// 响应状态码校验
pm.test("Status code is 200", function() {
    pm.response.to.have.status(200);
});

// 响应结构校验
pm.test("Response has correct structure", function() {
    var jsonData = pm.response.json();
    pm.expect(jsonData).to.have.property("code");
    pm.expect(jsonData).to.have.property("message");
    pm.expect(jsonData).to.have.property("data");
});

// 业务状态码校验
pm.test("Business code is 0", function() {
    var jsonData = pm.response.json();
    pm.expect(jsonData.code).to.eql(0);
});

// 数据字段校验
pm.test("User list has items", function() {
    var jsonData = pm.response.json();
    pm.expect(jsonData.data.list).to.be.an("array");
    pm.expect(jsonData.data.list.length).to.be.above(0);
});

// 响应时间校验
pm.test("Response time < 500ms", function() {
    pm.expect(pm.response.responseTime).to.be.below(500);
});
```

### Newman 执行
```bash
# 执行Collection
newman run user-api.json -e test.json

# 详细报告
newman run user-api.json -e test.json --reporters cli,html,json

# 输出HTML报告
newman run user-api.json -e test.json --reporters html --reporter-html-export report.html

# CI执行
newman run user-api.json -e test.json \
  --reporters cli \
  --reporter-cli-no-failures \
  --timeout 60000
```

### CI/CD 集成
```yaml
# GitLab CI
api-test:
  stage: test
  image: postman/newman:alpine
  script:
    - newman run tests/api/collections/user-api.json
      -e tests/api/environments/${CI_ENVIRONMENT_NAME}.json
      --reporters cli,html
      --reporter-html-export tests/api/reports/report.html
  artifacts:
    paths:
      - tests/api/reports/
```

## UI自动化

### Playwright 配置
```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] },
    },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Page Object 模式
```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly userNameInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.userNameInput = page.locator('[name="userName"]');
    this.passwordInput = page.locator('[name="password"]');
    this.loginButton = page.locator('button[type="submit"]');
    this.errorMessage = page.locator('.ant-message-error');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(userName: string, password: string) {
    await this.userNameInput.fill(userName);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }

  async expectErrorMessage(message: string) {
    await this.errorMessage.waitFor();
    expect(await this.errorMessage.textContent()).toContain(message);
  }
}
```

### 测试用例
```typescript
// tests/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

test.describe('用户登录', () => {
  test('正常登录', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('test', 'password123');
    
    // 验证登录成功
    await expect(page).toHaveURL(/.*home/);
    await expect(page.locator('.user-name')).toHaveText('test');
  });

  test('用户名错误', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('wronguser', 'password123');
    
    await loginPage.expectErrorMessage('用户不存在');
  });

  test('密码错误', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('test', 'wrongpassword');
    
    await loginPage.expectErrorMessage('密码错误');
  });

  test('表单校验', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    
    // 空用户名
    await loginPage.passwordInput.fill('password');
    await loginPage.loginButton.click();
    await expect(page.locator('.ant-form-item-explain-error'))
      .toHaveText('请输入用户名');
    
    // 空密码
    await loginPage.userNameInput.fill('test');
    await loginPage.passwordInput.clear();
    await loginPage.loginButton.click();
    await expect(page.locator('.ant-form-item-explain-error'))
      .toHaveText('请输入密码');
  });
});
```

### 订单流程测试
```typescript
// tests/order.spec.ts
import { test, expect } from '@playwright/test';

test.describe('订单流程', () => {
  test.beforeEach(async ({ page }) => {
    // 登录
    await page.goto('/login');
    await page.locator('[name="userName"]').fill('test');
    await page.locator('[name="password"]').fill('password123');
    await page.locator('button[type="submit"]').click();
    await expect(page).toHaveURL(/.*home/);
  });

  test('创建订单', async ({ page }) => {
    // 进入商品列表
    await page.goto('/product/list');
    
    // 选择商品
    await page.locator('.product-item').first().click();
    
    // 加入购物车
    await page.locator('button:has-text("加入购物车")').click();
    
    // 进入购物车
    await page.goto('/cart');
    
    // 结算
    await page.locator('button:has-text("结算")').click();
    
    // 确认订单
    await page.locator('button:has-text("提交订单")').click();
    
    // 验证订单创建成功
    await expect(page.locator('.order-id')).toBeVisible();
    await expect(page.locator('.order-status')).toHaveText('待支付');
  });

  test('支付订单', async ({ page }) => {
    // 进入订单详情
    await page.goto('/order/detail/123');
    
    // 支付
    await page.locator('button:has-text("支付")').click();
    
    // 选择支付方式
    await page.locator('.payment-method-alipay').click();
    
    // 确认支付
    await page.locator('button:has-text("确认支付")').click();
    
    // 验证支付成功
    await expect(page.locator('.order-status')).toHaveText('已支付');
  });
});
```

### 执行命令
```bash
# 执行所有测试
npx playwright test

# 执行指定文件
npx playwright test login.spec.ts

# 执行指定浏览器
npx playwright test --project=chromium

# 执行指定测试
npx playwright test -g "正常登录"

# UI模式
npx playwright test --ui

# 生成报告
npx playwright show-report
```

## 执行时机

### 执行策略
| 类型 | 执行时机 | 说明 |
|------|----------|------|
| 接口自动化 | 每次提交 | CI自动执行 |
| UI自动化 | 每日定时 | 夜间执行 |
| 全量回归 | 版本发布前 | 发布前执行 |
| 冒烟测试 | 部署后 | 验证核心功能 |

### 定时执行
```yaml
# GitLab CI 定时任务
api-test-daily:
  stage: test
  only:
    - schedules
  trigger:
    - "0 2 * * *"  # 每天凌晨2点
  script:
    - newman run tests/api/collections/full-api.json
      -e tests/api/environments/test.json
      --reporters html

ui-test-daily:
  stage: test
  only:
    - schedules
  trigger:
    - "0 3 * * *"  # 每天凌晨3点
  script:
    - npx playwright test
```

## 测试报告

### HTML 报告模板
```html
<!DOCTYPE html>
<html>
<head>
  <title>自动化测试报告</title>
  <style>
    .summary { padding: 20px; background: #f5f5f5; }
    .passed { color: #52c41a; }
    .failed { color: #f5222d; }
    .test-item { margin: 10px 0; padding: 10px; border: 1px solid #d9d9d9; }
  </style>
</head>
<body>
  <h1>自动化测试报告</h1>
  
  <div class="summary">
    <p>执行时间: {{date}}</p>
    <p>执行环境: {{environment}}</p>
    <p>总用例数: {{total}}</p>
    <p class="passed">通过: {{passed}}</p>
    <p class="failed">失败: {{failed}}</p>
    <p>通过率: {{passRate}}%</p>
  </div>
  
  <h2>详细结果</h2>
  {{#each tests}}
  <div class="test-item">
    <p>用例: {{name}}</p>
    <p>状态: <span class="{{status}}">{{status}}</span></p>
    <p>耗时: {{duration}}ms</p>
    {{#if error}}
    <p>错误: {{error}}</p>
    {{/if}}
  </div>
  {{/each}}
</body>
</html>
```

### 钉钉通知
```typescript
// 测试结果通知
async function sendTestReport(results) {
  const message = results.failed > 0
    ? `🔴 自动化测试失败
       通过: ${results.passed}
       失败: ${results.failed}
       通过率: ${results.passRate}%
       详情: ${reportUrl}`
    : `✅ 自动化测试通过
       通过: ${results.passed}
       通过率: ${results.passRate}%`;

  await fetch('https://oapi.dingtalk.com/robot/send?access_token=xxx', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      msgtype: 'text',
      text: { content: message }
    })
  });
}
```

## 测试数据

### 测试数据管理
```typescript
// 测试数据工厂
export class TestDataFactory {
  static createUser(overrides = {}) {
    return {
      userName: 'testuser',
      password: 'password123',
      email: 'test@example.com',
      ...overrides
    };
  }

  static createProduct(overrides = {}) {
    return {
      name: '测试商品',
      price: 100,
      stock: 50,
      ...overrides
    };
  }

  static createOrder(overrides = {}) {
    return {
      userId: 1,
      productId: 1,
      quantity: 1,
      ...overrides
    };
  }
}

// 使用
test('创建订单', async ({ page }) => {
  const orderData = TestDataFactory.createOrder({ quantity: 5 });
  // ...
});
```

### 数据清理
```typescript
// 测试后数据清理
test.afterEach(async ({ page }) => {
  // 清理测试数据
  await page.goto('/api/test/cleanup');
});

// 或使用API清理
test.afterAll(async ({ request }) => {
  await request.post('/api/test/cleanup', {
    data: { testId: 'test-001' }
  });
});
```

## DO - 推荐做法

```typescript
// ✓ 使用Page Object模式
const loginPage = new LoginPage(page);
await loginPage.login('test', 'password');

// ✓ 测试数据使用工厂
const user = TestDataFactory.createUser();

// ✓ 响应校验完整
pm.test("Status code is 200", () => pm.response.to.have.status(200));
pm.test("Business code is 0", () => pm.expect(jsonData.code).to.eql(0));

// ✓ 测试后清理数据
test.afterEach(async () => await cleanupData());

// ✓ 定时执行自动化测试
schedules: ["0 2 * * *"]

// ✓ CI集成自动化测试
script: - newman run collections/*.json

// ✓ 测试失败有通知
await sendTestReport(results);
```

## DON'T - 禁止做法

```typescript
// ✗ 无Page Object直接定位
await page.locator('[name="userName"]').fill('test');  // 禁止

// ✓ 正确做法
const loginPage = new LoginPage(page);
await loginPage.login('test', 'password');

// ✗ 硬编码测试数据
const user = { userName: 'testuser', password: 'password123' };  // 禁止

// ✓ 正确做法
const user = TestDataFactory.createUser();

// ✗ 校验不完整
pm.test("Response OK", () => {});  // 禁止，无实际校验

// ✓ 正确做法
pm.test("Status is 200", () => pm.response.to.have.status(200));
pm.test("Code is 0", () => pm.expect(jsonData.code).to.eql(0));

// ✗ 无数据清理
// 无 afterAll/afterEach  // 禁止

// ✓ 正确做法
test.afterEach(async () => await cleanupData());

// ✗ 无定时执行
// 无 schedules 配置  // 禁止

// ✓ 正确做法
schedules: ["0 2 * * *"]

// ✗ 无CI集成
// 无 test stage  // 禁止

// ✓ 正确做法
api-test:
  stage: test
  script: - newman run collections/*.json

// ✗ 失败无通知
// 无钉钉/邮件通知  // 禁止

// ✓ 正确做法
await sendTestReport(results);
```

## 最佳实践

### 自动化分层
- 接口自动化：API层验证
- UI自动化：界面层验证
- 单元测试：代码层验证

### 执行策略
- 接口：每次提交执行
- UI：每日定时执行
- 全量：发布前执行

### 数据管理
- 测试数据工厂生成
- 测试后自动清理
- 环境数据隔离