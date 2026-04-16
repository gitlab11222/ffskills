# 交互设计规范

## 交互原则

### 反馈原则
| 原则 | 说明 |
|------|------|
| 即时性 | 操作后100ms内反馈 |
| 明确性 | 反馈内容清晰明确 |
| 相关性 | 反馈与操作直接相关 |
| 一致性 | 相同操作反馈一致 |

### 可控原则
| 原则 | 说明 |
|------|------|
| 可预测 | 用户能预测操作结果 |
| 可撤销 | 错误操作可撤销 |
| 可中断 | 长操作可中断 |
| 可恢复 | 异常状态可恢复 |

### 简化原则
| 原则 | 说明 |
|------|------|
| 减少步骤 | 最简路径完成任务 |
| 减少输入 | 自动填充、默认值 |
| 减少选择 | 合理默认值 |
| 减少记忆 | 信息可见不隐藏 |

## 反馈设计

### 反馈时机
| 操作 | 反馈 | 延迟 | 形式 |
|------|------|------|------|
| 按钮点击 | Loading | < 100ms | 按钮状态变化 |
| 表单提交 | 进度条 | < 200ms | 进度展示 |
| 数据保存 | Toast | < 500ms | 提示消息 |
| 文件上传 | 进度条 | 立即 | 进度百分比 |
| 操作成功 | Toast | 立即 | 2s自动消失 |
| 操作失败 | Modal | 立即 | 需手动关闭 |
| 网络异常 | Toast | 立即 | 廦续提示 |

### Loading 状态
```typescript
// 按钮Loading
<Button type="primary" loading={loading}>
  {loading ? '提交中...' : '提交'}
</Button>

// 页面Loading
<Spin size="large" tip="加载中...">
  <Content />
</Spin>

// 列表Loading
<Skeleton active paragraph={{ rows: 5 }} />

// 卡片Loading
<Card loading={loading}>
  <Content />
</Card>
```

### Toast 提示
```typescript
// 成功提示
message.success('操作成功', 2);  // 2秒消失

// 错误提示
message.error('操作失败，请稍后重试', 3);  // 3秒消失

// 警告提示
message.warning('数据未保存，请确认');

// 信息提示
message.info('新功能已上线');

// 加载提示
const hideLoading = message.loading('正在处理...', 0);
// 完成后关闭
hideLoading();
message.success('处理完成');
```

### Modal 确认
```typescript
// 删除确认
Modal.confirm({
  title: '确认删除？',
  content: '删除后数据无法恢复',
  okText: '确认',
  okType: 'danger',
  cancelText: '取消',
  onOk: () => handleDelete(),
});

// 危险操作确认
Modal.warning({
  title: '风险提示',
  content: '此操作可能导致数据丢失',
  okText: '我已了解风险',
});
```

## 表单交互

### 输入反馈
```typescript
// 实时校验
<Input
  status={error ? 'error' : ''}
  onChange={(e) => validate(e.target.value)}
/>

// 输入提示
<ProFormText
  name="phone"
  label="手机号"
  placeholder="请输入11位手机号"
  rules={[
    { required: true, message: '请输入手机号' },
    { pattern: /^1[3-9]\d{9}$/, message: '手机号格式不正确' },
  ]}
/>
```

### 自动填充
```typescript
// 搜索自动完成
<AutoComplete
  options={searchOptions}
  onSearch={handleSearch}
  placeholder="输入关键字搜索"
/>

// 地址自动填充
<Input
  value={address}
  onChange={(e) => fetchAddressSuggestions(e.target.value)}
/>
```

### 表单保存
```typescript
// 自动保存（草稿）
const autoSave = useDebounceFn(() => {
  saveDraft(formValues);
}, { wait: 3000 });

// 手动保存提示
if (hasUnsavedChanges) {
  Modal.confirm({
    title: '有未保存的数据',
    content: '离开页面将丢失未保存的数据',
    okText: '保存',
    cancelText: '不保存',
  });
}
```

### 批量操作
```typescript
// 批量选择
<Table
  rowSelection={{
    selectedRowKeys,
    onChange: setSelectedRowKeys,
  }}
/>

// 批量操作按钮
<Button
  disabled={selectedRowKeys.length === 0}
  onClick={handleBatchAction}
>
  批量操作 ({selectedRowKeys.length})
</Button>
```

## 列表交互

### 空状态
```typescript
// 无数据
<Empty
  image={Empty.PRESENTED_IMAGE_SIMPLE}
  description="暂无数据"
>
  <Button type="primary">新建</Button>
</Empty>

// 搜索无结果
<Empty
  description={
    <span>
      未找到"<Text strong>{searchKeyword}</Text>"相关结果
      <Button onClick={clearSearch}>清除搜索</Button>
    </span>
  }
/>
```

