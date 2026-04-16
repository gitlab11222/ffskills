# C# 编码指南

## 命名约定

### PascalCase
- 类、接口、枚举、结构体
- 公共属性、方法、事件
- 命名空间

```csharp
public class UserService { }
public interface IUserRepository { }
public enum UserStatus { }
public struct Point { }
public string UserName { get; set; }
public void GetUserById(long userId) { }
public event EventHandler<UserCreatedEventArgs> UserCreated;
namespace MyApp.Services { }
```

### camelCase
- 局部变量、参数
- 私有字段（推荐 `_camelCase`）

```csharp
public void ProcessOrder(long orderId)
{
    var order = await GetOrderAsync(orderId);
    int itemCount = order.Items.Count;
}

// 私有字段推荐格式
private readonly IUserRepository _userRepository;
private int _counter;
```

### 常量命名
```csharp
// 公共常量：PascalCase
public const int MaxPageSize = 100;
public const string DefaultDateFormat = "yyyy-MM-dd";

// 私有常量：_PascalCase 或 PascalCase
private const int _MaxRetryCount = 3;
private const int MaxRetryCount = 3;
```

## C# 12 特性

### 主构造函数
```csharp
// 传统方式
public class UserService
{
    private readonly IUserRepository _repo;
    public UserService(IUserRepository repo) => _repo = repo;
}

// C# 12 主构造函数
public class UserService(IUserRepository repo)
{
    public User? GetUser(long id) => repo.GetById(id);
}
```

### 集合表达式
```csharp
// 传统方式
var list = new List<int> { 1, 2, 3 };
var array = new int[] { 1, 2, 3 };

// C# 12 集合表达式
List<int> list = [1, 2, 3];
int[] array = [1, 2, 3];
HashSet<int> set = [1, 2, 3];

// Spread 操作符
var combined = [..list, 4, 5, 6];
```

### 默认 Lambda 参数
```csharp
// C# 12 支持 Lambda 默认参数
var add = (int a, int b = 1) => a + b;
add(5);      // 6
add(5, 2);   // 7
```

### 内联数组
```csharp
// C# 12 内联数组（性能优化）
[InlineArray(3)]
public struct Buffer<T>
{
    private T _element0;
}

var buffer = new Buffer<int>();
buffer[0] = 1;
buffer[1] = 2;
buffer[2] = 3;
```

## 异步编程

### Async 命名
```csharp
// Async 方法以 Async 结尾
public async Task<User?> GetUserAsync(long userId);
public async Task<List<User>> GetListAsync();
public async Task<bool> SaveAsync();

// Event handler 以 EventHandlerAsync 结尾
public async Task HandleUserCreatedEventHandlerAsync(UserCreatedEventArgs e);
```

### ConfigureAwait
```csharp
// 类库代码：使用 ConfigureAwait(false)
public class UserService
{
    public async Task<User?> GetUserAsync(long userId)
    {
        return await _repo.GetByIdAsync(userId).ConfigureAwait(false);
    }
}

// UI/ASP.NET Core：不需要 ConfigureAwait
public class UserController : ControllerBase
{
    public async Task<IActionResult> GetUser(long userId)
    {
        var user = await _userService.GetUserAsync(userId);
        return Ok(user);
    }
}
```

### Async 最佳实践
```csharp
// ✓ 正确：异步方法返回 Task
public async Task<User> GetUserAsync(long id)
{
    return await _repo.GetByIdAsync(id);
}

// ✗ 错误：async void（仅用于 event handler）
public async void GetUser(long id)  // 禁止
{
    var user = await _repo.GetByIdAsync(id);
}

// ✓ 正确：并行执行
var userTask = GetUserAsync(userId);
var orderTask = GetOrderAsync(orderId);
await Task.WhenAll(userTask, orderTask);
var user = await userTask;
var order = await orderTask;

// ✗ 错误：顺序执行（不必要）
var user = await GetUserAsync(userId);
var order = await GetOrderAsync(orderId);  // 可并行

// ✓ 正确：避免不必要的 async
public Task<User?> GetUserAsync(long id) => _repo.GetByIdAsync(id);

// ✗ 错误：不必要的 async/await
public async Task<User?> GetUserAsync(long id)
{
    return await _repo.GetByIdAsync(id);  // 多余
}
```

## 异常处理

