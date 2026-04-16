# UI设计规范

## 设计系统

### 颜色系统

#### 主色调
| 用途 | 色值 | 变量名 | 说明 |
|------|------|--------|------|
| 主色 | #1890FF | @primary-color | 主要按钮、链接、强调元素 |
| 主色浅 | #40A9FF | @primary-color-hover | Hover状态 |
| 主色深 | #096DD9 | @primary-color-active | Active状态 |
| 主色背景 | #E6F7FF | @primary-color-bg | 主色背景区域 |

#### 功能色
| 用途 | 色值 | 变量名 | 说明 |
|------|------|--------|------|
| 成功 | #52C41A | @success-color | 成功状态、正向反馈 |
| 成功浅 | #73D13D | @success-color-hover | Hover状态 |
| 成功背景 | #F6FFED | @success-color-bg | 成功背景区域 |
| 警告 | #FAAD14 | @warning-color | 警告状态、风险提示 |
| 警告浅 | #FFC53D | @warning-color-hover | Hover状态 |
| 警告背景 | #FFFBE6 | @warning-color-bg | 警告背景区域 |
| 错误 | #F5222D | @error-color | 错误状态、危险操作 |
| 错误浅 | #FF4D4F | @error-color-hover | Hover状态 |
| 错误背景 | #FFF1F0 | @error-color-bg | 错误背景区域 |
| 信息 | #1890FF | @info-color | 信息提示 |

#### 中性色
| 用途 | 艱值 | 变量名 | 说明 |
|------|------|--------|------|
| 主文字 | #333333 | @text-color | 正文、标题 |
| 次文字 | #666666 | @text-color-secondary | 次要文字 |
| 辅助文字 | #999999 | @text-color辅助 | 辅助说明 |
| 禁用文字 | #CCCCCC | @disabled-color | 禁用状态 |
| 边框 | #D9D9D9 | @border-color | 边框、分割线 |
| 边框浅 | #E8E8E8 | @border-color-secondary | 浅边框 |
| 分割线 | #F0F0F0 | @divider-color | 分割线 |
| 背景 | #F5F5F5 | @background-color | 页面背景 |
| 白色 | #FFFFFF | @white-color | 白色背景 |

#### 渐变色
```less
// 主色渐变
.gradient-primary {
  background: linear-gradient(135deg, #1890FF 0%, #096DD9 100%);
}

// 功能色渐变
.gradient-success {
  background: linear-gradient(135deg, #52C41A 0%, #389E0D 100%);
}

.gradient-warning {
  background: linear-gradient(135deg, #FAAD14 0%, #D48806 100%);
}
```

### 字体系统

#### 字体家族
```less
// 主字体
@font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto,
  'Helvetica Neue', Arial, 'Noto Sans', sans-serif,
  'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol';

// 代码字体
@font-family-code: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, Courier, monospace;
```

#### 字号规范
| 类型 | 字号 | 行高 | 权重 | 变量名 | 说明 |
|------|------|------|------|--------|------|
| H1标题 | 24px | 1.4 | 600 | @font-size-h1 | 页面主标题 |
| H2标题 | 20px | 1.4 | 600 | @font-size-h2 | 区块标题 |
| H3标题 | 18px | 1.4 | 600 | @font-size-h3 | 小标题 |
| H4标题 | 16px | 1.5 | 600 | @font-size-h4 | 段落标题 |
| 正文 | 14px | 1.5 | 400 | @font-size-base | 正文内容 |
| 辅助文字 | 12px | 1.5 | 400 | @font-size-sm | 辅助说明 |
| 小文字 | 10px | 1.5 | 400 | @font-size-xs | 极小文字 |

#### 字重规范
| 权重 | 说明 | 使用场景 |
|------|------|----------|
| 400 | Regular | 正文、辅助文字 |
| 500 | Medium | 次标题 |
| 600 | SemiBold | 主标题、强调文字 |
| 700 | Bold | 重要标题 |

### 间距系统

