---
name: new-product
description: 商品管理功能（电商场景，包含 SKU、规格）
userInvoke: true
---

# 商品管理 Skill

创建电商商品管理功能，包含：
- 商品基本信息（名称、价格、库存、状态）
- SKU 规格（多规格商品）
- 商品分类关联
- 商品图片
- 商品详情

## 使用方式

```
/new-product [选项]
```

选项：
- `--with-sku` - 启用 SKU 多规格
- `--with-images` - 启用商品图片
- `--with-category` - 启用分类关联（默认启用）

示例：
```
/new-product --with-sku --with-images
```

## 数据模型

### 商品主表 (products)
```sql
CREATE TABLE `products` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `name` VARCHAR(200) NOT NULL COMMENT '商品名称',
  `category_id` BIGINT UNSIGNED NOT NULL COMMENT '分类ID',
  `brand_id` BIGINT UNSIGNED NULL COMMENT '品牌ID',
  `main_image` VARCHAR(500) NULL COMMENT '主图URL',
  `description` TEXT NULL COMMENT '商品描述',
  `detail` TEXT NULL COMMENT '商品详情（富文本）',
  `price` DECIMAL(18,4) NOT NULL COMMENT '价格',
  `original_price` DECIMAL(18,4) NULL COMMENT '原价',
  `stock` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '总库存',
  `sku_count` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT 'SKU数量',
  `status` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态：1=在售 2=下架 3=预售',
  `sort_order` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '排序',
  `sales_count` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '销量',
  `is_hot` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否热销',
  `is_new` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否新品',
  `is_recommend` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否推荐',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  `updated_at` DATETIME(3) NULL ON UPDATE CURRENT_TIMESTAMP(3),
  `is_deleted` TINYINT(1) NOT NULL DEFAULT 0,
  `version` INT UNSIGNED NOT NULL DEFAULT 0,
  INDEX `idx_category_id` (`category_id`),
  INDEX `idx_brand_id` (`brand_id`),
  INDEX `idx_status` (`status`),
  INDEX `idx_sort_order` (`sort_order`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品表';
```

### SKU 表 (product_skus) - 启用 --with-sku
```sql
CREATE TABLE `product_skus` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `product_id` BIGINT UNSIGNED NOT NULL COMMENT '商品ID',
  `sku_code` VARCHAR(100) NOT NULL COMMENT 'SKU编码',
  `spec_json` JSON NOT NULL COMMENT '规格JSON',
  `price` DECIMAL(18,4) NOT NULL COMMENT 'SKU价格',
  `stock` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT 'SKU库存',
  `image` VARCHAR(500) NULL COMMENT 'SKU图片',
  `status` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  `updated_at` DATETIME(3) NULL ON UPDATE CURRENT_TIMESTAMP(3),
  `is_deleted` TINYINT(1) NOT NULL DEFAULT 0,
  UNIQUE INDEX `uniq_sku_code` (`sku_code`),
  INDEX `idx_product_id` (`product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品SKU表';
