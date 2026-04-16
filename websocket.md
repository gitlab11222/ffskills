# WebSocket 使用规范

## WebSocket 基础

### 适用场景
| 场景 | 说明 | 推荐 |
|------|------|------|
| 实时消息 | 即时通讯、消息推送 | ✅ 推荐 |
| 实时数据 | 股票行情、数据监控 | ✅ 推荐 |
| 协作编辑 | 多人协同编辑 | ✅ 推荐 |
| 实时游戏 | 游戏状态同步 | ✅ 推荐 |
| 文件传输 | 大文件传输 | ⚠️ 考虑HTTP |
| 普通请求 | 一般API请求 | ❌ 不推荐 |

### 不适用场景
| 场景 | 替代方案 |
|------|----------|
| 短连接请求 | HTTP REST API |
| 文件上传 | HTTP multipart |
| 批量操作 | HTTP批量接口 |
| 低频更新 | 定时刷新 |

## 前端 WebSocket

### 连接管理
```typescript
// WebSocket 连接类
class WebSocketClient {
  private ws: WebSocket | null = null;
  private url: string;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectInterval = 5000;
  private heartbeatInterval = 30000;
  private heartbeatTimer: NodeJS.Timeout | null = null;

  constructor(url: string) {
    this.url = url;
  }

  // 连接
  connect() {
    this.ws = new WebSocket(this.url);
    
    // 连接成功
    this.ws.onopen = () => {
      console.log('WebSocket连接成功');
      this.reconnectAttempts = 0;
      this.startHeartbeat();
      this.onOpen?.();
    };

    // 接收消息
    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.handleMessage(data);
    };

    // 连接错误
    this.ws.onerror = (error) => {
      console.error('WebSocket连接错误', error);
      this.onError?.(error);
    };

    // 连接关闭
    this.ws.onclose = () => {
      console.log('WebSocket连接关闭');
      this.stopHeartbeat();
      this.reconnect();
      this.onClose?.();
    };
  }

  // 重连
  reconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('WebSocket重连失败，已达最大重连次数');
      return;
    }

    this.reconnectAttempts++;
    console.log(`WebSocket重连第${this.reconnectAttempts}次`);

    setTimeout(() => {
      this.connect();
    }, this.reconnectInterval);
  }

  // 心跳
  startHeartbeat() {
    this.heartbeatTimer = setInterval(() => {
      if (this.ws?.readyState === WebSocket.OPEN) {
        this.send({ type: 'ping', timestamp: Date.now() });
      }
    }, this.heartbeatInterval);
  }

  stopHeartbeat() {
    if (this.heartbeatTimer) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = null;
    }
  }

  // 发送消息
  send(data: any) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    } else {
      console.warn('WebSocket未连接，消息发送失败');
      // 缓存消息，连接成功后发送
      this.pendingMessages.push(data);
    }
  }

  // 处理消息
  handleMessage(data: any) {
    switch (data.type) {
      case 'pong':
        // 心跳响应
        break;
      case 'message':
        this.onMessage?.(data.payload);
        break;
      case 'notification':
        this.onNotification?.(data.payload);
        break;
      default:
        console.warn('未知的消息类型', data.type);
    }
  }

  // 关闭连接
  close() {
    this.stopHeartbeat();
    this.ws?.close();
    this.ws = null;
  }

  // 回调函数
  onOpen?: () => void;
  onMessage?: (data: any) => void;
  onNotification?: (data: any) => void;
  onError?: (error: any) => void;
  onClose?: () => void;

  // 待发送消息队列
  private pendingMessages: any[] = [];
}
```

