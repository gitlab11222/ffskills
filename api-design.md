# API 设计规范

## URL 设计

### 基本原则
- **本项目统一 POST**: 所有 API 使用 POST 方法（非标准 RESTful）
- **路径格式**: `/api/{module}/{action}`
- **路径命名**: 小写 camelCase

### URL 示例
```
/api/user/list          # 用户列表
/api/user/create        # 创建用户
/api/user/update        # 更新用户
/api/user/delete        # 删除用户
/api/user/detail        # 用户详情
/api/order/create       # 创建订单
/api/order/cancel       # 取消订单
/api/payment/pay        # 发起支付
/api/payment/callback   # 支付回调
```

## 请求设计

### Request 模型
```csharp
// 基础请求
public class PageRequest
{
    /// <summary>页码（从1开始）</summary>
    [Range(1, int.MaxValue)]
    public int Page { get; set; } = 1;

    /// <summary>每页条数</summary>
    [Range(1, 100)]
    public int PageSize { get; set; } = 20;
}

// 列表查询请求
public class UserListRequest : PageRequest
{
    /// <summary>用户名（模糊搜索）</summary>
    [MaxLength(50)]
    public string? UserName { get; set; }

    /// <summary>状态</summary>
    public UserStatus? Status { get; set; }

    /// <summary>创建时间开始</summary>
    public DateTime? CreatedAtStart { get; set; }

    /// <summary>创建时间结束</summary>
    public DateTime? CreatedAtEnd { get; set; }
}

// 创建请求
public class CreateUserRequest
{
    /// <summary>用户名</summary>
    [Required(ErrorMessage = "用户名不能为空")]
    [MaxLength(50, ErrorMessage = "用户名最长50字符")]
    public string UserName { get; set; } = string.Empty;

    /// <summary>邮箱</summary>
    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;

    /// <summary>手机号</summary>
    [RegularExpression(@"^1[3-9]\d{9}$", ErrorMessage = "手机号格式不正确")]
    public string? Phone { get; set; }

    /// <summary>状态</summary>
    [Required]
    public UserStatus Status { get; set; }
}

// 更新请求
public class UpdateUserRequest : CreateUserRequest
{
    /// <summary>用户ID</summary>
    [Required]
    public long Id { get; set; }
}
```

### 参数校验
```csharp
// 使用 DataAnnotations
[ApiController]
public class UserController : ApiControllerBase
{
    [HttpPost("create")]
    public async Task<IActionResult> Create(
        [FromServices] IUserService userService,
        [FromBody] CreateUserRequest request)
    {
        // 自动校验（ModelState）
        if (!ModelState.IsValid)
        {
            var errors = ModelState
                .Where(e => e.Value!.Errors.Count > 0)
                .Select(e => $"{e.Key}: {e.Value!.Errors.First().ErrorMessage}")
                .ToList();
            throw new BusinessException($"参数校验失败: {string.Join(", ", errors)}");
        }

        return Ok(await userService.CreateAsync(request));
    }
}

// 或使用全局过滤器自动处理
public class ValidationFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            var errors = context.ModelState
                .Where(e => e.Value!.Errors.Count > 0)
                .Select(e => e.Value!.Errors.First().ErrorMessage)
                .ToList();
            context.Result = Ok(DataResponse.Fail($"参数校验失败: {string.Join(", ", errors)}", 400));
        }
    }
}
```

## 响应设计

### 统一响应信封
```csharp
public class DataResponse<T>
{
    /// <summary>状态码（0=成功，其他=错误）</summary>
    public int Code { get; set; }

    /// <summary>提示信息</summary>
    public string Message { get; set; } = string.Empty;

    /// <summary>消息类型（0=info, 1=success, 2=warning, 3=error）</summary>
    public int MessageType { get; set; }

    /// <summary>业务数据</summary>
    public T? Data { get; set; }

    // 静态工厂方法
    public static DataResponse<T> Success(T data, string message = "操作成功")
        => new() { Code = 0, Message = message, MessageType = 1, Data = data };

    public static DataResponse<T> Fail(string message, int code = 1, int messageType = 3)
        => new() { Code = code, Message = message, MessageType = messageType };
}

// 无数据响应
public class DataResponse : DataResponse<object>
{
    public static DataResponse Success(string message = "操作成功")
        => Success(null, message);
}
```