```

### 商品图片表 (product_images) - 启用 --with-images
```sql
CREATE TABLE `product_images` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `product_id` BIGINT UNSIGNED NOT NULL COMMENT '商品ID',
  `image_url` VARCHAR(500) NOT NULL COMMENT '图片URL',
  `sort_order` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '排序',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  INDEX `idx_product_id` (`product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品图片表';
```

### 规格模板表 (spec_templates) - 启用 --with-sku
```sql
CREATE TABLE `spec_templates` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `name` VARCHAR(100) NOT NULL COMMENT '模板名称',
  `specs_json` JSON NOT NULL COMMENT '规格定义JSON',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  `updated_at` DATETIME(3) NULL ON UPDATE CURRENT_TIMESTAMP(3),
  `is_deleted` TINYINT(1) NOT NULL DEFAULT 0
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='规格模板表';
```

## 规格数据结构

### 规格 JSON 格式
```json
[
  {
    "name": "颜色",
    "values": ["红色", "蓝色", "黑色"]
  },
  {
    "name": "尺寸",
    "values": ["S", "M", "L", "XL"]
  }
]
```

### SKU 规格 JSON 格式
```json
{
  "颜色": "红色",
  "尺寸": "M"
}
```

## 后端代码生成

### 实体类

```csharp
// Product.cs
public class Product : BaseEntity
{
    [MaxLength(200)]
    public string Name { get; set; } = string.Empty;
    
    public long CategoryId { get; set; }
    
    public long? BrandId { get; set; }
    
    [MaxLength(500)]
    public string? MainImage { get; set; }
    
    public string? Description { get; set; }
    
    public string? Detail { get; set; }
    
    [Column(TypeName = "decimal(18,4)")]
    public decimal Price { get; set; }
    
    [Column(TypeName = "decimal(18,4)")]
    public decimal? OriginalPrice { get; set; }
    
    public int Stock { get; set; }
    
    public int SkuCount { get; set; }
    
    public ProductStatus Status { get; set; } = ProductStatus.Active;
    
    public int SortOrder { get; set; }
    
    public int SalesCount { get; set; }
    
    public bool IsHot { get; set; }
    
    public bool IsNew { get; set; }
    
    public bool IsRecommend { get; set; }
}

// ProductSku.cs
public class ProductSku : BaseEntity
{
    public long ProductId { get; set; }
    
    [MaxLength(100)]
    public string SkuCode { get; set; } = string.Empty;
    
    public JsonDocument SpecJson { get; set; } = JsonDocument.Parse("{}");
    
    [Column(TypeName = "decimal(18,4)")]
    public decimal Price { get; set; }
    
    public int Stock { get; set; }
    
    [MaxLength(500)]
    public string? Image { get; set; }
    
    public SkuStatus Status { get; set; } = SkuStatus.Active;
}
```

### 服务接口

```csharp
public interface IProductService : IAutoDIScoped
{
    // 商品 CRUD
    Task<DataResponse<PagedList<ProductDto>>> GetListAsync(ProductListRequest request);
    Task<DataResponse<ProductDetailDto>> GetDetailAsync(long id);
    Task<DataResponse<long>> CreateAsync(CreateProductRequest request);
    Task<DataResponse<bool>> UpdateAsync(UpdateProductRequest request);
    Task<DataResponse<bool>> DeleteAsync(long id);
    
    // SKU 管理
    Task<DataResponse<List<ProductSkuDto>>> GetSkuListAsync(long productId);
    Task<DataResponse<bool>> SaveSkuListAsync(long productId, List<SaveSkuRequest> skus);
    
    // 库存管理
    Task<DataResponse<bool>> UpdateStockAsync(long skuId, int quantity);
    Task<DataResponse<bool>> DeductStockAsync(long skuId, int quantity);
}
```

## 前端代码生成

### 商品列表页

```typescript
// pages/product/index.tsx
const columns: ProColumns<API.Product>[] = [
  { title: '商品图片', dataIndex: 'mainImage', width: 80, valueType: 'image' },
  { title: '商品名称', dataIndex: 'name', width: 200, ellipsis: true },
  { title: '分类', dataIndex: 'categoryName', width: 100 },
  { title: '价格', dataIndex: 'price', width: 100, valueType: 'money' },
  { title: '库存', dataIndex: 'stock', width: 80 },
  { title: '销量', dataIndex: 'salesCount', width: 80 },
  { title: '状态', dataIndex: 'status', width: 80, valueEnum: ProductStatusMap },
  { title: '创建时间', dataIndex: 'createdAt', width: 160, valueType: 'dateTime' },
  // 操作列...
];
```

### SKU 编辑组件

```typescript
// components/SkuEditor.tsx
interface SkuEditorProps {
  specTemplates: SpecTemplate[];
  value?: ProductSku[];
  onChange?: (skus: ProductSku[]) => void;
}

const SkuEditor: React.FC<SkuEditorProps> = ({ specTemplates, value, onChange }) => {
  // SKU 规格选择器
  // SKU 价格/库存编辑表格
  // 自动生成 SKU 组合
};
```

## 特殊业务逻辑

### SKU 自动生成
```csharp
public async Task<List<ProductSku>> GenerateSkusAsync(List<SpecItem> specs)
{
    var combinations = GenerateCombinations(specs);
    return combinations.Select(c => new ProductSku
    {
        SkuCode = GenerateSkuCode(c),
        SpecJson = JsonDocument.Parse(JsonSerializer.Serialize(c)),
        Price = 0,
        Stock = 0,
        Status = SkuStatus.Active
    }).ToList();
}

// 组合算法
private List<Dictionary<string, string>> GenerateCombinations(List<SpecItem> specs)
{
    // 多规格笛卡尔积
}
```

### 库存扣减（分布式锁）
```csharp
public async Task<bool> DeductStockAsync(long skuId, int quantity)
{
    await using var lock = await _redisLock.WaitAsync(
        RedisKey.InventoryLock + skuId,
        TimeSpan.FromSeconds(30));
    
    if (!lock.IsAcquired)
        throw new BusinessException("操作频繁，请稍后重试");
    
    var sku = await _skuRepo.GetByIdAsync(skuId);
    if (sku.Stock < quantity)
        throw new BusinessException("库存不足", BusinessStatusCodeEnum.InventoryNotEnough);
    
    sku.Stock -= quantity;
    await _skuRepo.UpdateAsync(sku);
    
    // 更新商品总库存
    await UpdateProductTotalStockAsync(sku.ProductId);
    
    return true;
}
```

## 输出清单

启用 `--with-sku`：
1. ✅ 商品表创建
2. ✅ SKU 表创建
3. ✅ 规格模板表创建
4. ✅ SKU 编辑组件
5. ✅ SKU 自动生成逻辑

启用 `--with-images`：
1. ✅ 商品图片表创建
2. ✅ 图片上传组件
3. ✅ 图片排序功能

## 测试要点

- [ ] 商品创建成功
- [ ] 商品编辑成功
- [ ] SKU 规格选择
- [ ] SKU 价格/库存设置
- [ ] 库存扣减（并发安全）
- [ ] 商品图片上传
- [ ] 商品分类关联
- [ ] 商品状态切换