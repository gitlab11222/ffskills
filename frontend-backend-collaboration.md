# 前后端协作规范

## 命名协作规范

### JSON 字段命名规范
| 规则 | 说明 |
|------|------|
| 后端返回驼峰命名 | API响应JSON使用camelCase |
| 前端接收驼峰命名 | 前端统一使用camelCase处理 |
| 禁止不一致命名 | 避免后端驼峰前端小写联调不通 |

### 前后端命名对照
```typescript
// ✓ 正确：前后端命名一致
// 后端响应
{
  "userId": 123,
  "userName": "张三",
  "createdAt": "2024-01-01"
}

// 前端接收
interface User {
  userId: number;
  userName: string;
  createdAt: string;
}

// ✗ 错误：命名不一致导致联调失败
// 后端返回 camelCase: userId
// 前端用 snake_case: user_id  // 禁止，无法正确映射

// ✗ 错误：后端 snake_case 前端 camelCase
// 后端返回: user_id
// 前端期望: userId  // 禁止，需要额外转换
```

### 后端 JSON 配置
```csharp
// Program.cs - JSON序列化配置
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        // 使用驼峰命名（camelCase）
        options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        // 保持字典键原样
        options.JsonSerializerOptions.DictionaryKeyPolicy = null;
    });

// 实体类定义（PascalCase）
public class User
{
    public long UserId { get; set; }      // 序列化为 userId
    public string UserName { get; set; }  // 序列化为 userName
    public DateTime CreatedAt { get; set; }  // 序列化为 createdAt
}
```

### 前端类型定义
```typescript
// 使用 camelCase 定义类型
interface User {
  userId: number;
  userName: string;
  createdAt: string;
}

// API调用
export async function getUser(id: number) {
  const res = await request.get<DataResponse<User>>(`/api/user/${id}`);
  // res.data.userId 可直接使用，无需转换
}
```

## 测试要求规范

### 测试类型强制要求
| 类型 | 强制要求 | 说明 |
|------|----------|------|
| 功能测试 | 必须 | 所有功能必须测试 |
| 安全测试 | 必须 | 防止SQL注入、越权等漏洞 |
| 性能测试 | 必须 | 防止内存泄漏、响应超时 |

### 安全测试必检项
| 检测项 | 说明 | 检测方法 |
|--------|------|----------|
| SQL注入 | 参数拼接SQL | 输入 `' OR 1=1--` |
| XSS注入 | 脚本注入 | 输入 `<script>alert(1)</script>` |
| 越权访问 | 水平/垂直越权 | 用用户A访问用户B数据 |
| 未授权访问 | 无Token访问 | 删除Token访问需授权接口 |
| 参数篡改 | 修改关键参数 | 修改orderId等参数 |

```markdown
// ✓ 安全测试必须执行
功能测试通过 → 安全测试 → 性能测试 → 验收

// ✗ 只做功能测试
功能测试 → 验收  // 禁止，漏掉安全漏洞

// ✓ 安全测试覆盖
- [ ] SQL注入测试
- [ ] XSS注入测试
- [ ] 越权访问测试
- [ ] 未授权访问测试
```

### 性能测试必检项
| 检测项 | 说明 | 检测方法 |
|--------|------|----------|
| 内存泄漏 | 长时间运行内存增长 | 压力测试监控内存 |
| 响应时间 | 接口响应超时 | 接口性能测试 |
| 并发性能 | 高并发稳定性 | 并发压力测试 |
| 数据库性能 | 慢查询检测 | 查询性能分析 |

```markdown
// ✓ 性能测试必须执行
- [ ] 内存泄漏检测
- [ ] 接口响应时间 < 200ms
- [ ] 并发测试通过

// ✗ 只做功能测试
功能完成就发布  // 禁止，可能存在性能问题
```

## 数据库索引规范

### 查询条件必须建索引
| 字段类型 | 必须建索引 | 说明 |
|----------|------------|------|
| WHERE 条件字段 | 必须 | 查询条件必须有索引 |
| JOIN 关联字段 | 必须 | 关联查询必须有索引 |
| ORDER BY 字段 | 推荐 | 排序字段推荐建索引 |
| GROUP BY 字段 | 推荐 | 分组字段推荐建索引 |

```sql
-- ✓ 查询条件有索引
SELECT * FROM orders WHERE user_id = 123;  -- user_id 有索引

-- ✗ 查询条件无索引
SELECT * FROM orders WHERE user_id = 123;  -- user_id 无索引，禁止

-- ✓ 创建索引
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- ✓ 复合索引（多个查询条件）
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

### 索引检查流程
```markdown
开发新接口流程：
1. 分析查询条件
2. 检查是否有索引
3. 无索引则创建索引
4. 执行 EXPLAIN 验证
5. 确认索引生效
```

### EXPLAIN 验证
```sql
-- 执行 EXPLAIN 检查索引
EXPLAIN SELECT * FROM orders WHERE user_id = 123;

