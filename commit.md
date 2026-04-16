---
name: commit
description: Git 提交（中文 commit message）
userInvoke: true
---

# Git 提交 Skill

按照团队规范生成 Git commit message（中文）。

## 使用方式

```
/commit
/commit --amend  # 修改上次提交
```

## Commit Message 格式

```
<类型>: <描述>

[可选的详细描述]

[可选的 Footers]
```

### 类型说明

| 类型 | 说明 | 示例 |
|------|------|------|
| 新增 | 新功能 | 新增: 商品管理功能 |
| 修改 | 功能修改 | 修改: 优化商品列表查询 |
| 修复 | Bug修复 | 修复: 修复订单库存扣减问题 |
| 重构 | 代码重构 | 重构: 重构用户服务架构 |
| 优化 | 性能优化 | 优化: 优化商品查询性能 |
| 文档 | 文档更新 | 文档: 更新 API 文档 |
| 测试 | 测试相关 | 测试: 添加商品服务单元测试 |
| 构建 | 构建/部署 | 构建: 更新 Dockerfile |
| 配置 | 配置修改 | 配置: 更新 Redis 连接配置 |

### 描述规范

1. 使用中文
2. 简洁明了（≤50字）
3. 说明"做了什么"，不说明"为什么"
4. 以动词开头（新增、修改、修复、优化）

### 详细描述（可选）

当需要补充说明时，在第二行添加详细描述：
```
修改: 商品列表查询

- 添加分类筛选
- 添加价格区间筛选
- 优化分页查询性能
```

### Footers（可选）

关联 Issue 或 PR：
```
修复: 订单库存扣减问题

Issue: #123
PR: #456
```

## 自动生成规则

根据变更内容自动判断类型：

### 类型判断逻辑
```
新增文件 + 新功能代码 → 新增
修改现有文件 → 修改
修复 Bug 相关 → 修复
重构/重命名 → 重构
性能相关 → 优化
文档文件 → 文档
测试文件 → 测试
Dockerfile/CI → 构建
配置文件 → 配置
```

### 描述生成逻辑
1. 分析变更文件
2. 提取关键改动
3. 生成简洁描述

## 示例

### 示例 1: 新增功能
```
变更文件:
+ Modules/Product/Entities/Product.cs
+ Modules/Product/Services/ProductService.cs
+ Controllers/ProductController.cs
+ pages/product/index.tsx

Commit Message:
新增: 商品管理功能
```

### 示例 2: Bug修复
```
变更文件:
M Modules/Order/Services/OrderService.cs

变更内容: 修复库存扣减逻辑

Commit Message:
修复: 订单库存扣减并发问题
```

### 示例 3: 性能优化
```
变更文件:
M Modules/User/Services/UserService.cs

变更内容: 添加 AsNoTracking

Commit Message:
优化: 用户查询性能

- 列表查询添加 AsNoTracking
- 减少 EF Core 跟踪开销
```

### 示例 4: 重构
```
变更文件:
R Modules/User/Services/UserService.cs → Services/User/UserService.cs

Commit Message:
重构: 调整服务文件结构
```

## 执行步骤

1. 运行 `git status` 查看变更
2. 运行 `git diff` 分析变更内容
3. 自动判断类型
4. 生成 commit message
5. 执行 `git commit`

## 注意事项

1. Commit message 必须使用中文
2. 类型使用预设类型，不可自定义
3. 描述简洁，不超过50字
4. 必须先 add 再 commit
5. 不要 commit 敏感文件（.env, credentials）