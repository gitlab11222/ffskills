# 前端编码规则

## HTTP 请求

### 统一请求工具
```typescript
// 只使用 @/utils/request 导出的 axios 实例
// 禁止使用 Umi 内置 request 或其他请求库

import { request } from '@/utils/request';

// API 函数放在页面同级 service.ts
export async function getUserList(params: UserListParams) {
  return request.post<DataResponse<UserListResponse>>('/api/user/list', params);
}

export async function getUserDetail(id: number) {
  return request.get<DataResponse<UserDto>>(`/api/user/${id}`);
}

export async function createUser(data: CreateUserRequest) {
  return request.post<DataResponse<number>>('/api/user/create', data);
}

export async function updateUser(data: UpdateUserRequest) {
  return request.post<DataResponse<boolean>>('/api/user/update', data);
}
```

### 请求配置
```typescript
// src/utils/request.ts
import axios from 'axios';

const request = axios.create({
  baseURL: '/api',
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// 请求拦截器
request.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// 响应拦截器
request.interceptors.response.use(
  (response) => {
    const { data } = response;
    // code !== 0 时显示错误信息
    if (data.code !== 0) {
      message.error(data.message);
      return Promise.reject(new Error(data.message));
    }
    return data;
  },
  (error) => {
    if (error.response?.status === 401) {
      // 跳转登录
      history.push('/login');
    }
    return Promise.reject(error);
  }
);

export { request };
```

## 组件使用

### 列表页 - MyProTable
```typescript
// 禁止直接使用 ProTable，必须使用 MyProTable
import MyProTable from '@/components/MyProTable';

const UserList: React.FC = () => {
  const actionRef = useRef<ActionType>();
  const access = useAccess();

  const columns: ProColumns<API.User>[] = [
    {
      title: '用户名',
      dataIndex: 'userName',
      width: 120,  // 列必须指定 width
    },
    {
      title: '状态',
      dataIndex: 'status',
      width: 100,
      valueEnum: UserStatusMap,  // 使用配对的 Map
    },
    {
      title: '创建时间',
      dataIndex: 'createdAt',
      width: 180,
      valueType: 'dateTime',
    },
    {
      title: '操作',
      valueType: 'option',
      width: 150,
      render: (_, record) => [
        <a key="edit" onClick={() => history.push(`/user/edit/${record.id}`)}>
          编辑
        </a>,
        <Popconfirm
          key="delete"
          title="确定删除？"
          onConfirm={() => handleDelete(record.id)}
        >
          <a>删除</a>
        </Popconfirm>,
      ],
    },
  ];

  return (
    <MyProTable<API.User>
      actionRef={actionRef}
      columns={columns}
      request={async (params) => {
        const res = await getUserList({
          page: params.current,
          pageSize: params.pageSize,
          ...params,
        });
        return {
          data: res.data?.list || [],
          total: res.data?.total || 0,
          success: res.code === 0,
        };
      }}
      rowKey="id"
      search={{ labelWidth: 'auto' }}
      options={{ reload: true, density: true, setting: true }}
    />
  );
};
```

### 弹窗表单 - ModalForm
```typescript
import { ModalForm, ProFormText, ProFormSelect } from '@ant-design/pro-components';

const CreateModal: React.FC<{ trigger: React.ReactNode; onSuccess: () => void }> = ({
  trigger,
  onSuccess,
}) => {
  return (
    <ModalForm
      title="新建用户"
      trigger={trigger}
      width={600}
      onFinish={async (values) => {
        const res = await createUser(values);
        if (res.code === 0) {
          message.success('创建成功');
          onSuccess();
          return true;
        }
        return false;
      }}
    >
      <ProFormText
        name="userName"
        label="用户名"
        rules={[{ required: true, message: '请输入用户名' }]}
      />
      <ProFormSelect
        name="status"
        label="状态"
        valueEnum={UserStatusMap}
        rules={[{ required: true }]}
      />
    </ModalForm>
  );
};
```

### 文件上传 - MyUpload
```typescript
import MyUpload from '@/components/MyUpload';

<ProForm.Item name="avatar" label="头像">
  <MyUpload
    accept="image/*"
    maxSize={2 * 1024 * 1024}  // 2MB
    maxCount={1}
  />
</ProForm.Item>
```

## 页面文件结构

```
pages/
└── user/
    ├── index.tsx          # 列表页
    ├── create.tsx         # 创建/编辑页
    ├── service.ts         # API 接口
    ├── data.d.ts          # 类型定义
    ├── enum.ts            # 枚举定义
    └── components/        # 页面组件
        └── UserDetail.tsx
```

## 类型定义

### 页面级类型
```typescript
// data.d.ts - 使用 declare namespace
declare namespace API {
  interface User {
    id: number;
    userName: string;
    status: UserStatus;
    createdAt: string;
    updatedAt?: string;
  }

  interface UserListParams {
    page: number;
    pageSize: number;
    userName?: string;
    status?: UserStatus;
  }

  interface UserListResponse {
    list: User[];
    total: number;
  }

  interface CreateUserRequest {
    userName: string;
    status: UserStatus;
  }
}
```

### 全局类型
```typescript
// src/global.d.ts
declare namespace API {
  interface DataResponse<T = any> {
    code: number;
    message: string;
    messageType: number;
    data: T;
  }

  interface PagedList<T> {
    list: T[];
    total: number;
    pageIndex: number;
    pageSize: number;
  }
}
```