-- 检查结果
-- key 列显示使用的索引名
-- type 列为 ref/range 说明索引生效
-- type 列为 ALL 说明全表扫描，禁止
```

## 代码优雅规范

### 代码复杂度原则
| 原则 | 说明 |
|------|------|
| 简洁优先 | 代码尽量简洁，不要太复杂 |
| 可读性 | 代码要易于理解和维护 |
| 复杂度评估 | 代码太复杂可以咨询是否改需求 |
| 需求协商 | 不要无脑完成所有需求 |

```markdown
// ✓ 简洁代码
if (user.Status == UserStatus.Active)
{
    return true;
}

// ✗ 复杂嵌套（考虑咨询改需求）
if (user != null)
{
    if (user.Status != null)
    {
        if (user.Status.Value == UserStatus.Active)
        {
            if (user.Role != null)
            {
                if (user.Role.Permissions != null)
                {
                    // 5层嵌套，太复杂，咨询改需求
                }
            }
        }
    }
}

// ✓ 复杂代码咨询改需求
发现代码逻辑太复杂 → 咨询用户 → 评估是否简化需求

// ✗ 无脑完成复杂需求
用户提复杂需求 → 无脑实现 → 代码难以维护  // 禁止
```

### 代码复杂度评估
```markdown
## 复杂度评估标准

| 维度 | 简洁 | 可接受 | 需咨询 |
|------|------|--------|--------|
| 嵌套层数 | ≤ 2 | 3 | ≥ 4 |
| 方法行数 | ≤ 20 | 20-50 | ≥ 50 |
| 条件分支 | ≤ 3 | 3-5 | ≥ 6 |
| 参数数量 | ≤ 3 | 3-5 | ≥ 6 |

## 复杂代码处理流程
发现复杂代码 → 评估复杂度 → 超过阈值 → 咨询用户 → 简化需求/重构代码
```

## 界面交互规范

### 时间查询条件规范
| 规则 | 说明 |
|------|------|
| 默认时间范围 | 时间查询默认1天范围 |
| 时间范围可选 | 支持自定义时间范围 |
| 时间格式统一 | 使用统一的日期格式 |

```typescript
// ✓ 时间查询默认1天
const [timeRange, setTimeRange] = useState<[string, string]>(() => {
  const end = dayjs().format('YYYY-MM-DD');
  const start = dayjs().subtract(1, 'day').format('YYYY-MM-DD');
  return [start, end];
});

// 查询参数
const params = {
  startTime: timeRange[0],
  endTime: timeRange[1],
};

// ✗ 时间查询无默认值
const [timeRange, setTimeRange] = useState<[string, string]>(['', '']);  // 禁止
```

### 查询分页统计导出规范
| 规则 | 说明 |
|------|------|
| 有查询就有分页 | 列表必须支持分页 |
| 有分页就有统计 | 分页列表显示总数 |
| 有查询就有导出 | 列表支持数据导出 |

```typescript
// ✓ 查询+分页+统计+导出完整
<MyProTable
  columns={columns}
  request={async (params) => {
    const res = await getList(params);
    return {
      data: res.data.list,
      total: res.data.total,  // 有统计
      success: res.code === 0,
    };
  }}
  toolBarRender={() => [
    <Button key="export" onClick={handleExport}>导出</Button>,  // 有导出
  ]}
/>

// ✗ 只有查询无分页
<List dataSource={data} />  // 禁止，无分页

// ✗ 有分页无统计
<MyProTable request={async () => {
  return { data: res.data.list };  // 禁止，缺少 total
}} />

// ✗ 有查询无导出
<MyProTable />  // 禁止，缺少导出按钮
```

### 弹窗与页面选择规范
| 场景 | 选择 | 说明 |
|------|------|------|
| 内容少（≤ 5字段） | 弹窗Modal | 简单表单用弹窗 |
| 内容多（> 5字段） | 新页面 | 复杂表单跳转新页面 |
| 关联数据多 | 新页面 | 需要多步操作的跳转页面 |
| 快速操作 | 弹窗Modal | 快速新增/编辑用弹窗 |

```typescript
// ✓ 内容少用弹窗（≤ 5字段）
<ModalForm title="修改状态" width={400}>
  <ProFormSelect name="status" label="状态" />
  <ProFormTextArea name="remark" label="备注" />
</ModalForm>

// ✓ 内容多用新页面（> 5字段）
<Button onClick={() => history.push('/order/create')}>新建订单</Button>

// ✗ 内容多用弹窗
<ModalForm title="新建订单" width={800}>
  {/* 10+字段，弹窗太拥挤 */}  // 禁止，应跳转新页面
</ModalForm>
```

### 弹窗大小动态调整
| 内容量 | 弹窗宽度 | 说明 |
|--------|----------|------|
| 少量内容 | 400px | 2-3个字段 |
| 中量内容 | 600px | 4-6个字段 |
| 较多内容 | 800px | 7-10个字段 |
| 大量内容 | 新页面 | > 10个字段 |

```typescript
// ✓ 弹窗宽度根据内容动态调整
const getModalWidth = (fieldCount: number) => {
  if (fieldCount <= 3) return 400;
  if (fieldCount <= 6) return 600;
  if (fieldCount <= 10) return 800;
  return null;  // > 10字段跳转新页面
};