### BusinessException
```csharp
// 业务异常（预期错误）
public class BusinessException : Exception
{
    public int ErrorCode { get; }

    public BusinessException(string message, int errorCode = 1)
        : base(message)
    {
        ErrorCode = errorCode;
    }
}

// 使用
public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
{
    if (request.Quantity <= 0)
        throw new BusinessException("数量必须大于0", BusinessStatusCodeEnum.InvalidParam);

    var inventory = await GetInventoryAsync(request.ProductId);
    if (inventory.Quantity < request.Quantity)
        throw new BusinessException("库存不足", BusinessStatusCodeEnum.InventoryNotEnough);

    // ...
}
```

### 防御性编程
```csharp
// 参数校验
public async Task<User> CreateUserAsync(CreateUserRequest request)
{
    // Guard clauses
    if (request == null)
        throw new ArgumentNullException(nameof(request));

    if (string.IsNullOrWhiteSpace(request.UserName))
        throw new BusinessException("用户名不能为空");

    // 主流程
    var user = new User { UserName = request.UserName };
    await _repo.AddAsync(user);
    return user;
}
```

### 异常捕获
```csharp
// ✓ 正确：捕获具体异常
try
{
    await ProcessOrderAsync(orderId);
}
catch (BusinessException ex)
{
    // 业务错误：记录日志，返回错误信息
    _logger.LogWarning(ex, "业务处理失败: {Message}", ex.Message);
    return DataResponse.Fail(ex.Message, ex.ErrorCode);
}
catch (DbUpdateException ex)
{
    // 数据库错误：记录日志，返回系统错误
    _logger.LogError(ex, "数据库更新失败");
    return DataResponse.Fail("系统错误，请稍后重试", BusinessStatusCodeEnum.DatabaseError);
}

// ✗ 错误：捕获所有异常
try
{
    await ProcessOrderAsync(orderId);
}
catch (Exception ex)  // 太宽泛
{
    _logger.LogError(ex, "未知错误");
    throw;
}

// ✗ 错误：吞掉异常
try
{
    await ProcessOrderAsync(orderId);
}
catch
{
    // 没有任何处理
}
```

## LINQ 最佳实践

### EF Core 查询优化
```csharp
// ✓ 正确：AsNoTracking 只读查询
var users = await _context.Users
    .AsNoTracking()
    .Where(e => e.Status == UserStatus.Active)
    .ToListAsync();

// ✓ 正确：Select 只查询需要的字段
var userIds = await _context.Users
    .AsNoTracking()
    .Where(e => e.Status == UserStatus.Active)
    .Select(e => e.Id)
    .ToListAsync();

// ✓ 正确：分页查询
var orders = await _context.Orders
    .AsNoTracking()
    .Where(e => e.UserId == userId)
    .OrderByDescending(e => e.CreatedAt)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();

// ✗ 错误：ToList 后再过滤（内存操作）
var allOrders = await _context.Orders.ToListAsync();
var filtered = allOrders.Where(e => e.UserId == userId);  // 内存操作

// ✗ 错误：Include 不需要的关联
var orders = await _context.Orders
    .Include(e => e.User)          // 不需要
    .Include(e => e.Items)         // 不需要
    .ToListAsync();
```

### 内存集合处理
```csharp
// List/IEnumerable 操作
var users = GetUsers();

// ✓ 正确：链式操作（延迟执行）
var activeUsers = users
    .Where(e => e.Status == UserStatus.Active)
    .Select(e => new UserDto { Id = e.Id, Name = e.UserName })
    .ToList();  // 最后才执行

// ✓ 正确：复用 IEnumerable
var baseQuery = users.Where(e => e.Status == UserStatus.Active);
var count = baseQuery.Count();
var list = baseQuery.Take(10).ToList();
```

## 依赖注入

### 构造函数注入
```csharp
// 推荐：构造函数注入
public class UserService
{
    private readonly IUserRepository _repo;
    private readonly ILogger<UserService> _logger;

    public UserService(IUserRepository repo, ILogger<UserService> logger)
    {
        _repo = repo;
        _logger = logger;
    }
}

// C# 12 主构造函数注入
public class UserService(IUserRepository repo, ILogger<UserService> logger)
{
    public User? GetUser(long id) => repo.GetById(id);
}
```

### 服务生命周期
```csharp
// Singleton: 单例（整个应用生命周期）
public interface IConfigService : IAutoDIable { }

// Scoped: 作用域（每次请求）
public interface IUserService : IAutoDIScoped { }

// Transient: 瞬态（每次获取新实例）
public interface IEmailService : IAutoDIableTransient { }
```

