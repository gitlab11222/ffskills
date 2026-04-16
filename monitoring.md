# 监控告警规范

## 监控架构

### 监控体系
```
应用层 → 中间件层 → 基础设施层
   ↓         ↓           ↓
指标采集 → 数据存储 → 可视化展示 → 告警通知
```

### 监控组件
| 组件 | 用途 | 推荐工具 |
|------|------|----------|
| 指标采集 | 性能指标、业务指标 | Prometheus Agent |
| 日志采集 | 应用日志、系统日志 | Serilog + 阿里云SLS |
| APM | 应用性能监控 | SkyWalking / Elastic APM |
| 可视化 | 监控面板展示 | Grafana |
| 告警 | 告警通知 | Prometheus AlertManager + 钉钉 |

## 应用监控

### 关键指标
| 指标类型 | 指标名称 | 说明 | 告警阈值 |
|----------|----------|------|----------|
| 性能指标 | CPU使用率 | CPU占用百分比 | > 80% 告警，> 90% 紧急 |
| 性能指标 | 内存使用率 | 内存占用百分比 | > 80% 告警，> 90% 紧急 |
| 性能指标 | GC频率 | GC执行频率 | > 10次/分钟 告警 |
| 性能指标 | 线程数 | 当前线程数量 | > 200 告警 |
| 接口指标 | 请求量 QPS | 每秒请求数 | 骤降50% 告警 |
| 接口指标 | 响应时间 RT | 平均响应时间 | > 500ms 告警 |
| 接口指标 | 错误率 | HTTP错误比例 | > 1% 告警，> 5% 紧急 |
| 接口指标 | 慢请求 | 响应时间 > 1s | 每分钟 > 10次 告警 |

### Serilog 配置
```csharp
// Program.cs
builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .Enrich.WithProperty("Application", "MyApp")
    .Enrich.WithProperty("Environment", context.HostingEnvironment.EnvironmentName)
    .WriteTo.Console()
    .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day)
    // Prometheus 指标导出
    .WriteTo.Prometheus(config =>
    {
        config.Path = "/metrics";
        config.Port = 80;
    })
    // 阿里云 SLS
    .WriteTo.AliyunSls(
        endpoint: context.Configuration["Sls:Endpoint"],
        accessKeyId: context.Configuration["Sls:AccessKeyId"],
        accessKeySecret: context.Configuration["Sls:AccessKeySecret"],
        project: context.Configuration["Sls:Project"],
        logStore: context.Configuration["Sls:LogStore"])
);

// 添加 Prometheus 端点
app.MapPrometheusScrapingEndpoint("/metrics");
```

### 结构化日志
```csharp
// 业务日志模板
_logger.LogInformation("订单创建: OrderId={OrderId}, UserId={UserId}, Amount={Amount}, Status={Status}",
    order.Id, order.UserId, order.TotalAmount, order.Status);

// 错误日志模板
_logger.LogError(ex, "订单处理失败: OrderId={OrderId}, Error={Error}",
    orderId, ex.Message);

// 性能日志模板
_logger.LogInformation("接口性能: Method={Method}, Path={Path}, Duration={Duration}ms, StatusCode={StatusCode}",
    context.Request.Method, context.Request.Path, duration, context.Response.StatusCode);
```

## 中间件监控

### MySQL 监控指标
| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| 连接数 | 当前连接数 | > 80% 最大连接数 告警 |
| QPS | 每秒查询数 | > 5000 告警 |
| 慢查询 | 执行时间 > 500ms | > 10次/分钟 告警 |
| 死锁 | 死锁次数 | > 1次/小时 告警 |
| 主从延迟 | 主从同步延迟 | > 1s 告警 |
| 表空间 | 表空间使用率 | > 80% 告警 |

### Redis 监控指标
| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| 内存使用率 | 内存占用百分比 | > 80% 告警 |
| 连接数 | 当前连接数 | > 1000 告警 |
| 命令执行数 | 每秒命令数 | > 10000 告警 |
| 键命中率 | 缓存命中率 | < 80% 告警 |
| 阻塞命令 | 阻塞命令数 | > 5 告警 |
| 慢日志 | 执行时间 > 10ms | > 10次/分钟 告警 |

