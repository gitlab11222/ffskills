---
name: gen-crud
description: 生成全栈 CRUD 代码（后端 API + 前端页面 + 数据库表）
userInvoke: true
---

# CRUD 生成 Skill

生成完整的 CRUD 功能，包含：
- 后端：实体、DTO、服务、Controller、数据库迁移
- 前端：类型定义、枚举、API、列表页、表单页、路由
- 数据库：表设计、索引

## 使用方式

```
/gen-crud <实体名> <字段列表>
```

示例：
```
/gen-crud Product name:string price:decimal stock:int status:int category_id:long
/gen-crud Order user_id:long total_amount:decimal status:int created_at:datetime
```

## 参数说明

| 参数 | 格式 | 示例 |
|------|------|------|
| 实体名 | PascalCase | Product, Order, User |
| 字段列表 | `name:type` 逗号分隔 | `name:string,age:int` |

### 支持的字段类型

| 类型 | C# 类型 | MySQL 类型 | 前端类型 |
|------|---------|-----------|----------|
| string | string | VARCHAR(200) | string |
| int | int | INT | number |
| long | long | BIGINT | number |
| decimal | decimal | DECIMAL(18,4) | number |
| bool | bool | TINYINT(1) | boolean |
| datetime | DateTime | DATETIME(3) | string |
| enum | 自定义 | INT | number |
| text | string | TEXT | string |

## 执行步骤

### 1. 理解需求
- 解析实体名和字段列表
- 确认是否需要分类、状态等枚举

### 2. 生成后端代码

#### 实体类
```csharp
// Modules/{Module}/Entities/{Entity}.cs
public class Product : BaseEntity
{
    [MaxLength(200)]
    public string Name { get; set; } = string.Empty;
    
    [Column(TypeName = "decimal(18,4)")]
    public decimal Price { get; set; }
    
    public int Stock { get; set; }
    
    public ProductStatus Status { get; set; } = ProductStatus.Active;
    
    public long CategoryId { get; set; }
}
```

#### DTO
```csharp
// Modules/{Module}/Dtos/ProductListRequest.cs
public class ProductListRequest : PageRequest
{
    public string? Name { get; set; }
    public ProductStatus? Status { get; set; }
    public long? CategoryId { get; set; }
}

// Modules/{Module}/Dtos/ProductDto.cs
public class ProductDto
{
    public long Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public ProductStatus Status { get; set; }
    public string StatusText { get; set; }
    public long CategoryId { get; set; }
    public string CategoryName { get; set; }
    public DateTime CreatedAt { get; set; }
}

// Modules/{Module}/Dtos/CreateProductRequest.cs
public class CreateProductRequest
{
    [Required]
    [MaxLength(200)]
    public string Name { get; set; }
    
    [Required]
    [Range(0, double.MaxValue)]
    public decimal Price { get; set; }
    
    [Required]
    [Range(0, int.MaxValue)]
    public int Stock { get; set; }
    
    [Required]
    public ProductStatus Status { get; set; }
    
    [Required]
    public long CategoryId { get; set; }
}
```

#### 服务接口
```csharp
// Modules/{Module}/Interfaces/IProductService.cs
public interface IProductService : IAutoDIScoped
{
    Task<DataResponse<PagedList<ProductDto>>> GetListAsync(ProductListRequest request);
    Task<DataResponse<ProductDto>> GetDetailAsync(long id);
    Task<DataResponse<long>> CreateAsync(CreateProductRequest request);
    Task<DataResponse<bool>> UpdateAsync(UpdateProductRequest request);
    Task<DataResponse<bool>> DeleteAsync(long id);
}
```

#### 服务实现
```csharp
// Modules/{Module}/Services/ProductService.cs
public class ProductService : IProductService
{
    private readonly IProductRepository _repo;
    private readonly ICategoryRepository _categoryRepo;
    private readonly ILogger<ProductService> _logger;

    public async Task<DataResponse<PagedList<ProductDto>>> GetListAsync(ProductListRequest request)
    {
        // 实现查询逻辑...
    }

    public async Task<DataResponse<long>> CreateAsync(CreateProductRequest request)
    {
        // 实现创建逻辑...
    }
}
```

