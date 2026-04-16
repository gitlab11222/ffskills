---
name: new-category
description: 商品分类树（多级分类）
userInvoke: true
---

# 商品分类 Skill

创建商品分类管理功能，支持：
- 多级分类（无限层级）
- 分类树结构
- 分类排序
- 分类图标

## 使用方式

```
/new-category [选项]
```

选项：
- `--max-level=N` - 最大层级（默认3级）

示例：
```
/new-category --max-level=3
```

## 数据模型

### 分类表 (categories)
```sql
CREATE TABLE `categories` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `parent_id` BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '父分类ID（0=顶级）',
  `name` VARCHAR(100) NOT NULL COMMENT '分类名称',
  `icon` VARCHAR(500) NULL COMMENT '分类图标',
  `level` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '层级（1/2/3）',
  `sort_order` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '排序',
  `path` VARCHAR(500) NOT NULL COMMENT '分类路径（1/2/3）',
  `product_count` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '商品数量',
  `status` TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '状态：1=正常 2=禁用',
  `created_at` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  `updated_at` DATETIME(3) NULL ON UPDATE CURRENT_TIMESTAMP(3),
  `is_deleted` TINYINT(1) NOT NULL DEFAULT 0,
  `version` INT UNSIGNED NOT NULL DEFAULT 0,
  INDEX `idx_parent_id` (`parent_id`),
  INDEX `idx_level` (`level`),
  INDEX `idx_path` (`path`),
  INDEX `idx_sort_order` (`sort_order`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品分类表';
```

## 后端代码

### 实体类
```csharp
public class Category : BaseEntity
{
    public long ParentId { get; set; } = 0;
    
    [MaxLength(100)]
    public string Name { get; set; } = string.Empty;
    
    [MaxLength(500)]
    public string? Icon { get; set; }
    
    public byte Level { get; set; } = 1;
    
    public int SortOrder { get; set; }
    
    [MaxLength(500)]
    public string Path { get; set; } = string.Empty;  // 如 "1/5/12"
    
    public int ProductCount { get; set; }
    
    public CategoryStatus Status { get; set; } = CategoryStatus.Active;
}
```

### DTO
```csharp
public class CategoryTreeDto
{
    public long Id { get; set; }
    public long ParentId { get; set; }
    public string Name { get; set; }
    public string? Icon { get; set; }
    public int Level { get; set; }
    public int SortOrder { get; set; }
    public string Path { get; set; }
    public int ProductCount { get; set; }
    public CategoryStatus Status { get; set; }
    public List<CategoryTreeDto> Children { get; set; } = [];
}

public class CreateCategoryRequest
{
    [Required]
    [MaxLength(100)]
    public string Name { get; set; }
    
    public long ParentId { get; set; } = 0;
    
    [MaxLength(500)]
    public string? Icon { get; set; }
    
    public int SortOrder { get; set; }
}
```

### 服务接口
```csharp
public interface ICategoryService : IAutoDIScoped
{
    /// <summary>获取分类树</summary>
    Task<DataResponse<List<CategoryTreeDto>>> GetTreeAsync();
    
    /// <summary>获取子分类列表</summary>
    Task<DataResponse<List<CategoryDto>>> GetChildrenAsync(long parentId);
    
    /// <summary>创建分类</summary>
    Task<DataResponse<long>> CreateAsync(CreateCategoryRequest request);
    
    /// <summary>更新分类</summary>
    Task<DataResponse<bool>> UpdateAsync(UpdateCategoryRequest request);
    
    /// <summary>删除分类（含子分类检查）</summary>
    Task<DataResponse<bool>> DeleteAsync(long id);
    
    /// <summary>移动分类</summary>
    Task<DataResponse<bool>> MoveAsync(long id, long newParentId);
}
```

### 树形结构构建
```csharp
public async Task<List<CategoryTreeDto>> GetTreeAsync()
{
    var allCategories = await _repo.GetListAsync(e => !e.IsDeleted && e.Status == CategoryStatus.Active);
    
    // 构建树
    return BuildTree(allCategories, 0);
}

private List<CategoryTreeDto> BuildTree(List<Category> all, long parentId)
{
    return all
        .Where(e => e.ParentId == parentId)
        .OrderBy(e => e.SortOrder)
        .Select(e => new CategoryTreeDto
        {
            Id = e.Id,
            ParentId = e.ParentId,
            Name = e.Name,
            Icon = e.Icon,
            Level = e.Level,
            SortOrder = e.SortOrder,
            Path = e.Path,
            ProductCount = e.ProductCount,
            Status = e.Status,
            Children = BuildTree(all, e.Id)
        })
        .ToList();
}
```