### 分页响应
```csharp
public class PagedList<T>
{
    /// <summary>数据列表</summary>
    public List<T> List { get; set; } = [];

    /// <summary>总条数</summary>
    public long Total { get; set; }

    /// <summary>当前页码</summary>
    public int PageIndex { get; set; }

    /// <summary>每页条数</summary>
    public int PageSize { get; set; }

    /// <summary>总页数</summary>
    public int TotalPages => (int)Math.Ceiling(Total / (double)PageSize);
}
```

### 响应示例
```json
// 成功响应（无数据）
{
  "code": 0,
  "message": "操作成功",
  "messageType": 1,
  "data": null
}

// 成功响应（有数据）
{
  "code": 0,
  "message": "查询成功",
  "messageType": 1,
  "data": {
    "id": 12345,
    "userName": "张三",
    "status": 1
  }
}

// 成功响应（分页）
{
  "code": 0,
  "message": "查询成功",
  "messageType": 1,
  "data": {
    "list": [...],
    "total": 1000,
    "pageIndex": 1,
    "pageSize": 20,
    "totalPages": 50
  }
}

// 错误响应
{
  "code": 400,
  "message": "参数校验失败: 用户名不能为空",
  "messageType": 3,
  "data": null
}
```

## Controller 标准

### 标准 Controller 模板
```csharp
/// <summary>用户管理</summary>
[ApiController]
[Route("api/[controller]")]
[ApiExplorerSettings(GroupName = "v1")]
public class UserController : ApiControllerBase
{
    /// <summary>获取用户列表</summary>
    [HttpPost("list")]
    [ProducesResponseType(typeof(DataResponse<PagedList<UserDto>>), 200)]
    public async Task<IActionResult> GetList(
        [FromServices] IUserService userService,
        [FromBody] UserListRequest request)
        => Ok(await userService.GetListAsync(request));

    /// <summary>获取用户详情</summary>
    [HttpPost("detail")]
    [ProducesResponseType(typeof(DataResponse<UserDto>), 200)]
    public async Task<IActionResult> GetDetail(
        [FromServices] IUserService userService,
        [FromBody] DetailRequest request)
        => Ok(await userService.GetDetailAsync(request.Id));

    /// <summary>创建用户</summary>
    [HttpPost("create")]
    [ProducesResponseType(typeof(DataResponse<long>), 200)]
    public async Task<IActionResult> Create(
        [FromServices] IUserService userService,
        [FromBody] CreateUserRequest request)
        => Ok(await userService.CreateAsync(request));

    /// <summary>更新用户</summary>
    [HttpPost("update")]
    [ProducesResponseType(typeof(DataResponse<bool>), 200)]
    public async Task<IActionResult> Update(
        [FromServices] IUserService userService,
        [FromBody] UpdateUserRequest request)
        => Ok(await userService.UpdateAsync(request));

    /// <summary>删除用户</summary>
    [HttpPost("delete")]
    [ProducesResponseType(typeof(DataResponse<bool>), 200)]
    public async Task<IActionResult> Delete(
        [FromServices] IUserService userService,
        [FromBody] DeleteRequest request)
        => Ok(await userService.DeleteAsync(request.Id));
}
```

### ApiControllerBase
```csharp
/// <summary>API控制器基类</summary>
[ApiController]
public abstract class ApiControllerBase : Controller
{
    // 公共方法可在此定义
}
```

## Swagger 文档

### API 分组
```csharp
// Program.cs
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "API 文档",
        Version = "v1",
        Description = "系统 API 文档"
    });

    // JWT 认证
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme.",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "Bearer" }
            },
            Array.Empty<string>()
        }
    });

    // XML 注释
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    if (File.Exists(xmlPath))
        options.IncludeXmlComments(xmlPath);
});

// 启用 Swagger UI
app.UseSwagger();
app.UseSwaggerUI(options =>
{
    options.SwaggerEndpoint("/swagger/v1/swagger.json", "API v1");
    options.RoutePrefix = "swagger";
});
```

### API 注释
```csharp
/// <summary>用户管理</summary>
[ApiController]
[Route("api/[controller]")]
public class UserController : ApiControllerBase
{
    /// <summary>
    /// 获取用户列表
    /// </summary>
    /// <param name="userService">用户服务</param>
    /// <param name="request">查询参数</param>
    /// <returns>用户分页列表</returns>
    [HttpPost("list")]
    [ProducesResponseType(typeof(DataResponse<PagedList<UserDto>>), 200)]
    public async Task<IActionResult> GetList(...) => ...;
}
```

