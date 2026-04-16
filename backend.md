# 后端编码规则

## 项目结构

### Domain 层
```
Domain/
├── Common/           # 共享基础设施
│   ├── Entities/     # 基础实体
│   ├── Enums/        # 公共枚举
│   ├── Exceptions/   # 自定义异常
│   └── Extensions/   # 扩展方法
├── Modules/          # 业务模块
│   └── {Module}/
│       ├── Entities/     # 实体类
│       ├── Enums/        # 模块枚举
│       ├── Dtos/         # DTO（Request/Response）
│       ├── Interfaces/   # 服务接口
│       ├── Services/     # 服务实现
│       ├── Consumers/    # Redis消费者
│       └── Options/      # 配置类
└── Infrastructure/   # 基础设施
    ├── Redis/        # Redis配置
    ├── RabbitMQ/     # 消息队列
    └── External/     # 外部服务
```

### WebApi 层
```
WebApi/
├── Controllers/      # API控制器
├── Filters/          # 过滤器
├── Middlewares/      # 中间件
├── Extensions/       # 启动扩展
└── Program.cs        # 入口
```

## 依赖注入

### 自动注册规则
```csharp
// 必须继承 IAutoDIable 族接口
public interface IUserService : IAutoDIable { }           // 单例
public interface ITransientService : IAutoDIableTransient { }  // 瞬态
public interface IScopedService : IAutoDIScoped { }       // 作用域

// 实现类自动注册
public class UserService : IUserService { }

// Program.cs 自动注册
builder.Services.AddAutoDIServices([typeof(Program).Assembly]);
```

### 多实现策略模式
```csharp
// 1. 使用 [NameDI] 标记实现
[NameDI("alipay")]
public class AlipayService : IPaymentService { }

[NameDI("wechat")]
public class WechatPayService : IPaymentService { }

// 2. 通过工厂获取
public class PaymentServiceFactory : IMulInstanceFactory<IPaymentService>
{
    public IPaymentService GetInstance(string name) => name switch
    {
        "alipay" => _alipayService,
        "wechat" => _wechatPayService,
        _ => throw new BusinessException("不支持的支付方式")
    };
}
```

## Controller 规范

### 基类继承
```csharp
// 必须继承 ApiControllerBase
[ApiController, Route("api/[controller]")]
public class UserController : ApiControllerBase
{
    // 使用 [FromServices] 注入服务
    [HttpPost("list")]
    public async Task<IActionResult> GetList(
        [FromServices] IUserService userService,
        [FromBody] UserListRequest request)
        => Ok(await userService.GetListAsync(request));
}
```

### 响应格式
```csharp
// HTTP 状态码始终 200
// 统一响应信封
public class DataResponse<T>
{
    public int Code { get; set; }              // 0=成功, 其他=错误码
    public string Message { get; set; }         // 提示信息
    public int MessageType { get; set; }       // 0=info, 1=success, 2=warning, 3=error
    public T? Data { get; set; }               // 业务数据
}

// 成功响应
return Ok(DataResponse.Success(data));

// 失败响应
return Ok(DataResponse.Fail("参数错误", BusinessStatusCodeEnum.InvalidParam));

// 分页响应
return Ok(DataResponse.Success(new PagedList<T>
{
    List = list,
    Total = total,
    PageIndex = pageIndex,
    PageSize = pageSize
}));
```

### 异常处理
```csharp
// 预期业务错误：抛 BusinessException
throw new BusinessException("用户名已存在", BusinessStatusCodeEnum.UserExists);

// 禁止抛原生 Exception
// ❌ throw new Exception("系统错误");

// 全局异常处理中间件自动捕获
app.UseExceptionHandler("/error");
```

## 配置管理

### Options 模式
```csharp
// 1. 定义配置类（实现 IConfigurableOptions）
public class JwtOptions : IConfigurableOptions
{
    public string Secret { get; set; } = string.Empty;
    public int ExpireMinutes { get; set; } = 1440;
    public string Issuer { get; set; } = string.Empty;
}

// 2. appsettings.json（类名去掉Options后缀 = 配置节名）
{
  "Jwt": {
    "Secret": "your-secret-key",
    "ExpireMinutes": 1440,
    "Issuer": "your-app"
  }
}

// 3. Program.cs 自动绑定
builder.Services.Configure<JwtOptions>(builder.Configuration.GetSection("Jwt"));

// 4. 使用
public class UserService
{
    private readonly JwtOptions _jwtOptions;
    public UserService(IOptions<JwtOptions> jwtOptions) => _jwtOptions = jwtOptions.Value;
}
```

## 数据访问

### 实体定义
```csharp
// 实现 IBaseEntity 接口
public class User : IBaseEntity<long>
{
    public long Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public bool IsDeleted { get; set; }
}

// DbContext 配置
public class AppDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 软删除过滤器
        modelBuilder.Entity<User>().HasQueryFilter(e => !e.IsDeleted);

        // 索引
        modelBuilder.Entity<User>().HasIndex(e => e.UserName).IsUnique();
    }
}
```

### CRUD 操作
```csharp
// 使用 BaseRepository<T>
public class UserRepository : BaseRepository<User>, IUserRepository
{
    public UserRepository(AppDbContext db) : base(db) { }
}

// 基础方法
await _userRepo.GetByIdAsync(id);
await _userRepo.GetListAsync(e => e.Status == UserStatus.Active);
await _userRepo.AddAsync(new User { ... });
await _userRepo.UpdateAsync(user);
await _userRepo.DeleteAsync(id);  // 软删除
```