## 基础设施监控

### 服务器指标
| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| CPU使用率 | CPU占用 | > 80% 告警 |
| 内存使用率 | 内存占用 | > 85% 告警 |
| 磁盘使用率 | 磁盘占用 | > 85% 告警 |
| 网络带宽 | 入/出流量 | > 80% 告警 |
| TCP连接数 | TCP连接数 | > 500 告警 |
| 进程数 | 运行进程数 | 异常增长 告警 |

### 容器/K8s指标
| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| Pod状态 | Running/Failed | Failed 告警 |
| 容器重启数 | 重启次数 | > 3次/小时 告警 |
| Pod CPU | Pod CPU使用 | > 80% 告警 |
| Pod 内存 | Pod 内存使用 | > 80% 告警 |
| PVC使用率 | 存储使用 | > 85% 告警 |
| Node状态 | Node Ready | Not Ready 告警 |

## 业务监控

### 业务指标定义
| 指标 | 定义 | 计算方式 | 告警阈值 |
|------|------|----------|----------|
| 订单量 | 每分钟创建订单数 | COUNT(*) | 骤降50% 告警 |
| 支付成功率 | 支付成功/支付请求 | 成功数/总数 | < 95% 告警 |
| 转化率 | 下单用户/访问用户 | 下单数/访客数 | < 5% 告警 |
| 客单价 | 平均订单金额 | 总金额/订单数 | 骤降20% 告警 |
| 用户活跃数 | 每小时活跃用户 | COUNT(DISTINCT user_id) | 骤降30% 告警 |

### 业务指标上报
```csharp
// Prometheus 业务指标定义
public class BusinessMetrics
{
    private static readonly Counter OrderCreated = Metrics.CreateCounter(
        "order_created_total",
        "订单创建总数",
        "status", "channel");

    private static readonly Gauge OrderAmount = Metrics.CreateGauge(
        "order_amount_sum",
        "订单金额总计",
        "currency");

    private static readonly Counter PaymentProcessed = Metrics.CreateCounter(
        "payment_processed_total",
        "支付处理总数",
        "status", "method");

    public static void RecordOrderCreated(string status, string channel)
    {
        OrderCreated.WithLabels(status, channel).Inc();
    }

    public static void RecordPaymentProcessed(string status, string method)
    {
        PaymentProcessed.WithLabels(status, method).Inc();
    }
}

// 使用
BusinessMetrics.RecordOrderCreated("success", "mobile");
BusinessMetrics.RecordPaymentProcessed("success", "alipay");
```

## 告警配置

### 告警级别定义
| 级别 | 定义 | 响应时间 | 通知渠道 |
|------|------|----------|----------|
| 🔴 P0 紧急 | 服务不可用、核心功能故障 | 5分钟 | 电话 + 钉钉 + 邮件 |
| 🟡 P1 严重 | 性能严重下降、部分功能异常 | 30分钟 | 钉钉 + 邮件 |
| 🟢 P2 一般 | 性能轻微下降、非核心问题 | 4小时 | 邮件 |

### Prometheus AlertRules
```yaml
# alert-rules.yml
groups:
- name: application-alerts
  rules:
  # CPU告警
  - alert: HighCPUUsage
    expr: process_cpu_seconds_total > 0.8
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "CPU使用率过高"
      description: "服务 {{ $labels.service }} CPU使用率 > 80%，当前值: {{ $value }}"

  # 内存告警
  - alert: HighMemoryUsage
    expr: process_resident_memory_bytes / process_virtual_memory_bytes > 0.85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "内存使用率过高"
      description: "服务 {{ $labels.service }} 内存使用率 > 85%"

  # 错误率告警
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "错误率过高"
      description: "服务 {{ $labels.service }} 错误率 > 5%，当前值: {{ $value }}"

  # 响应时间告警
  - alert: SlowResponse
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 0.5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "响应时间过长"
      description: "服务 {{ $labels.service }} P95响应时间 > 500ms"

- name: database-alerts
  rules:
  # MySQL连接数告警
  - alert: MySQLHighConnections
    expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections > 0.8
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "MySQL连接数过高"
      description: "MySQL连接使用率 > 80%"

  # Redis内存告警
  - alert: RedisHighMemory
    expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.8
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Redis内存过高"
      description: "Redis内存使用率 > 80%"
```