### 创建分类（路径计算）
```csharp
public async Task<DataResponse<long>> CreateAsync(CreateCategoryRequest request)
{
    var parent = request.ParentId > 0
        ? await _repo.GetByIdAsync(request.ParentId)
        : null;
    
    if (parent != null && parent.Level >= _maxLevel)
        throw new BusinessException($"最多支持{_maxLevel}级分类");
    
    var category = new Category
    {
        ParentId = request.ParentId,
        Name = request.Name,
        Icon = request.Icon,
        Level = (byte)((parent?.Level ?? 0) + 1),
        SortOrder = request.SortOrder,
        Path = parent != null ? $"{parent.Path}/{parent.Id}" : "",
        Status = CategoryStatus.Active
    };
    
    await _repo.AddAsync(category);
    return DataResponse.Success(category.Id);
}
```

## 前端代码

### 分类树选择器
```typescript
// components/CategoryTreeSelect.tsx
import { TreeSelect } from 'antd';

interface CategoryTreeSelectProps {
  value?: number;
  onChange?: (value: number) => void;
  placeholder?: string;
}

const CategoryTreeSelect: React.FC<CategoryTreeSelectProps> = ({
  value,
  onChange,
  placeholder = '请选择分类'
}) => {
  const { data: tree } = useRequest(getCategoryTree);

  return (
    <TreeSelect
      value={value}
      onChange={onChange}
      treeData={convertToTreeData(tree?.data || [])}
      placeholder={placeholder}
      allowClear
      showSearch
      treeDefaultExpandAll
    />
  );
};

// 转换数据格式
function convertToTreeData(list: API.CategoryTree[]): TreeData[] {
  return list.map(item => ({
    key: item.id,
    value: item.id,
    title: item.name,
    children: item.children ? convertToTreeData(item.children) : []
  }));
}
```

### 分类管理页
```typescript
// pages/category/index.tsx
const CategoryManage: React.FC = () => {
  const { data: tree } = useRequest(getCategoryTree);

  return (
    <Row gutter={16}>
      <Col span={6}>
        <Card title="分类树">
          <Tree
            treeData={tree?.data || []}
            onSelect={handleSelect}
            draggable
            onDrop={handleDrop}  // 支持拖拽排序
          />
        </Card>
      </Col>
      <Col span={18}>
        <Card title="分类详情">
          {/* 分类编辑表单 */}
        </Card>
      </Col>
    </Row>
  );
};
```

## 特殊功能

### 拖拽排序
```typescript
// 前端拖拽处理
const handleDrop = async (info: DropEvent) => {
  const targetNode = info.node;
  const dragNode = info.dragNode;
  
  // 更新排序或移动分类
  await updateCategorySort({
    id: dragNode.key,
    parentId: targetNode.key,
    sortOrder: calculateNewSortOrder(info)
  });
  
  actionRef.current?.reload();
};
```

### 商品数量统计
```csharp
// 定时任务更新分类商品数量
public async Task UpdateProductCountAsync()
{
    var categories = await _categoryRepo.GetListAsync();
    
    foreach (var category in categories)
    {
        // 统计该分类及其子分类的商品总数
        var count = await _productRepo.CountAsync(e => 
            e.CategoryId == category.Id || e.CategoryId.ToString().StartsWith(category.Path));
        
        category.ProductCount = count;
        await _categoryRepo.UpdateAsync(category);
    }
}
```

## 输出清单

1. ✅ 分类表创建
2. ✅ 分类实体类
3. ✅ 分类树 DTO
4. ✅ 分类服务（树构建、路径计算）
5. ✅ 分类 Controller
6. ✅ 前端分类树选择器
7. ✅ 前端分类管理页
8. ✅ 拖拽排序支持

## 测试要点

- [ ] 一级分类创建
- [ ] 二级分类创建
- [ ] 三级分类创建
- [ ] 层级限制检查
- [ ] 分类删除（含子分类检查）
- [ ] 分类移动
- [ ] 分类拖拽排序
- [ ] 分类树选择器
- [ ] 商品数量统计