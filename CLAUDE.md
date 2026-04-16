# 研发工作流配置

基于团队技术栈定制的 Claude Code 开发工作流配置。

## 技术栈

### 后端
- **框架**: .NET 8 / C# 12
- **Web API**: ASP.NET Core Web API
- **ORM**: Entity Framework Core 8 + Pomelo MySQL Provider
- **IoC**: Autofac
- **缓存/队列/分布式锁**: Redis (StackExchange.Redis)
- **任务调度**: Hangfire / Quartz.NET
- **延时任务**: Redis 延迟队列 / Hangfire Delayed Jobs
- **日志**: Serilog + 阿里云 SLS / Seq

### 前端
- **框架**: React 18 + TypeScript 5
- **UI组件**: Ant Design 5 Pro
- **构建工具**: Umi.js 4 / Vite
- **状态管理**: Umi `@@initialState` + ahooks
- **HTTP请求**: Axios

### 数据库
- **主库**: MySQL 8.x
- **缓存**: Redis 7.x

## 工作模式 (GSD)

**G**oal-oriented - 目标驱动，明确交付物
**S**pec-driven - 规范先行，遵循本文档
**D**ata-aware - 数据敏感，注意性能和安全
**V**erify-first - 先读后写，理解再改

## 规则索引

| 规则 | 用途 |
|------|------|
| [backend.md](.claude/rules/backend.md) | 后端编码规则 |
| [frontend.md](.claude/rules/frontend.md) | 前端编码规则 |
| [mysql.md](.claude/rules/mysql.md) | MySQL 设计规范 |
| [redis.md](.claude/rules/redis.md) | Redis 使用规范 |
| [api-design.md](.claude/rules/api-design.md) | API 设计规范 |
| [csharp.md](.claude/rules/csharp.md) | C# 编码指南 |
| [task-scheduling.md](.claude/rules/task-scheduling.md) | 任务调度规范 |
| [deployment.md](.claude/rules/deployment.md) | 部署与构建规则 |
| [testing.md](.claude/rules/testing.md) | 测试规范 |
| [unit-test.md](.claude/rules/unit-test.md) | 单元测试规范 |
| [automated-test.md](.claude/rules/automated-test.md) | 自动化测试规范 |
| [test-data.md](.claude/rules/test-data.md) | 测试数据管理规范 |
| [release-process.md](.claude/rules/release-process.md) | 发布流程规范 |
| [requirement-doc.md](.claude/rules/requirement-doc.md) | 需求文档规范 |
| [acceptance.md](.claude/rules/acceptance.md) | 验收规范 |
| [retrospect.md](.claude/rules/retrospect.md) | 复盘规范 |
| [architecture-design.md](.claude/rules/architecture-design.md) | 架构设计规范 |
| [risk-management.md](.claude/rules/risk-management.md) | 风险管理规范 |
| [quality-gate.md](.claude/rules/quality-gate.md) | 质量门禁规范 |
| [change-management.md](.claude/rules/change-management.md) | 变更管理规范 |
| [monitoring.md](.claude/rules/monitoring.md) | 监控告警规范 |
| [kubernetes.md](.claude/rules/kubernetes.md) | K8s部署规范 |
| [database-backup.md](.claude/rules/database-backup.md) | 数据库备份规范 |
| [slow-query.md](.claude/rules/slow-query.md) | 慢查询优化规范 |
| [ui-design.md](.claude/rules/ui-design.md) | UI设计规范 |
| [interaction-design.md](.claude/rules/interaction-design.md) | 交互设计规范 |
| [tracking.md](.claude/rules/tracking.md) | 埋点规范 |
| [websocket.md](.claude/rules/websocket.md) | WebSocket使用规范 |
| [workflow-principles.md](.claude/rules/workflow-principles.md) | 研发工作流核心原则 |
| [frontend-backend-collaboration.md](.claude/rules/frontend-backend-collaboration.md) | 前后端协作规范 |

## Agent 索引

