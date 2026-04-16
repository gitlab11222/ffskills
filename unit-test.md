# 单元测试规范

## 测试框架

### 测试工具
| 工具 | 用途 | 说明 |
|------|------|------|
| xUnit | 单元测试框架 | 推荐使用 |
| Moq | Mock框架 | 模拟依赖对象 |
| FluentAssertions | 断言库 | 流式断言语法 |
| TestContainers | 集成测试 | 容器化测试环境 |

### NuGet 包
```xml
<PackageReference Include="xunit" Version="2.5.0" />
<PackageReference Include="xunit.runner.visualstudio" Version="2.5.0" />
<PackageReference Include="Moq" Version="4.20.0" />
<PackageReference Include="FluentAssertions" Version="6.12.0" />
<PackageReference Include="TestContainers" Version="3.7.0" />
```

## 测试命名规范

### 方法命名
```csharp
// 格式: MethodName_Scenario_ExpectedResult
[Fact]
public void CreateOrder_库存充足_创建成功()
{
    // ...
}

[Fact]
public void CreateOrder_库存不足_返回错误()
{
    // ...
}

[Fact]
public void CreateOrder_数量为零_抛出异常()
{
    // ...
}
```

### 类命名
```csharp
// 格式: {被测类名}Tests
public class OrderServiceTests { }
public class UserServiceTests { }
public class PaymentServiceTests { }
```

### 文件结构
```
Tests/
├── Domain.Tests/           # Domain层测试
│   ├── Services/
│   │   ├── OrderServiceTests.cs
│   │   └── UserServiceTests.cs
│   └── Entities/
│   │   └── OrderTests.cs
├── WebApi.Tests/           # WebApi层测试
│   ├── Controllers/
│   │   └── UserControllerTests.cs
│   └── Middlewares/
│   │   └── ExceptionMiddlewareTests.cs
```

## 测试结构

### AAA模式
```csharp
[Fact]
public async Task GetOrder_订单存在_返回订单详情()
{
    // Arrange（准备）
    var orderId = 1L;
    var expectedOrder = new Order { Id = orderId, Status = OrderStatus.Pending };
    var mockRepo = new Mock<IOrderRepository>();
    mockRepo.Setup(x => x.GetByIdAsync(orderId))
        .ReturnsAsync(expectedOrder);
    var service = new OrderService(mockRepo.Object);

    // Act（执行）
    var result = await service.GetOrderAsync(orderId);

    // Assert（断言）
    result.Should().NotBeNull();
    result.Id.Should().Be(orderId);
    result.Status.Should().Be(OrderStatus.Pending);
}
```

### 构造函数测试
```csharp
[Fact]
public void 构造函数_依赖为空_抛出异常()
{
    // Arrange
    IOrderRepository? repo = null;

    // Act
    Action act = () => new OrderService(repo!);

    // Assert
    act.Should().Throw<ArgumentNullException>()
        .WithParameterName("repo");
}
```

## 测试覆盖要求

### 覆盖率标准
| 优先级 | 覆盖率 | 说明 |
|--------|--------|------|
| P0 核心逻辑 | 100% | 核心业务逻辑必须覆盖 |
| P1 重要逻辑 | ≥ 80% | 重要业务逻辑 |
| P2 一般逻辑 | ≥ 60% | 一般业务逻辑 |
| 工具类 | 100% | 工具方法必须覆盖 |

### 必测场景
| 场景 | 说明 |
|------|------|
| 正常流程 | 正常输入，正常输出 |
| 异常流程 | 异常输入，错误处理 |
| 边界值 | 最小值、最大值、临界值 |
| 空值/null | 参数为空处理 |
| 并发场景 | 多线程并发访问 |
| 重复操作 | 重复提交、幂等性 |

## Mock 规范

### Mock 创建
```csharp
// 使用 Moq 创建 Mock
var mockRepo = new Mock<IOrderRepository>();
var mockLogger = new Mock<ILogger<OrderService>>();

// Mock 方法返回
mockRepo.Setup(x => x.GetByIdAsync(1L))
    .ReturnsAsync(new Order { Id = 1L });

// Mock 方法抛出异常
mockRepo.Setup(x => x.GetByIdAsync(999L))
    .ThrowsAsync(new BusinessException("订单不存在"));

// Mock 验证调用
mockRepo.Verify(x => x.AddAsync(It.IsAny<Order>()), Times.Once);
mockRepo.Verify(x => x.UpdateAsync(It.Is<Order>(o => o.Status == OrderStatus.Cancelled)), Times.Once);
```

