# 埋点规范

## 埋点分类

### 埋点类型
| 类型 | 说明 | 示例 |
|------|------|------|
| 页面埋点 | 页面访问、浏览行为 | 页面PV、UV、停留时长 |
| 点击埋点 | 用户点击行为 | 按钮点击、链接点击 |
| 曝光埋点 | 元素曝光展示 | 商品曝光、广告曝光 |
| 事件埋点 | 业务事件触发 | 订单创建、支付成功 |
| 性能埋点 | 页面性能数据 | 加载时间、渲染时间 |

### 埋点优先级
| 优先级 | 说明 | 覆盖范围 |
|--------|------|----------|
| P0 | 核心业务流程 | 用户登录、订单流程、支付流程 |
| P1 | 重要业务行为 | 商品浏览、搜索、筛选 |
| P2 | 优化迭代需求 | UI交互、用户体验 |

## 埋点命名规范

### 命名格式
```
{项目}.{模块}.{页面}.{动作}.{对象}

例如：
myapp.order.detail.click.submit_btn      # 订单详情页点击提交按钮
myapp.product.list.view.product_card      # 商品列表页浏览商品卡片
myapp.search.result.submit.search_form    # 搜索结果页提交搜索
```

### 命名规则
| 部分 | 规则 | 示例 |
|------|------|------|
| 项目 | 英文小写 | myapp |
| 模块 | 英文小写 | order、product |
| 页面 | 英文小写 | detail、list |
| 动作 | 英文小写 | click、view、submit |
| 对象 | 英文小写 | btn、card、form |

### 动作类型
| 动作 | 说明 | 场景 |
|------|------|------|
| view | 浏览/曝光 | 页面浏览、元素曝光 |
| click | 点击 | 按钮点击、链接点击 |
| submit | 提交 | 表单提交、搜索提交 |
| scroll | 滚动 | 页面滚动、列表滚动 |
| play | 播放 | 视频播放、音频播放 |
| close | 关闭 | 弹窗关闭、页面关闭 |
| load | 加载 | 页面加载、图片加载 |

## 前端埋点

### 埋点SDK集成
```typescript
// src/utils/tracking.ts
import { Tracker } from '@anthropic/tracker-sdk';

// 初始化埋点SDK
export const tracker = new Tracker({
  appId: 'myapp',
  reportUrl: 'https://tracking.example.com/api/report',
  enable: true,
  debug: process.env.NODE_ENV === 'development',
});

// 自动采集页面访问
tracker.autoTrackPageView();

// 自动采集点击事件
tracker.autoTrackClick();
```

### 页面埋点
```typescript
// 页面访问埋点
import { tracker } from '@/utils/tracking';

// 页面进入
useEffect(() => {
  tracker.track('myapp.order.list.view.page', {
    page_name: '订单列表',
    page_url: '/order/list',
    referer: document.referrer,
    user_id: currentUser.id,
  });
}, []);

// 页面停留时长
useEffect(() => {
  const startTime = Date.now();
  
  return () => {
    const duration = Date.now() - startTime;
    tracker.track('myapp.order.list.leave.page', {
      page_name: '订单列表',
      duration_ms: duration,
    });
  };
}, []);
```

### 点击埋点
```typescript
// 按钮点击埋点
<Button
  onClick={() => {
    // 业务逻辑
    handleSubmit();
    
    // 埋点上报
    tracker.track('myapp.order.detail.click.submit_btn', {
      order_id: orderId,
      order_status: orderStatus,
      button_name: '提交订单',
    });
  }}
>
  提交订单
</Button>

// 链接点击埋点
<a
  href="/product/detail"
  onClick={() => {
    tracker.track('myapp.product.list.click.product_link', {
      product_id: productId,
      product_name: productName,
      link_url: '/product/detail',
    });
  }}
>
  {productName}
</a>
```

### 曝光埋点
```typescript
// 商品曝光埋点
import { useInView } from 'react-intersection-observer';

const ProductCard: React.FC<{ product: Product }> = ({ product }) => {
  const [ref, inView] = useInView({
    threshold: 0.5,  // 50%可见时触发
    triggerOnce: true,  // 只触发一次
  });

  useEffect(() => {
    if (inView) {
      tracker.track('myapp.product.list.view.product_card', {
        product_id: product.id,
        product_name: product.name,
        product_price: product.price,
        position: product.position,
      });
    }
  }, [inView]);

  return <div ref={ref}>{/* 商品卡片内容 */}</div>;
};
```

