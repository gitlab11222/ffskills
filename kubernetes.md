# Kubernetes 部署规范

## K8s 资源规范

### 命名规范
| 资源 | 命名格式 | 示例 |
|------|----------|------|
| Namespace | {team}-{env} | product-dev |
| Deployment | {service-name} | order-service |
| Service | {service-name} | order-service |
| ConfigMap | {service-name}-config | order-service-config |
| Secret | {service-name}-secret | order-service-secret |
| Ingress | {service-name}-ingress | order-service-ingress |
| HPA | {service-name}-hpa | order-service-hpa |

### Namespace 划分
```yaml
# Namespace 定义
apiVersion: v1
kind: Namespace
metadata:
  name: product-dev
  labels:
    team: product
    environment: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: product-test
  labels:
    team: product
    environment: test
---
apiVersion: v1
kind: Namespace
metadata:
  name: product-prod
  labels:
    team: product
    environment: prod
```

## Deployment 配置

### 标准 Deployment 模板
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: product-prod
  labels:
    app: order-service
    version: v1.0.0
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 最多多1个Pod
      maxUnavailable: 0    # 最少保持全部可用
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        version: v1.0.0
    spec:
      # 优雅终止时间
      terminationGracePeriodSeconds: 30
      
      # 容器配置
      containers:
      - name: order-service
        image: registry.example.com/order-service:v1.0.0
        
        # 镜像拉取策略
        imagePullPolicy: IfNotPresent
        
        # 资源限制
        resources:
          limits:
            cpu: "1"
            memory: "1Gi"
          requests:
            cpu: "500m"
            memory: "512Mi"
        
        # 端口配置
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        
        # 环境变量
        envFrom:
        - configMapRef:
            name: order-service-config
        - secretRef:
            name: order-service-secret
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        
        # 存活探针
        livenessProbe:
          httpGet:
            path: /hc
            port: 80
          initialDelaySeconds: 30   # 启动后等待30秒
          periodSeconds: 10         # 每10秒检查
          timeoutSeconds: 5         # 超时5秒
          failureThreshold: 3       # 失败3次重启
        
        # 就绪探针
        readinessProbe:
          httpGet:
            path: /hc
            port: 80
          initialDelaySeconds: 5    # 启动后等待5秒
          periodSeconds: 5          # 每5秒检查
          timeoutSeconds: 3         # 超时3秒
          failureThreshold: 3       # 失败3次标记未就绪
        
        # 启动探针（慢启动应用）
        startupProbe:
          httpGet:
            path: /hc
            port: 80
          initialDelaySeconds: 0
          periodSeconds: 10
          failureThreshold: 30      # 等待300秒启动
        
        # 卷挂载
        volumeMounts:
        - name: logs
          mountPath: /app/logs
        - name: config
          mountPath: /app/config
      
      # 卷定义
      volumes:
      - name: logs
        emptyDir: {}
      - name: config
        configMap:
          name: order-service-config
      
      # 镜像拉取密钥
      imagePullSecrets:
      - name: registry-secret
      
      # 节点选择
      nodeSelector:
        team: product
      
      # 容忍配置
      tolerations:
      - key: "team"
        operator: "Equal"
        value: "product"
        effect: "NoSchedule"
      
      # 亲和性配置
      affinity:
        podAntiAffinity:           # Pod反亲和，分散部署
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: order-service
              topologyKey: kubernetes.io/hostname
```

### 资源配置标准
| 服务类型 | CPU Request | CPU Limit | Memory Request | Memory Limit |
|----------|-------------|-----------|----------------|--------------|
| API服务 | 500m | 1 | 512Mi | 1Gi |
| 计算服务 | 1 | 2 | 1Gi | 2Gi |
| 后台任务 | 200m | 500m | 256Mi | 512Mi |
| 轻量服务 | 100m | 300m | 128Mi | 256Mi |

## Service 配置

### ClusterIP Service（内部服务）
```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: product-prod
  labels:
    app: order-service
spec:
  type: ClusterIP
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 80
    name: http
```

### NodePort Service（需外部访问）
```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service-external
  namespace: product-prod
spec:
  type: NodePort
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080  # 范围: 30000-32767
```

### Headless Service（StatefulSet）
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: product-prod
spec:
  type: ClusterIP
  clusterIP: None    # Headless
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

## Ingress 配置

### 标准 Ingress 模板
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-service-ingress
  namespace: product-prod
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/order
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
```

### 路径规则
```yaml
# 精确匹配
- path: /api/order/create
  pathType: Exact

# 前缀匹配
- path: /api/order
  pathType: Prefix

# 正则匹配（需 Nginx Ingress 支持）
annotations:
  nginx.ingress.kubernetes.io/use-regex: "true"
- path: /api/order/[0-9]+
  pathType: ImplementationSpecific
```

## ConfigMap 配置

### 文件型 ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: product-prod
data:
  # 环境变量
  ASPNETCORE_ENVIRONMENT: "Production"
  LOG_LEVEL: "Warning"
  
  # 配置文件
  appsettings.json: |
    {
      "ConnectionStrings": {
        "Default": "Server=mysql-service;Port=3306;Database=order_db;"
      },
      "Redis": {
        "ConnectionString": "redis-service:6379"
      }
    }
```

### 命令行创建 ConfigMap
```bash
# 从文件创建
kubectl create configmap app-config --from-file=appsettings.json

# 从目录创建
kubectl create configmap app-config --from-file=./config/

# 从键值创建
kubectl create configmap app-config --from-literal=LOG_LEVEL=Warning
```

## Secret 配置

### Secret 模板
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secret
  namespace: product-prod
type: Opaque
data:
  # Base64 编码
  DB_PASSWORD: cGFzc3dvcmQxMjM=          # password123
  REDIS_PASSWORD: cmVkaXNwYXNz           # redispass
  JWT_SECRET: and0c2VjcmV0a2V5          # jwtsecretkey
stringData:
  # 明文（自动编码）
  API_KEY: apikey123
```

