# Redis 使用规范

## 基础配置

### 连接配置
```csharp
// appsettings.json
{
  "Redis": {
    "ConnectionString": "localhost:6379,password=xxx,defaultDatabase=0",
    "InstanceName": "MyApp:"
  }
}

// Program.cs
builder.Services.AddRedis(builder.Configuration.GetSection("Redis"));
```

### 客户端使用
```csharp
public class RedisService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IDatabase _db;

    public RedisService(IConnectionMultiplexer redis)
    {
        _redis = redis;
        _db = redis.GetDatabase();
    }
}
```

## 键命名规范

### 命名规则
```
{应用}:{模块}:{业务}:{标识}

例如：
myapp:user:info:12345         # 用户信息缓存
myapp:order:lock:12345        # 订单处理锁
myapp:order:queue:create      # 订单创建队列
myapp:session:token:abc123    # 会话Token
```

### 键前缀常量
```csharp
public static class RedisKey
{
    public const string Prefix = "myapp:";

    // 缓存
    public const string UserCache = Prefix + "user:info:";
    public const string ConfigCache = Prefix + "config:";
    public const string PermissionCache = Prefix + "permission:";

    // 分布式锁
    public const string OrderLock = Prefix + "order:lock:";
    public const string PaymentLock = Prefix + "payment:lock:";
    public const string InventoryLock = Prefix + "inventory:lock:";

    // 队列
    public const string OrderCreateQueue = Prefix + "queue:order:create";
    public const string OrderCallbackQueue = Prefix + "queue:order:callback";
    public const string PaymentCallbackQueue = Prefix + "queue:payment:callback";

    // 延迟队列
    public const string DelayQueue = Prefix + "delay:queue";
    public const string DelayQueueProcess = Prefix + "delay:queue:process";

    // 计数器
    public const string RequestCount = Prefix + "count:request:";
    public const string ApiLimitCount = Prefix + "limit:api:";
}
```

## 缓存使用

### 缓存策略

#### Cache-Aside（旁路缓存）
```csharp
public class UserService
{
    private readonly IDatabase _redis;
    private readonly IUserRepository _repo;

    public async Task<User?> GetUserAsync(long userId)
    {
        var cacheKey = RedisKey.UserCache + userId;

        // 1. 先查缓存
        var cached = await _redis.StringGetAsync(cacheKey);
        if (cached.HasValue)
            return JsonSerializer.Deserialize<User>(cached!);

        // 2. 查数据库
        var user = await _repo.GetByIdAsync(userId);
        if (user == null) return null;

        // 3. 写入缓存
        await _redis.StringSetAsync(
            cacheKey,
            JsonSerializer.Serialize(user),
            TimeSpan.FromMinutes(30));

        return user;
    }

    public async Task UpdateUserAsync(User user)
    {
        // 1. 更新数据库
        await _repo.UpdateAsync(user);

        // 2. 删除缓存（让下次查询重新加载）
        var cacheKey = RedisKey.UserCache + user.Id;
        await _redis.KeyDeleteAsync(cacheKey);
    }
}
```

#### Write-Through（穿透写入）
```csharp
public async Task CreateUserAsync(User user)
{
    var cacheKey = RedisKey.UserCache + user.Id;

    // 1. 写入缓存
    await _redis.StringSetAsync(
        cacheKey,
        JsonSerializer.Serialize(user),
        TimeSpan.FromMinutes(30));

    // 2. 写入数据库
    await _repo.AddAsync(user);
}
```

### 缓存过期
```csharp
// 绝对过期
await _redis.StringSetAsync(key, value, TimeSpan.FromMinutes(30));

// 滑动过期（需要额外实现）
await _redis.StringSetAsync(key, value);
await _redis.KeyExpireAsync(key, TimeSpan.FromMinutes(30), ExpireWhen.HasExpiry);

// 不设置过期（需手动管理）
await _redis.StringSetAsync(key, value);
```

### 缓存穿透防护
```csharp
public async Task<User?> GetUserAsync(long userId)
{
    var cacheKey = RedisKey.UserCache + userId;

    // 查缓存
    var cached = await _redis.StringGetAsync(cacheKey);
    if (cached.HasValue)
    {
        // 空值标记
        if (cached == "NULL") return null;
        return JsonSerializer.Deserialize<User>(cached!);
    }

    // 查数据库
    var user = await _repo.GetByIdAsync(userId);

    // 缓存空值（短时间，防止穿透）
    if (user == null)
    {
        await _redis.StringSetAsync(cacheKey, "NULL", TimeSpan.FromMinutes(5));
        return null;
    }

    // 缓存正常值
    await _redis.StringSetAsync(cacheKey, JsonSerializer.Serialize(user), TimeSpan.FromMinutes(30));
    return user;
}
```