### 复杂查询（Dapper）
```csharp
// 使用 IUnitOfWork
public class OrderService
{
    private readonly IUnitOfWork _uow;

    public async Task<PagedList<OrderDto>> GetComplexListAsync(OrderListRequest request)
    {
        var sql = @"
            SELECT o.*, u.UserName
            FROM Orders o
            LEFT JOIN Users u ON o.UserId = u.Id
            WHERE o.Status = @Status
            ORDER BY o.CreatedAt DESC
            LIMIT @PageSize OFFSET @Offset";

        var countSql = @"
            SELECT COUNT(*)
            FROM Orders o
            WHERE o.Status = @Status";

        var list = await _uow.QueryAsync<OrderDto>(sql, new { Status = request.Status, PageSize = request.PageSize, Offset = offset });
        var total = await _uow.ExecuteScalarAsync<int>(countSql, new { Status = request.Status });

        return new PagedList<OrderDto> { List = list, Total = total };
    }
}
```

## Redis 使用

### 分布式锁
```csharp
public class OrderService
{
    private readonly IRedisLock _redisLock;

    public async Task<bool> ProcessOrderAsync(long orderId)
    {
        // 获取分布式锁（必须设置超时）
        await using var lockResult = await _redisLock.WaitAsync(
            $"{RedisKey.OrderLock}:{orderId}",
            TimeSpan.FromSeconds(30));  // 锁超时时间

        if (!lockResult.IsAcquired)
        {
            throw new BusinessException("操作频繁，请稍后重试", BusinessStatusCodeEnum.TooManyRequests);
        }

        // 执行业务逻辑...
        return true;
    }
}
```

### 缓存
```csharp
public class UserService
{
    private readonly IDatabase _redis;

    public async Task<User?> GetUserAsync(long userId)
    {
        var cacheKey = $"{RedisKey.UserCache}:{userId}";

        // 先查缓存
        var cached = await _redis.StringGetAsync(cacheKey);
        if (cached.HasValue)
            return JsonSerializer.Deserialize<User>(cached!);

        // 查数据库
        var user = await _userRepo.GetByIdAsync(userId);
        if (user == null) return null;

        // 写入缓存（设置过期时间）
        await _redis.StringSetAsync(
            cacheKey,
            JsonSerializer.Serialize(user),
            TimeSpan.FromMinutes(30));

        return user;
    }
}
```

### 队列消费者
```csharp
// 定义队列常量
public static class RedisKey
{
    public const string OrderCreateQueue = "queue:order:create";
    public const string OrderCallbackQueue = "queue:order:callback";
}

// 实现消费者
[RedisConsumer(RedisKey.OrderCreateQueue, IntervalSeconds = 5)]
public class OrderCreateConsumer : IRedisConsumer
{
    public async Task ExecuteAsync(string message)
    {
        var order = JsonSerializer.Deserialize<OrderMessage>(message);
        // 处理订单...
    }
}
```

## Program.cs 标准启动顺序

```csharp
var builder = WebApplication.CreateBuilder(args);

// 1. Autofac 容器
builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory());

// 2. 基础设施（日志、配置）
builder.Services.AddSerilog(builder.Configuration);
builder.Services.AddOptions();

// 3. Web 框架
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// 4. 认证授权
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => { /* JWT配置 */ });
builder.Services.AddAuthorization();

// 5. 数据层
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseMySql(connectionString, ServerVersion.AutoDetect(connectionString)));
builder.Services.AddDapper();
builder.Services.AddRedis(builder.Configuration);

// 6. 自动DI
builder.Services.AddAutoDIServices([typeof(Program).Assembly]);

var app = builder.Build();

// 中间件管道
app.UseSwagger();
app.UseSwaggerUI();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

## 命名约定

### 项目命名
```
{公司}.{产品}.{模块}.{层}
例如：Company.Product.User.Domain
      Company.Product.User.WebApi
```

### 接口命名
```csharp
// 服务接口：I{Service}Service
public interface IUserService : IAutoDIable { }

// 仓储接口：I{Entity}Repository
public interface IUserRepository { }
```

### 实现命名
```csharp
// 服务实现：{Service}Service
public class UserService : IUserService { }

// 仓储实现：{Entity}Repository
public class UserRepository : BaseRepository<User>, IUserRepository { }
```

### 请求/响应命名
```csharp
// Request：{Action}Request
public class UserListRequest { }

// Response：{Action}Response 或 {Entity}Dto
public class UserListResponse { }
public class UserDto { }
```

### Controller 命名
```csharp
// {Entity}Controller
public class UserController : ApiControllerBase { }
```

## Dockerfile 标准

```dockerfile
# 3阶段构建
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["Company.Product.User.WebApi/Company.Product.User.WebApi.csproj", "Company.Product.User.WebApi/"]
RUN dotnet restore "Company.Product.User.WebApi/Company.Product.User.WebApi.csproj"
COPY . .
WORKDIR "/src/Company.Product.User.WebApi"
RUN dotnet build "Company.Product.User.WebApi.csproj" -c Release -o /app/build

# Stage 2: Publish
FROM build AS publish
RUN dotnet publish "Company.Product.User.WebApi.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Stage 3: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
# 设置时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
# 删除开发配置
RUN rm -f appsettings.Development.json
EXPOSE 80
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Company.Product.User.WebApi.dll"]
```