### Mock 禁止事项
```csharp
// ✗ 禁止 Mock 数据库（集成测试使用真实数据库）
var mockDbContext = new Mock<AppDbContext>();  // 禁止

// ✓ 集成测试使用 TestContainers
[Fact]
public async Task 集成测试_真实数据库()
{
    using var container = new MySqlContainer("mysql:8.0");
    await container.StartAsync();
    var connectionString = container.GetConnectionString();
    // 使用真实连接测试
}

// ✗ 禁止 Mock 静态方法
Mock.Setup(typeof(DateTime), "Now");  // 禁止

// ✓ 使用包装类注入
public interface IDateTimeProvider
{
    DateTime Now { get; }
}
public class DateTimeProvider : IDateTimeProvider
{
    public DateTime Now => DateTime.UtcNow;
}

// 测试时 Mock
var mockTime = new Mock<IDateTimeProvider>();
mockTime.Setup(x => x.Now).Returns(new DateTime(2024, 1, 1));
```

## 异步测试规范

### async/await 测试
```csharp
[Fact]
public async Task CreateOrder_正常创建_返回订单ID()
{
    // Act
    var orderId = await _service.CreateOrderAsync(request);

    // Assert
    orderId.Should().BeGreaterThan(0);
}

// 异步方法必须返回 Task，禁止 async void
[Fact]
public async Task AsyncVoid_禁止使用()  // 正确
{
    await Task.Delay(100);
}

[Fact]
public void AsyncVoid_禁止使用()  // ✗ 禁止 async void
{
    // 错误
}
```

### 超时设置
```csharp
[Fact(Timeout = 5000)]  // 5秒超时
public async Task 长时间操作_设置超时()
{
    await _service.ProcessLargeDataAsync();
}
```

## 参数化测试

### Theory + InlineData
```csharp
[Theory]
[InlineData(1, 100, true)]    // 正常数量
[InlineData(0, 100, false)]   // 数量为0
[InlineData(101, 100, false)] // 超出库存
[InlineData(-1, 100, false)]  // 负数
public void ValidateQuantity_各种数量_返回验证结果(int quantity, int stock, bool expected)
{
    var result = _service.ValidateQuantity(quantity, stock);
    result.Should().Be(expected);
}
```

### Theory + MemberData
```csharp
public static IEnumerable<object[]> OrderTestData =>
    new List<object[]>
    {
        new object[] { 1L, OrderStatus.Pending, true },
        new object[] { 1L, OrderStatus.Completed, false },
        new object[] { 999L, OrderStatus.Pending, false },
    };

[Theory]
[MemberData(nameof(OrderTestData))]
public void CancelOrder_各种状态_返回结果(long orderId, OrderStatus status, bool expected)
{
    // ...
}
```

### Theory + ClassData
```csharp
public class OrderTestDataClass : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { 1L, OrderStatus.Pending, true };
        yield return new object[] { 1L, OrderStatus.Completed, false };
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(OrderTestDataClass))]
public void CancelOrder_使用类数据(long orderId, OrderStatus status, bool expected)
{
    // ...
}
```

## 集成测试

### 数据库集成测试
```csharp
// 使用 TestContainers
public class OrderIntegrationTests : IAsyncLifetime
{
    private MySqlContainer _container;
    private AppDbContext _dbContext;

    public async Task InitializeAsync()
    {
        _container = new MySqlContainer("mysql:8.0");
        await _container.StartAsync();
        
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseMySql(_container.GetConnectionString(), ServerVersion.AutoDetect(_container.GetConnectionString()))
            .Options;
        _dbContext = new AppDbContext(options);
        await _dbContext.Database.EnsureCreatedAsync();
    }

    public async Task DisposeAsync()
    {
        await _dbContext.DisposeAsync();
        await _container.DisposeAsync();
    }

    [Fact]
    public async Task 创建订单_真实数据库_保存成功()
    {
        var order = new Order { UserId = 1L, Status = OrderStatus.Pending };
        _dbContext.Orders.Add(order);
        await _dbContext.SaveChangesAsync();

        var saved = await _dbContext.Orders.FindAsync(order.Id);
        saved.Should().NotBeNull();
        saved.Status.Should().Be(OrderStatus.Pending);
    }
}
```

### API 集成测试
```csharp
// 使用 WebApplicationFactory
public class UserApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;

    public UserApiTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task 获取用户列表_返回分页数据()
    {
        var response = await _client.PostAsJsonAsync("/api/user/list", new { page = 1, pageSize = 10 });
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        var result = await response.Content.ReadFromJsonAsync<DataResponse<PagedList<UserDto>>>();
        result.Code.Should().Be(0);
        result.Data.List.Should().NotBeEmpty();
    }
}
```

## 测试数据准备