## 枚举模式

### 枚举定义
```typescript
// enum.ts - 中文标签 + 配对 Map

// 枚举定义（标签中文）
export enum UserStatus {
  Active = 1,
  Inactive = 2,
  Banned = 3,
}

// 配对 Map（用于 ProTable valueEnum）
export const UserStatusMap = new Map<UserStatus, HasBadgeStatusType>([
  [UserStatus.Active, { text: '正常', status: 'Success' }],
  [UserStatus.Inactive, { text: '未激活', status: 'Warning' }],
  [UserStatus.Banned, { text: '已禁用', status: 'Error' }],
]);

// 下拉选项
export const UserStatusOptions = [
  { label: '正常', value: UserStatus.Active },
  { label: '未激活', value: UserStatus.Inactive },
  { label: '已禁用', value: UserStatus.Banned },
];
```

### 枚举使用
```typescript
// ProTable 列定义
{
  title: '状态',
  dataIndex: 'status',
  valueEnum: UserStatusMap,
}

// ProFormSelect
<ProFormSelect
  name="status"
  label="状态"
  options={UserStatusOptions}
/>

// 代码判断
if (user.status === UserStatus.Active) {
  // ...
}
```

## 路由配置

### 路由定义
```typescript
// routes.ts - 定义在根目录
export default [
  {
    path: '/user',
    name: '用户管理',
    icon: 'UserOutlined',
    routes: [
      { path: '/user/list', name: '用户列表', component: './user' },
      { path: '/user/create', name: '新建用户', component: './user/create', hideInMenu: true },
      { path: '/user/edit/:id', name: '编辑用户', component: './user/create', hideInMenu: true },
    ],
  },
];
```

### 路由规范
- 路径使用小写 camelCase
- 新建/编辑使用同一组件（create.tsx）
- 编辑页面使用 `:id` 参数

## 状态管理

### 全局状态
```typescript
// 使用 Umi @@initialState
// src/app.ts
export async function getInitialState(): Promise<{
  currentUser?: API.User;
  permissions?: string[];
}> {
  const currentUser = await fetchCurrentUser();
  const permissions = await fetchPermissions();
  return { currentUser, permissions };
}

// 组件中使用
const { initialState } = useModel('@@initialState');
```

### 局部状态
```typescript
// 使用 ahooks
import { useRequest, useBoolean, useDebounceFn } from 'ahooks';

const { data, loading, run } = useRequest(getUserList, {
  manual: true,
});
```

### 禁止使用
- Redux / MobX / Zustand（项目统一使用 Umi + ahooks）

## 权限控制

### 权限定义
```typescript
// src/access.ts
export default function (initialState: { permissions?: string[] }) {
  const { permissions } = initialState || {};
  return {
    hasPermissions: (code: string) => permissions?.includes(code),
  };
}
```

### 权限使用
```typescript
// 组件中使用
const access = useAccess();

// 按钮权限
{access.hasPermissions('user:create') && (
  <Button type="primary">新建用户</Button>
)}

// 路由权限
{
  path: '/user',
  access: 'hasPermissions',
  accessParams: ['user:view'],
}
```

## 表单模式

### 新建/编辑共用组件
```typescript
// create.tsx
import { ProForm, ProFormText } from '@ant-design/pro-components';

const UserForm: React.FC = () => {
  const params = useParams<{ id }>();
  const isEdit = !!params.id;

  // 编辑时获取详情
  const { data: userDetail } = useRequest(
    () => getUserDetail(Number(params.id)),
    { ready: isEdit, refreshDeps: [params.id] }
  );

  return (
    <ProForm
      initialValues={userDetail?.data}
      onFinish={async (values) => {
        const res = isEdit
          ? await updateUser({ ...values, id: Number(params.id) })
          : await createUser(values);
        if (res.code === 0) {
          message.success(isEdit ? '编辑成功' : '创建成功');
          history.back();
        }
      }}
    >
      <ProFormText name="userName" label="用户名" rules={[{ required: true }]} />
    </ProForm>
  );
};
```

## 代码风格

### 配置
- 包管理器：pnpm
- 代码风格：ESLint + Prettier
- 缩进：2 空格
- 引号：单引号
- 分号：不加分号
- 样式：CSS Modules

### 示例
```typescript
// ✓ 正确
import { useState } from 'react'
import styles from './index.less'

const UserList: React.FC = () => {
  const [loading, setLoading] = useState(false)
  return <div className={styles.container}>...</div>
}

// ✗ 错误
import { useState } from "react";
import './index.css'

const UserList: React.FC = () => {
  const [loading, setLoading] = useState(false);
  return <div className="container">...</div>
}
```

## 环境配置

### 运行时配置注入
```html
<!-- public/configs.js -->
window.appConfigs = {
  apiBaseURL: '/api',
  wsURL: 'ws://localhost:8080',
};
```

```typescript
// src/app.ts
declare global {
  interface Window {
    appConfigs: {
      apiBaseURL: string;
      wsURL: string;
    };
  }
}

// 使用
const { apiBaseURL } = window.appConfigs;
```

### 禁止构建时硬编码
```typescript
// ✗ 错误 - 构建时写死
const apiUrl = process.env.API_URL;

// ✓ 正确 - 运行时注入
const apiUrl = window.appConfigs.apiBaseURL;
```