### 无限滚动
```typescript
// 无限滚动列表
<List
  dataSource={list}
  renderItem={(item) => <ListItem data={item} />}
  loadMore={
    <div style={{ textAlign: 'center' }}>
      {hasMore && (
        <Button onClick={loadMore}>加载更多</Button>
      )}
    </div>
  }
/>
```

### 下拉刷新
```typescript
// 移动端下拉刷新
<PullToRefresh
  onRefresh={handleRefresh}
  refreshing={refreshing}
>
  <List />
</PullToRefresh>
```

## 弹窗交互

### Modal 规范
| 类型 | 用途 | 尺寸 | 关闭方式 |
|------|------|------|----------|
| 确认框 | 二选一确认 | 400px | 按钮/ESC |
| 信息框 | 信息展示 | 600px | 按钮/X |
| 表单框 | 表单填写 | 800px | 按钮/X |
| 全屏框 | 复杂内容 | 100% | 按钮 |

### Drawer 规范
| 类型 | 用途 | 宽度 | 关闭方式 |
|------|------|------|----------|
| 信息抽屉 | 信息展示 | 400px | X/ESC |
| 表单抽屉 | 表单填写 | 600px | X/ESC |
| 大抽屉 | 复杂内容 | 800px | X/ESC |

### 弹窗代码
```typescript
// Modal弹窗
<Modal
  title="新建用户"
  open={visible}
  width={600}
  onCancel={handleClose}
  onOk={handleSubmit}
  confirmLoading={loading}
  destroyOnClose  // 关闭时销毁
  maskClosable={false}  // 禁止点击遮罩关闭
>
  <Form />
</Modal>

// Drawer抽屉
<Drawer
  title="用户详情"
  open={visible}
  width={600}
  onClose={handleClose}
  destroyOnClose
>
  <Detail />
</Drawer>
```

## 动效设计

### 动效时长
| 类型 | 时长 | 说明 |
|------|------|------|
| Hover反馈 | 100-200ms | 悬浮变色 |
| 按钮点击 | 100-200ms | 点击缩放 |
| Toast出现 | 200ms | 淡入 |
| Toast消失 | 200ms | 淡出 |
| 弹窗展开 | 300-500ms | 缩放+淡入 |
| 弹窗关闭 | 200-300ms | 缩放+淡出 |
| 抽屉展开 | 300-400ms | 滑动 |
| 抽屉关闭 | 200-300ms | 滑动 |
| 页面切换 | 300-400ms | 淡入淡出 |

### 动效代码
```typescript
// CSS过渡
.fade-enter {
  opacity: 0;
}
.fade-enter-active {
  opacity: 1;
  transition: opacity 200ms ease-out;
}
.fade-exit {
  opacity: 1;
}
.fade-exit-active {
  opacity: 0;
  transition: opacity 200ms ease-in;
}

// React过渡
<CSSTransition
  in={visible}
  timeout={200}
  classNames="fade"
  unmountOnExit
>
  <Modal />
</CSSTransition>

// Framer Motion
<motion.div
  initial={{ opacity: 0, scale: 0.9 }}
  animate={{ opacity: 1, scale: 1 }}
  exit={{ opacity: 0, scale: 0.9 }}
  transition={{ duration: 0.3 }}
>
  <Content />
</motion.div>
```

### 禁用动效
```typescript
// 无障碍：尊重用户动效偏好
const prefersReducedMotion = window.matchMedia(
  '(prefers-reduced-motion: reduce)'
).matches;

if (prefersReducedMotion) {
  // 禁用动效
  return <div>{content}</div>;
} else {
  // 启用动效
  return <motion.div>{content}</motion.div>;
}
```

## 错误处理

### 错误提示位置
| 类型 | 位置 | 形式 |
|------|------|------|
| 表单校验错误 | 输入框下方 | 红色文字 |
| 接口请求错误 | Toast | 错误提示 |
| 业务逻辑错误 | Toast | 错误提示 |
| 系统异常 | Modal | 详细说明 |
| 网络异常 | Toast | 廦续提示 |

### 错误恢复
```typescript
// 错误边界
<ErrorBoundary
  fallback={<ErrorPage onRetry={handleRetry} />}
  onError={(error) => logError(error)}
>
  <App />
</ErrorBoundary>

// 请求错误处理
try {
  const result = await api.createOrder(params);
  message.success('创建成功');
} catch (error) {
  if (error.code === 400) {
    message.error('参数错误，请检查输入');
  } else if (error.code === 1001) {
    message.error('登录已过期，请重新登录');
    history.push('/login');
  } else {
    message.error('系统异常，请稍后重试');
  }
}

// 网络异常重试
const { data, error, retry } = useRequest(api.getData, {
  retryCount: 3,
  retryInterval: 1000,
});
if (error) {
  return <ErrorState onRetry={retry} />;
}
```