### 测试数据工厂
```csharp
public class OrderTestDataFactory
{
    public static Order CreatePendingOrder(long userId = 1L)
    {
        return new Order
        {
            UserId = userId,
            Status = OrderStatus.Pending,
            TotalAmount = 100.00m,
            CreatedAt = DateTime.UtcNow
        };
    }

    public static Order CreateCompletedOrder(long userId = 1L)
    {
        return new Order
        {
            UserId = userId,
            Status = OrderStatus.Completed,
            TotalAmount = 100.00m,
            CreatedAt = DateTime.UtcNow,
            CompletedAt = DateTime.UtcNow
        };
    }

    public static List<Order> CreateOrders(int count)
    {
        return Enumerable.Range(1, count)
            .Select(i => CreatePendingOrder(i))
            .ToList();
    }
}

// 使用
[Fact]
public void 测试使用工厂数据()
{
    var order = OrderTestDataFactory.CreatePendingOrder();
    // ...
}
```

## 断言规范

### FluentAssertions 使用
```csharp
// 基本断言
result.Should().NotBeNull();
result.Id.Should().Be(1);
result.Name.Should().Be("张三");
result.Status.Should().Be(OrderStatus.Pending);

// 字符串断言
result.UserName.Should().StartWith("张");
result.Email.Should().EndWith("@example.com");
result.Phone.Should().MatchRegex("^1[3-9]\\d{9}$");

// 数值断言
result.Amount.Should().BeGreaterThan(0);
result.Amount.Should().BeLessThan(1000);
result.Amount.Should().BeApproximately(100.00m, 0.01m);

// 集合断言
result.List.Should().NotBeEmpty();
result.List.Should().HaveCount(10);
result.List.Should().Contain(o => o.Id == 1);
result.List.Should().BeInAscendingOrder(o => o.CreatedAt);

// 异常断言
Action act = () => _service.CreateOrderAsync(nullRequest);
act.Should().Throw<ArgumentNullException>();
act.Should().Throw<BusinessException>()
    .WithMessage("库存不足");

// 时间断言
result.CreatedAt.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromSeconds(1));
result.CreatedAt.Should().BeAfter(new DateTime(2024, 1, 1));
```

## 测试执行

### 本地执行
```bash
# 执行所有测试
dotnet test

# 执行指定项目
dotnet test Tests/Domain.Tests

# 执行指定类
dotnet test --filter "FullyQualifiedName~OrderServiceTests"

# 执行指定方法
dotnet test --filter "FullyQualifiedName~OrderServiceTests.CreateOrder_正常创建_返回订单ID"

# 生成覆盖率报告
dotnet test --collect:"XPlat Code Coverage"
```

### CI 集成
```yaml
# GitLab CI
test:
  stage: test
  script:
    - dotnet restore
    - dotnet build
    - dotnet test --no-build --collect:"XPlat Code Coverage"
  coverage: '/Total Coverage: \d+\.\d+/'
```

## DO - 推荐做法

```csharp
// ✓ 测试方法命名清晰
[Fact]
public void CreateOrder_库存不足_返回错误() { }

// ✓ 使用 AAA 模式
[Fact]
public void Test()
{
    // Arrange
    var service = new OrderService();
    
    // Act
    var result = service.Create(request);
    
    // Assert
    result.Should().NotBeNull();
}

// ✓ 使用测试数据工厂
var order = OrderTestDataFactory.CreatePendingOrder();

// ✓ 使用 FluentAssertions
result.Should().NotBeNull();
result.Id.Should().BeGreaterThan(0);

// ✓ 集成测试使用真实数据库
using var container = new MySqlContainer("mysql:8.0");
```

## DON'T - 禁止做法

```csharp
// ✗ 测试命名不清晰
[Fact]
public void Test1() { }  // 禁止

// ✓ 正确做法
[Fact]
public void CreateOrder_库存不足_返回错误() { }

// ✗ Mock 数据库
var mockDb = new Mock<AppDbContext>();  // 禁止

// ✓ 集成测试使用真实数据库
using var container = new MySqlContainer();

// ✗ 测试无断言
[Fact]
public void Test()
{
    _service.CreateOrder(request);  // 无断言，禁止
}

// ✓ 每个测试都有断言
[Fact]
public void Test()
{
    var result = _service.CreateOrder(request);
    result.Should().NotBeNull();
}

// ✗ 测试有多个断言场景
[Fact]
public void 测试所有场景()
{
    // 测试正常
    // 测试异常
    // 测试边界
    // 一个测试多个场景，禁止
}

// ✓ 每个场景独立测试
[Fact]
public void CreateOrder_正常创建_返回成功() { }

[Fact]
public void CreateOrder_库存不足_返回错误() { }

[Fact]
public void CreateOrder_数量为零_抛出异常() { }
```

## 最佳实践

### 测试隔离
- 每个测试独立，不依赖其他测试
- 测试顺序不影响结果
- 测试后清理数据

### 测试速度
- 单元测试 < 100ms
- 集成测试 < 1s
- 使用并行测试提高效率

### 测试维护
- 定期重构测试代码
- 删除过时测试
- 更新测试数据

### 覆盖率监控
- CI 自动生成覆盖率报告
- 覆盖率阈值检查
- 低覆盖率告警