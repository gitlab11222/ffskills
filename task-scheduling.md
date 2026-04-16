# 任务调度规范

## Hangfire 配置

### 安装与配置
```csharp
// NuGet 包
// Hangfire
// Hangfire.MySqlStorage (MySQL 存储)

// appsettings.json
{
  "Hangfire": {
    "ConnectionString": "Server=localhost;Database=mydb;Uid=root;Pwd=xxx;",
    "WorkerCount": 5
  }
}

// Program.cs
builder.Services.AddHangfire(configuration => configuration
    .UseStorage(new MySqlStorage(
        builder.Configuration["Hangfire:ConnectionString"],
        new MySqlStorageOptions
        {
            TransactionIsolationLevel = IsolationLevel.ReadCommitted,
            QueuePollInterval = TimeSpan.FromSeconds(15),
            JobExpirationCheckInterval = TimeSpan.FromHours(1),
            CountersAggregateInterval = TimeSpan.FromMinutes(5),
            PrepareSchemaIfNecessary = true,
            DashboardJobListLimit = 50000,
            TransactionTimeout = TimeSpan.FromMinutes(1),
            TablesPrefix = "Hangfire"
        })));

builder.Services.AddHangfireServer(options =>
{
    options.WorkerCount = builder.Configuration.GetValue<int>("Hangfire:WorkerCount", 5);
    options.Queues = new[] { "default", "critical", "normal" };
});

// Dashboard（可选）
app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    Authorization = new[] { new HangfireAuthorizationFilter() }
});
```

### 任务定义

#### 即时任务
```csharp
// 立即执行
BackgroundJob.Enqueue(() => SendEmailAsync(userId, "欢迎注册"));

// 带参数
BackgroundJob.Enqueue<IEmailService>(x => x.SendAsync(userId, "订单已创建"));
```

#### 延迟任务
```csharp
// 延迟执行（指定时间）
BackgroundJob.Schedule(() => CancelOrderAsync(orderId), TimeSpan.FromMinutes(30));

// 指定时间执行
var executeAt = DateTime.UtcNow.AddHours(1);
BackgroundJob.Schedule(() => SendReminderAsync(userId), executeAt);

// 通过服务接口
BackgroundJob.Schedule<IOrderService>(x => x.CancelTimeoutOrderAsync(orderId), TimeSpan.FromMinutes(30));
```

#### 定时任务（Recurring）
```csharp
// Cron 表达式
RecurringJob.AddOrUpdate("DailyReport", () => GenerateDailyReportAsync(), Cron.Daily(8, 0));
RecurringJob.AddOrUpdate("HourlyCheck", () => CheckPendingOrdersAsync(), Cron.Hourly(0));
RecurringJob.AddOrUpdate("WeeklyCleanup", () => CleanupOldDataAsync(), Cron.Weekly(DayOfWeek.Sunday, 2, 0));

// 自定义 Cron
RecurringJob.AddOrUpdate("CustomJob", () => DoWorkAsync(), "0 0 12 * * ?");  // 每天12点

// 通过服务接口
RecurringJob.AddOrUpdate<IReportService>("DailyReport", x => x.GenerateDailyAsync(), Cron.Daily(8));
```

#### 继续/批量任务
```csharp
// 继续 任务
var jobId = BackgroundJob.Enqueue(() => CreateOrderAsync(orderId));
BackgroundJob.ContinueWith(jobId, () => SendOrderNotificationAsync(orderId));

// 批量任务
var batchJobs = new List<Job>
{
    Job.FromMethod(() => ProcessOrder1Async()),
    Job.FromMethod(() => ProcessOrder2Async()),
    Job.FromMethod(() => ProcessOrder3Async())
};
var batchId = BatchJob.Start(batchJobs);
```

### 任务队列优先级
```csharp
// 定义队列
[Queue("critical")]
public async Task CriticalTaskAsync() { }

[Queue("normal")]
public async Task NormalTaskAsync() { }

// 任务放入指定队列
BackgroundJob.Enqueue(() => CriticalTaskAsync());  // critical 队列优先执行
```