### 接口设计
```csharp
// ✓ 正确：接口隔离
public interface IUserRepository
{
    Task<User?> GetByIdAsync(long id);
    Task<List<User>> GetListAsync();
    Task AddAsync(User user);
    Task UpdateAsync(User user);
}

// ✗ 错误：万能接口
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(long id);
    Task<List<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(long id);
    Task SaveAsync();
    // ...太多方法
}
```

## AutoMapper

### Profile 配置
```csharp
public class UserProfile : Profile
{
    public UserProfile()
    {
        CreateMap<User, UserDto>()
            .ForMember(dest => dest.StatusText,
                opt => opt.MapFrom(src => src.Status.GetDescription()));

        CreateMap<CreateUserRequest, User>()
            .ForMember(dest => dest.Id, opt => opt.Ignore())
            .ForMember(dest => dest.CreatedAt, opt => opt.Ignore());

        CreateMap<User, UserListDto>()
            .ForMember(dest => dest.OrderCount,
                opt => opt.MapFrom(src => src.Orders.Count));
    }
}

// 注册
builder.Services.AddAutoMapper(typeof(UserProfile).Assembly);
```

### 使用
```csharp
public class UserService
{
    private readonly IMapper _mapper;

    public async Task<UserDto> GetUserAsync(long id)
    {
        var user = await _repo.GetByIdAsync(id);
        return _mapper.Map<UserDto>(user);
    }

    public async Task<List<UserListDto>> GetListAsync()
    {
        var users = await _repo.GetListAsync();
        return _mapper.Map<List<UserListDto>>(users);
    }
}
```

## 日志规范

### Serilog 配置
```csharp
// Program.cs
builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .WriteTo.Console()
    .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day)
    .WriteTo.Seq("http://localhost:5341")  // 或阿里云 SLS
);
```

### 结构化日志
```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // ✓ 正确：结构化日志
        _logger.LogInformation("创建订单: UserId={UserId}, ProductId={ProductId}, Quantity={Quantity}",
            request.UserId, request.ProductId, request.Quantity);

        var order = await _repo.AddAsync(new Order { ... });

        _logger.LogInformation("订单创建成功: OrderId={OrderId}", order.Id);

        return order;
    }

    // ✗ 错误：字符串拼接
    _logger.LogInformation($"创建订单: UserId={request.UserId}");  // 不推荐
}
```

### 日志级别
```csharp
// Trace: 详细跟踪信息
_logger.LogTrace("进入方法: GetUserAsync({UserId})", userId);

// Debug: 调试信息
_logger.LogDebug("查询参数: {Request}", request);

// Information: 重要业务事件
_logger.LogInformation("订单创建成功: OrderId={OrderId}", orderId);

// Warning: 潜在问题
_logger.LogWarning("库存不足: ProductId={ProductId}, Required={Required}, Available={Available}",
    productId, required, available);

// Error: 错误
_logger.LogError(ex, "订单处理失败: OrderId={OrderId}", orderId);

// Critical: 严重错误
_logger.LogCritical(ex, "数据库连接失败");
```

### 敏感信息保护
```csharp
// ✓ 正确：不记录敏感信息
_logger.LogInformation("用户登录: UserName={UserName}", userName);

// ✗ 错误：记录敏感信息
_logger.LogInformation("用户登录: Password={Password}", password);  // 禁止

// 使用 Structured Logging 的过滤器
builder.Services.AddLogging(builder =>
{
    builder.AddFilter("Microsoft", LogLevel.Warning);
    builder.AddFilter(log =>
    {
        // 过滤敏感信息
        if (log.State?.ToString()?.Contains("password") ?? false)
            return false;
        return true;
    });
});
```

## 性能注意事项

### AsNoTracking
```csharp
// 只读查询使用 AsNoTracking
var users = await _context.Users
    .AsNoTracking()
    .ToListAsync();
```

### 分页
```csharp
// 大数据量必须分页
var orders = await _context.Orders
    .AsNoTracking()
    .OrderByDescending(e => e.CreatedAt)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

### 批量操作
```csharp
// EF Core 8 批量更新
await _context.Users
    .Where(e => e.Status == UserStatus.Inactive)
    .ExecuteUpdateAsync(e => e.SetProperty(u => u.IsDeleted, true));

// EF Core 8 批量删除
await _context.Users
    .Where(e => e.IsDeleted)
    .ExecuteDeleteAsync();
```

### 连接池
```csharp
// 连接字符串配置
"Server=localhost;Database=mydb;Uid=root;Pwd=xxx;Pooling=true;MinimumPoolSize=5;MaximumPoolSize=100;"
```