### React 封装
```typescript
// useWebSocket Hook
import { useEffect, useRef, useState } from 'react';

export function useWebSocket(url: string) {
  const clientRef = useRef<WebSocketClient | null>(null);
  const [connected, setConnected] = useState(false);
  const [messages, setMessages] = useState<any[]>([]);

  useEffect(() => {
    const client = new WebSocketClient(url);
    
    client.onOpen = () => setConnected(true);
    client.onClose = () => setConnected(false);
    client.onMessage = (data) => {
      setMessages((prev) => [...prev, data]);
    };

    client.connect();
    clientRef.current = client;

    return () => {
      client.close();
    };
  }, [url]);

  const send = (data: any) => {
    clientRef.current?.send(data);
  };

  return { connected, messages, send };
}

// 使用示例
const Chat: React.FC = () => {
  const { connected, messages, send } = useWebSocket('wss://example.com/ws');

  return (
    <div>
      <div>连接状态: {connected ? '已连接' : '未连接'}</div>
      <div>
        {messages.map((msg, i) => (
          <div key={i}>{msg.content}</div>
        ))}
      </div>
      <button onClick={() => send({ type: 'message', content: 'Hello' })}>
        发送
      </button>
    </div>
  );
};
```

### 消息订阅
```typescript
// 消息订阅管理
class MessageSubscription {
  private subscriptions: Map<string, Set<(data: any) => void>> = new Map();

  // 订阅
  subscribe(topic: string, callback: (data: any) => void) {
    if (!this.subscriptions.has(topic)) {
      this.subscriptions.set(topic, new Set());
    }
    this.subscriptions.get(topic)?.add(callback);
    
    // 返回取消订阅函数
    return () => {
      this.subscriptions.get(topic)?.delete(callback);
    };
  }

  // 发布
  publish(topic: string, data: any) {
    const callbacks = this.subscriptions.get(topic);
    if (callbacks) {
      callbacks.forEach((cb) => cb(data));
    }
  }

  // 清理
  clear(topic?: string) {
    if (topic) {
      this.subscriptions.delete(topic);
    } else {
      this.subscriptions.clear();
    }
  }
}

// WebSocket 消息订阅
const subscription = new MessageSubscription();

wsClient.onMessage = (data) => {
  subscription.publish(data.topic, data.payload);
};

// 使用
const unsubscribe = subscription.subscribe('order_update', (data) => {
  console.log('订单更新', data);
});

// 取消订阅
unsubscribe();
```

## 后端 WebSocket

### SignalR 配置（.NET）
```csharp
// Program.cs
builder.Services.AddSignalR()
    .AddJsonProtocol(options =>
    {
        options.PayloadSerializerOptions.PropertyNamingPolicy = null;
    });

// 配置WebSocket
app.MapHub<MessageHub>("/ws/message");
app.MapHub<OrderHub>("/ws/order");

// Hub 定义
public class MessageHub : Hub
{
    private readonly IMessageService _messageService;

    // 连接
    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier;
        await Groups.AddToGroupAsync(Context.ConnectionId, $"user_{userId}");
        await base.OnConnectedAsync();
    }

    // 断开连接
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier;
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"user_{userId}");
        await base.OnDisconnectedAsync(exception);
    }

    // 发送消息
    public async Task SendMessage(MessageRequest request)
    {
        var message = await _messageService.CreateAsync(request);
        
        // 发送给指定用户
        await Clients.User(request.ReceiverId.ToString())
            .SendAsync("ReceiveMessage", message);
        
        // 或发送给群组
        await Clients.Group($"group_{request.GroupId}")
            .SendAsync("ReceiveMessage", message);
    }

    // 加入群组
    public async Task JoinGroup(long groupId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"group_{groupId}");
    }

    // 退出群组
    public async Task LeaveGroup(long groupId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"group_{groupId}");
    }
}

// 订单状态推送Hub
public class OrderHub : Hub
{
    // 推送订单状态更新
    public async Task PushOrderStatus(long orderId, OrderStatus status)
    {
        await Clients.Group($"order_{orderId}")
            .SendAsync("OrderStatusUpdate", new { orderId, status });
    }

    // 订阅订单
    public async Task SubscribeOrder(long orderId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"order_{orderId}");
    }

    // 取消订阅
    public async Task UnsubscribeOrder(long orderId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"order_{orderId}");
    }
}
```