### 任务服务定义
```csharp
/// <summary>任务调度服务接口</summary>
public interface IScheduledTaskService : IAutoDIScoped
{
    /// <summary>订单超时取消任务</summary>
    Task ScheduleOrderTimeoutCancelAsync(long orderId, TimeSpan delay);

    /// <summary>支付超时关闭任务</summary>
    Task SchedulePaymentTimeoutAsync(long paymentId, TimeSpan delay);

    /// <summary>发送通知任务</summary>
    Task ScheduleNotificationAsync(long notificationId, DateTime executeAt);
}

/// <summary>任务调度服务实现</summary>
public class ScheduledTaskService : IScheduledTaskService
{
    public async Task ScheduleOrderTimeoutCancelAsync(long orderId, TimeSpan delay)
    {
        BackgroundJob.Schedule<IOrderService>(
            x => x.CancelTimeoutOrderAsync(orderId),
            delay);
    }

    public async Task SchedulePaymentTimeoutAsync(long paymentId, TimeSpan delay)
    {
        BackgroundJob.Schedule<IPaymentService>(
            x => x.TimeoutCancelAsync(paymentId),
            delay);
    }

    public async Task ScheduleNotificationAsync(long notificationId, DateTime executeAt)
    {
        BackgroundJob.Schedule<INotificationService>(
            x => x.SendAsync(notificationId),
            executeAt);
    }
}
```

### 任务执行服务
```csharp
/// <summary>订单服务 - 任务执行方法</summary>
public class OrderService : IOrderService
{
    /// <summary>取消超时订单（由 Hangfire 调用）</summary>
    [AutomaticRetry(Attempts = 3, DelaysInSeconds = new[] { 60, 300, 900 })]
    public async Task CancelTimeoutOrderAsync(long orderId)
    {
        await using var lockResult = await _redisLock.WaitAsync(
            RedisKey.OrderLock + orderId,
            TimeSpan.FromSeconds(30));

        if (!lockResult.IsAcquired)
        {
            _logger.LogWarning("订单取消任务获取锁失败: OrderId={OrderId}", orderId);
            return;
        }

        var order = await _orderRepo.GetByIdAsync(orderId);
        if (order == null || order.Status != OrderStatus.Pending)
        {
            _logger.LogInformation("订单已处理，跳过取消: OrderId={OrderId}, Status={Status}",
                orderId, order?.Status);
            return;
        }

        order.Status = OrderStatus.Cancelled;
        order.CancelReason = "超时未支付";
        order.CancelledAt = DateTime.UtcNow;
        await _orderRepo.UpdateAsync(order);

        _logger.LogInformation("订单超时取消成功: OrderId={OrderId}", orderId);
    }
}
```

### 任务重试策略
```csharp
// 方法级重试配置
[AutomaticRetry(Attempts = 3)]  // 默认重试间隔: 1分钟, 2分钟, 4分钟...
public async Task ProcessOrderAsync(long orderId) { }

// 自定义重试间隔
[AutomaticRetry(Attempts = 5, DelaysInSeconds = new[] { 10, 30, 60, 300, 900 })]
public async Task CriticalProcessAsync(long id) { }

// 禁用重试
[AutomaticRetry(Attempts = 0)]
public async Task NoRetryTaskAsync() { }
```

### Dashboard 权限控制
```csharp
public class HangfireAuthorizationFilter : IDashboardAuthorizationFilter
{
    public bool Authorize(DashboardContext context)
    {
        var httpContext = context.GetHttpContext();

        // 仅管理员可访问
        var user = httpContext.User;
        return user.IsInRole("Admin");
    }
}

// 配置
app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    Authorization = new[] { new HangfireAuthorizationFilter() },
    DashboardTitle = "任务调度中心"
});
```

## Quartz.NET 配置（备选方案）