### 搜索埋点
```typescript
// 搜索提交埋点
const handleSearch = async (keywords: string) => {
  // 埋点上报
  tracker.track('myapp.search.home.submit.search_form', {
    keywords,
    search_type: 'normal',
    filters: JSON.stringify(searchFilters),
  });

  // 业务逻辑
  const result = await searchProducts(keywords);
  setSearchResult(result);

  // 搜索结果埋点
  tracker.track('myapp.search.result.view.result_page', {
    keywords,
    result_count: result.total,
    page_number: 1,
  });
};
```

## 后端埋点

### 业务事件埋点
```csharp
// 订单创建埋点
public class OrderService
{
    private readonly ITrackingService _tracking;

    public async Task<long> CreateOrderAsync(CreateOrderRequest request)
    {
        var order = new Order { ... };
        await _orderRepo.AddAsync(order);

        // 埋点上报
        _tracking.Track("myapp.order.create.success.event", new
        {
            order_id = order.Id,
            user_id = order.UserId,
            product_id = order.ProductId,
            order_amount = order.TotalAmount,
            order_status = order.Status,
            payment_method = request.PaymentMethod,
        });

        return order.Id;
    }
}
```

### 支付埋点
```csharp
// 支付成功埋点
public class PaymentService
{
    public async Task<bool> ProcessPaymentAsync(long orderId)
    {
        var result = await _paymentClient.Pay(orderId);
        
        if (result.Success)
        {
            // 埋点上报
            _tracking.Track("myapp.payment.success.event", new
            {
                order_id = orderId,
                payment_amount = result.Amount,
                payment_method = result.Method,
                payment_channel = result.Channel,
                payment_duration_ms = result.DurationMs,
            });
        }
        else
        {
            // 支付失败埋点
            _tracking.Track("myapp.payment.fail.event", new
            {
                order_id = orderId,
                error_code = result.ErrorCode,
                error_message = result.ErrorMessage,
            });
        }

        return result.Success;
    }
}
```

## 埋点参数

### 公共参数
| 参数 | 类型 | 说明 | 必填 |
|------|------|------|------|
| user_id | long | 用户ID | 是 |
| session_id | string | 会话ID | 是 |
| device_id | string | 设备ID | 是 |
| platform | string | 平台(web/app) | 是 |
| app_version | string | 应用版本 | 是 |
| os_version | string | 系统版本 | 是 |
| device_type | string | 设备类型 | 是 |
| network_type | string | 网络类型 | 是 |
| timestamp | long | 时间戳 | 是 |

### 页面参数
| 参数 | 类型 | 说明 | 必填 |
|------|------|------|------|
| page_name | string | 页面名称 | 是 |
| page_url | string | 页面URL | 是 |
| referer | string | 来源页面 | 否 |
| duration_ms | long | 停留时长 | 否 |

### 点击参数
| 参数 | 类型 | 说明 | 必填 |
|------|------|------|------|
| button_name | string | 按钮名称 | 是 |
| button_type | string | 按钮类型 | 否 |
| position | int | 元素位置 | 否 |

### 商品参数
| 参数 | 类型 | 说明 | 必填 |
|------|------|------|------|
| product_id | long | 商品ID | 是 |
| product_name | string | 商品名称 | 是 |
| product_price | decimal | 商品价格 | 是 |
| category_id | long | 分类ID | 否 |
| brand_id | long | 品牌ID | 否 |

### 订单参数
| 参数 | 类型 | 说明 | 必填 |
|------|------|------|------|
| order_id | long | 订单ID | 是 |
| order_amount | decimal | 订单金额 | 是 |
| order_status | int | 订单状态 | 是 |
| payment_method | string | 支付方式 | 否 |

## 上报时机

### 上报时机定义
| 场景 | 上报时机 | 说明 |
|------|----------|------|
| 页面访问 | 页面进入时 | 立即上报 |
| 页面停留 | 页面离开时 | 计算时长后上报 |
| 点击事件 | 点击时 | 立即上报 |
| 曝光事件 | 元素可见时 | 立即上报 |
| 业务事件 | 事件完成时 | 立即上报 |

### 上报方式
| 方式 | 说明 | 适用场景 |
|------|------|----------|
| 同步上报 | 立即发送 | 重要事件 |
| 异步上报 | 缓存后发送 | 一般事件 |
| 批量上报 | 批量打包发送 | 曝光事件 |

### 异步上报实现
```typescript
// 埋点缓存队列
class TrackingQueue {
  private queue: TrackingEvent[] = [];
  private maxQueueSize = 20;
  private flushInterval = 5000;

  push(event: TrackingEvent) {
    this.queue.push(event);
    
    if (this.queue.length >= this.maxQueueSize) {
      this.flush();
    }
  }

  flush() {
    if (this.queue.length === 0) return;
    
    const events = [...this.queue];
    this.queue = [];
    
    // 批量上报
    fetch('/api/tracking/batch', {
      method: 'POST',
      body: JSON.stringify(events),
    });
  }

  // 定时刷新
  startFlushTimer() {
    setInterval(() => this.flush(), this.flushInterval);
  }
}
```