### 创建 Secret
```bash
# 从文件创建
kubectl create secret generic db-secret --from-file=password.txt

# 从键值创建
kubectl create secret generic db-secret --from-literal=password=password123

# 从 Docker Registry 创建
kubectl create secret docker-registry registry-secret \
  --docker-server=registry.example.com \
  --docker-username=admin \
  --docker-password=password \
  --docker-email=admin@example.com
```

## HPA 配置

### HPA 模板
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: product-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  # CPU 自动扩缩
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  # 内存自动扩缩
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # 自定义指标
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 100
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

## StatefulSet 配置

### StatefulSet 模板（MySQL）
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: product-prod
spec:
  serviceName: mysql-service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

## Job 配置

### Job 模板（一次性任务）
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
  namespace: product-prod
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: data-migration
    spec:
      containers:
      - name: migration
        image: registry.example.com/migration:v1.0.0
        command: ["dotnet", "Migration.dll"]
      restartPolicy: Never      # 成功后不重启
      backoffLimit: 4           # 失败重试次数
```

### CronJob 模板（定时任务）
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
  namespace: product-prod
spec:
  schedule: "0 8 * * *"        # 每天8点
  concurrencyPolicy: Forbid    # 禁止并发
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report
            image: registry.example.com/report:v1.0.0
            command: ["dotnet", "Report.dll"]
          restartPolicy: OnFailure
```

## 部署策略

### RollingUpdate（滚动更新）
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # 每次最多多1个Pod
    maxUnavailable: 0    # 保持全部可用
```

### Blue/Green 部署
```yaml
# Blue 版本
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-blue
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: order-service
        version: blue

# Green 版本
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-green
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: order-service
        version: green

# Service 切换
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
    version: blue  # 切换为 green 即完成切换
```

### Canary 部署
```yaml
# 稳定版本
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-stable
spec:
  replicas: 9    # 90%

# 金丝雀版本
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-canary
spec:
  replicas: 1    # 10%

# Ingress 流量分配（Nginx）
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-weight: "10"
```

## 命令速查

### 部署命令
```bash
# 应用配置
kubectl apply -f deployment.yaml

# 查看部署状态
kubectl rollout status deployment/order-service

# 查看部署历史
kubectl rollout history deployment/order-service

# 回滚到上一版本
kubectl rollout undo deployment/order-service

# 回滚到指定版本
kubectl rollout undo deployment/order-service --to-revision=2

# 暂停部署
kubectl rollout pause deployment/order-service

# 继续部署
kubectl rollout resume deployment/order-service

# 查看Pod状态
kubectl get pods -l app=order-service

# 查看Pod详情
kubectl describe pod order-service-xxx

# 查看Pod日志
kubectl logs order-service-xxx

# 实时查看日志
kubectl logs -f order-service-xxx

# 进入Pod
kubectl exec -it order-service-xxx -- /bin/bash

# 端口转发
kubectl port-forward order-service-xxx 8080:80
```

### 资源查看
```bash
# 查看所有资源
kubectl get all -n product-prod

# 查看Deployment
kubectl get deployments -n product-prod

# 查看Service
kubectl get services -n product-prod

# 查看Ingress
kubectl get ingress -n product-prod

# 查看ConfigMap
kubectl get configmaps -n product-prod

# 查看Secret
kubectl get secrets -n product-prod

# 查看HPA
kubectl get hpa -n product-prod

# 查看资源使用
kubectl top pods -n product-prod
kubectl top nodes
```

## DO - 推荐做法

```yaml
# ✓ 设置资源限制
resources:
  limits:
    cpu: "1"
    memory: "1Gi"
  requests:
    cpu: "500m"
    memory: "512Mi"

# ✓ 设置健康检查
livenessProbe:
  httpGet:
    path: /hc
    port: 80

readinessProbe:
  httpGet:
    path: /hc
    port: 80

# ✓ 滚动更新配置
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

# ✓ 设置副本数
replicas: 3

# ✓ 使用 ConfigMap/Secret
envFrom:
- configMapRef:
    name: app-config
- secretRef:
    name: app-secret
```

## DON'T - 禁止做法

```yaml
# ✗ 不设置资源限制
# 无 resources 配置  // 禁止

# ✓ 正确做法
resources:
  limits:
    cpu: "1"
    memory: "1Gi"

# ✗ 健康检查无超时
livenessProbe:
  httpGet:
    path: /hc
    port: 80
  # 无 timeoutSeconds  // 禁止

# ✓ 正确做法
livenessProbe:
  httpGet:
    path: /hc
    port: 80
  timeoutSeconds: 5

# ✗ 滚动更新允许不可用
maxUnavailable: 1  // 禁止，服务中断

# ✓ 正确做法
maxUnavailable: 0

# ✗ Secret 明文存储
data:
  password: password123  // 禁止，未Base64编码

# ✓ 正确做法
data:
  password: cGFzc3dvcmQxMjM=
# 或
stringData:
  password: password123

# ✗ 单副本无冗余
replicas: 1  // 禁止，无高可用

# ✓ 正确做法
replicas: 3
```

## 最佳实践

### 高可用配置
- 副本数 ≥ 3
- Pod反亲和分散部署
- 跨可用区部署
- 健康检查配置
- 自动扩缩容配置

### 安全配置
- 资源限制防OOM
- Secret加密存储
- NetworkPolicy隔离
- RBAC权限控制
- 镜像安全扫描

### 资源优化
- 合理设置Request/Limit
- 自动扩缩容
- 资源配额管理
- 定期清理无用资源