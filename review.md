---
name: review
description: 代码审查（基于团队规范）
userInvoke: true
---

# 代码审查 Skill

基于团队规范审查代码，检查是否符合规则。

## 使用方式

```
/review [文件路径]
/review [目录路径]
/review --diff  # 审查当前变更
```

示例：
```
/review Modules/User/Services/UserService.cs
/review pages/product/
/review --diff
```

## 审查维度

### 1. 后端代码审查

#### DI 规范
- [ ] 服务接口继承 `IAutoDIable` 族接口
- [ ] 禁止手动注册（`builder.Services.AddScoped`）
- [ ] Controller 使用 `[FromServices]` 注入

#### Controller 规范
- [ ] 继承 `ApiControllerBase`
- [ ] 使用 POST 方法
- [ ] 返回统一响应格式（`DataResponse`）
- [ ] Swagger 注释完整

#### 异常处理
- [ ] 使用 `BusinessException`，禁止原生 `Exception`
- [ ] 错误码使用 `BusinessStatusCodeEnum`
- [ ] 全局异常处理中间件配置

#### 数据访问
- [ ] 只读查询使用 `AsNoTracking`
- [ ] 禁止 N+1 查询
- [ ] 大数据量分页
- [ ] 批量操作使用 `ExecuteUpdateAsync`

#### Redis 使用
- [ ] 键命名规范（`{应用}:{模块}:{业务}:{标识}`）
- [ ] 分布式锁设置超时
- [ ] 锁必须释放（`await using`）
- [ ] 缓存设置过期时间

#### 日志规范
- [ ] 结构化日志（参数化）
- [ ] 禁止字符串拼接
- [ ] 不记录敏感信息
- [ ] 日志级别正确

#### 命名规范
- [ ] 接口：`I{Service}Service`
- [ ] 实现：`{Service}Service`
- [ ] Request：`{Action}Request`
- [ ] Response：`{Action}Response` 或 `{Entity}Dto`

### 2. 前端代码审查

#### HTTP 请求
- [ ] 使用 `@/utils/request`
- [ ] 禁止 Umi 内置 request
- [ ] 返回类型定义完整

#### 组件使用
- [ ] 列表页使用 `MyProTable`
- [ ] 禁止直接使用 `ProTable`
- [ ] ProForm 组件使用正确

#### 列定义
- [ ] 每列指定 `width`
- [ ] 状态列使用 `valueEnum` + Map
- [ ] 操作列 `fixed: 'right'`

#### 枚举规范
- [ ] 中文标签
- [ ] 配对 Map 定义
- [ ] 下拉 Options 定义

#### 类型安全
- [ ] 禁止 `any` 类型
- [ ] 类型定义完整
- [ ] namespace 定义正确

#### 状态管理
- [ ] 使用 `@@initialState` 或 `ahooks`
- [ ] 禁止 Redux/MobX

#### 命名规范
- [ ] 文件：小写 camelCase
- [ ] 组件：PascalCase
- [ ] 类型：namespace API

### 3. 数据库审查

#### 表设计
- [ ] 必备字段完整（id, created_at, updated_at, is_deleted, version）
- [ ] 字段命名：小写下划线
- [ ] 字段类型正确
- [ ] 字符集：utf8mb4

#### 索引设计
- [ ] 外键有索引
- [ ] 查询条件字段有索引
- [ ] 唯一约束有索引
- [ ] 索引数量合理（≤5）

#### 查询优化
- [ ] 分页查询
- [ ] 避免 SELECT *
- [ ] 大表修改低峰期执行

### 4. API 设计审查

#### URL 设计
- [ ] 格式：`/api/{module}/{action}`
- [ ] 小写 camelCase
- [ ] 统一 POST

#### 请求设计
- [ ] 分页继承 `PageRequest`
- [ ] 参数校验（DataAnnotations）
- [ ] 参数注释完整

#### 响应设计
- [ ] 统一响应信封
- [ ] HTTP 200
- [ ] 分页响应格式

#### 安全设计
- [ ] 认证配置
- [ ] 权限控制
- [ ] 输入校验
- [ ] 敏感信息过滤

## 审查报告格式

```markdown
# 代码审查报告

## 审查范围
- 文件：{文件路径}
- 时间：{时间}

## 审查结果

### ✅ 符合规范
- DI 自动注册正确
- Controller 继承 ApiControllerBase
- 响应格式统一

### ❌ 不符合规范

#### 问题 1: 未使用 AsNoTracking
**位置**: UserService.cs:45
**代码**:
```csharp
var users = await _context.Users.ToListAsync();
```
**建议**:
```csharp
var users = await _context.Users.AsNoTracking().ToListAsync();
```

#### 问题 2: 列未指定 width
**位置**: product/index.tsx:25
**代码**:
```typescript
{ title: '商品名称', dataIndex: 'name' }
```
**建议**:
```typescript
{ title: '商品名称', dataIndex: 'name', width: 200 }
```

### ⚠️ 建议改进
- 添加日志记录
- 优化查询性能

## 统计
- 符合规范：15 项
- 不符合规范：2 项
- 建议改进：3 项
- 综合评分：85/100
```

## 执行步骤

1. 解析审查范围（文件/目录/diff）
2. 读取相关代码
3. 检查各项规范
4. 输出审查报告

## 注意事项

1. 审查基于团队规范文档
2. 问题需给出具体位置和建议
3. 区分"必须修改"和"建议改进"
4. 给出综合评分