### 缓存击穿防护（分布式锁）
```csharp
public async Task<User?> GetUserAsync(long userId)
{
    var cacheKey = RedisKey.UserCache + userId;

    // 查缓存
    var cached = await _redis.StringGetAsync(cacheKey);
    if (cached.HasValue && cached != "NULL")
        return JsonSerializer.Deserialize<User>(cached!);

    // 获取分布式锁（防止并发击穿）
    var lockKey = RedisKey.OrderLock + $"cache:{userId}";
    await using var lock = await _redisLock.WaitAsync(lockKey, TimeSpan.FromSeconds(10));

    if (!lock.IsAcquired)
    {
        // 等待其他线程加载完成后再次查询缓存
        await Task.Delay(100);
        return await GetUserAsync(userId);
    }

    // Double Check
    cached = await _redis.StringGetAsync(cacheKey);
    if (cached.HasValue && cached != "NULL")
        return JsonSerializer.Deserialize<User>(cached!);

    // 查数据库并缓存
    var user = await _repo.GetByIdAsync(userId);
    await _redis.StringSetAsync(
        cacheKey,
        user == null ? "NULL" : JsonSerializer.Serialize(user),
        user == null ? TimeSpan.FromMinutes(5) : TimeSpan.FromMinutes(30));

    return user;
}
```

## 分布式锁

### 标准用法
```csharp
public class OrderService
{
    private readonly IRedisLock _redisLock;

    public async Task<bool> ProcessOrderAsync(long orderId)
    {
        // 获取分布式锁（必须设置超时）
        await using var lockResult = await _redisLock.WaitAsync(
            RedisKey.OrderLock + orderId,
            TimeSpan.FromSeconds(30));  // 锁超时时间

        if (!lockResult.IsAcquired)
        {
            throw new BusinessException("操作频繁，请稍后重试");
        }

        // 执行业务逻辑
        // ...

        return true;
    }
}
```

### RedisLock 实现
```csharp
public interface IRedisLock
{
    Task<LockResult> WaitAsync(string key, TimeSpan expiry, TimeSpan? wait = null, TimeSpan? retry = null);
}

public class RedisLock : IRedisLock
{
    private readonly IDatabase _db;

    public async Task<LockResult> WaitAsync(string key, TimeSpan expiry, TimeSpan? wait = null, TimeSpan? retry = null)
    {
        wait ??= TimeSpan.FromSeconds(10);
        retry ??= TimeSpan.FromMilliseconds(50);

        var startTime = DateTime.UtcNow;
        var lockId = Guid.NewGuid().ToString();

        while (DateTime.UtcNow - startTime < wait)
        {
            // SET NX PX（原子操作）
            var acquired = await _db.StringSetAsync(
                key,
                lockId,
                expiry,
                When.NotExists,
                CommandFlags.None);

            if (acquired)
            {
                return new LockResult(key, lockId, true, _db);
            }

            await Task.Delay(retry.Value);
        }

        return new LockResult(key, lockId, false, _db);
    }
}

public class LockResult : IAsyncDisposable
{
    public string Key { get; }
    public string LockId { get; }
    public bool IsAcquired { get; }
    private readonly IDatabase _db;

    public async ValueTask DisposeAsync()
    {
        if (!IsAcquired) return;

        // Lua脚本保证原子释放（只释放自己持有的锁）
        var script = @"
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end";

        await _db.ScriptEvaluateAsync(script, new RedisKey[] { Key }, new RedisValue[] { LockId });
    }
}
```

### 锁使用注意事项
1. **必须设置超时**: 防止死锁
2. **必须释放锁**: 使用 `await using` 或手动释放
3. **原子释放**: 使用 Lua 脚本，只释放自己持有的锁
4. **锁粒度**: 范围越小越好，避免大范围锁

## 消息队列

### 队列常量定义
```csharp
public static class RedisKey
{
    public const string OrderCreateQueue = "queue:order:create";
    public const string OrderCallbackQueue = "queue:order:callback";
    public const string PaymentCallbackQueue = "queue:payment:callback";
    public const string NotificationQueue = "queue:notification";
}
```

### 生产者
```csharp
public class OrderProducer
{
    private readonly IDatabase _redis;

    public async Task EnqueueOrderAsync(OrderMessage message)
    {
        // LPUSH 入队（左入）
        await _redis.ListLeftPushAsync(
            RedisKey.OrderCreateQueue,
            JsonSerializer.Serialize(message));
    }
}
```

