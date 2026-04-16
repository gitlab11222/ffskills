---
name: new-api
description: 创建新的 API 接口
userInvoke: true
---

# API 接口生成 Skill

快速创建新的 API 接口，遵循统一规范。

## 使用方式

```
/new-api <模块名> <操作名> [参数列表]
```

示例：
```
/new-api user list
/new-api product detail id:long
/new-api order create user_id:long,total_amount:decimal,items:list
```

## 参数说明

| 参数 | 说明 | 示例 |
|------|------|------|
| 模块名 | 模块/Controller名 | user, product, order |
| 操作名 | API操作名 | list, create, update, delete, detail |
| 参数列表 | 请求参数 | id:long,name:string |

## API 操作类型

| 操作 | HTTP方法 | URL格式 | Request | Response |
|------|---------|---------|---------|----------|
| list | POST | `/api/{module}/list` | PageRequest | PagedList |
| detail | POST | `/api/{module}/detail` | DetailRequest | EntityDto |
| create | POST | `/api/{module}/create` | CreateRequest | long |
| update | POST | `/api/{module}/update` | UpdateRequest | bool |
| delete | POST | `/api/{module}/delete` | DeleteRequest | bool |
| custom | POST | `/api/{module}/{action}` | CustomRequest | CustomDto |

## 执行步骤

### 1. 创建 Request DTO
```csharp
// 列表查询
public class {Module}ListRequest : PageRequest
{
    // 查询参数...
}

// 详情查询
public class DetailRequest
{
    [Required]
    public long Id { get; set; }
}

// 创建请求
public class Create{Module}Request
{
    // 创建参数...
}
```

### 2. 创建 Response DTO
```csharp
public class {Module}Dto
{
    public long Id { get; set; }
    // 其他字段...
}
```

### 3. 服务接口方法
```csharp
public interface I{Module}Service : IAutoDIScoped
{
    Task<DataResponse<PagedList<{Module}Dto>>> Get{Action}Async({Module}{Action}Request request);
}
```

### 4. Controller 方法
```csharp
[HttpPost("{action}")]
[ProducesResponseType(typeof(DataResponse<T>), 200)]
public async Task<IActionResult> {Action}(
    [FromServices] I{Module}Service svc,
    [FromBody] {Module}{Action}Request request)
    => Ok(await svc.Get{Action}Async(request));
```

### 5. 前端 API
```typescript
export async function {module}{Action}(params: API.{Module}{Action}Params) {
  return request.post<API.DataResponse<T>>(`/api/{module}/{action}`, params);
}
```

## 代码模板

### Controller 方法模板
```csharp
/// <summary>{操作说明}</summary>
[HttpPost("{action}")]
[ProducesResponseType(typeof(DataResponse<{ReturnType}>), 200)]
public async Task<IActionResult> {Action}(
    [FromServices] I{Module}Service svc,
    [FromBody] {Module}{Action}Request request)
{
    // 参数校验（ModelState 自动校验）
    if (!ModelState.IsValid)
    {
        var errors = ModelState
            .Where(e => e.Value!.Errors.Count > 0)
            .Select(e => $"{e.Key}: {e.Value!.Errors.First().ErrorMessage}")
            .ToList();
        throw new BusinessException($"参数校验失败: {string.Join(", ", errors)}");
    }

    return Ok(await svc.Get{Action}Async(request));
}
```

### 服务方法模板
```csharp
public async Task<DataResponse<{ReturnType}>> Get{Action}Async({Module}{Action}Request request)
{
    _logger.LogInformation("{Module}{Action}: {Request}", "{Module}", "{Action}", request);

    try
    {
        // 业务逻辑
        var result = await DoWorkAsync(request);
        
        _logger.LogInformation("{Module}{Action}成功: {Result}", "{Module}", "{Action}", result);
        return DataResponse.Success(result);
    }
    catch (BusinessException ex)
    {
        _logger.LogWarning(ex, "{Module}{Action}业务错误: {Message}", "{Module}", "{Action}", ex.Message);
        throw;  // 全局异常处理
    }
}
```

## 输出清单

1. ✅ Request DTO 创建
2. ✅ Response DTO 创建（如需要）
3. ✅ 服务接口方法添加
4. ✅ 服务实现方法添加
5. ✅ Controller 方法添加
6. ✅ Swagger 注释添加
7. ✅ 前端 API 函数添加

## 注意事项

1. 所有 API 使用 POST 方法
2. HTTP 状态码始终返回 200
3. 响应格式统一为 DataResponse
4. 参数校验使用 DataAnnotations
5. 业务错误抛 BusinessException