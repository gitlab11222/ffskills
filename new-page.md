---
name: new-page
description: 创建前端页面（列表页 + 表单页）
userInvoke: true
---

# 前端页面生成 Skill

快速创建前端页面，包含列表页、表单页、路由配置。

## 使用方式

```
/new-page <页面名> <字段列表>
```

示例：
```
/new-page user name:string,email:string,status:int
/new-page product name:string,price:decimal,stock:int,category_id:long
/new-page order user_id:long,total_amount:decimal,status:int,created_at:datetime
```

## 页面文件结构

```
pages/{page}/
├── index.tsx        # 列表页
├── create.tsx       # 创建/编辑页
├── service.ts       # API 函数
├── data.d.ts        # 类型定义
├── enum.ts          # 枚举定义
└── components/      # 页面组件
    └── SearchForm.tsx
```

## 执行步骤

### 1. 创建类型定义 (data.d.ts)
```typescript
declare namespace API {
  interface {Entity} {
    id: number;
    // 字段定义...
    createdAt: string;
  }

  interface {Entity}ListParams {
    page: number;
    pageSize: number;
    // 查询参数...
  }

  interface {Entity}ListResponse {
    list: {Entity}[];
    total: number;
  }

  interface Create{Entity}Request {
    // 创建参数...
  }
}
```

### 2. 创建枚举定义 (enum.ts)
```typescript
export enum {Entity}Status {
  Active = 1,
  Inactive = 2,
}

export const {Entity}StatusMap = new Map<{Entity}Status, HasBadgeStatusType>([
  [{Entity}Status.Active, { text: '正常', status: 'Success' }],
  [{Entity}Status.Inactive, { text: '禁用', status: 'Error' }],
]);

export const {Entity}StatusOptions = [
  { label: '正常', value: {Entity}Status.Active },
  { label: '禁用', value: {Entity}Status.Inactive },
];
```

### 3. 创建 API 函数 (service.ts)
```typescript
import { request } from '@/utils/request';

export async function get{Entity}List(params: API.{Entity}ListParams) {
  return request.post<API.DataResponse<API.{Entity}ListResponse>>('/api/{entity}/list', params);
}

export async function get{Entity}Detail(params: { id: number }) {
  return request.post<API.DataResponse<API.{Entity}>>('/api/{entity}/detail', params);
}

export async function create{Entity}(data: API.Create{Entity}Request) {
  return request.post<API.DataResponse<number>>('/api/{entity}/create', data);
}

export async function update{Entity}(data: API.Update{Entity}Request) {
  return request.post<API.DataResponse<boolean>>('/api/{entity}/update', data);
}

export async function delete{Entity}(params: { id: number }) {
  return request.post<API.DataResponse<boolean>>('/api/{entity}/delete', params);
}
```

### 4. 创建列表页 (index.tsx)
```typescript
import React, { useRef } from 'react';
import { history, useAccess } from 'umi';
import { Button, Popconfirm, message } from 'antd';
import MyProTable from '@/components/MyProTable';
import type { ActionType, ProColumns } from '@ant-design/pro-components';
import { get{Entity}List, delete{Entity} } from './service';
import { {Entity}StatusMap } from './enum';
import type { API } from './data.d';

const {Entity}List: React.FC = () => {
  const actionRef = useRef<ActionType>();
  const access = useAccess();

  const handleDelete = async (id: number) => {
    const res = await delete{Entity}({ id });
    if (res.code === 0) {
      message.success('删除成功');
      actionRef.current?.reload();
    }
  };

  const columns: ProColumns<API.{Entity}>[] = [
    {
      title: 'ID',
      dataIndex: 'id',
      width: 80,
      hideInSearch: true,
    },
    // 字段列（自动生成）...
    {
      title: '创建时间',
      dataIndex: 'createdAt',
      width: 180,
      valueType: 'dateTime',
      hideInSearch: true,
      sorter: true,
    },
    {
      title: '操作',
      valueType: 'option',
      width: 150,
      fixed: 'right',
      render: (_, record) => [
        <a key="edit" onClick={() => history.push(`/{entity}/edit/${record.id}`)}>
          编辑
        </a>,
        access.hasPermissions('{entity}:delete') && (
          <Popconfirm
            key="delete"
            title="确定删除？"
            onConfirm={() => handleDelete(record.id)}
          >
            <a>删除</a>
          </Popconfirm>
        ),
      ],
    },
  ];

  return (
    <MyProTable<API.{Entity}>
      actionRef={actionRef}
      columns={columns}
      request={async (params, sort) => {
        const res = await get{Entity}List({
          page: params.current,
          pageSize: params.pageSize,
          // 查询参数...
        });
        return {
          data: res.data?.list || [],
          total: res.data?.total || 0,
          success: res.code === 0,
        };
      }}
      rowKey="id"
      search={{
        labelWidth: 'auto',
      }}
      options={{
        reload: true,
        density: true,
        setting: true,
      }}
      toolBarRender={() => [
        access.hasPermissions('{entity}:create') && (
          <Button
            key="create"
            type="primary"
            onClick={() => history.push('/{entity}/create')}
          >
            新建
          </Button>
        ),
      ]}
    />
  );
};

export default {Entity}List;
```