### 消息推送服务
```csharp
// 消息推送服务
public interface IPushService
{
    Task PushToUser(long userId, string method, object data);
    Task PushToGroup(string groupName, string method, object data);
    Task PushToAll(string method, object data);
}

public class PushService : IPushService
{
    private readonly IHubContext<MessageHub> _hubContext;

    public PushService(IHubContext<MessageHub> hubContext)
    {
        _hubContext = hubContext;
    }

    public async Task PushToUser(long userId, string method, object data)
    {
        await _hubContext.Clients.User(userId.ToString())
            .SendAsync(method, data);
    }

    public async Task PushToGroup(string groupName, string method, object data)
    {
        await _hubContext.Clients.Group(groupName)
            .SendAsync(method, data);
    }

    public async Task PushToAll(string method, object data)
    {
        await _hubContext.Clients.All.SendAsync(method, data);
    }
}

// 业务中使用
public class OrderService
{
    private readonly IPushService _pushService;

    public async Task UpdateOrderStatusAsync(long orderId, OrderStatus status)
    {
        var order = await _orderRepo.GetByIdAsync(orderId);
        order.Status = status;
        await _orderRepo.UpdateAsync(order);

        // 推送订单状态更新
        await _pushService.PushToGroup($"order_{orderId}", "OrderStatusUpdate", new
        {
            orderId,
            status,
            updatedAt = DateTime.UtcNow
        });
    }
}
```

## 消息协议

### 消息格式
```typescript
// 消息格式定义
interface WebSocketMessage {
  type: string;        // 消息类型
  topic: string;       // 消息主题
  payload: any;        // 消息内容
  timestamp: number;   // 时间戳
}

// 消息类型定义
enum MessageType {
  // 连接相关
  CONNECT = 'connect',
  DISCONNECT = 'disconnect',
  PING = 'ping',
  PONG = 'pong',
  
  // 业务消息
  MESSAGE = 'message',
  NOTIFICATION = 'notification',
  
  // 状态更新
  ORDER_UPDATE = 'order_update',
  PAYMENT_UPDATE = 'payment_update',
  
  // 订阅管理
  SUBSCRIBE = 'subscribe',
  UNSUBSCRIBE = 'unsubscribe',
}
```

### 消息示例
```json
// 连接消息
{
  "type": "connect",
  "payload": {
    "user_id": 123,
    "token": "xxx"
  },
  "timestamp": 1704067200000
}

// 心跳消息
{
  "type": "ping",
  "timestamp": 1704067200000
}

// 业务消息
{
  "type": "message",
  "topic": "order_update",
  "payload": {
    "order_id": 1,
    "status": "paid",
    "updated_at": "2024-01-01T00:00:00Z"
  },
  "timestamp": 1704067200000
}

// 订阅消息
{
  "type": "subscribe",
  "topic": "order_1",
  "timestamp": 1704067200000
}
```

## 心跳机制

### 心跳配置
| 参数 | 值 | 说明 |
|------|-----|------|
| 心跳间隔 | 30秒 | 每30秒发送心跳 |
| 心跳超时 | 60秒 | 超时未响应断开 |
| 重连间隔 | 5秒 | 断开后5秒重连 |
| 最大重连次数 | 5次 | 超过则停止重连 |