### AlertManager 配置
```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  routes:
  - match:
      severity: critical
    receiver: 'critical'
  - match:
      severity: warning
    receiver: 'warning'

receivers:
- name: 'default'
  email_configs:
  - to: 'team@example.com'

- name: 'critical'
  email_configs:
  - to: 'team@example.com'
  webhook_configs:
  - url: 'https://oapi.dingtalk.com/robot/send?access_token=xxx'
    send_resolved: true

- name: 'warning'
  email_configs:
  - to: 'team@example.com'
  webhook_configs:
  - url: 'https://oapi.dingtalk.com/robot/send?access_token=xxx'
```

### 钉钉告警模板
```csharp
public class DingTalkAlertService
{
    private readonly string _webhook;
    private readonly HttpClient _httpClient;

    public async Task SendAlertAsync(AlertLevel level, string title, string content)
    {
        var message = level switch
        {
            AlertLevel.Critical => $"🔴 **紧急告警**\n\n**{title}**\n\n{content}\n\n⏰ {DateTime.Now:yyyy-MM-dd HH:mm:ss}",
            AlertLevel.Warning => $"🟡 **严重告警**\n\n**{title}**\n\n{content}\n\n⏰ {DateTime.Now:yyyy-MM-dd HH:mm:ss}",
            AlertLevel.Info => $"🟢 **一般告警**\n\n**{title}**\n\n{content}\n\n⏰ {DateTime.Now:yyyy-MM-dd HH:mm:ss}",
            _ => content
        };

        var payload = new
        {
            msgtype = "text",
            text = new { content = message }
        };

        await _httpClient.PostAsJsonAsync(_webhook, payload);
    }
}

// 告警级别
public enum AlertLevel
{
    Critical,  // 紧急
    Warning,   // 严重
    Info       // 一般
}
```

## 告警处理流程

### 告警响应流程
```
告警触发 → 通知发送 → 值班人员响应 → 问题排查 → 问题处理 → 告警恢复
```

### 响应时间要求
| 级别 | 响应时间 | 处理时间 | 说明 |
|------|----------|----------|------|
| 🔴 P0 紧急 | 5分钟 | 30分钟内解决或升级 | 核心功能不可用 |
| 🟡 P1 严重 | 30分钟 | 4小时内解决 | 重要功能异常 |
| 🟢 P2 一般 | 4小时 | 24小时内解决 | 非核心问题 |

### 告警记录模板
```markdown
## 告警记录：{告警编号}

### 基本信息
- **告警时间**: {YYYY-MM-DD HH:mm:ss}
- **告警级别**: 🔴/🟡/🟢
- **告警内容**: {内容}
- **影响范围**: {范围}
- **值班人员**: {姓名}

### 处理过程
| 时间 | 操作 | 结果 | 操作人 |
|------|------|------|--------|
| {时间} | {操作} | {结果} | {姓名} |

### 原因分析
{根因分析}

### 解决方案
{解决方案}

### 预防措施
{预防措施}

### 告警恢复
- **恢复时间**: {YYYY-MM-DD HH:mm:ss}
- **恢复确认**: {确认人}
```

## Grafana 面板配置