#### 基础间距
| 级别 | 像素 | 变量名 | 说明 |
|------|------|--------|------|
| xs | 4px | @spacing-xs | 极小间距 |
| sm | 8px | @spacing-sm | 小间距 |
| md | 16px | @spacing-md | 中间距 |
| lg | 24px | @spacing-lg | 大间距 |
| xl | 32px | @spacing-xl | 极大间距 |
| xxl | 48px | @spacing-xxl | 超大间距 |

#### 组件间距
```less
// 按钮间距
@button-padding: 8px 16px;
@button-padding-sm: 4px 8px;
@button-padding-lg: 12px 24px;

// 卡片间距
@card-padding: 24px;
@card-padding-sm: 16px;

// 表格间距
@table-cell-padding: 16px;
@table-cell-padding-sm: 8px;
```

### 尺寸系统

#### 按钮尺寸
| 尺寸 | 高度 | 内边距 | 字号 | 说明 |
|------|------|--------|------|------|
| Small | 24px | 4px 8px | 12px | 小按钮 |
| Default | 32px | 8px 16px | 14px | 默认按钮 |
| Large | 40px | 12px 24px | 16px | 大按钮 |

#### 输入框尺寸
| 尺寸 | 高度 | 内边距 | 字号 | 说明 |
|------|------|--------|------|------|
| Small | 24px | 4px 8px | 12px | 小输入框 |
| Default | 32px | 8px 12px | 14px | 默认输入框 |
| Large | 40px | 12px 16px | 16px | 大输入框 |

#### 圆角规范
| 类型 | 像素 | 变量名 | 说明 |
|------|------|--------|------|
| 无圆角 | 0px | @border-radius-zero | 直角 |
| 小圆角 | 2px | @border-radius-sm | 小圆角 |
| 默认圆角 | 4px | @border-radius-base | 默认圆角 |
| 大圆角 | 8px | @border-radius-lg | 大圆角 |
| 胶囊 | 32px | @border-radius-pill | 胶囊形 |

### 阴影系统

#### 阴影级别
| 级别 | 像素 | 变量名 | 说明 |
|------|------|--------|------|
| 1级 | 0 2px 8px rgba(0, 0, 0, 0.08) | @shadow-1 | 卡片阴影 |
| 2级 | 0 4px 12px rgba(0, 0, 0, 0.12) | @shadow-2 | 弹窗阴影 |
| 3级 | 0 8px 24px rgba(0, 0, 0, 0.16) | @shadow-3 | 抽屉阴影 |

#### 特殊阴影
```less
// 卡片阴影
.card-shadow {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

// 弹窗阴影
.modal-shadow {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.12);
}

// 悬浮阴影
.hover-shadow {
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);
  transition: box-shadow 0.3s;
}
```

## 响应式设计

### 断点定义
| 设备 | 断点 | 变量名 | 说明 |
|------|------|--------|------|
| 手机 | < 576px | @screen-xs | 小屏手机 |
| 手机 | ≥ 576px | @screen-sm | 大屏手机 |
| 平板 | ≥ 768px | @screen-md | 平板 |
| PC | ≥ 992px | @screen-lg | 桌面 |
| 大屏 | ≥ 1200px | @screen-xl | 大屏桌面 |

### 响应式布局
```less
// 响应式容器
.container {
  width: 100%;
  padding: 0 @spacing-md;
  
  @media (min-width: @screen-sm) {
    max-width: 540px;
  }
  
  @media (min-width: @screen-md) {
    max-width: 720px;
  }
  
  @media (min-width: @screen-lg) {
    max-width: 960px;
  }
  
  @media (min-width: @screen-xl) {
    max-width: 1140px;
  }
}

// 响应式栅格
.row {
  display: flex;
  flex-wrap: wrap;
}

.col {
  flex: 1;
  
  @media (max-width: @screen-md) {
    flex: 0 0 100%;  // 移动端单列
  }
}

// 响应式隐藏
.hide-mobile {
  @media (max-width: @screen-md) {
    display: none;
  }
}

.hide-desktop {
  @media (min-width: @screen-md) {
    display: none;
  }
}
```

