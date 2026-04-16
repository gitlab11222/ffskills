# 部署与构建规则

## Dockerfile 标准

### 后端 .NET 8 Dockerfile
```dockerfile
# 3阶段构建
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# 复制项目文件
COPY ["MyApp.WebApi/MyApp.WebApi.csproj", "MyApp.WebApi/"]
COPY ["MyApp.Domain/MyApp.Domain.csproj", "MyApp.Domain/"]

# 还原依赖
RUN dotnet restore "MyApp.WebApi/MyApp.WebApi.csproj"

# 复制源码
COPY . .

# 构建
WORKDIR "/src/MyApp.WebApi"
RUN dotnet build "MyApp.WebApi.csproj" -c Release -o /app/build

# Stage 2: Publish
FROM build AS publish
RUN dotnet publish "MyApp.WebApi.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Stage 3: Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app

# 设置时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 创建日志目录
RUN mkdir -p /app/logs

# 暴露端口
EXPOSE 80
EXPOSE 443

# 复制发布文件
COPY --from=publish /app/publish .

# 入口
ENTRYPOINT ["dotnet", "MyApp.WebApi.dll"]
```

### 前端 React Dockerfile
```dockerfile
# Stage 1: Build
FROM node:18-alpine AS build
WORKDIR /app

# 安装依赖
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm && pnpm install --frozen-lockfile

# 复制源码
COPY . .

# 构建
RUN pnpm build

# Stage 2: Runtime (Nginx)
FROM nginx:alpine AS final
WORKDIR /usr/share/nginx/html

# 复制构建产物
COPY --from=build /app/dist .

# 复制 Nginx 配置
COPY nginx.conf /etc/nginx/nginx.conf

# 复制运行时配置
COPY configs.js /usr/share/nginx/html/configs.js

# 暴露端口
EXPOSE 80

# 入口
CMD ["nginx", "-g", "daemon off;"]
```

### Nginx 配置
```nginx
# nginx.conf
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    gzip  on;
    gzip_min_length 1k;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    server {
        listen       80;
        server_name  localhost;

        root   /usr/share/nginx/html;
        index  index.html index.htm;

        # SPA 路由
        location / {
            try_files $uri $uri/ /index.html;
        }

        # API 反向代理
        location /api {
            proxy_pass http://backend:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # 健康检查
        location /hc {
            return 200 'OK';
            add_header Content-Type text/plain;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

## CI/CD 配置

### GitLab CI 环境映射
```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_REGISTRY: registry.example.com
  IMAGE_NAME: $CI_REGISTRY/$CI_PROJECT_PATH

# 后端构建
backend-build:
  stage: build
  image: mcr.microsoft.com/dotnet/sdk:8.0
  script:
    - dotnet restore
    - dotnet build -c Release
    - dotnet test --no-build -c Release
    - dotnet publish -c Release -o publish
  artifacts:
    paths:
      - publish/

# 前端构建
frontend-build:
  stage: build
  image: node:18-alpine
  script:
    - npm install -g pnpm
    - pnpm install --frozen-lockfile
    - pnpm build
  artifacts:
    paths:
      - dist/

# 部署到开发环境
deploy-dev:
  stage: deploy
  only:
    - develop
  script:
    - docker build -t $IMAGE_NAME:dev -f Dockerfile.backend .
    - docker push $IMAGE_NAME:dev
    - kubectl apply -f k8s/dev/

# 部署到测试环境
deploy-test:
  stage: deploy
  only:
    - /^release\/.*$/
  script:
    - docker build -t $IMAGE_NAME:test -f Dockerfile.backend .
    - docker push $IMAGE_NAME:test
    - kubectl apply -f k8s/test/

# 部署到生产环境
deploy-prod:
  stage: deploy
  only:
    - /^rel-.*$/
  when: manual
  script:
    - docker build -t $IMAGE_NAME:$CI_COMMIT_TAG -f Dockerfile.backend .
    - docker push $IMAGE_NAME:$CI_COMMIT_TAG
    - kubectl apply -f k8s/prod/
```

### 环境分支映射规则
| 分支/标签 | 环境 | 说明 |
|----------|------|------|
| develop | dev | 开发环境，自动部署 |
| release/* | test | 测试环境，自动部署 |
| beta/* | beta/it | 集成测试环境 |
| master + hotfix/* | pre | 预发布环境 |
| rel-* 标签 | prod | 生产环境，手动触发 |

## NuGet 版本管理

### Directory.Build.props
```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <LangVersion>12</LangVersion>

    <!-- NuGet 包统一版本 -->
    <FLNextCoreVersion>1.0.0</FLNextCoreVersion>
    <EFCoreVersion>8.0.0</EFCoreVersion>
    <PomeloVersion>8.0.0</PomeloVersion>
    <SerilogVersion>3.1.1</SerilogVersion>
    <HangfireVersion>1.8.6</HangfireVersion>
    <AutoMapperVersion>12.0.1</AutoMapperVersion>
  </PropertyGroup>
</Project>
```

### 项目引用
```xml
<!-- Domain.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="$(EFCoreVersion)" />
    <PackageReference Include="Pomelo.EntityFrameworkCore.MySql" Version="$(PomeloVersion)" />
    <PackageReference Include="AutoMapper" Version="$(AutoMapperVersion)" />
  </ItemGroup>
</Project>
```

## 前端构建

### 构建命令
```bash
# 安装依赖
pnpm install --frozen-lockfile

# 开发模式
pnpm dev

# 构建
pnpm build