#### Controller
```csharp
// Controllers/ProductController.cs
[ApiController, Route("api/[controller]")]
public class ProductController : ApiControllerBase
{
    [HttpPost("list")]
    public async Task<IActionResult> GetList(
        [FromServices] IProductService svc,
        [FromBody] ProductListRequest request)
        => Ok(await svc.GetListAsync(request));

    [HttpPost("create")]
    public async Task<IActionResult> Create(
        [FromServices] IProductService svc,
        [FromBody] CreateProductRequest request)
        => Ok(await svc.CreateAsync(request));
}
```

### 3. 生成前端代码

#### 类型定义
```typescript
// pages/product/data.d.ts
declare namespace API {
  interface Product {
    id: number;
    name: string;
    price: number;
    stock: number;
    status: ProductStatus;
    statusText: string;
    categoryId: number;
    categoryName: string;
    createdAt: string;
  }

  interface ProductListParams {
    page: number;
    pageSize: number;
    name?: string;
    status?: ProductStatus;
    categoryId?: number;
  }
}
```

#### 枚举定义
```typescript
// pages/product/enum.ts
export enum ProductStatus {
  Active = 1,
  Inactive = 2,
  OutOfStock = 3,
}

export const ProductStatusMap = new Map<ProductStatus, HasBadgeStatusType>([
  [ProductStatus.Active, { text: '在售', status: 'Success' }],
  [ProductStatus.Inactive, { text: '下架', status: 'Warning' }],
  [ProductStatus.OutOfStock, { text: '缺货', status: 'Error' }],
]);

export const ProductStatusOptions = [
  { label: '在售', value: ProductStatus.Active },
  { label: '下架', value: ProductStatus.Inactive },
  { label: '缺货', value: ProductStatus.OutOfStock },
];
```

#### API 定义
```typescript
// pages/product/service.ts
import { request } from '@/utils/request';

export async function getProductList(params: API.ProductListParams) {
  return request.post<API.DataResponse<API.ProductListResponse>>('/api/product/list', params);
}

export async function createProduct(data: API.CreateProductRequest) {
  return request.post<API.DataResponse<number>>('/api/product/create', data);
}
```

#### 列表页
```typescript
// pages/product/index.tsx
// 完整列表页代码
```

#### 表单页
```typescript
// pages/product/create.tsx
// 完整表单页代码
```

### 4. 数据库迁移

```bash
# 添加迁移
dotnet ef migrations add Add{Entity}Table

# 生成 SQL 脚本
dotnet ef migrations script
```

### 5. 路由配置

```typescript
// routes.ts
{
  path: '/product',
  name: '商品管理',
  icon: 'ShoppingOutlined',
  routes: [
    { path: '/product/list', name: '商品列表', component: './product' },
    { path: '/product/create', name: '新建商品', component: './product/create', hideInMenu: true },
    { path: '/product/edit/:id', name: '编辑商品', component: './product/create', hideInMenu: true },
  ],
}
```

## 输出清单

完成后输出：
1. ✅ 实体类创建完成
2. ✅ DTO 创建完成
3. ✅ 服务接口创建完成
4. ✅ 服务实现创建完成
5. ✅ Controller 创建完成
6. ✅ 数据库迁移脚本生成
7. ✅ 前端类型定义创建完成
8. ✅ 前端枚举定义创建完成
9. ✅ 前端 API 创建完成
10. ✅ 前端列表页创建完成
11. ✅ 前端表单页创建完成
12. ✅ 路由配置更新完成

## 注意事项

1. 字段类型 `enum` 需要额外指定枚举定义
2. 外键字段（如 `category_id:long`）会自动关联
3. 必备字段（id、created_at、updated_at、is_deleted、version）自动添加
4. 列表页每列宽度根据字段类型自动设置
5. 状态字段自动生成枚举和下拉选项