### 适配策略
| 场景 | 手机 | 平板 | PC |
|------|------|------|-----|
| 列表页 | 单列卡片 | 双列卡片 | 表格 |
| 表单页 | 全屏表单 | 全屏表单 | 居中表单 |
| 详情页 | 全屏详情 | 全屏详情 | 侧边详情 |
| 导航 | 底部导航 | 侧边导航 | 顶部导航 |

## 组件规范

### 按钮规范
```less
// 按钮类型
.btn-primary {
  background: @primary-color;
  color: @white-color;
  border-color: @primary-color;
  
  &:hover {
    background: @primary-color-hover;
    border-color: @primary-color-hover;
  }
  
  &:active {
    background: @primary-color-active;
    border-color: @primary-color-active;
  }
  
  &:disabled {
    background: @disabled-color;
    border-color: @disabled-color;
    cursor: not-allowed;
  }
}

.btn-default {
  background: @white-color;
  color: @text-color;
  border-color: @border-color;
  
  &:hover {
    color: @primary-color;
    border-color: @primary-color;
  }
}

.btn-danger {
  background: @error-color;
  color: @white-color;
  border-color: @error-color;
}

// 按钮图标
.btn-icon {
  padding: 8px;
  
  .anticon {
    font-size: 16px;
  }
}
```

### 输入框规范
```less
// 输入框样式
.input {
  height: 32px;
  padding: 8px 12px;
  border: 1px solid @border-color;
  border-radius: @border-radius-base;
  font-size: @font-size-base;
  
  &:hover {
    border-color: @primary-color-hover;
  }
  
  &:focus {
    border-color: @primary-color;
    box-shadow: 0 0 0 2px rgba(24, 144, 255, 0.2);
  }
  
  &:disabled {
    background: @disabled-color;
    cursor: not-allowed;
  }
  
  &.error {
    border-color: @error-color;
    
    &:focus {
      box-shadow: 0 0 0 2px rgba(245, 34, 45, 0.2);
    }
  }
}

// 输入框前缀/后缀
.input-prefix,
.input-suffix {
  background: @background-color;
  border-right: 1px solid @border-color;
  padding: 8px 12px;
}
```

### 表格规范
```less
// 表格样式
.table {
  width: 100%;
  border-collapse: collapse;
  
  th {
    background: @background-color;
    padding: 16px;
    font-weight: 600;
    border-bottom: 1px solid @border-color;
  }
  
  td {
    padding: 16px;
    border-bottom: 1px solid @border-color-secondary;
  }
  
  tr:hover td {
    background: @primary-color-bg;
  }
}

// 表格固定列
.table-fixed {
  th, td {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
}
```

### 卡片规范
```less
// 卡片样式
.card {
  background: @white-color;
  border-radius: @border-radius-lg;
  box-shadow: @shadow-1;
  padding: @card-padding;
  
  .card-header {
    padding-bottom: @spacing-md;
    border-bottom: 1px solid @divider-color;
    
    .card-title {
      font-size: @font-size-h4;
      font-weight: 600;
    }
  }
  
  .card-body {
    padding: @spacing-md 0;
  }
  
  .card-footer {
    padding-top: @spacing-md;
    border-top: 1px solid @divider-color;
  }
}
```

## 图标规范

### 图标尺寸
| 尺寸 | 像素 | 说明 |
|------|------|------|
| Small | 12px | 小图标 |
| Default | 16px | 默认图标 |
| Large | 24px | 大图标 |
| Extra | 32px | 超大图标 |

### 图标颜色
| 场景 | 颜色 | 说明 |
|------|------|------|
| 默认 | @text-color | 默认图标 |
| 主色 | @primary-color | 强调图标 |
| 成功 | @success-color | 成功图标 |
| 警告 | @warning-color | 警告图标 |
| 错误 | @error-color | 错误图标 |
| 禁用 | @disabled-color | 禁用图标 |

## 动效规范