### 应用监控面板
```yaml
# Grafana Dashboard JSON
{
  "title": "应用监控面板",
  "panels": [
    {
      "title": "CPU使用率",
      "type": "gauge",
      "targets": [
        {
          "expr": "process_cpu_seconds_total",
          "legendFormat": "{{service}}"
        }
      ],
      "thresholds": [
        { "value": 0, "color": "green" },
        { "value": 80, "color": "yellow" },
        { "value": 90, "color": "red" }
      ]
    },
    {
      "title": "请求量 QPS",
      "type": "graph",
      "targets": [
        {
          "expr": "rate(http_requests_total[5m])",
          "legendFormat": "{{service}}"
        }
      ]
    },
    {
      "title": "响应时间 P95",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
          "legendFormat": "{{service}}"
        }
      ]
    },
    {
      "title": "错误率",
      "type": "graph",
      "targets": [
        {
          "expr": "rate(http_requests_total{status=~\"5..\"}[5m]) / rate(http_requests_total[5m])",
          "legendFormat": "{{service}}"
        }
      ]
    }
  ]
}
```

### 业务监控面板
```yaml
{
  "title": "业务监控面板",
  "panels": [
    {
      "title": "订单量",
      "type": "graph",
      "targets": [
        {
          "expr": "rate(order_created_total[1m])",
          "legendFormat": "{{status}}"
        }
      ]
    },
    {
      "title": "支付成功率",
      "type": "gauge",
      "targets": [
        {
          "expr": "sum(payment_processed_total{status=\"success\"}) / sum(payment_processed_total)",
          "legendFormat": "成功率"
        }
      ]
    },
    {
      "title": "用户活跃数",
      "type": "graph",
      "targets": [
        {
          "expr": "user_active_count",
          "legendFormat": "活跃用户"
        }
      ]
    }
  ]
}
```

## 监控巡检

### 每日巡检清单
```markdown
# 监控每日巡检

## 应用层
- [ ] CPU使用率 < 80%
- [ ] 内存使用率 < 80%
- [ ] 错误率 < 1%
- [ ] 响应时间 P95 < 500ms
- [ ] 无异常错误日志

## 中间件层
- [ ] MySQL连接数正常
- [ ] Redis内存使用 < 80%
- [ ] 无慢查询告警

## 基础设施层
- [ ] 服务器资源充足
- [ ] 网络带宽正常
- [ ] 存储空间充足

## 业务指标
- [ ] 订单量正常
- [ ] 支付成功率 > 95%
- [ ] 用户活跃数正常
```

## DO - 推荐做法

```csharp
// ✓ 结构化日志包含关键信息
_logger.LogInformation("订单创建: OrderId={OrderId}, UserId={UserId}", orderId, userId);

// ✓ 业务指标上报
BusinessMetrics.RecordOrderCreated("success", "mobile");

// ✓ 告警分级处理
if (errorRate > 0.05)
    await SendCriticalAlert("错误率过高", $"当前值: {errorRate}");

// ✓ 监控面板关键指标
CPU、内存、QPS、响应时间、错误率

// ✓ 告警有响应记录
| 时间 | 操作 | 结果 | 操作人 |
```

## DON'T - 禁止做法

```csharp
// ✗ 日志无关键信息
_logger.LogInformation("订单创建");  // 禁止

// ✓ 正确做法
_logger.LogInformation("订单创建: OrderId={OrderId}", orderId);

// ✗ 告警无分级
await SendAlert("有问题");  // 禁止

// ✓ 正确做法
await SendCriticalAlert("错误率过高", $"当前值: {errorRate}");

// ✗ 告警阈值过高（无告警）
threshold: 99%  // 禁止，阈值应合理

// ✓ 正确做法
threshold: 80% warning, 90% critical

// ✗ 告警无响应流程
告警后无处理  // 禁止

// ✓ 正确做法
告警 → 响应 → 处理 → 记录 → 预防

// ✗ 监控面板指标过多
100个指标面板  // 禁止，关注核心指标

// ✓ 正确做法
10-20个核心指标面板
```

## 最佳实践

### 监控金字塔
1. **基础设施监控**: 服务器、网络、存储
2. **中间件监控**: MySQL、Redis、消息队列
3. **应用监控**: CPU、内存、QPS、响应时间、错误率
4. **业务监控**: 订单量、转化率、支付成功率

### 告警收敛
- 相同告警合并，避免告警风暴
- 告警静默期设置，避免重复告警
- 告警分组，批量处理

### 监控文档化
- 监控指标说明文档
- 告警处理手册
- 监控面板使用指南