### 5. 创建表单页 (create.tsx)
```typescript
import React from 'react';
import { useParams, history } from 'umi';
import { message, Spin } from 'antd';
import { ProForm, ProFormText, ProFormSelect, ProFormDigit } from '@ant-design/pro-components';
import { useRequest } from 'ahooks';
import { get{Entity}Detail, create{Entity}, update{Entity} } from './service';
import { {Entity}StatusOptions } from './enum';

const {Entity}Form: React.FC = () => {
  const params = useParams<{ id: string }>();
  const isEdit = !!params.id;

  const { data: detail, loading } = useRequest(
    () => get{Entity}Detail({ id: Number(params.id) }),
    { ready: isEdit, refreshDeps: [params.id] }
  );

  return (
    <Spin spinning={loading}>
      <ProForm
        initialValues={detail?.data}
        onFinish={async (values) => {
          const res = isEdit
            ? await update{Entity}({ ...values, id: Number(params.id) })
            : await create{Entity}(values);

          if (res.code === 0) {
            message.success(isEdit ? '编辑成功' : '创建成功');
            history.back();
          }
          return true;
        }}
      >
        {/* 表单字段（自动生成）... */}
        <ProFormText
          name="name"
          label="名称"
          rules={[{ required: true, message: '请输入名称' }]}
        />
        <ProFormSelect
          name="status"
          label="状态"
          options={{Entity}StatusOptions}
          rules={[{ required: true }]}
        />
      </ProForm>
    </Spin>
  );
};

export default {Entity}Form;
```

### 6. 路由配置
```typescript
// routes.ts 新增
{
  path: '/{entity}',
  name: '{Entity}管理',
  icon: 'UserOutlined',
  routes: [
    { path: '/{entity}/list', name: '{Entity}列表', component: './{entity}' },
    { path: '/{entity}/create', name: '新建{Entity}', component: './{entity}/create', hideInMenu: true },
    { path: '/{entity}/edit/:id', name: '编辑{Entity}', component: './{entity}/create', hideInMenu: true },
  ],
}
```

## 字段类型映射

| 类型 | ProForm组件 | ProTable valueType | 宽度 |
|------|------------|-------------------|------|
| string | ProFormText | text | 120-200 |
| int | ProFormDigit | digit | 80 |
| long | ProFormDigit | digit | 80 |
| decimal | ProFormDigit | money | 100 |
| bool | ProFormSwitch | - | 60 |
| datetime | - | dateTime | 180 |
| enum | ProFormSelect | - | 100 |
| text | ProFormTextArea | - | 200 |
| image | - | image | 80 |
| foreign | ProFormSelect | - | 100 |

## 输出清单

1. ✅ 类型定义创建 (data.d.ts)
2. ✅ 枚举定义创建 (enum.ts)
3. ✅ API 函数创建 (service.ts)
4. ✅ 列表页创建 (index.tsx)
5. ✅ 表单页创建 (create.tsx)
6. ✅ 路由配置更新
7. ✅ 权限标识添加

## 注意事项

1. 列表页必须使用 MyProTable
2. 每列必须指定 width
3. 状态字段自动生成枚举和 Map
4. 外键字段生成下拉选择器
5. 创建/编辑共用 create.tsx
6. 时间字段使用 valueType: 'dateTime'