### 消费者
```csharp
[RedisConsumer(RedisKey.OrderCreateQueue, IntervalSeconds = 5)]
public class OrderCreateConsumer : IRedisConsumer
{
    private readonly IOrderService _orderService;

    public async Task ExecuteAsync(string message)
    {
        var orderMsg = JsonSerializer.Deserialize<OrderMessage>(message);
        if (orderMsg == null) return;

        // 处理订单
        await _orderService.ProcessOrderAsync(orderMsg.OrderId);
    }
}
```

### 消费者基类
```csharp
public interface IRedisConsumer
{
    Task ExecuteAsync(string message);
}

[AttributeUsage(AttributeTargets.Class)]
public class RedisConsumerAttribute : Attribute
{
    public string QueueName { get; }
    public int IntervalSeconds { get; } = 5;
    public int BatchSize { get; } = 10;

    public RedisConsumerAttribute(string queueName)
    {
        QueueName = queueName;
    }
}

// 消费者后台服务
public class RedisConsumerHostedService : BackgroundService
{
    private readonly IDatabase _redis;
    private readonly IEnumerable<IRedisConsumer> _consumers;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            foreach (var consumer in _consumers)
            {
                var attr = consumer.GetType().GetCustomAttribute<RedisConsumerAttribute>();
                if (attr == null) continue;

                // RPOP 出队（右出）
                var messages = await _redis.ListRightPopAsync(attr.QueueName, attr.BatchSize);

                foreach (var msg in messages)
                {
                    if (msg.HasValue)
                    {
                        await consumer.ExecuteAsync(msg!);
                    }
                }
            }

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}
```

## 延迟队列

### 实现方案

#### 方案一：ZSET + 定时扫描
```csharp
public class DelayQueueService
{
    private readonly IDatabase _redis;

    // 添加延迟任务
    public async Task AddDelayTaskAsync<T>(T data, TimeSpan delay)
    {
        var executeAt = DateTime.UtcNow.Add(delay);
        var score = executeAt.Ticks;

        await _redis.SortedSetAddAsync(
            RedisKey.DelayQueue,
            JsonSerializer.Serialize(new DelayTask<T>
            {
                Data = data,
                ExecuteAt = executeAt,
            }),
            score);
    }

    // 获取到期任务
    public async Task<List<string>> GetDueTasksAsync()
    {
        var now = DateTime.UtcNow.Ticks;

        // 查询 score <= now 的任务
        var tasks = await _redis.SortedSetRangeByScoreAsync(
            RedisKey.DelayQueue,
            start: 0,
            stop: now,
            take: 100);

        // 删除已取出的任务
        if (tasks.Any())
        {
            await _redis.SortedSetRemoveRangeByScoreAsync(
                RedisKey.DelayQueue,
                start: 0,
                stop: now);
        }

        return tasks.Select(t => (string)t!).ToList();
    }
}

public class DelayTask<T>
{
    public T Data { get; set; }
    public DateTime ExecuteAt { get; set; }
}
```

#### 方案二：Redis Keyspace Notifications
```csharp
// 配置 Redis 启用键过期通知
// redis.conf: notify-keyspace-events Ex

public class ExpiryNotificationService
{
    private readonly IConnectionMultiplexer _redis;

    public async Task SubscribeKeyExpiryAsync()
    {
        var subscriber = _redis.GetSubscriber();

        // 订阅键过期事件
        await subscriber.SubscribeAsync("__keyevent@0__:expired", (channel, key) =>
        {
            // 处理过期事件
            Console.WriteLine($"Key expired: {key}");
        });
    }

    // 设置带过期的事件键
    public async Task ScheduleTaskAsync(string taskId, TimeSpan delay)
    {
        var key = $"delay:task:{taskId}";
        await _redis.GetDatabase().StringSetAsync(key, taskId, delay);
    }
}
```

### 延迟队列消费者
```csharp
[RedisConsumer(RedisKey.DelayQueueProcess, IntervalSeconds = 5)]
public class DelayQueueConsumer : IRedisConsumer
{
    private readonly IDatabase _redis;
    private readonly IServiceProvider _services;

    public async Task ExecuteAsync(string message)
    {
        // 获取到期任务
        var dueTasks = await GetDueTasksAsync();

        foreach (var taskJson in dueTasks)
        {
            var task = JsonSerializer.Deserialize<DelayTaskMessage>(taskJson);
            if (task == null) continue;

            // 根据任务类型分发处理
            switch (task.Type)
            {
                case "OrderCancel":
                    var orderService = _services.GetRequiredService<IOrderService>();
                    await orderService.CancelOrderAsync(task.OrderId);
                    break;
                case "PaymentTimeout":
                    var paymentService = _services.GetRequiredService<IPaymentService>();
                    await paymentService.TimeoutCancelAsync(task.PaymentId);
                    break;
            }
        }
    }

    private async Task<List<string>> GetDueTasksAsync()
    {
        var now = DateTime.UtcNow.Ticks;
        var db = _redis;
        var tasks = await db.SortedSetRangeByScoreAsync(
            RedisKey.DelayQueue,
            start: 0,
            stop: now,
            take: 100);

        if (tasks.Any())
        {
            await db.SortedSetRemoveRangeByScoreAsync(
                RedisKey.DelayQueue,
                start: 0,
                stop: now);
        }

        return tasks.Select(t => (string)t!).ToList();
    }
}

public class DelayTaskMessage
{
    public string Type { get; set; }
    public long OrderId { get; set; }
    public long PaymentId { get; set; }
    public DateTime ExecuteAt { get; set; }
}
```