### 安装与配置
```csharp
// NuGet 包
// Quartz
// Quartz.Extensions.DependencyInjection
// Quartz.Serialization.Json

// appsettings.json
{
  "Quartz": {
    "quartz.scheduler.instanceName": "MyScheduler",
    "quartz.scheduler.instanceId": "AUTO",
    "quartz.threadPool.threadCount": 10,
    "quartz.jobStore.type": "Quartz.Impl.AdoJobStore.AdoJobStore",
    "quartz.jobStore.driverDelegateType": "Quartz.Impl.AdoJobStore.MySqlDelegate",
    "quartz.jobStore.dataSource": "myDs",
    "quartz.dataSource.myDs.connectionString": "Server=localhost;Database=mydb;Uid=root;Pwd=xxx;",
    "quartz.dataSource.myDs.provider": "MySql"
  }
}

// Program.cs
builder.Services.AddQuartz(q =>
{
    q.UseMicrosoftDependencyInjectionJobFactory();
    q.UseSimpleTypeLoader();
    q.UseJsonSerializer();

    // 配置存储
    q.UsePersistentStore(options =>
    {
        options.UseProperties = true;
        options.RetryInterval = TimeSpan.FromSeconds(15);
        options.UseMySqlConnector(provider =>
        {
            provider.ConnectionString = builder.Configuration["Quartz:quartz.dataSource.myDs.connectionString"];
            provider.TablePrefix = "QRTZ_";
        });
        options.UseJsonSerializer();
    });

    // 注册 Job
    q.ScheduleJob<DailyReportJob>(trigger => trigger
        .WithIdentity("DailyReportTrigger")
        .StartNow()
        .WithCronSchedule("0 0 8 * * ?"));  // 每天8点
});

builder.Services.AddQuartzHostedService(options =>
{
    options.WaitForJobsToComplete = true;
});
```

### Job 定义
```csharp
/// <summary>每日报表任务</summary>
[DisallowConcurrentExecution]  // 禁止并发执行
[PersistJobDataAfterExecution] // 执行后保存 JobDataMap
public class DailyReportJob : IJob
{
    private readonly IReportService _reportService;
    private readonly ILogger<DailyReportJob> _logger;

    public DailyReportJob(IReportService reportService, ILogger<DailyReportJob> logger)
    {
        _reportService = reportService;
        _logger = logger;
    }

    public async Task Execute(IJobExecutionContext context)
    {
        _logger.LogInformation("开始执行每日报表任务: {Time}", DateTime.UtcNow);

        try
        {
            await _reportService.GenerateDailyReportAsync();
            _logger.LogInformation("每日报表任务执行成功");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "每日报表任务执行失败");
            throw;  // Quartz 会自动重试
        }
    }
}
```

### 动态调度
```csharp
public class QuartzSchedulerService
{
    private readonly ISchedulerFactory _schedulerFactory;

    public async Task ScheduleJobAsync<TJob>(string jobName, string cronExpression) where TJob : IJob
    {
        var scheduler = await _schedulerFactory.GetScheduler();

        var job = JobBuilder.Create<TJob>()
            .WithIdentity(jobName)
            .Build();

        var trigger = TriggerBuilder.Create()
            .WithIdentity($"{jobName}_Trigger")
            .WithCronSchedule(cronExpression)
            .Build();

        await scheduler.ScheduleJob(job, trigger);
    }

    public async Task ScheduleDelayedJobAsync<TJob>(string jobName, TimeSpan delay, JobDataMap data) where TJob : IJob
    {
        var scheduler = await _schedulerFactory.GetScheduler();

        var job = JobBuilder.Create<TJob>()
            .WithIdentity(jobName)
            .UsingJobData(data)
            .Build();

        var trigger = TriggerBuilder.Create()
            .WithIdentity($"{jobName}_Trigger")
            .StartAt(DateTime.UtcNow.Add(delay))
            .Build();

        await scheduler.ScheduleJob(job, trigger);
    }
}
```

## 任务调度最佳实践