### 心跳实现
```typescript
// 心跳检测
class HeartbeatManager {
  private heartbeatInterval = 30000;
  private heartbeatTimeout = 60000;
  private lastPongTime = 0;
  private heartbeatTimer: NodeJS.Timeout | null = null;
  private timeoutTimer: NodeJS.Timeout | null = null;

  start(ws: WebSocketClient) {
    this.lastPongTime = Date.now();
    
    // 定时发送心跳
    this.heartbeatTimer = setInterval(() => {
      ws.send({ type: 'ping', timestamp: Date.now() });
    }, this.heartbeatInterval);

    // 定时检查超时
    this.timeoutTimer = setInterval(() => {
      if (Date.now() - this.lastPongTime > this.heartbeatTimeout) {
        console.warn('WebSocket心跳超时');
        ws.close();
        ws.reconnect();
      }
    }, this.heartbeatInterval);
  }

  // 收到pong响应
  onPong() {
    this.lastPongTime = Date.now();
  }

  stop() {
    if (this.heartbeatTimer) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = null;
    }
    if (this.timeoutTimer) {
      clearInterval(this.timeoutTimer);
      this.timeoutTimer = null;
    }
  }
}
```

## 断线重连

### 重连策略
| 策略 | 说明 |
|------|------|
| 立即重连 | 断开后立即尝试重连 |
| 延迟重连 | 断开后延迟N秒重连 |
| 渐进重连 | 重连间隔逐渐增加 |

### 渐进重连实现
```typescript
class ProgressiveReconnect {
  private baseInterval = 1000;
  private maxInterval = 30000;
  private attempts = 0;

  reconnect(ws: WebSocketClient) {
    this.attempts++;
    
    // 渐进间隔：1s, 2s, 4s, 8s, 16s, 30s
    const interval = Math.min(
      this.baseInterval * Math.pow(2, this.attempts - 1),
      this.maxInterval
    );

    console.log(`WebSocket重连第${this.attempts}次，间隔${interval}ms`);

    setTimeout(() => {
      ws.connect();
    }, interval);
  }

  reset() {
    this.attempts = 0;
  }
}
```

## DO - 推荐做法

```typescript
// ✓ 连接管理完整
connect() → onopen → onmessage → onerror → onclose → reconnect()

// ✓ 心跳机制
定时发送ping，检测pong响应，超时重连

// ✓ 断线重连
自动重连，渐进间隔，最大次数限制

// ✓ 消息订阅管理
subscribe(topic, callback) → unsubscribe()

// ✓ 状态连接监控
connected状态实时显示

// ✓ 消息队列缓存
未连接时缓存消息，连接后发送

// ✓ 错误处理完整
连接错误、消息错误、超时处理

// ✓ 组件卸载关闭
useEffect return中关闭连接
```

## DON'T - 禁止做法

```typescript
// ✗ 无连接管理
new WebSocket(url);  // 禁止，无onopen/onclose处理

// ✓ 正确做法
ws.onopen = () => { };
ws.onclose = () => reconnect();

// ✗ 无心跳机制
// 无ping/pong  // 禁止，无法检测连接状态

// ✓ 正确做法
定时发送ping，检测pong响应

// ✗ 无断线重连
onclose = () => {};  // 禁止，无重连

// ✓ 正确做法
onclose = () => reconnect();

// ✗ 无消息订阅管理
// 直接监听消息  // 禁止，无法取消订阅

// ✓ 正确做法
subscribe(topic, callback)
unsubscribe()

// ✗ 无连接状态监控
// 不显示连接状态  // 禁止

// ✓ 正确做法
显示 connected状态

// ✗ 无消息缓存
// 未连接时消息丢失  // 禁止

// ✓ 正确做法
pendingMessages缓存

// ✗ 无错误处理
// 不处理error  // 禁止

// ✓ 正确做法
onerror = (error) => { };

// ✗ 组件卸载未关闭
// useEffect无return  // 禁止，连接泄漏

// ✓ 正确做法
useEffect(() => {
  const ws = new WebSocket();
  return () => ws.close();
});
```

## 最佳实践

### 连接管理
- 连接生命周期完整
- 心跳机制健壮
- 断线自动重连

### 消息管理
- 消息格式规范
- 订阅机制清晰
- 队列缓存可靠

### 错误处理
- 连接错误处理
- 消息错误处理
- 超时检测处理