<ModalForm
  title="编辑用户"
  width={getModalWidth(4)}  // 4字段用600px
>
  {/* 字段 */}
</ModalForm>

// ✗ 弹窗宽度固定
<ModalForm width={600}>  // 禁止，不考虑内容量
```

### 页面风格统一规范
| 规则 | 说明 |
|------|------|
| 表单布局统一 | 标签+输入框布局方式一致 |
| 列表样式统一 | 表格/卡片风格一致 |
| 按钮位置统一 | 操作按钮位置一致 |
| 颜色风格统一 | 主色调一致 |

```typescript
// ✓ 表单布局统一（标签和输入框同一行）
<ProForm
  layout="horizontal"  // 水平布局：标签+输入框同一行
  labelCol={{ span: 4 }}
  wrapperCol={{ span: 20 }}
>
  <ProFormText name="name" label="名称" />
  <ProFormText name="email" label="邮箱" />
</ProForm>

// ✓ 所有表单使用相同的布局配置
// 不允许某个页面是水平布局，另一个是垂直布局

// ✗ 页面风格不统一
// 页面A: 标签+输入框同一行
// 页面B: 标签+输入框分两行  // 禁止，风格不统一

// ✓ 全局统一配置
const formLayout = {
  layout: 'horizontal',
  labelCol: { span: 4 },
  wrapperCol: { span: 20 },
};

// 所有表单使用统一配置
```

### 列表页面统一规范
```typescript
// ✓ 列表页面统一结构
const ListPage: React.FC = () => {
  return (
    <MyProTable
      columns={columns}
      request={request}
      search={{
        labelWidth: 'auto',  // 统一搜索栏配置
      }}
      toolBarRender={() => [
        <Button key="create" type="primary" onClick={handleCreate}>新建</Button>,
        <Button key="export" onClick={handleExport}>导出</Button>,
      ]}
    />
  );
};

// ✓ 所有列表页使用相同组件和配置
// - 使用 MyProTable
// - 搜索栏 labelWidth: 'auto'
// - 操作按钮位置统一
```

## DO - 推荐做法

```markdown
// ✓ 前后端命名一致（驼峰）
后端返回 userId，前端用 userId

// ✓ 三种测试必须执行
功能测试 + 安全测试 + 性能测试

// ✓ 查询条件必须建索引
WHERE user_id = 123 → user_id 有索引

// ✓ 代码简洁，复杂时咨询
代码嵌套 ≤ 3层，复杂需求咨询改需求

// ✓ 时间查询默认1天范围
startTime: 今天-1天, endTime: 今天

// ✓ 查询+分页+统计+导出完整
列表必须有分页、显示总数、导出按钮

// ✓ 内容少弹窗，内容多新页面
≤ 5字段用弹窗，> 5字段跳转页面

// ✓ 弹窗宽度动态调整
根据字段数量动态设置宽度

// ✓ 页面风格统一
所有表单使用相同的布局配置
```

## DON'T - 禁止做法

```markdown
// ✗ 前后端命名不一致
后端返回 userId，前端用 user_id  // 禁止

// ✓ 正确做法
前后端统一使用驼峰命名

// ✗ 只做功能测试
功能测试 → 发布  // 禁止，漏安全漏洞

// ✓ 正确做法
功能测试 + 安全测试 + 性能测试 → 发布

// ✗ 查询条件无索引
WHERE user_id = 123  // user_id 无索引，禁止

// ✓ 正确做法
CREATE INDEX idx_user_id ON orders(user_id);

// ✗ 无脑完成复杂需求
用户提复杂需求 → 直接实现  // 禁止

// ✓ 正确做法
评估复杂度 → 咨询用户 → 简化需求或重构

// ✗ 时间查询无默认值
startTime: '', endTime: ''  // 禁止

// ✓ 正确做法
默认1天范围：startTime: 昨天, endTime: 今天

// ✗ 查询无分页/统计/导出
列表无分页、无总数、无导出  // 禁止

// ✓ 正确做法
分页 + 总数统计 + 导出按钮

// ✗ 内容多用弹窗
10+字段用弹窗  // 禁止，太拥挤

// ✓ 正确做法
> 5字段跳转新页面

// ✗ 弹窗宽度固定
width=600（不考虑内容）  // 禁止

// ✓ 正确做法
根据字段数量动态调整宽度

// ✗ 页面风格不统一
A页面水平布局，B页面垂直布局  // 禁止

// ✓ 正确做法
所有页面使用统一的布局配置
```

## 最佳实践

### 前后端协作
- 命名规范统一（驼峰命名）
- API文档同步更新
- 联调前确认字段命名

### 测试完备
- 功能测试必须
- 安全测试必须（SQL注入、越权）
- 性能测试必须（内存泄漏）

### 索引管理
- 查询条件必须有索引
- 新接口必须检查索引
- 使用 EXPLAIN 验证

### 代码质量
- 代码简洁可读
- 复杂度超阈值咨询用户
- 不无脑完成需求

### 界面统一
- 时间查询默认1天
- 查询+分页+统计+导出
- 弹窗/页面按内容选择
- 风格布局统一