### 动效时长
| 类型 | 时长 | 变量名 | 说明 |
|------|------|--------|------|
| 微动效 | 100ms | @duration-fast | Hover、Click反馈 |
| 短动效 | 200ms | @duration-base | 通用过渡 |
| 中动效 | 300ms | @duration-normal | 弹窗展开 |
| 长动效 | 500ms | @duration-slow | 页面切换 |

### 动效曲线
| 曲线 | 值 | 说明 |
|------|-----|------|
| ease | cubic-bezier(0.25, 0.1, 0.25, 1) | 通用 |
| ease-in | cubic-bezier(0.42, 0, 1, 1) | 进入 |
| ease-out | cubic-bezier(0, 0, 0.58, 1) | 退出 |
| ease-in-out | cubic-bezier(0.42, 0, 0.58, 1) | 进出 |

### 常用动效
```less
// 淡入淡出
.fade-enter {
  opacity: 0;
}
.fade-enter-active {
  opacity: 1;
  transition: opacity @duration-normal;
}

// 缩放
.scale-enter {
  transform: scale(0.9);
  opacity: 0;
}
.scale-enter-active {
  transform: scale(1);
  opacity: 1;
  transition: all @duration-normal ease-out;
}

// 滑动
.slide-up-enter {
  transform: translateY(100%);
}
.slide-up-enter-active {
  transform: translateY(0);
  transition: transform @duration-normal ease-out;
}
```

## 设计交付

### 设计稿命名
```
{项目}-{模块}-{页面}-{状态}

例如：
product-order-list-default.figma
product-order-detail-loading.figma
product-order-create-error.figma
```

### 设计稿标注
| 内容 | 说明 |
|------|------|
| 尺寸 | 宽高、间距、字号 |
| 颜色 | 艱值、变量名 |
| 字体 | 字号、字重、行高 |
| 状态 | Default/Hover/Active/Disabled |
| 交互 | 动效、过渡 |

### 设计稿切图
| 类型 | 格式 | 说明 |
|------|------|------|
| 图标 | SVG |矢量图标 |
| 图片 | PNG/WebP | 位图图片 |
| 背景 | PNG/WebP | 背景图 |

## DO - 推荐做法

```less
// ✓ 使用变量定义颜色
@primary-color: #1890FF;
.btn { background: @primary-color; }

// ✓ 遵循间距系统
padding: @spacing-md;  // 16px

// ✓ 遵循字号系统
font-size: @font-size-base;  // 14px

// ✓ 遵循响应式断点
@media (min-width: @screen-md) { }

// ✓ 状态完整定义
.btn {
  &:hover { }
  &:active { }
  &:disabled { }
}

// ✓ 动效时长合理
transition: all @duration-normal;  // 300ms
```

## DON'T - 禁止做法

```less
// ✗ 硬编码颜色
.btn { background: #1890FF; }  // 禁止

// ✓ 正确做法
.btn { background: @primary-color; }

// ✗ 间距不规范
padding: 15px;  // 禁止，不符合间距系统

// ✓ 正确做法
padding: @spacing-md;  // 16px

// ✗ 字号不规范
font-size: 13px;  // 禁止

// ✓ 正确做法
font-size: @font-size-base;  // 14px

// ✗ 无响应式设计
// 无 @media 定义  // 禁止

// ✓ 正确做法
@media (max-width: @screen-md) { }

// ✗ 状态定义不完整
.btn:hover { }
// 缺少 active/disabled  // 禁止

// ✓ 正确做法
.btn {
  &:hover { }
  &:active { }
  &:disabled { }
}

// ✗ 动效时长过长
transition: all 1s;  // 禁止，太慢

// ✓ 正确做法
transition: all @duration-normal;  // 300ms
```

## 最佳实践

### 设计一致性
- 使用统一颜色变量
- 使用统一间距系统
- 使用统一字号系统
- 状态定义完整

### 响应式优先
- 移动端优先设计
- 断点使用规范
- 适配策略明确

### 交付标准化
- 设计稿命名规范
- 标注完整清晰
- 切图格式正确