### 任务设计原则
```csharp
// 1. 任务方法必须幂等
[AutomaticRetry(Attempts = 3)]
public async Task CancelTimeoutOrderAsync(long orderId)
{
    // 检查是否已处理
    var order = await _orderRepo.GetByIdAsync(orderId);
    if (order?.Status != OrderStatus.Pending)
        return;  // 已处理，跳过

    // 处理...
}

// 2. 任务方法必须获取分布式锁
public async Task ProcessOrderAsync(long orderId)
{
    await using var lock = await _redisLock.WaitAsync(
        RedisKey.OrderLock + orderId,
        TimeSpan.FromSeconds(30));

    if (!lock.IsAcquired)
        throw new Exception("获取锁失败");  // Hangfire 会重试
}

// 3. 任务必须有详细日志
public async Task Execute(IJobExecutionContext context)
{
    _logger.LogInformation("任务开始: JobKey={JobKey}, FireTime={FireTime}",
        context.JobDetail.Key, context.FireTimeUtc);

    try
    {
        await DoWorkAsync();
        _logger.LogInformation("任务成功: JobKey={JobKey}", context.JobDetail.Key);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "任务失败: JobKey={JobKey}", context.JobDetail.Key);
        throw;
    }
}

// 4. 禁止并发执行（必要时）
[DisallowConcurrentExecution]  // Quartz
[Queue("critical")]  // Hangfire: 使用独立队列，WorkerCount=1
public async Task CriticalTaskAsync() { }
```

### 任务取消与更新
```csharp
// Hangfire 删除/更新任务
BackgroundJob.Delete(jobId);
RecurringJob.RemoveIfExists("DailyReport");
RecurringJob.AddOrUpdate("DailyReport", () => GenerateReportAsync(), Cron.Hourly());

// Quartz 删除/更新任务
var scheduler = await _schedulerFactory.GetScheduler();
await scheduler.DeleteJob(new JobKey(jobName));
await scheduler.RescheduleJob(new TriggerKey(triggerName), newTrigger);
```

### 任务监控
```csharp
// Hangfire Dashboard 内置监控
// 访问 /hangfire 查看：
// - 执行中的任务
// - 失败的任务
// - 待执行的任务
// - 定时任务列表

// 自定义监控（发送告警）
public class HangfireMonitorService
{
    public async Task CheckFailedJobsAsync()
    {
        using var connection = JobStorage.Current.GetConnection();
        var failedJobs = connection.GetFailedJobs(0, 100);

        if (failedJobs.Count > 10)
        {
            await SendAlertAsync($"Hangfire 失败任务过多: {failedJobs.Count}");
        }
    }
}
```

## 延时任务场景

### 常见延时任务
| 场景 | 延时时间 | 处理逻辑 |
|------|---------|---------|
| 订单超时取消 | 30分钟 | 取消订单、释放库存 |
| 支付超时关闭 | 15分钟 | 关闭支付、取消订单 |
| 活动开始提醒 | 根据活动时间 | 发送通知 |
| 催付提醒 | 5分钟 | 发送催付短信 |
| 自动评价 | 7天 | 自动好评 |
| 数据清理 | 每天2点 | 清理过期数据 |

### 延时任务服务封装
```csharp
/// <summary>延时任务服务</summary>
public interface IDelayTaskService
{
    /// <summary>订单超时取消</summary>
    Task ScheduleOrderTimeout(long orderId, TimeSpan delay = TimeSpan.FromMinutes(30));

    /// <summary>支付超时关闭</summary>
    Task SchedulePaymentTimeout(long paymentId, TimeSpan delay = TimeSpan.FromMinutes(15));

    /// <summary>发送延时通知</summary>
    Task ScheduleNotification(long userId, string content, DateTime executeAt);

    /// <summary>取消延时任务</summary>
    Task CancelTask(string taskId);
}

public class DelayTaskService : IDelayTaskService
{
    private readonly IScheduledTaskService _scheduledTask;

    public Task ScheduleOrderTimeout(long orderId, TimeSpan delay)
        => _scheduledTask.ScheduleOrderTimeoutCancelAsync(orderId, delay);

    public Task SchedulePaymentTimeout(long paymentId, TimeSpan delay)
        => _scheduledTask.SchedulePaymentTimeoutAsync(paymentId, delay);

    public Task ScheduleNotification(long userId, string content, DateTime executeAt)
        => BackgroundJob.Schedule<INotificationService>(
            x => x.SendAsync(userId, content),
            executeAt);

    public Task CancelTask(string taskId)
        => BackgroundJob.Delete(taskId);
}