## 键盘交互

### 快捷键
| 快捷键 | 功能 | 说明 |
|--------|------|------|
| Enter | 提交表单 | 表单提交 |
| ESC | 关闭弹窗 | Modal/Drawer关闭 |
| Tab | 切换焦点 | 表单切换 |
| Ctrl+S | 保存 | 保存当前内容 |
| Ctrl+C | 复制 | 复制选中内容 |
| Ctrl+V | 粘贴 | 粘贴内容 |

### 键盘导航
```typescript
// Tab焦点管理
<Input autoFocus />  // 自动聚焦
<Input ref={inputRef} />
inputRef.current.focus();  // 手动聚焦

// Tab顺序
<Input tabIndex={1} />
<Input tabIndex={2} />
<Input tabIndex={3} />

// 禁用Tab
<Input tabIndex={-1} />

// 键盘事件
<Input
  onKeyDown={(e) => {
    if (e.key === 'Enter') {
      handleSubmit();
    }
  }}
/>
```

## 手势交互

### 手势类型
| 手势 | 用途 | 说明 |
|------|------|------|
| 点击 | 选择/提交 | 单击 |
| 双击 | 放大/选择 | 图片放大 |
| 长按 | 菜单/选择 | 显示操作菜单 |
| 滑动 | 切换/滚动 | 页面切换 |
| 拖拽 | 移动/排序 | 拖动排序 |
| 缩放 | 放大/缩小 | 图片缩放 |

### 手势代码
```typescript
// 长按
<Pressable
  onLongPress={() => showMenu()}
  delayLongPress={500}
>
  <Card />
</Pressable>

// 滑动
<Swipeable
  onSwipedLeft={() => nextPage()}
  onSwipedRight={() => prevPage()}
>
  <Page />
</Swipeable>

// 拖拽排序
<SortableList
  items={items}
  onSortEnd={(newItems) => setItems(newItems)}
/>
```

## DO - 推荐做法

```typescript
// ✓ 操作即时反馈
<Button loading={loading}>提交</Button>

// ✓ Toast自动消失
message.success('操作成功', 2);

// ✓ 确认危险操作
Modal.confirm({
  title: '确认删除？',
  okType: 'danger',
});

// ✓ 动效时长合理
transition: opacity 200ms ease-out;

// ✓ 空状态有引导
<Empty>
  <Button>新建</Button>
</Empty>

// ✓ 错误可恢复
<ErrorState onRetry={retry} />

// ✓ 快捷键支持
onKeyDown={(e) => {
  if (e.key === 'Enter') handleSubmit();
}

// ✓ 无障碍动效偏好
if (prefersReducedMotion) {
  // 禁用动效
}
```

## DON'T - 禁止做法

```typescript
// ✗ 反馈延迟过大
<Button onClick={async () => {
  await api.submit();
  setLoading(true);  // 禁止，反馈太晚
}}

// ✓ 正确做法
<Button loading={loading} onClick={handleSubmit}>

// ✗ Toast不消失
message.error('错误', 0);  // 禁止，持续显示

// ✓ 正确做法
message.error('错误', 3);

// ✗ 危险操作无确认
<Button onClick={handleDelete}>删除</Button>  // 禁止

// ✓ 正确做法
Modal.confirm({ okType: 'danger' });

// ✗ 动效时长过长
transition: opacity 1s;  // 禁止，太慢

// ✓ 正确做法
transition: opacity 200ms ease-out;

// ✗ 空状态无引导
<Empty description="暂无数据" />  // 禁止

// ✓ 正确做法
<Empty>
  <Button>新建</Button>
</Empty>

// ✗ 错误无恢复方案
message.error('系统异常');  // 禁止，无重试

// ✓ 正确做法
<ErrorState onRetry={retry} />

// ✗ 无键盘支持
// 无 onKeyDown  // 禁止

// ✓ 正确做法
onKeyDown={(e) => {
  if (e.key === 'Enter') handleSubmit();
}

// ✗ 无障碍忽略
// 无动效偏好检查  // 禁止

// ✓ 正确做法
if (prefersReducedMotion) {
  // 禁用动效
}
```

## 最佳实践

### 反馈设计
- 操作100ms内反馈
- Loading状态明确
- Toast时长合理
- 错误有恢复方案

### 表单设计
- 实时校验反馈
- 自动填充辅助
- 草稿自动保存
- 离开提示未保存

### 动效设计
- 时长200-300ms
- 曲线ease-out
- 尊重用户偏好
- 状态切换平滑