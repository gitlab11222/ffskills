---
name: security-review
description: 安全专项审查，检查SQL注入、XSS、越权等安全漏洞
userInvoke: true
---

# 安全审查 Skill

对代码进行安全专项审查，检查常见安全漏洞和安全隐患。

## 使用方式

```
/security-review <目录路径>
/security-review --type <审查类型>
/security-review --depth <深度>
```

## 参数说明

| 参数 | 格式 | 说明 | 示例 |
|------|------|------|------|
| 目录路径 | 路径 | 要审查的代码目录 | Modules/User |
| --type | 全部/API/前端/后端 | 审查类型 | API |
| --depth | quick/standard/deep | 审查深度 | standard |

## 审查类型

| 类型 | 检查范围 | 适用场景 |
|------|----------|----------|
| 全部 | 全栈安全检查 | 发布前安全审查 |
| API | 后端API安全检查 | 接口安全专项 |
| 前端 | 前端代码安全检查 | 前端安全专项 |
| 后端 | 后端代码安全检查 | 后端安全专项 |

## 审查深度

| 深度 | 检查项数 | 适用场景 |
|------|----------|----------|
| quick | 核心检查项 | 快速筛查 |
| standard | 标准检查项 | 常规审查 |
| deep | 全面检查项 | 发布前审查 |

## 执行步骤

### 1. SQL注入检查

#### 1.1 检查要点
- 参数拼接SQL语句
- 动态SQL构建
- Raw SQL查询
- 存储过程调用

#### 1.2 检查模式
```csharp
// 危险模式识别
// ✗ 错误: 直接拼接SQL
var sql = $"SELECT * FROM users WHERE name = '{name}'";

// ✗ 错误: 动态构建SQL
var sql = "SELECT * FROM " + tableName + " WHERE id = " + id;

// ✗ 错误: Raw SQL参数未正确传参
var users = await _context.Users.FromSqlRaw($"SELECT * FROM users WHERE name = '{name}'").ToListAsync();

// ✓ 正确: 参数化查询
var users = await _context.Users.FromSqlRaw("SELECT * FROM users WHERE name = {0}", name).ToListAsync();
```

### 2. XSS注入检查

#### 2.1 检查要点
- 输入未转义直接输出
- 富文本未过滤
- URL参数未编码
- JavaScript动态渲染

#### 2.2 检查模式
```typescript
// ✗ 错误: 直接渲染用户输入
<div>{userInput}</div>

// ✗ 错误: innerHTML直接赋值
element.innerHTML = userInput;

// ✗ 错误: URL参数未编码
const url = `/api?name=${name}`;

// ✓ 正确: 使用 sanitizeHtml
import sanitizeHtml from 'sanitize-html';
const safeContent = sanitizeHtml(userInput);

// ✓ 正确: React 自动转义
<div>{userInput}</div>  // React默认转义

// ✓ 正确: URL编码
const url = `/api?name=${encodeURIComponent(name)}`;
```

### 3. 越权检查

#### 3.1 检查要点
- 数据访问权限校验
- 操作权限校验
- 角色/用户隔离
- 资源归属校验

#### 3.2 检查模式
```csharp
// ✗ 错误: 无权限校验
public async Task<OrderDto> GetOrder(long orderId)
{
    return await _orderRepo.GetByIdAsync(orderId);  // 任何用户可访问任何订单
}

// ✓ 正确: 校验数据归属
public async Task<OrderDto> GetOrder(long orderId)
{
    var order = await _orderRepo.GetByIdAsync(orderId);
    if (order.UserId != _currentUser.Id)
        throw new BusinessException("无权限访问", BusinessStatusCodeEnum.NoPermission);
    return order;
}

// ✓ 正确: 使用权限过滤器
[Authorize(Policy = "OrderView")]
public async Task<OrderDto> GetOrder(long orderId) { }
```

### 4. 认证鉴权检查

#### 4.1 检查要点
- 接口认证配置
- Token校验逻辑
- Session管理
- 密码安全处理

#### 4.2 检查模式
```csharp
// ✗ 错误: 接口无认证保护
[HttpPost("list")]
public async Task<IActionResult> GetList() { }  // 无 [Authorize]

// ✗ 错误: 密码明文存储
user.Password = password;  // 明文存储

// ✗ 错误: Token无校验
var token = request.Headers["Authorization"];  // 无有效性校验

// ✓ 正确: 接口认证保护
[Authorize]
[HttpPost("list")]
public async Task<IActionResult> GetList() { }

// ✓ 正确: 密码加密存储
user.PasswordHash = BCrypt.Net.BCrypt.HashPassword(password);

// ✓ 正确: Token校验
// JWT自动校验（配置在Startup）
```

### 5. CSRF检查

#### 5.1 检查要点
- 表单CSRF Token
- AJAX请求CSRF防护
- 跨站请求验证

#### 5.2 检查模式
```typescript
// ✗ 错误: 表单无CSRF Token
<form method="POST" action="/api/submit">
    <input name="data" />
    <button>提交</button>
</form>

// ✓ 正确: 添加CSRF Token
<form method="POST" action="/api/submit">
    <input type="hidden" name="_csrf" value={csrfToken} />
    <input name="data" />
    <button>提交</button>
</form>

// ✓ 正确: AJAX请求携带Token
headers: {
    'X-CSRF-Token': csrfToken
}
```

### 6. 文件上传检查

#### 6.1 检查要点
- 文件类型校验
- 文件大小限制
- 文件名安全处理
- 存储路径安全