## 埋点清单

### 埋点清单模板
```markdown
# 埋点清单：{项目}-{模块}

## 页面埋点
| 埋点ID | 埋点名称 | 命名 | 参数 | 上报时机 |
|--------|----------|------|------|----------|
| T001 | 订单列表页浏览 | myapp.order.list.view.page | page_name, page_url, referer | 页面进入 |
| T002 | 订单详情页浏览 | myapp.order.detail.view.page | page_name, page_url, order_id | 页面进入 |

## 点击埋点
| 埋点ID | 埋点名称 | 命名 | 参数 | 上报时机 |
|--------|----------|------|------|----------|
| T003 | 创建订单按钮点击 | myapp.order.list.click.create_btn | button_name, referer | 点击时 |
| T004 | 提交订单按钮点击 | myapp.order.detail.click.submit_btn | button_name, order_id | 点击时 |

## 曝光埋点
| 埅点ID | 埅点名称 | 命名 | 参数 | 上报时机 |
|--------|----------|------|------|----------|
| T005 | 商品卡片曝光 | myapp.product.list.view.product_card | product_id, product_name, position | 元素可见 |

## 事件埋点
| 埅点ID | 埅点名称 | 命名 | 参数 | 上报时机 |
|--------|----------|------|------|----------|
| T006 | 订单创建成功 | myapp.order.create.success.event | order_id, order_amount | 订单创建成功 |
| T007 | 支付成功 | myapp.payment.success.event | order_id, payment_amount | 支付成功 |
```

## 数据分析

### 分析指标
| 指标 | 说明 | 计算方式 |
|------|------|----------|
| PV | 页面浏览量 | 页面访问次数 |
| UV | 页面访客数 | 唯一用户数 |
| 转化率 | 转化完成率 | 完成数/访问数 |
| 点击率 | 点击/浏览比例 | 点击数/PV |
| 停留时长 | 页面停留时间 | 离开时间-进入时间 |

### 分析报表
| 报表 | 说明 | 数据来源 |
|------|------|----------|
| 用户路径 | 用户行为路径 | 页面埋点 |
| 功能使用 | 功能使用情况 | 点击埋点 |
| 商品热度 | 商品浏览热度 | 曝光埋点 |
| 转化漏斗 | 转化漏斗分析 | 事件埋点 |

## DO - 推荐做法

```typescript
// ✓ 埅点命名规范
myapp.order.detail.click.submit_btn

// ✓ 公共参数完整
user_id, session_id, device_id, platform

// ✓ 业务参数完整
order_id, order_amount, order_status

// ✓ 上报时机明确
页面进入时立即上报
点击时立即上报

// ✓ 埅点清单维护
// 埅点清单模板记录所有埋点

// ✓ 重要事件同步上报
订单创建、支付成功立即上报

// ✓ 曝光事件批量上报
商品曝光批量打包上报
```

## DON'T - 禁止做法

```typescript
// ✗ 埅点命名不规范
order_submit  // 禁止，缺少项目和模块

// ✓ 正确做法
myapp.order.detail.click.submit_btn

// ✗ 公共参数缺失
// 缺少 user_id  // 禁止

// ✓ 正确做法
user_id, session_id, device_id, platform, app_version

// ✗ 业务参数缺失
tracker.track('myapp.order.create', {});  // 禁止，无参数

// ✓ 正确做法
tracker.track('myapp.order.create.success.event', {
  order_id, order_amount, order_status
});

// ✗ 上报时机不明确
// 不定义上报时机  // 禁止

// ✓ 正确做法
页面进入时立即上报

// ✗ 无埋点清单
// 无埋点记录  // 禁止

// ✓ 正确做法
# 埅点清单模板

// ✗ 所有事件同步上报
// 重要和不重要都立即上报  // 禁止，影响性能

// ✓ 正确做法
重要事件同步上报，一般事件异步上报

// ✗ 曝光无批量
// 每次曝光立即上报  // 禁止，影响性能

// ✓ 正确做法
曝光事件批量打包上报
```

## 最佳实践

### 埅点规划
- 埅点命名规范统一
- 埅点参数完整明确
- 埅点清单维护更新

### 埅点实现
- SDK统一集成
- 自动采集优先
- 手动埋点补充

### 埅点上报
- 重要事件同步上报
- 一般事件异步上报
- 曝光事件批量上报