## 安全设计

### 认证
```csharp
// JWT 认证配置
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = jwtOptions.Issuer,
            ValidAudience = jwtOptions.Audience,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtOptions.Secret))
        };
    });

// Controller 使用
[Authorize]
[ApiController]
public class UserController : ApiControllerBase
{
    [AllowAnonymous]  // 允许匿名访问
    [HttpPost("login")]
    public async Task<IActionResult> Login(...) => ...;

    [Authorize(Roles = "Admin")]  // 需要管理员角色
    [HttpPost("delete")]
    public async Task<IActionResult> Delete(...) => ...;
}
```

### 权限控制
```csharp
// 权限策略
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("UserView", policy => policy.RequireClaim("Permission", "user:view"));
    options.AddPolicy("UserCreate", policy => policy.RequireClaim("Permission", "user:create"));
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));
});

// Controller 使用
[Authorize(Policy = "UserView")]
[HttpPost("list")]
public async Task<IActionResult> GetList(...) => ...;

[Authorize(Policy = "UserCreate")]
[HttpPost("create")]
public async Task<IActionResult> Create(...) => ...;
```

### 输入安全
```csharp
// SQL 注入防护（使用参数化查询）
var users = await _context.Users
    .FromSqlRaw("SELECT * FROM users WHERE user_name = {0}", userName)
    .ToListAsync();

// XSS 防护（前端使用 sanitize）
import sanitizeHtml from 'sanitize-html';
const safeContent = sanitizeHtml(userInput, {
  allowedTags: ['b', 'i', 'em', 'strong'],
  allowedAttributes: {}
});

// 敏敏信息加密
public class UserService
{
    public string HashPassword(string password)
        => BCrypt.Net.BCrypt.HashPassword(password);

    public bool VerifyPassword(string password, string hash)
        => BCrypt.Net.BCrypt.Verify(password, hash);
}
```

### 输出安全
```csharp
// 敏感字段过滤
public class UserDto
{
    public long Id { get; set; }
    public string UserName { get; set; }
    // 不返回密码、盐值等敏感字段
    // public string PasswordHash { get; set; }  // 不暴露
}

// AutoMapper 配置
public class UserProfile : Profile
{
    public UserProfile()
    {
        CreateMap<User, UserDto>()
            .ForMember(dest => dest.PasswordHash, opt => opt.Ignore());
    }
}
```

## 错误码设计

### 错误码分段
```
0       = 成功
400-499 = 参数错误
500-599 = 系统错误
1000-1999 = 认证/授权错误
2000-2999 = 业务错误
```

### 错误码枚举
```csharp
public enum BusinessStatusCodeEnum
{
    /// <summary>成功</summary>
    Success = 0,

    /// <summary>参数无效</summary>
    InvalidParam = 400,

    /// <summary>参数缺失</summary>
    MissingParam = 401,

    /// <summary>参数格式错误</summary>
    FormatError = 402,

    /// <summary>系统内部错误</summary>
    SystemError = 500,

    /// <summary>数据库错误</summary>
    DatabaseError = 501,

    /// <summary>未登录</summary>
    NotLogin = 1000,

    /// <summary>登录过期</summary>
    TokenExpired = 1001,

    /// <summary>无权限</summary>
    NoPermission = 1002,

    /// <summary>用户已存在</summary>
    UserExists = 2001,

    /// <summary>用户不存在</summary>
    UserNotFound = 2002,

    /// <summary>密码错误</summary>
    PasswordError = 2003,

    /// <summary>账户已禁用</summary>
    AccountDisabled = 2004,

    /// <summary>操作频繁</summary>
    TooManyRequests = 2005,

    /// <summary>订单不存在</summary>
    OrderNotFound = 2100,

    /// <summary>订单状态错误</summary>
    OrderStatusError = 2101,

    /// <summary>库存不足</summary>
    InventoryNotEnough = 2102,
}
```

### 错误码使用
```csharp
// 业务异常
throw new BusinessException("用户已存在", BusinessStatusCodeEnum.UserExists);

// 响应
return Ok(DataResponse.Fail("用户已存在", BusinessStatusCodeEnum.UserExists));
```