# 预览构建产物
pnpm preview
```

### 运行时配置注入
```html
<!-- public/configs.js -->
window.appConfigs = {
  apiBaseURL: '/api',
  wsURL: 'ws://localhost:8080',
  environment: 'development'
};
```

```html
<!-- public/index.html -->
<!DOCTYPE html>
<html>
<head>
  <script src="/configs.js"></script>
</head>
<body>
  <div id="root"></div>
</body>
</html>
```

```typescript
// src/utils/config.ts
declare global {
  interface Window {
    appConfigs: {
      apiBaseURL: string;
      wsURL: string;
      environment: string;
    };
  }
}

export const config = window.appConfigs;
```

## 健康检查

### 后端健康检查
```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddMySql(connectionString, name: "mysql")
    .AddRedis(redisConnectionString, name: "redis");

app.MapHealthChecks("/hc", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var response = JsonSerializer.Serialize(new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                duration = e.Value.Duration.TotalMilliseconds
            })
        });
        await context.Response.WriteAsync(response);
    }
});
```

### 前端健康检查
```nginx
# Nginx 健康检查端点
location /hc {
    return 200 'OK';
    add_header Content-Type text/plain;
}
```

## 配置管理

### 后端 appsettings.json
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "ConnectionStrings": {
    "Default": "Server=localhost;Port=3306;Database=mydb;Uid=root;Pwd=xxx;",
    "Redis": "localhost:6379,password=xxx,defaultDatabase=0"
  },
  "Jwt": {
    "Secret": "your-secret-key",
    "Issuer": "myapp",
    "Audience": "myapp",
    "ExpireMinutes": 1440
  },
  "Hangfire": {
    "WorkerCount": 5
  },
  "Redis": {
    "InstanceName": "MyApp:"
  },
  "Serilog": {
    "WriteTo": [
      { "Name": "Console" },
      { "Name": "File", "Args": { "path": "logs/log-.txt", "rollingInterval": "Day" } }
    ]
  }
}
```

### 环境配置覆盖
```json
// appsettings.Development.json (本地开发)
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Port=3306;Database=mydb_dev;Uid=root;Pwd=xxx;"
  }
}

// appsettings.Test.json (测试环境)
{
  "ConnectionStrings": {
    "Default": "Server=test-db;Port=3306;Database=mydb_test;Uid=root;Pwd=xxx;"
  }
}

// appsettings.Production.json (生产环境)
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  }
}
```

### 前端 configs.js
```javascript
// configs.js (运行时注入)
window.appConfigs = {
  // API 配置
  apiBaseURL: '/api',

  // WebSocket 配置
  wsURL: 'ws://backend:8080',

  // 环境标识
  environment: 'production',

  // 功能开关
  features: {
    enableDarkMode: true,
    enableNotifications: true
  }
};
```

## 部署前检查清单

### 后端服务
```markdown
- [ ] 配置文件正确（appsettings.json）
- [ ] 数据库连接正常
- [ ] Redis 连接正常
- [ ] 健康检查端点正常（/hc）
- [ ] Swagger 仅开发环境启用
- [ ] 日志输出正常
- [ ] 删除 appsettings.Development.json（生产环境）
- [ ] Dockerfile 时区设置正确
- [ ] NuGet 包版本统一
```

### 前端应用
```markdown
- [ ] 运行时配置正确（configs.js）
- [ ] API 反向代理配置正确
- [ ] 健康检查端点正常（/hc）
- [ ] 构建产物完整
- [ ] Nginx 配置正确
- [ ] gzip 压缩启用
- [ ] SPA 路由正常
```

### 跨服务变更
```markdown
- [ ] 依赖服务已部署（如 flnext-core）
- [ ] 数据库迁移已完成
- [ ] Redis 队列常量一致
- [ ] API 接口变更已通知前端
- [ ] 配置变更已同步
```

## 监控与告警

### Serilog 配置
```csharp
// 阿里云 SLS 日志
builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithProperty("Application", "MyApp.WebApi")
    .WriteTo.Console()
    .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day)
    // 阿里云 SLS
    .WriteTo.AliyunSls(
        endpoint: context.Configuration["Sls:Endpoint"],
        accessKeyId: context.Configuration["Sls:AccessKeyId"],
        accessKeySecret: context.Configuration["Sls:AccessKeySecret"],
        project: context.Configuration["Sls:Project"],
        logStore: context.Configuration["Sls:LogStore"])
);
```

### TraceId 跨服务传播
```csharp
// 中间件生成 TraceId
public class TraceIdMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        var traceId = context.Request.Headers["X-Trace-Id"].FirstOrDefault()
            ?? Guid.NewGuid().ToString("N");

        context.Items["TraceId"] = traceId;
        context.Response.Headers["X-Trace-Id"] = traceId;

        using (LogContext.PushProperty("TraceId", traceId))
        {
            await _next(context);
        }
    }
}

// 注册
app.UseMiddleware<TraceIdMiddleware>();
```

### 告警通知
```csharp
// 钉钉告警服务
public class DingTalkAlertService
{
    public async Task SendAlertAsync(string message)
    {
        var webhook = _config["DingTalk:Webhook"];
        var payload = new
        {
            msgtype = "text",
            text = new { content = $"【告警】{message}" }
        };

        await _httpClient.PostAsJsonAsync(webhook, payload);
    }
}

// 使用：关键错误告警
_logger.LogError(ex, "订单处理失败: OrderId={OrderId}", orderId);
await _dingTalk.SendAlertAsync($"订单处理失败: {orderId}");
```