| Agent | 用途 |
|-------|------|
| [architect.md](.claude/agents/architect.md) | 技术方案设计、系统架构 |
| [architect-enhanced.md](.claude/agents/architect-enhanced.md) | 技术决策、项目体量评估 |
| [backend-dev.md](.claude/agents/backend-dev.md) | .NET 后端开发 |
| [frontend-dev.md](.claude/agents/frontend-dev.md) | React 前端开发 |
| [database-dev.md](.claude/agents/database-dev.md) | MySQL 数据库设计 |
| [code-reviewer.md](.claude/agents/code-reviewer.md) | 代码审查 |
| [product-manager.md](.claude/agents/product-manager.md) | 需求分析、原型设计、业务流程 |
| [tester.md](.claude/agents/tester.md) | 功能测试、安全测试、性能测试 |
| [project-manager.md](.claude/agents/project-manager.md) | 立项评审、排期管理、复盘总结 |
| [qa-engineer.md](.claude/agents/qa-engineer.md) | 测试计划、测试执行、质量评估 |
| [ops-engineer.md](.claude/agents/ops-engineer.md) | 环境部署、发布上线、监控运维 |
| [ux-designer.md](.claude/agents/ux-designer.md) | 原型设计、UI设计、交互设计 |
| [requirements-analyst.md](.claude/agents/requirements-analyst.md) | 需求收集、需求分析、需求文档 |

## Skills 索引

| Skill | 用途 | 调用方式 |
|-------|------|----------|
| [gen-crud.md](.claude/skills/gen-crud.md) | 全栈 CRUD 生成 | `/gen-crud <实体名> <字段>` |
| [new-product.md](.claude/skills/new-product.md) | 商品管理（含 SKU） | `/new-product --with-sku` |
| [new-category.md](.claude/skills/new-category.md) | 商品分类树 | `/new-category` |
| [new-api.md](.claude/skills/new-api.md) | API 接口生成 | `/new-api <模块> <操作>` |
| [new-page.md](.claude/skills/new-page.md) | 前端页面生成 | `/new-page <页面名> <字段>` |
| [review.md](.claude/skills/review.md) | 代码审查 | `/review [文件路径]` |
| [commit.md](.claude/skills/commit.md) | Git 提交（中文） | `/commit` |
| [generate-testcase.md](.claude/skills/generate-testcase.md) | 测试用例生成 | `/generate-testcase --from-api <模块>` |
| [create-wbs.md](.claude/skills/create-wbs.md) | WBS/排期计划表生成 | `/create-wbs --team <成员>` |
| [create-checklist.md](.claude/skills/create-checklist.md) | Checklist生成 | `/create-checklist --type验收` |
| [security-review.md](.claude/skills/security-review.md) | 安全专项审查 | `/security-review <目录>` |

## 核心模式速查

### 依赖注入（自动注册）
```csharp
// 必须继承 IAutoDIable 族接口
public interface IUserService : IAutoDIable { }
public class UserService : IUserService { }

// Program.cs 自动注册
builder.Services.AddAutoDIServices([typeof(Program).Assembly]);
```

### Controller 规范
```csharp
[ApiController, Route("api/[controller]")]
public class UserController : ApiControllerBase
{
    [HttpPost("list")]
    public async Task<IActionResult> GetList([FromServices] IUserService svc,
        [FromBody] UserListRequest request)
        => Ok(await svc.GetListAsync(request));
}
```

### 统一响应信封
```csharp
// HTTP 状态码始终 200
// 响应格式: { code: 0, message: "", messageType: 0, data: T }
return Ok(DataResponse.Success(data));
return Ok(DataResponse.Fail("业务错误", BusinessStatusCodeEnum.InvalidParam));
```

### 业务异常
```csharp
// 禁止抛原生 Exception
throw new BusinessException("错误信息", BusinessStatusCodeEnum.InvalidParam);
```

### Redis 分布式锁
```csharp
await using var lockResult = await _redisLock.WaitAsync(
    RedisKey.LockPrefix + key,
    TimeSpan.FromSeconds(30));
if (!lockResult.IsAcquired)
    throw new BusinessException("操作频繁，请稍后重试");
```

### Redis 延迟队列
```csharp
// 生产者：延迟 5 分钟执行
await _redis.Database.ListLeftPushAsync(
    RedisKey.DelayQueue,
    JsonSerializer.Serialize(new { orderId, executeAt = DateTime.UtcNow.AddMinutes(5) }));

// 消费者：定时扫描到期任务
[RedisConsumer("delay:queue:process", IntervalSeconds = 5)]
public class DelayQueueConsumer : IRedisConsumer { }
```

### 前端请求
```typescript
// 只用 @/utils/request
import { request } from '@/utils/request';

export async function getUserList(params: UserListParams) {
  return request.post<DataResponse<UserListResponse>>('/api/user/list', params);
}
```

### 前端列表页
```typescript
// 使用 MyProTable，禁止直接用 ProTable
<MyProTable<API.User>
  columns={columns}
  request={async (params) => {
    const res = await getUserList(params);
    return { data: res.data?.list || [], total: res.data?.total || 0, success: res.code === 0 };
  }}
/>
```

## 语言约定