#### 6.2 检查模式
```csharp
// ✗ 错误: 无类型校验
public async Task Upload(IFormFile file)
{
    await SaveFile(file);  // 任何文件都可上传
}

// ✗ 错误: 保留原始文件名
var fileName = file.FileName;  // 可能包含恶意字符

// ✓ 正确: 类型校验
var allowedTypes = new[] { ".jpg", ".png", ".gif" };
var ext = Path.GetExtension(file.FileName).ToLower();
if (!allowedTypes.Contains(ext))
    throw new BusinessException("不支持的文件类型");

// ✓ 正确: 重命名文件
var fileName = $"{Guid.NewGuid()}{ext}";

// ✓ 正确: 大小限制
if (file.Length > 10 * 1024 * 1024)  // 10MB
    throw new BusinessException("文件过大");
```

### 7. 敏感信息检查

#### 7.1 检查要点
- 密码/密钥泄露
- 日志敏感信息
- 响应敏感字段
- 配置文件安全

#### 7.2 检查模式
```csharp
// ✗ 错误: 日志记录密码
_logger.LogInformation("用户登录: password={Password}", password);

// ✗ 错误: 响应包含密码
return Ok(new UserDto { PasswordHash = user.PasswordHash });

// ✗ 错误: 硬编码密钥
var key = "my-secret-key-12345";

// ✓ 正确: 不记录敏感信息
_logger.LogInformation("用户登录: UserName={UserName}", userName);

// ✓ 正确: 过滤敏感字段
return Ok(new UserDto { Id = user.Id, Name = user.Name });  // 不返回密码

// ✓ 正确: 从配置读取密钥
var key = _config["Jwt:Secret"];
```

## 安全审查报告模板

```markdown
# 安全审查报告：{审查范围}

## 基本信息
- **审查时间**: {YYYY-MM-DD HH:mm}
- **审查范围**: {目录/模块}
- **审查类型**: 全部/API/前端/后端
- **审查深度**: quick/standard/deep
- **审查人**: {姓名}

---

## 审查结果汇总

| 检查类型 | 检查项数 | 发现问题 | 严重级别 | 状态 |
|----------|----------|----------|----------|------|
| SQL注入 | {数量} | {数量} | 🔴/🟡/🟢 | {状态} |
| XSS注入 | {数量} | {数量} | 🔴/🟡/🟢 | {状态} |
| 越权漏洞 | {数量} | {数量} | 🔴/🟡/🟢 | {状态} |
| 认证鉴权 | {数量} | {数量} | 🔴/🟡/🟢 | {状态} |
| CSRF | {数量} | {数量} | 🔴/🟡/🟢 | {状态} |
| 文件上传 | {数量} | {数量} | 🔴/🟡/🟢 | {状态} |
| 敏感信息 | {数量} | {数量} | 🔴/🟡/🟢 | {状态} |
| **总计** | **{数量}** | **{数量}** | - | - |

---

## 详细问题清单

### 🔴 SQL注入漏洞

#### 问题1: {文件路径}:{行号}
**问题描述**: {描述}

**问题代码**:
```csharp
{问题代码片段}
```

**修复建议**:
```csharp
{修复代码示例}
```

**修复优先级**: 立即修复

---

### 🔴 XSS注入漏洞

#### 问题1: {文件路径}:{行号}
**问题描述**: {描述}

**问题代码**:
```typescript
{问题代码片段}
```

**修复建议**:
```typescript
{修复代码示例}
```

**修复优先级**: 立即修复

---

### 🔴 越权漏洞

#### 问题1: {文件路径}:{行号}
**问题描述**: {描述}

**问题代码**:
```csharp
{问题代码片段}
```

**修复建议**:
```csharp
{修复代码示例}
```

**修复优先级**: 立即修复

---

### 🟡 认证鉴权问题

#### 问题1: {文件路径}:{行号}
**问题描述**: {描述}

**问题代码**:
```csharp
{问题代码片段}
```

**修复建议**:
```csharp
{修复代码示例}
```

**修复优先级**: 发布前修复

---

## 安全评分

### 评分维度
| 维度 | 权重 | 得分 | 说明 |
|------|------|------|------|
| SQL注入防护 | 30% | {分数} | {说明} |
| XSS防护 | 20% | {分数} | {说明} |
| 越权防护 | 20% | {分数} | {说明} |
| 认证鉴权 | 15% | {分数} | {说明} |
| 其他安全 | 15% | {分数} | {说明} |

### 总评分
- **安全评分**: {分数}/100
- **安全等级**: 🔴高风险 / 🟡中风险 / 🟢低风险

---

## 审查结论

### 通过条件
- [ ] 无🔴高风险问题
- [ ] 🟡中风险问题 ≤ 3
- [ ] 安全评分 ≥ 70

### 最终结论
- **审查结果**: ✅ 通过 / ❌ 不通过 / ⚠️ 需修复后通过
- **发布建议**: 可以发布 / 需修复后发布 / 不建议发布

---

## 修复计划

| 问题 | 级别 | 修复人 | 修复时间 | 验证时间 |
|------|------|--------|----------|----------|
| {问题} | 🔴 | {姓名} | {日期} | {日期} |
| {问题} | 🟡 | {姓名} | {日期} | {日期} |

---

## 附录
- 审查代码清单
- 安全测试用例
```

## 输出清单

1. ✅ 安全审查报告（Markdown格式）
2. ✅ 问题清单（按严重级别分类）
3. ✅ 修复建议（带代码示例）
4. ✅ 安全评分
5. ✅ 审查结论

## 注意事项

1. 🔴高风险问题必须立即修复
2. 🟡中风险问题需发布前修复
3. 修复后需重新验证
4. 安全审查需定期执行
5. 发布前必须通过安全审查