### 常见延迟任务场景
```csharp
// 订单超时取消（30分钟）
await _delayQueue.AddDelayTaskAsync(new DelayTaskMessage
{
    Type = "OrderCancel",
    OrderId = orderId,
}, TimeSpan.FromMinutes(30));

// 支付超时关闭（15分钟）
await _delayQueue.AddDelayTaskAsync(new DelayTaskMessage
{
    Type = "PaymentTimeout",
    PaymentId = paymentId,
}, TimeSpan.FromMinutes(15));

// 定时通知推送
await _delayQueue.AddDelayTaskAsync(new DelayTaskMessage
{
    Type = "Notification",
    NotificationId = notificationId,
}, TimeSpan.FromHours(1));
```

## 计数器与限流

### 计数器
```csharp
public class CounterService
{
    private readonly IDatabase _redis;

    // 增加计数
    public async Task<long> IncrementAsync(string key)
    {
        return await _redis.StringIncrementAsync(key);
    }

    // 设置计数并过期
    public async Task SetCountAsync(string key, long value, TimeSpan expiry)
    {
        await _redis.StringSetAsync(key, value, expiry);
    }

    // 获取计数
    public async Task<long> GetCountAsync(string key)
    {
        var value = await _redis.StringGetAsync(key);
        return value.HasValue ? (long)value : 0;
    }
}
```

### API 限流
```csharp
public class RateLimitService
{
    private readonly IDatabase _redis;

    // 滑动窗口限流
    public async Task<bool> IsAllowedAsync(string key, int limit, TimeSpan window)
    {
        var now = DateTime.UtcNow.Ticks;
        var windowStart = now - window.Ticks;

        // 1. 删除窗口外的记录
        await _redis.SortedSetRemoveRangeByScoreAsync(key, 0, windowStart);

        // 2. 获取当前窗口内的计数
        var count = await _redis.SortedSetLengthAsync(key);

        if (count >= limit)
            return false;

        // 3. 添加当前请求
        await _redis.SortedSetAddAsync(key, Guid.NewGuid().ToString(), now);
        await _redis.KeyExpireAsync(key, window);

        return true;
    }

    // 固定窗口限流（简单版）
    public async Task<bool> IsAllowedSimpleAsync(string key, int limit, TimeSpan window)
    {
        var count = await _redis.StringIncrementAsync(key);

        if (count == 1)
        {
            await _redis.KeyExpireAsync(key, window);
        }

        return count <= limit;
    }
}
```

## 最佳实践

### DO - 推荐做法
```csharp
// ✓ 合理设置过期时间
await _redis.StringSetAsync(key, value, TimeSpan.FromMinutes(30));

// ✓ 使用连接复用
builder.Services.AddSingleton<IConnectionMultiplexer>(redis);

// ✓ 键命名规范
var key = $"{RedisKey.Prefix}:user:{userId}";

// ✓ 异步操作
await _redis.StringGetAsync(key);

// ✓ 批量操作（Pipeline）
var batch = _redis.CreateBatch();
batch.StringSetAsync(key1, value1);
batch.StringSetAsync(key2, value2);
batch.Execute();
```

### DON'T - 禁止做法
```csharp
// ✗ 不设置过期时间（可能导致内存溢出）
await _redis.StringSetAsync(key, value);

// ✗ Big Key（单个值过大）
await _redis.StringSetAsync(key, bigJsonString); // > 10KB

// ✗ 阻塞操作
var result = _redis.StringGet(key); // 同步阻塞

// ✗ 在循环中单独操作
foreach (var item in items)
{
    await _redis.StringSetAsync(key + item.Id, item); // N次网络请求
}

// ✓ 批量操作
var batch = _redis.CreateBatch();
foreach (var item in items)
{
    batch.StringSetAsync(key + item.Id, item);
}
batch.Execute();
```