| 场景 | 语言 |
|------|------|
| 代码标识符 | 英文 |
| 业务文案/UI文本 | 中文 |
| 枚举标签 | 中文 |
| Git commit | 中文 |
| 代码注释 | 中文 |
| API文档 | 中文 |

## 开发流程（确认驱动）

### 新增功能
```
需求分析 → 原型设计 → 原型确认签字 → 数据库设计 → 数据库确认签字 → 代码开发 → 自测验证 → 代码审查
   ↑                                              ↑
   └─────────────── 变更需重新确认 ────────────────┘
```

| 阶段 | 产出物 | 确认方式 | 说明 |
|------|--------|----------|------|
| 需求分析 | PRD文档、字段定义 | 需求评审签字 | 需求来源、背景、验收标准 |
| 原型设计 | 页面原型、交互流程 | 原型评审签字 | 产品+UX+研发确认 |
| 原型确认 | 签字确认表 | 必须签字 | 确认后方可进入数据库设计 |
| 数据库设计 | 表结构、索引设计 | 数据库评审签字 | 架构师+DBA+后端确认 |
| 数据库确认 | 签字确认表 | 必须签字 | 确认后方可进入代码开发 |
| 代码开发 | 完整代码（后端→前端） | - | 后端完成后自测，再前端 |
| 自测验证 | 自测报告 | 转测试单 | 功能、边界、异常测试 |
| 代码审查 | 审查意见 | - | 修复问题后提交 |

### 修改功能
```
阅读代码 → 评估影响 → 重新确认 → 代码修改 → 回归测试 → 代码审查
              ↓
         影响原型/数据库？ → 重新走确认流程
```

### 开发流程详细说明

#### 阶段1：需求分析
- 阅读PRD文档，理解业务背景和目标
- 识别用户角色、使用场景
- 拆解功能模块、定义优先级
- 输出：需求分析报告

#### 阶段2：原型设计（UX/产品）
- 页面布局设计（列表页/详情页/表单页）
- 交互流程设计（弹窗/抽屉/页面跳转）
- 字段定义（类型、长度、校验规则）
- 状态流转设计
- 输出：原型设计稿

#### 阶段3：原型确认（必须签字）
- 产品、UX、研发共同评审
- 检查：页面布局、交互流程、功能入口、字段定义
- 输出：签字确认表
- **禁止跳过此阶段直接进入数据库设计**

#### 阶段4：数据库设计（后端/DBA）
- 根据确认后的原型设计表结构
- 设计字段、类型、约束
- 设计索引（查询条件、关联字段）
- 输出：表结构设计文档

#### 阶段5：数据库确认（必须签字）
- 架构师、DBA、研发共同评审
- 检查：表结构、字段定义、索引设计
- 输出：签字确认表
- **禁止跳过此阶段直接进入代码开发**

#### 阶段6：代码开发
- 后端：实体 → DTO → 服务 → Controller
- 前端：API → 页面 → 组件
- 先后端自测，再前端开发
- 输出：完整代码

#### 阶段7：自测验证
- 功能测试、边界测试、异常测试
- 输出：自测报告

#### 阶段8：代码审查
- 提交审查，修复问题
- 合并代码

## 快速开始

### 生成 CRUD（推荐）
```
/gen-crud Product name:string price:decimal stock:int status:int category_id:long
```
自动生成：实体 + DTO + 服务 + Controller + 前端页面 + 数据库迁移

### 商品管理（电商场景）
```
/new-product --with-sku --with-images
```
生成完整的商品管理功能，包含 SKU 规格、商品图片

### 商品分类树
```
/new-category
```
生成多级分类管理，支持拖拽排序

### 单独生成 API
```
/new-api product list
/new-api order create user_id:long,total_amount:decimal
```

### 单独生成前端页面
```
/new-page user name:string,email:string,status:int
```

### 代码审查
```
/review Modules/Product/Services/ProductService.cs
/review pages/product/
/review --diff
```

### Git 提交
```
/commit
```

### 测试用例生成
```
/generate-testcase --from-api user
/generate-testcase --from-fields name:string,status:int
```
生成：正向用例 + 逆向用例 + 边界用例 + 安全用例

### WBS排期生成
```
/create-wbs --team 张三:前端,李四:后端,王五:测试
```
生成：WBS分解表 + 工时汇总 + 里程碑规划 + 详细排期

### Checklist生成
```
/create-checklist --type验收
/create-checklist --type发布
/create-checklist --type上线
```
生成：验收Checklist + 发布Checklist + 上线验证清单

### 安全审查
```
/security-review Modules/User
/security-review --depth deep
```
检查：SQL注入 + XSS + 越权 + 认证鉴权 + 敏感信息