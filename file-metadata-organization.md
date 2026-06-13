# Cloudreve 文件目录与元数据组织

## 一、目录层级模型

### 1.1 数据库 Schema：File 实体

文件目录采用 **邻接表（Adjacency List）** 模型，即每个 File 记录通过 `file_children` 字段指向其父目录的 ID，由此构成树形层级。

**核心字段**（定义于 [file.go](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/ent/schema/file.go#L22-L49)）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | int | 文件类型（文件夹=0, 普通文件=1，对应 `types.FileType`） |
| `name` | string | 文件/文件夹名称 |
| `owner_id` | int | 所属用户 ID |
| `size` | int64 | 文件逻辑大小（文件夹为 0） |
| `primary_entity` | int | 主版本实体的 ID（可选） |
| `file_children` | int | 父目录 ID（即 parent_id，可选） |
| `is_symbolic` | bool | 是否为符号链接文件夹 |
| `props` | JSON | 文件属性（含视图设置等，`types.FileProps`） |
| `storage_policy_files` | int | 存储策略 ID（可选） |

**Edge 关系**（[file.go#L53-L72](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/ent/schema/file.go#L53-L72)）：

```
File ──< children     (File 1:N File，自引用)
File ──> parent       (File N:1 File，通过 file_children)
File ──< metadata     (File 1:N Metadata)
File ──< entities     (File 1:N Entity)
File ──< shares       (File 1:N Share)
File ──< direct_links (File 1:N DirectLink)
File ──> owner        (File N:1 User)
File ──> storage_policies (File N:1 StoragePolicy)
```

**唯一性约束索引**（[file.go#L76-L84](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/ent/schema/file.go#L76-L84)）：

```go
index.Fields("file_children", "name").Unique()   // 同一目录下名称唯一
index.Fields("file_children", "type", "updated_at")
index.Fields("file_children", "type", "created_at")
index.Fields("file_children", "type", "size")
```

关键约束：**同一父目录下（file_children 相同）文件名（name）必须唯一**，不区分文件/文件夹类型。

### 1.2 内存中的树结构：dbfs.File

运行时，File 的数据库记录被包装为 [dbfs.File](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L44-L57) 结构体，在内存中构成真正的树：

```go
type File struct {
    Model             *ent.File          // 数据库模型
    Children          map[string]*File   // 子文件映射（key=name）
    Parent            *File              // 父目录指针
    Path              [2]*fs.URI         // [0]=owner视角URI, [1]=user视角URI
    OwnerModel        *ent.User          // 所有者
    IsUserRoot        bool               // 是否为用户根目录
    CapabilitiesBs    *boolset.BooleanSet // 当前导航器的能力集
    FileExtendedInfo  *fs.FileExtendedInfo
    FileFolderSummary *fs.FolderSummary
    disableView       bool
    mu                *sync.Mutex
}
```

**Path 双索引机制**（[file.go#L74-L76](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L74-L76)）：

```go
pathIndexRoot = 0  // Owner 视角路径（从文件实际所有者的根目录出发）
pathIndexUser = 1  // User 视角路径（从当前操作用户的视角出发）
```

- `pathIndexRoot`：用于锁定和内部操作，始终从文件所有者的根出发
- `pathIndexUser`：用于前端展示，可能是共享视图或回收站视图

### 1.3 四种导航器（Navigator）与文件系统视图

系统定义了四种文件系统视图，每种有独立的导航器实现（[dbfs.go#L731-L741](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L731-L741)）：

| 文件系统 | 导航器 | 说明 |
|---------|--------|------|
| `my` | myNavigator | 用户私有文件系统，完整能力 |
| `share` | shareNavigator | 共享文件系统，只读为主 |
| `trash` | trashNavigator | 回收站，受限操作（删除/恢复） |
| `sharedWithMe` | sharedWithMeNavigator | 他人共享给我的文件，仅列表/下载 |

每种导航器有不同的能力集（[navigator.go#L110-L150](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L110-L150)），在操作前进行能力校验。

### 1.4 根目录与用户根

- **数据库根目录**：每个用户的根文件夹 name 为空字符串（`RootFolderName = ""`），没有父级（`file_children` 为 NULL）
- **用户根（UserRoot）**：导航器在 `To()` 方法中通过 `n.root.IsUserRoot = true` 标记
- **用户根的 Path 初始化**（[my_navigator.go#L106-L111](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/my_navigator.go#L106-L111)）：

```go
n.root = newFile(nil, rootFile)
rootPath := path.Root()
n.root.Path[pathIndexRoot], n.root.Path[pathIndexUser] = rootPath, rootPath
n.root.OwnerModel = targetUser
n.root.IsUserRoot = true
```

### 1.5 路径遍历（walk）

**向下遍历**（[navigator.go#L177-L203](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L177-L203)）：

1. 先检查内存缓存（`root.Children[next]`）
2. 缓存未命中时查询数据库 `GetChildFile(ctx, model, ownerID, next, isLeaf)`
3. 符号链接文件夹不可遍历进入（返回 `ErrSymbolicFolderFound`）

**向上遍历**（[navigator.go#L206-L231](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L206-L231)）：

- `findRoot` 循环调用 `walkUp` 直到找不到父级
- `walkUp` 通过 `GetParentFile` 获取父目录

**Walk 递归遍历**（[navigator.go#L270-L353](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L270-L353)）：

- 按层级遍历，每层收集文件夹后批量查询子级
- 支持 `limit`（文件数量限制）和 `depth`（深度限制）
- 跳过符号链接文件夹

---

## 二、元数据（Metadata）系统

### 2.1 数据库 Schema

Metadata 是独立实体，通过外键 `file_id` 关联到 File（[metadata.go](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/ent/schema/metadata.go)）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 元数据键名 |
| `value` | text | 元数据值（最大 65535 字符） |
| `file_id` | int | 所属文件 ID |
| `is_public` | bool | 是否公开（非 owner 可见） |

**唯一约束**：`index.Fields("file_id", "name").Unique()` — 同一文件下键名唯一。

**软删除**：Metadata 继承 `CommonMixin`，支持软删除（`deleted_at` 字段），查询时自动过滤已删除记录。

### 2.2 元数据的分类与键名约定

元数据键名采用 `category:key` 格式，主要分类（定义于 [metadata.go](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/manager/metadata.go#L27-L35)）：

| 前缀 | 说明 | 示例 |
|------|------|------|
| `sys:` | 系统内部元数据 | `sys:restore_uri`, `sys:expected_collect_time`, `sys:shared_owner`, `sys:shared_redirect`, `sys:fulltext_index` |
| `thumb:` | 缩略图相关 | `thumb:disabled` |
| `customize:` | 用户自定义 | `customize:icon_color`, `customize:emoji` |
| `tag:` | 标签 | `tag:任意标签名` |
| `props:` | 自定义属性 | `props:属性ID` |
| `dav:` | WebDAV 相关 | 预留 |

**系统元数据常量**（[file.go#L60-L73](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L60-L73)）：

```go
MetadataSysPrefix           = "sys:"
MetadataUploadSessionPrefix = "sys:upload_session"
MetadataUploadSessionID     = "sys:upload_session_id"
MetadataRestoreUri          = "sys:restore_uri"           // 回收站恢复路径
MetadataExpectedCollectTime = "sys:expected_collect_time"  // 预计清理时间
MetadataSharedOwner         = "sys:shared_owner"           // 共享所有者
FullTextIndexKey            = "sys:fulltext_index"          // 全文索引标记
```

### 2.3 元数据读取（Eager Loading）

元数据通过 **Context Key** 控制是否在查询时预加载（[file_utils.go#L331-L358](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file_utils.go#L331-L358)）：

```go
func withFileEagerLoading(ctx context.Context, q *ent.FileQuery) *ent.FileQuery {
    if v, ok := ctx.Value(LoadFileMetadata{}).(bool); ok && v {
        q.WithMetadata()   // 加载所有元数据（含私有）
    }
    if v, ok := ctx.Value(LoadFilePublicMetadata{}).(bool); ok && v {
        q.WithMetadata(func(m *ent.MetadataQuery) {
            m.Where(metadata.IsPublic(true))  // 仅加载公开元数据
        })
    }
    ...
}
```

**Context Key 列表**（[file.go#L33-L40](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file.go#L33-L40)）：

| Context Key | 说明 |
|-------------|------|
| `LoadFileEntity` | 预加载实体（版本数据） |
| `LoadFileMetadata` | 预加载全部元数据（owner 操作时使用） |
| `LoadFilePublicMetadata` | 仅预加载公开元数据（`is_public=true`，非 owner 视图使用） |
| `LoadFileShare` | 预加载分享信息 |
| `LoadFileUser` | 预加载所有者信息 |
| `LoadFileDirectLink` | 预加载直链信息 |

#### 2.3.1 不同视图下的元数据加载策略

元数据加载由 DBFS 层的 `dbfsOption` 控制（[options.go#L13](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/options.go#L13)），通过 `WithFilePublicMetadata()` 选项开启。

**关键：默认列表入口默认加载公开元数据**

所有列表请求通过 `manager.List()` 进入（[operation.go#L55-L91](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/manager/operation.go#L55-L91)），该方法**默认**传入 `WithFilePublicMetadata()`：

```go
func (m *manager) List(ctx context.Context, path *fs.URI, args *ListArgs) (fs.File, *fs.ListFileResult, error) {
    opts := []fs.Option{
        fs.WithPageSize(args.PageSize),
        fs.WithOrderBy(args.Order),
        fs.WithOrderDirection(args.OrderDirection),
        dbfs.WithFilePublicMetadata(),       // ← 默认加载公开元数据
        dbfs.WithContextHint(),
        dbfs.WithFileShareIfOwned(),
    }
    ...
    return m.fs.List(ctx, path, opts...)
}
```

这意味着：**无论什么视图（my / share / trash / sharedWithMe），只要是列表请求，默认都会加载公开元数据**，不需要上层额外设置。

在 `DBFS.List` 中（[dbfs.go#L172-L174](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L172-L174)），这个选项被转换为 Context Key：

```go
if o.loadFilePublicMetadata {
    ctx = context.WithValue(ctx, inventory.LoadFilePublicMetadata{}, true)
}
```

#### 2.3.2 文件系统列表入口 vs 分享列表入口的元数据加载差异

系统存在两条独立的"列表"入口，元数据加载策略完全不同：

| 维度 | 文件系统列表（manager.List） | 分享列表（shareClient.List） |
|------|---------------------------|---------------------------|
| **调用入口** | `service/explorer/file.go` 的 `ListFileService.List` | `service/share/visit.go` 的 `ListShareService.List` / `ListInUserProfile` |
| **查询对象** | `ent.File`（文件记录） | `ent.Share`（分享记录），再通过 Edge 加载关联的 File |
| **元数据 Context Key** | `LoadFilePublicMetadata`（由 DBFS 层设置） | `LoadFileMetadata`（由 Service 层直接设置） |
| **元数据范围** | 仅公开元数据（`is_public=true`） | **全部元数据**（含私有，无 `is_public` 过滤） |
| **使用场景** | 文件浏览器（explorer）的所有文件列表 | 分享管理页面（查看我发出的分享）、用户资料页的公开分享 |

**分享列表入口的完整设置**（[visit.go#L142-L144](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/service/share/visit.go#L142-L144)）：

```go
ctx := context.WithValue(c, inventory.LoadShareUser{}, true)
ctx = context.WithValue(ctx, inventory.LoadShareFile{}, true)
ctx = context.WithValue(ctx, inventory.LoadFileMetadata{}, true)  // ← 全部元数据！
res, err := shareClient.List(ctx, args)
```

**关键差异原因**：
- 文件系统列表面向可能非 owner 的访问者，仅能看到公开元数据
- 分享列表由分享发起者（owner）查看自己的分享记录，需要完整的文件信息（包括 `sys:fulltext_index` 等系统元数据）
- 两条入口走完全独立的查询路径：`manager.List` → `DBFS.List` → `inventory.GetChildFiles` vs `shareClient.List` → `ent.Share.Query` → `WithFile()` → `WithMetadata()`

**我的文件（myNavigator）**：
- 列表查询：默认通过 `manager.List` 加载公开元数据
- 单个文件查询：根据调用方是否传入 `WithFilePublicMetadata()` 决定（如 [file.go#L642](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/service/explorer/file.go#L642)）
- 写操作（Rename/Copy/Move）：使用 `LoadFileMetadata` 加载全部元数据（[manage.go#L158](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L158)），确保能读取到 `sys:restore_uri`、`sys:fulltext_index` 等系统元数据

**共享视图（shareNavigator）**：
- 列表查询：通过 `manager.List` 默认加载公开元数据
- **路径定位**时不额外设置元数据 Context，依赖上游 Context
- **根目录加载**时设置 `LoadShareUser`、`LoadUserGroup`、`LoadShareFile`（[share_navigator.go#L115-L117](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go#L115-L117)），用于加载分享和用户信息，但不加载文件元数据
- 对于非 owner 访问者，元数据仅加载 `is_public=true` 的公开元数据
- 单文件分享（`singleFileShare=true`）：直接通过 `GetByID` 查询，元数据加载取决于调用方的 Context 设置

**回收站视图（trashNavigator）**：
- 列表查询：通过 `manager.List` 默认加载公开元数据
- 回收站文件**必须**加载元数据，因为 `DisplayName` 依赖 `sys:restore_uri` 元数据来显示原始文件名
- 关键：**`sys:restore_uri` 的 `is_public` 是 `true`**，因此通过公开元数据链路即可读取（详见 3.6.2 节说明）
- 回收站列表查询走 `parent == nil` 的全局搜索路径，查询无父级（孤儿）文件
- 每个结果文件通过 `newTrashUri()` 重新生成用户视角路径

**他人分享给我（sharedWithMeNavigator）**：
- 列表查询：通过 `manager.List` 默认加载公开元数据
- 设置 `args.SharedWithMe = true`（[sharewithme_navigator.go#L89](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/sharewithme_navigator.go#L89)），由 inventory 层过滤共享给当前用户的文件
- 元数据仅加载公开元数据（因为调用方仅传入 `WithFilePublicMetadata`）
- 路径以 `sharedWithMe` 文件系统前缀 + hashid 文件ID 构成

#### 2.3.3 列表过滤中的元数据过滤

搜索过滤时，元数据条件**仅匹配公开元数据**（[file_utils.go#L75-L88](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file_utils.go#L75-L88)）：

```go
q = q.Where(
    file.HasMetadataWith(
        metadata.And(
            metadata.IsPublic(true),    // 强制仅匹配公开元数据
            metadata.NameEQ(key),
            ...
        ),
    ),
)
```

这意味着：
- 私有元数据（`is_public=false`）**不会**出现在搜索过滤条件中
- 非 owner 用户无法通过搜索来探测他人的私有元数据
- 元数据搜索条件与元数据预加载的可见范围一致

#### 2.3.4 运行时读取与延迟加载

**运行时读取**（[file.go#L165-L172](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L165-L172)）：

```go
func (f *File) Metadata() map[string]string {
    return lo.Associate(f.Model.Edges.Metadata, func(item *ent.Metadata) (string, string) {
        return item.Name, item.Value
    })
}
```

从 Edges 中提取并转换为 `map[string]string`，若 Edges 未加载则返回 nil。

**延迟加载**：若元数据未预加载，可调用 `QueryMetadata` 单独查询（[file.go#L1007-L1014](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file.go#L1007-L1014)）：

```go
func (f *fileClient) QueryMetadata(ctx context.Context, root *ent.File) error {
    metadata, err := f.client.File.QueryMetadata(root).All(ctx)
    root.SetMetadata(metadata)
    return nil
}
```

### 2.4 元数据写入

**Upsert（插入或更新）**（[file.go#L707-L736](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file.go#L707-L736)）：

```go
func (f *fileClient) UpsertMetadata(ctx context.Context, file *ent.File, data map[string]string, privateMask map[string]bool) error {
    // 1. 校验值长度（最大 MaxMetadataLen=65535）
    // 2. 使用 OnConflictColumns(file_id, name) 实现冲突时更新
    f.client.Metadata.CreateBulk(...).OnConflictColumns(metadata.FieldFileID, metadata.FieldName).UpdateNewValues().Exec(ctx)
}
```

**Remove**（[file.go#L738-L751](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file.go#L738-L751)）：按 `file_id` + `name` 条件硬删除（跳过软删除拦截器）。

**PatchMetadata（批量修改）**（[props.go#L58-L150](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/props.go#L58-L150)）：

1. 获取 Navigator 并校验 `NavigatorCapabilityUpdateMetadata` 能力
2. 权限校验（仅 owner 可修改）
3. 根目录不可修改元数据
4. 获取文件锁
5. 将 patch 分为新增/更新（`metadataMap`）和删除（`deleted`）
6. 在事务中执行 Upsert + Remove
7. 如果 patch 标记了 `UpdateModifiedAt`，则同时更新文件的 `updated_at`

### 2.5 元数据验证

写入前需经过验证器链（[metadata.go#L65-L278](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/manager/metadata.go#L65-L278)），按键名前缀路由：

```
键名 "customize:emoji" → 查找 validators["customize"]["customize:emoji"] → 验证 emoji 在预设列表中
键名 "tag:xxx"         → 查找 validators["tag"]["*"] → 通配符验证器 → 校验颜色值
键名 "sys:shared_owner" → 查找 validators["sys"]["*"] → 通配符验证器 → 校验 hashid 格式
```

验证流程：先精确匹配 `validators[category][key]`，再匹配 `validators[category]["*"]`（通配符）。

---

## 三、列表过滤

### 3.1 文件列表查询流程

```
DBFS.List() → navigator.Children() → baseNavigator.children()
    → fileClient.GetChildFiles() → childFileQuery + withFileEagerLoading
        → cursorPagination / offsetPagination
```

### 3.2 查询构建：childFileQuery

[文件](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file_utils.go#L118-L155) 根据 roots 参数决定查询模式：

| roots 参数 | 查询策略 |
|-----------|---------|
| 单个 root | `QueryChildren(root)` — 查询直接子级 |
| nil root | 查询孤儿文件（无父级 + ownerID 过滤），用于回收站/共享 |
| 多个 root | `HasParentWith(file.IDIn(...))` — 批量查询多个父目录的子级 |

### 3.3 搜索过滤：searchQuery

[搜索](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file_utils.go#L23-L115) 支持多种条件组合：

**名称过滤**：
- 引号包围 → `NameContains` 精确包含
- 包含通配符 `*` → 转换为 SQL `LIKE` 模式
- 普通文本 → `NameContainsFold`（大小写不敏感）或 `NameContains`
- 支持多关键词 AND/OR 组合（`NameOperatorOr`）

**类型过滤**：`file.TypeEQ(int(*args.Type))`

**元数据过滤**：
```go
// 仅匹配公开元数据，值支持模糊匹配
metadata.And(metadata.IsPublic(true), metadata.NameEQ(key))
// 精确匹配（item.Exact=true）：
metadata.And(metadata.NameEQ(key), metadata.ValueEQ(value))
```

**其他条件**：大小范围（SizeGte/SizeLte）、创建/更新时间范围

### 3.4 分页策略

**游标分页（Cursor Pagination）**（[file_utils.go#L184-L255](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file_utils.go#L184-L255)）：

- 文件夹优先：先查文件夹，再查文件，混合时文件夹在前
- 支持三种查询模式：仅文件夹、仅文件、混合
- 游标基于排序字段值+ID 构成，编码为 Base64 JSON

**偏移分页（Offset Pagination）**（[file_utils.go#L258-L329](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file_utils.go#L258-L329)）：

- 先按类型分组计数（`GroupBy(file.FieldType)`）
- 然后分别查询文件夹和文件，文件夹排在前面
- 支持总条目数统计

**排序选项**（[file_utils.go#L379-L393](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file_utils.go#L379-L393)）：

| 排序字段 | 排序方式 |
|---------|---------|
| name | ByName + ByID 兜底 |
| size | BySize + ByID 兜底 |
| created_at | ByCreatedAt + ByID 兜底 |
| updated_at | ByUpdatedAt + ByID 兜底 |
| 默认 | ByID |

所有排序都以 ID 作为第二排序键保证稳定性。

### 3.5 列表过滤器（listFilter）

[baseNavigator](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L153-L164) 内置 `listFilter` 函数，在返回结果前对每个文件执行过滤：

```go
type fileFilter func(ctx context.Context, f *File) (*File, bool)
var defaultFilter = func(ctx context.Context, f *File) (*File, bool) { return f, true }
```

- 默认过滤器不过滤任何文件
- 特定导航器可设置自定义过滤器（如共享导航器只返回共享文件夹下的文件）

### 3.6 不同视图下的过滤差异

#### 3.6.1 共享视图（shareNavigator）

**根目录过滤**：
- 共享根目录是被分享的那个文件/文件夹（`share.Edges.File`），不是用户的完整根目录
- 单文件分享（`singleFileShare=true`）：列表直接返回该单个文件，不查询子级
- 文件夹分享：从分享文件夹出发，可正常向下遍历子级

**能力限制过滤**：
- `shareNavigatorCapability` 是共享视图的能力集，控制可执行操作
- 非 owner 用户不能修改/删除共享文件（能力校验层拦截）
- `disableView` 标志：若分享未开启预览权限且非 owner，则禁用视图功能

**路径过滤**：
- 用户视角路径从共享根开始，不暴露所有者的完整目录结构
- 分享根的 `Path[pathIndexUser]` 被设置为 `path.Root()`，相当于虚拟根

**元数据可见性**：
- 非 owner 用户只可见 `is_public=true` 的元数据
- 搜索过滤时元数据条件也仅匹配公开元数据

#### 3.6.2 回收站视图（trashNavigator）

**扁平树结构**：
- 回收站是"扁平"的，只有一层（`len(elements) > 1` 直接返回 `ErrPathNotExist`）
- 所有被删除的文件平铺展示，不保留原始目录层级
- `parent == nil` 传入 `children()`，触发全局孤儿文件查询

**查询过滤**：
- 通过 `GetChildFiles` 的 `ownerID` + `nil roots` 查询无父级文件
- 仅显示当前用户自己删除的文件（owner_id = 当前用户ID）

**路径重写**：
- 每个文件的用户视角路径被重写为 `newTrashUri(name)` → `cloudreve://trash/{name}`
- 不保留原始路径，只保留文件名用于展示

**名称显示与公开元数据链路**：

这是一个关键的设计细节：**回收站的 `DisplayName` 依赖 `sys:restore_uri`，但该元数据是公开的**。

**写入时设置 is_public=true**（[manage.go#L309-L317](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L309-L317)）：

```go
// SoftDelete 中写入回收站元数据
fc.UpsertMetadata(ctx, target.Model, map[string]string{
    MetadataRestoreUri: target.Uri(true).String(),          // 原始 URI
    MetadataExpectedCollectTime: strconv.FormatInt(...),     // 预计清理时间
}, nil);  // ← privateMask 为 nil！
```

**UpsertMetadata 中 is_public 的默认逻辑**（[file.go#L717-L726](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file.go#L717-L726)）：

```go
isPrivate := false
if privateMask != nil {
    _, isPrivate = privateMask[key]
}
SetIsPublic(!isPrivate)  // isPrivate=false → !false=true → is_public=true
```

当 `privateMask` 为 `nil` 时，`isPrivate` 保持默认值 `false`，因此 `SetIsPublic(!false)` = `SetIsPublic(true)`。

**读取链路**：
1. `manager.List()` 默认传入 `WithFilePublicMetadata()`
2. `DBFS.List` 设置 `LoadFilePublicMetadata=true`
3. `withFileEagerLoading` 只加载 `is_public=true` 的元数据
4. `MetadataRestoreUri` 的 `is_public=true`，因此被加载
5. `DisplayName()` 读取 `Metadata()[MetadataRestoreUri]` 显示原始名称

这就是为什么回收站列表不需要 `LoadFileMetadata`（加载全部元数据），仅通过 `LoadFilePublicMetadata` 就能正确显示原始文件名。

**restore_uri 虽是公开元数据但仅当前用户可见的边界原因**：

`is_public=true` 只是元数据层面的可见性标记，真正保证数据隔离的是 **SQL 查询的 WHERE 条件** 和 **文件系统导航器选择** 这两层边界：

**边界 1：owner_id 过滤（SQL 层）**

`trashNavigator.Children` 调用 `baseNavigator.children(ctx, nil, args)`，最终执行：

```go
// childFileQuery 中 root == nil 且 isSymbolic = false 时
predicates = append(predicates,
    file.NameNEQ(RootFolderName),
    file.OwnerIDEQ(ownerID),       // ← owner_id == 当前用户 ID
    file.Not(file.HasParent()),    // ← 无父级（已被 SoftDelete 清除）
)
```

`file.OwnerIDEQ(ownerID)` 确保 SQL 查询只返回 `owner_id = 当前用户ID` 的文件记录。即使用户 B 也有一条 `is_public=true` 的 `sys:restore_uri` 元数据，由于其 `file.owner_id != 用户A.ID`，用户 A 根本查不到那条 File 记录，自然不可能读取到其元数据。

**边界 2：回收站导航器的隔离（URI 层）**

只有 URI 的文件系统类型为 `FileSystemTrash` 时，`getNavigator` 才会选择 `trashNavigator`：

```go
case constants.FileSystemTrash:
    n = NewTrashNavigator(f.user, f.fileClient, f.l, config, f.hasher)
```

而回收站 URI 由当前用户请求时通过 `newTrashUri(name)` 生成，格式为 `cloudreve://trash/{name}`，不包含其他用户的信息。其他用户不可能构造出指向你回收站的合法 URI。

**边界 3：软删除时清除父级关联（文件关系层）**

`SoftDelete` 执行时（[manage.go#L296-L306](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L296-L306)）：

```go
fc.SoftDelete(ctx, target.Model)
// SoftDelete 内部：
newName := uuid.Must(uuid.NewV4())
UpdateOne(file).SetName(newName.String()).ClearParent().Save(ctx)
```

`ClearParent()` 将 `file_children` 置为 NULL，文件从原目录树中彻底脱离，变成"孤儿"。其他用户即使通过原路径也无法再访问到该文件。

**总结三层边界**：

| 边界层级 | 机制 | 作用 |
|---------|------|------|
| SQL 查询层 | `file.OwnerIDEQ(ownerID)` + `file.Not(file.HasParent())` | 只返回当前用户的无父级文件 |
| URI 路由层 | `FileSystemTrash` → `trashNavigator`，URI 由当前用户生成 | 其他用户无法访问你的回收站 URI |
| 文件关系层 | `ClearParent()` 清除 `file_children` | 文件从原目录树脱离，无法通过路径访问 |

`is_public=true` 只是"当文件被成功查询到时，这条元数据是否被附带加载"的标记，**不参与访问控制判断**。真正的访问控制在 SQL 的 WHERE 条件和文件系统导航层就已经完成了。这是一个典型的"先过滤行，再选择列"的安全模型。

#### 3.6.3 他人分享给我（sharedWithMeNavigator）

**根目录的双重身份**：

`sharedWithMeNavigator` 的根目录有两层含义，对应不同的对象：

| 概念 | 对应的数据库对象 | URI 路径 | 说明 |
|------|-----------------|---------|------|
| **实际数据库对象** | 当前用户自己的根文件夹（`t.fileClient.Root(ctx, t.user)`） | `newMyIDUri()` | 存在于 `file` 表，name=""，file_children=NULL |
| **虚拟根（用户视角）** | 无对应数据库对象（逻辑容器） | `newSharedWithMeUri("")` → `cloudreve://sharedWithMe` | 导航器的 Path[pathIndexUser] 被覆盖为此值 |

**根目录初始化代码**（[sharewithme_navigator.go#L70-L83](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/sharewithme_navigator.go#L70-L83)）：

```go
if t.root == nil {
    rootFile, err := t.fileClient.Root(ctx, t.user)
    t.root = newFile(nil, rootFile)
    rootPath := newSharedWithMeUri("")
    t.root.Path[pathIndexRoot], t.root.Path[pathIndexUser] = rootPath, rootPath
    t.root.OwnerModel = t.user
    t.root.IsUserRoot = true
}
```

**关键：虚拟根不代表真实的目录关系**
- `sharedWithMeNavigator` 的根目录在数据库中是用户自己的根文件夹
- 但 `To()` 方法限制 `len(elements) > 0` 直接报错，无法像普通目录那样向下遍历
- `Children()` 传入 `parent = nil`（不是 `t.root`），不基于虚拟根查询子级
- 因此虚拟根只是一个"容器"，不参与真实的目录关系

**列表查询过滤：inventory 层的 SharedWithMe 查询**

`sharedWithMeNavigator.Children` 设置 `args.SharedWithMe = true` 后，`baseNavigator.children` 调用：

```go
b.fileClient.GetChildFiles(ctx, &inventory.ListFileParameters{
    PaginationArgs: args.Page,
    SharedWithMe:   args.SharedWithMe,  // true
}, b.user.ID, model)  // model = nil（parent = nil）
```

进入 `childFileQuery` 后（[file_utils.go#L118-L155](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file_utils.go#L118-L155)），因 `root[0] == nil` 走孤儿查询分支，且 `isSymbolic = SharedWithMe = true`：

```go
predicates = append(predicates,
    file.NameNEQ(RootFolderName),
    file.OwnerIDEQ(ownerID),                 // ← 只查当前用户自己拥有的文件
    file.And(file.IsSymbolic(true),          // ← 且是 symbolic 链接
              file.FileChildrenNotNil()),    // ← 有父级（不是孤儿）
)
```

他人共享给"我"的文件，是以 **symbolic link 文件夹** 的形式存在于"我"的文件系统中的，有 `file_children`（父级指向某个逻辑位置），但 `owner_id` 是"我"，`is_symbolic=true`。实际文件内容通过 `sys:shared_redirect` 元数据指向原文件。

**列表结果是游离节点（Orphan Nodes）的原因**：

`baseNavigator.children` 中（[navigator.go#L260-L264](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L260-L264)）：

```go
Files: lo.FilterMap(children.Files, func(model *ent.File, index int) (*File, bool) {
    f := newFile(parent, model)   // ← parent = nil！
    return b.listFilter(ctx, f)
}),
```

因 `parent = nil`，在 `newFile` 中（[file.go#L329-L353](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L329-L353)）走 `else` 分支：

```go
if parent != nil {
    f.Parent = parent
    parent.Children[model.Name] = f          // 加入父节点 Children map
    f.Path[pathIndexUser] = parent.Path[pathIndexUser].Join(model.Name)   // 继承用户视角路径
    f.Path[pathIndexRoot] = parent.Path[pathIndexRoot].Join(model.Name)   // 继承 owner 视角路径
} else {
    f.mu = &sync.Mutex{}
    // Parent 为 nil
    // Children map 中无记录
    // Path[0] 和 Path[1] 都为 nil！
}
```

**游离节点的三个特征**：
1. `f.Parent = nil` — 不挂到任何父节点
2. 不在任何 `parent.Children` map 中 — 无法通过父节点按名称查找
3. `f.Path[pathIndexRoot]` 和 `f.Path[pathIndexUser]` 均为 nil — 无自动生成的路径缓存

之后 `sharedWithMeNavigator.Children` 才**手动覆盖**用户视角路径（[sharewithme_navigator.go#L96-L98](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/sharewithme_navigator.go#L96-L98)）：

```go
res.Files[i].Path[pathIndexUser] = newSharedWithMeUri(hashid.EncodeFileID(t.hasher, res.Files[i].Model.ID))
```

但 `Path[pathIndexRoot]` 永远保持 nil。

**Owner 视角路径缓存为什么不再代表通用规则，且 `Uri(true)` 也不能兜底**：

`Path[pathIndexRoot]`（owner 视角路径）在标准导航场景下通过 `newFile` 自动从父级继承，在各场景的状态如下：

| 场景 | Path[pathIndexRoot] 状态 | 原因 |
|------|------------------------|------|
| 标准 my 目录浏览 | 正常：`cloudreve://my/folder1/file.txt` | `parent != nil` 且 `parent.Path[pathIndexRoot] != nil`，自动继承 |
| sharedWithMe 列表 | **nil** | `parent = nil`，`newFile` 不设置；且虚拟根的 pathIndexRoot 被覆盖为 sharedWithMe 前缀，不是真实 owner 根 |
| 回收站列表 | **nil** | `parent = nil`，`newFile` 不设置；回收站 Children 只设置了 `Path[pathIndexUser]` |
| 递归搜索展开的文件夹 | 用户视角正常，owner 视角 nil | 递归搜索仅手动设置了 `Path[pathIndexUser]`，未设置 `Path[pathIndexRoot]` |
| 回收站 To() 单文件定位 | 被覆盖为 trash URI | `current.Path[pathIndexRoot] = current.Path[pathIndexUser]`（[trash_navigator.go#L95](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/trash_navigator.go#L95)），两者相同，都是 trash 虚拟路径 |

**关键：`Uri(true)` 也不能作为 owner 视角路径的通用兜底**

`Uri(isRoot bool)` 方法的完整逻辑（[file.go#L177-L199](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L177-L199)）：

```go
func (f *File) Uri(isRoot bool) *fs.URI {
    index := 1
    if isRoot {
        index = 0  // pathIndexRoot
    }
    // 短路条件：有缓存 或 无父级 → 直接返回缓存
    if f.Path[index] != nil || f.Parent == nil {
        return f.Path[index]  // ← 可能为 nil！
    }

    // 只有当有父级且 Path[index] 为 nil 时，才向上遍历拼接
    elements := make([]string, 0)
    parent := f
    for parent.Parent != nil && parent.Path[index] == nil {
        elements = append([]string{parent.Name()}, elements...)
        parent = parent.Parent
    }

    if parent.Path[index] == nil {
        return nil  // ← 最终仍可能为 nil
    }
    return parent.Path[index].Join(elements...)
}
```

**sharedWithMe 游离节点的失效链路**：

对于 sharedWithMe 列表结果（游离节点）：
1. `f.Parent == nil`（游离节点没有挂到任何父节点）
2. 命中短路条件 `f.Path[index] != nil || f.Parent == nil` 中的 `f.Parent == nil`
3. **直接返回 `f.Path[pathIndexRoot]`** — 而这个值是 nil！
4. `Uri(true)` **不会**尝试任何补救措施（如读取元数据），直接返回 nil

这是因为 `Uri()` 方法的设计前提是"**如果没有父级，那文件自身就是根，Path 缓存一定已经被设置过了**"。但 sharedWithMe 游离节点打破了这个前提：它们 `Parent == nil` 但**不是根**，且 Path 缓存也未被设置，所以 `Uri(true)` 对它们毫无办法。

对比 `Uri(false)`（用户视角）为什么可以正常工作：
- sharedWithMe navigator 在 Children 中**手动覆盖**了 `Path[pathIndexUser]`：
  `res.Files[i].Path[pathIndexUser] = newSharedWithMeUri(hashid.EncodeFileID(...))`
- 同样因为 `f.Parent == nil`，`Uri(false)` 走短路直接返回 `Path[pathIndexUser]`
- 但用户视角路径被手动设置了，所以返回有效值

**三种路径失效场景的完整闭合关系**：

```
┌───────────────────────────────────────────────────────────────────────┐
│                    sharedWithMe 虚拟路径                               │
│  问题：文件是 symbolic 链接，owner 视角路径指向原所有者的文件系统        │
│  解决：Path[pathIndexUser] = newSharedWithMeUri(hashid)                │
│        用 hashid 编码文件 ID 作为虚拟路径，跳过目录树解析                │
│  后果：Path[pathIndexRoot] 为 nil，Uri(true) 因 Parent==nil 短路返回 nil│
│        真实 owner 路径存储在 sys:shared_redirect 元数据中               │
│        Uri() 不读元数据，所以无法恢复                                    │
└────────────────────────────────────┬──────────────────────────────────┘
                                     │
                                     ▼
┌───────────────────────────────────────────────────────────────────────┐
│                  Uri(true) 的失效边界                                   │
│  短路条件：f.Path[index] != nil || f.Parent == nil                     │
│  游离节点：Parent == nil 且 Path[pathIndexRoot] == nil                  │
│  结果：Uri(true) 直接返回 nil，不尝试向上遍历或读取元数据               │
│  原因：Uri() 假设"无父级=是根节点=Path已手动设置"，游离节点打破此前提    │
└────────────────────────────────────┬──────────────────────────────────┘
                                     │
                                     ▼
┌───────────────────────────────────────────────────────────────────────┐
│                  restore_uri 的可见性设计                               │
│  问题：回收站文件被 SoftDelete，name 变为 UUID，file_children 被清除     │
│  解决：sys:restore_uri 元数据记录原始 owner 视角路径                    │
│        DisplayName() 读取 restore_uri 显示原始文件名                    │
│        恢复时用 restore_uri 定位目标目录                                │
│  关键：is_public=true 所以通过公开元数据链路即可读取                     │
│        但 OwnerIDEQ + ClearParent 两层边界保证仅 owner 可见             │
│        Uri(true) 对回收站文件也返回 nil（同 sharedWithMe 原因）         │
│        所以必须用元数据而非 Uri() 来恢复原始路径                         │
└───────────────────────────────────────────────────────────────────────┘
```

**三条设计规律相互闭合**：

1. **当目录树结构失效时（`Parent == nil`），`Uri()` 的基于树遍历的路径推导必然失效** — sharedWithMe 游离节点和回收站孤儿文件都属于这种情况
2. **树结构失效时，必须依赖元数据存储路径信息** — sharedWithMe 用 `sys:shared_redirect`，回收站用 `sys:restore_uri`
3. **但元数据的用途和可见性不同**：`shared_redirect` 用于访问时跳转（由 DBFS.SharedAddressTranslation 在遇到 symbolic folder 时读取，需要 QueryMetadata 兜底延迟加载），而 `restore_uri` 用于显示名称和恢复目标（仅 owner 自己可见，设为 `is_public=true` 可走通用公开元数据链路）

**核心结论**：
- `Path[pathIndexRoot]` 仅在"标准 my 文件系统的正常层级导航"场景下可靠
- `Uri(true)` 仅在 `Path[pathIndexRoot]` 有缓存 **或** `Parent != nil` 且向上能找到有缓存的祖先时可靠
- **对 sharedWithMe 和回收站的游离节点，两者都不可靠**，必须通过其他机制获取真实路径（虚拟路径编码 ID、元数据存储原始 URI）

**URI 与导航器的映射**（[dbfs.go#L738-L739](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L738-L739)）：

```go
case constants.FileSystemSharedWithMe:
    n = NewSharedWithMeNavigator(f.user, f.fileClient, f.l, config, f.hasher)
```

当 URI 的 `FileSystem()` 为 `sharedWithMe` 时，`getNavigator` 自动选择 `sharedWithMeNavigator`。

**Walk 未实现**：
- `Walk` 方法直接返回 `errors.New("not implemented")`
- 说明"他人分享给我"视图不支持递归遍历，因为它本质上是扁平列表

### 3.7 递归搜索

[递归搜索](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L355-L522) 实现了按**广度优先（BFS）层级推进**的搜索方式，核心数据结构是 `parents` 层级数组。

#### 3.7.1 核心数据结构

```go
parents := []map[int]*File{{parent.Model.ID: parent}}
```

- `parents` 是一个切片，每个元素代表一层的所有文件夹
- 每层是一个 `map[int]*File`，key 为文件夹 ID，value 为 `dbfs.File` 对象
- `parents[0]` = 搜索起始目录的子文件夹层
- `walkedFolder` 记录已展开的文件夹总数，受 `MaxRecursiveSearchedFolder` 限制

#### 3.7.2 层级推进函数 stepLevel

`stepLevel(level int)` 负责展开 `parents[level]` 层的所有子文件夹，追加到 `parents[level+1]`：

```
┌─────────────────────────────────────────────────────────────┐
│  调用 stepLevel(0)                                          │
│  parents[0] → 父级文件夹集合                                 │
│      ↓ GetChildFiles(FolderOnly=true) 批量查询子文件夹       │
│  parents[1] → 第 1 层所有文件夹（map）                       │
│      ↓ 如未取完，用 cursor 分页继续                           │
│  walkedFolder += len(parents[1])                            │
└─────────────────────────────────────────────────────────────┘
```

**关键细节**：
- 使用游标分页（`UseCursorPagination: true`）批量获取文件夹，避免一次性加载过多
- 每次只展开一层（level），文件夹追加到 `parents[level+1]`
- 若当前层文件夹全部取完（`NextPageToken == ""`），break 循环
- 若 `walkedFolder` 超过 `MaxRecursiveSearchedFolder` 上限，停止展开

**性能优化**：层级展开时**不加载元数据**：
```go
listCtx := context.WithValue(ctx, inventory.LoadFilePublicMetadata{}, nil)
```
因为文件夹仅用于路径导航，不需要元数据，减少查询开销。

#### 3.7.3 搜索流程

整体搜索遵循"**层级步进 + 层内搜索**"的模式：

**步骤 1：解析分页 Token**
```go
startLevel, innerPageToken, err := parseSearchPageToken(args.Page.PageToken)
```
Token 格式：`{level}|{innerToken}`，用 `searchTokenSeparator` 分隔
- `startLevel`：从第几层开始搜索
- `innerPageToken`：该层内的游标分页 Token

**步骤 2：前向步进层级**
```go
for level := 0; level < startLevel; level++ {
    stop, err := stepLevel(level)
    if stop { return &ListResult{}, nil }
}
```
从第 0 层开始，一层层展开到 `startLevel`，确保 `parents[startLevel]` 已就绪。

**步骤 3：层内搜索文件**
在 `parents[startLevel]` 层的所有文件夹中搜索匹配的文件（`MixedType: true`）：
```
  parents[startLevel] 中的所有文件夹
         ↓ GetChildFiles(Search + MixedType)
       匹配的文件 + 子文件夹
         ↓ FilterMap + listFilter
       结果集 res
```

**步骤 4：下一层推进**
- 当前层搜完（`PageToken == ""`）且结果未满一页 → `startLevel++`，调用 `stepLevel` 展开下一层
- 若 `stepLevel` 返回 `finished=true`（无更多文件夹），说明全部搜索完毕
- 若结果集填满一页（`len(res) == originalPageSize`），停止搜索，记录当前 `startLevel` 和 `PageToken`

#### 3.7.4 分页 Token 生成

```go
if walkedFolder <= b.config.MaxRecursiveSearchedFolder && !stop {
    searchRes.Pagination.NextPageToken = 
        fmt.Sprintf("%d%s%s", startLevel, searchTokenSeparator, args.Page.PageToken)
}
```

Token 编码了两个信息：
- **层级位置**：当前搜索到第几层（`startLevel`）
- **层内游标**：该层内的分页游标（`PageToken`）

当后续请求带上此 Token 时，解析后从对应层级和位置继续搜索。

#### 3.7.5 回收站搜索的特殊处理

当 `parent == nil` 时（回收站视图），搜索走"全局搜索"路径：
```go
children, err := b.fileClient.GetChildFiles(ctx, &inventory.ListFileParameters{
    PaginationArgs: args.Page,
    MixedType:      true,
    Search:         args.Search,
    SharedWithMe:   args.SharedWithMe,
}, b.user.ID, nil)
```
直接查询该用户所有无父级（孤儿）文件，不做层级递归。

---

## 四、重命名一致性

### 4.1 重命名完整流程

[DBFS.Rename](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L150-L248) 是一个典型的"**先锁后事务，事务提交后再更新内存**"模式，完整步骤按顺序排列如下：

```
┌───────────────────────────────────────────────────────────┐
│  阶段 1：路径定位与预加载                                   │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ 1. getNavigator — 校验 RenameFile + LockFile 能力    │  │
│  │ 2. ctx 注入 LoadFileMetadata=true — 预加载全部元数据 │  │
│  │ 3. getFileByPath — 导航器定位目标文件                │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────┬───────────────────────────────────┘
                        │
┌───────────────────────▼───────────────────────────────────┐
│  阶段 2：权限与合法性校验                                   │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ 4. owner 权限校验                                    │  │
│  │ 5. 根目录不可修改校验                                │  │
│  │ 6. validateFileName — 文件名格式/长度/非法字符        │  │
│  │ 7. 存储策略校验 — 扩展名白名单 + 文件名正则            │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────┬───────────────────────────────────┘
                        │
┌───────────────────────▼───────────────────────────────────┐
│  阶段 3：获取锁                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ 8. acquireByPath — 基于路径获取文件锁                │  │
│  │    ├── lockTupleFromUri — 生成锁键 (ns/root)        │  │
│  │    ├── 检查当前 session 是否已持有锁                  │  │
│  │    ├── ls.Create — 调用分布式锁服务创建锁            │  │
│  │    └── ensureConsistency — 锁后校验文件未被修改      │  │
│  │                                                       │  │
│  │ 9. defer Release — 函数退出时释放锁                  │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────┬───────────────────────────────────┘
                        │
┌───────────────────────▼───────────────────────────────────┐
│  阶段 4：数据库事务                                       │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ 10. inventory.WithTx — 开启事务 + 获取事务版 client  │  │
│  │ 11. fc.Rename — 更新文件名                           │  │
│  │ 12. 若扩展名变更 → fc.RemoveMetadata(thumb:disabled) │  │
│  │ 13. inventory.Commit — 提交事务                      │  │
│  │                                                       │  │
│  │ ⚠️ 任何一步出错 → inventory.Rollback 回滚事务        │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────┬───────────────────────────────────┘
                        │
┌───────────────────────▼───────────────────────────────────┐
│  阶段 5：内存树与副作用更新                                │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ 14. emitFileRenamed — 发出文件重命名事件             │  │
│  │ 15. target.Replace(updated) — 更新内存树结构         │  │
│  │     ├── 从 Parent.Children 删除旧名称                │  │
│  │     ├── 用新模型创建新 File 节点                      │  │
│  │     └── 新名称写入 Parent.Children                   │  │
│  │ 16. 若有全文索引 → 生成 IndexDiff                    │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬───────────────────────┘
                                    │
                            ┌───────▼───────┐
                            │  函数返回     │
                            │  (defer 解锁) │
                            └───────────────┘
```

### 4.2 锁与事务的时序关系

**为什么先加锁再开事务，而不是在事务内加锁？**

1. **分布式锁独立于数据库事务**：锁由独立的锁服务（`f.ls`）管理，不是数据库行锁
2. **事务前必须保证并发安全**：文件查询（`getFileByPath`）和事务之间存在时间窗口，期间文件可能被其他协程修改
3. **锁作为外部互斥机制**：锁确保在整个操作期间，其他写操作无法进入该路径

**ensureConsistency 的作用**（[lock.go#L220-L263](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/lock.go#L220-L263)）：

```go
func (f *DBFS) ensureConsistency(ctx context.Context, files ...*File) error {
    // 查询数据库中的最新状态
    // 对比 name / file_children / owner_id / type 是否变化
    // 若变化 → 返回 ErrModified
}
```

在"查询文件"和"获取锁"之间存在短暂的无保护窗口，`ensureConsistency` 通过二次校验填补这个窗口，确保文件在加锁前未被修改。

### 4.3 事务与内存树更新的时序

**为什么事务提交后才更新内存树？**

1. **事务可能回滚**：如果在事务内更新内存树，事务回滚时内存树已经变了，会导致不一致
2. **内存树无事务机制**：`dbfs.File` 的 `Children` map 是纯内存结构，没有回滚能力
3. **提交即永久**：事务提交成功后，数据库状态是最终状态，此时更新内存树才安全

**内存树更新的具体操作**（[file.go#L251-L272](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L251-L272)）：

```go
func (f *File) Replace(model *ent.File) *File {
    f.mu.Lock()
    delete(f.Parent.Children, f.Model.Name)  // 删除旧名称映射
    f.mu.Unlock()

    defer f.Recycle()
    replaced := newFile(f.Parent, model)      // 新节点自动写入 Parent.Children
    if f.IsRootFile() {
        replaced.Path[pathIndexUser] = f.Path[pathIndexUser]
    }
    return replaced
}
```

**关键要点**：
- `newFile` 构造函数内部会自动将新文件添加到父节点的 `Children` map 中
- 旧节点通过 `defer f.Recycle()` 标记为可回收
- 操作持有父节点的互斥锁（`f.mu`），保证并发安全

### 4.4 锁的层级与作用域

**锁键构成**（[lock.go#L317-L325](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/lock.go#L317-L325)）：

```go
func lockTupleFromUri(uri *fs.URI, u *ent.User, hasher hashid.Encoder) (string, string, string) {
    ns := id + "/" + string(uri.FileSystem())  // 命名空间：用户ID/文件系统类型
    root := uri.Path()                         // 锁根：路径
    return ns, root, ns + "/" + root           // 完整锁键
}
```

**锁的类型**：
- `ZeroDepth=false`：锁定路径及其所有子级（递归锁）
- `ZeroDepth=true`：仅锁定该路径本身（非递归）

**LockSession 栈结构**：
```go
type LockSession struct {
    Tokens     map[string]string  // 所有已获取的锁 token
    TokenStack [][]string         // 栈式管理，每层一个 token 列表
}
```
- 每次进入新的锁作用域，`TokenStack` push 一层
- `Release` 只释放当前栈层的锁
- 支持嵌套锁操作，内层释放不影响外层

### 4.5 重命名错误处理与回滚

| 阶段 | 失败操作 | 回滚方式 |
|------|---------|---------|
| 路径定位 | `getFileByPath` 失败 | 直接返回错误，无副作用 |
| 校验 | 文件名/扩展名非法 | 直接返回错误，无副作用 |
| 获取锁 | `acquireByPath` 失败 | 返回冲突错误，锁已自动释放 |
| 事务内 | `Rename` 或 `RemoveMetadata` 失败 | `inventory.Rollback(tx)` 回滚数据库 |
| 事务提交 | `Commit` 失败 | 返回错误，内存树未更新 |
| 内存树更新 | `Replace` 失败 | 理论上不会失败（纯内存操作） |
| 事件发送 | `emitFileRenamed` 失败 | 不影响重命名结果（事件异步） |

**原子性保证**：数据库事务保证了文件重命名 + 元数据删除的原子性；锁保证了并发安全；内存树更新在事务提交后执行，保证最终一致性。

### 4.6 文件名验证

[validateFileName](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/validator.go#L17-L31)：

```go
func validateFileName(name string) error {
    if len(name) >= 256 || len(name) == 0 {
        return fmt.Errorf("length of name must be between 1 and 255")
    }
    if strings.ContainsAny(name, "\\/:*?\"<>|") {
        return fmt.Errorf("name contains illegal characters")
    }
    if name == "." || name == ".." {
        return fmt.Errorf("name cannot be only dot")
    }
    return nil
}
```

**扩展名验证**（[validator.go#L34-L45](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/validator.go#L34-L45)）：根据存储策略的白名单/黑名单校验。

**文件名正则验证**（[validator.go#L47-L62](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/validator.go#L47-L62)）：根据存储策略的 `NameRegexp` 和 `IsNameRegexpDenyList` 设置校验。

### 4.7 数据库级一致性保证

**唯一约束**：`UNIQUE(file_children, name)` 确保同一目录下不存在同名文件。

**冲突处理**：Rename 调用 `fc.Rename()` 时，若违反唯一约束会返回 `ent.IsConstraintError`，上层转换为 `fs.ErrFileExisted`。

### 4.8 扩展名变更时的元数据清理

当文件重命名导致扩展名变更时（[manage.go#L219-L224](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L219-L224)）：

```go
if target.Type() == types.FileTypeFile && !strings.EqualFold(filepath.Ext(newName), filepath.Ext(oldName)) {
    if err := fc.RemoveMetadata(ctx, target.Model, ThumbDisabledKey); err != nil {
        // 回滚事务
    }
}
```

扩展名改变后，原有的 `thumb:disabled` 标记会被移除，因为新文件类型可能需要重新生成缩略图。

### 4.9 内存树结构的一致性维护

[File.Replace](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L251-L264) 方法确保内存中的树结构与数据库同步：

```go
func (f *File) Replace(model *ent.File) *File {
    f.mu.Lock()
    delete(f.Parent.Children, f.Model.Name)  // 删除旧名称的映射
    f.mu.Unlock()

    defer f.Recycle()
    replaced := newFile(f.Parent, model)      // 用新模型创建节点（新名称会写入 Parent.Children）
    if f.IsRootFile() {
        replaced.Path[pathIndexUser] = f.Path[pathIndexUser]  // 根文件保持用户路径不变
    }
    return replaced
}
```

关键步骤：
1. **删除旧映射**：从父节点的 `Children` map 中移除旧名称
2. **创建新节点**：`newFile` 会自动将新名称写入 `Parent.Children`
3. **路径继承**：根文件（`IsRootFile`）保持用户视角路径不变

### 4.10 软删除时的重命名

[SoftDelete](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/inventory/file.go#L409-L421) 将文件重命名为随机 UUID，同时清除父级关联：

```go
func (f *fileClient) SoftDelete(ctx context.Context, file *ent.File) error {
    newName := uuid.Must(uuid.NewV4())
    _, err := f.client.File.UpdateOne(file).
        SetName(newName.String()).
        ClearParent().
        Save(ctx)
    return err
}
```

同时写入恢复元数据：
```go
fc.UpsertMetadata(ctx, target.Model, map[string]string{
    MetadataRestoreUri: target.Uri(true).String(),         // 原始 URI
    MetadataExpectedCollectTime: strconv.FormatInt(...),    // 预计清理时间
}, nil)
```

### 4.11 从回收站恢复时的重命名

[moveFiles](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L967-L1007) 中，当文件从回收站移出时：

1. 检查是否存在 `MetadataRestoreUri` 元数据
2. 如果存在，将文件重命名回原始显示名称（`file.DisplayName()`）
3. 移除回收站相关元数据（`MetadataRestoreUri`, `MetadataExpectedCollectTime`）

**DisplayName** 的逻辑（[file.go#L86-L97](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L86-L97)）：

```go
func (f *File) DisplayName() string {
    if uri, ok := f.Metadata()[MetadataRestoreUri]; ok {
        restoreUri, _ := fs.NewUriFromString(uri)
        return path.Base(restoreUri.Path())  // 从恢复 URI 中提取原始文件名
    }
    return f.Name()  // 普通文件直接返回 Name
}
```

回收站中的文件实际名称是 UUID，但 `DisplayName` 从元数据中恢复原始名称。

### 4.12 全文索引一致性

重命名后如果文件有全文索引标记（`sys:fulltext_index`），会生成 `IndexDiff` 用于更新搜索引擎（[manage.go#L232-L247](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L232-L247)）：

```go
originalMetadata := target.Metadata()
newFile := target.Replace(updated)
var diff *fs.IndexDiff
if _, ok := originalMetadata[FullTextIndexKey]; ok {
    diff = &fs.IndexDiff{
        IndexToRename: []fs.IndexDiffRenameDetails{{
            Uri:      *newFile.Uri(false),
            FileID:   newFile.ID(),
            EntityID: newFile.PrimaryEntityID(),
        }},
    }
}
```

### 4.13 复制时的元数据排除

[copyFiles](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L867-L965) 中，复制操作会排除全文索引元数据：

```go
fc.Copy(ctx, &inventory.CopyParameter{
    Files:               ...,
    ExcludedMetadataKeys: []string{FullTextIndexKey},  // 不复制全文索引标记
    DstMap:              ...,
})
```

---

## 五、核心架构总结

```
┌─────────────────────────────────────────────────────────────┐
│                    Service Layer                             │
│  service/explorer/metadata.go  (PatchMetadataService)       │
│  service/explorer/file.go      (FileService)                │
│  service/admin/file.go         (AdminFileService)           │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                  Manager Layer                               │
│  pkg/filemanager/manager/metadata.go  (验证+路由)            │
│  pkg/filemanager/manager/manager.go   (核心调度)             │
│  pkg/filemanager/manager/entity.go    (实体管理)             │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                    DBFS Layer                                │
│  pkg/filemanager/fs/dbfs/                                   │
│  ├── dbfs.go          (DBFS 入口，文件系统操作)               │
│  ├── manage.go        (Create/Rename/Delete/Move/Copy)       │
│  ├── navigator.go     (Navigator 接口 + baseNavigator)       │
│  ├── my_navigator.go  (用户文件系统导航)                      │
│  ├── share_navigator.go (共享导航)                           │
│  ├── trash_navigator.go  (回收站导航)                        │
│  ├── file.go          (File 内存树结构)                      │
│  ├── props.go         (PatchMetadata/PatchProps)             │
│  ├── validator.go     (文件名校验)                           │
│  └── lock.go          (路径锁)                               │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                 Inventory Layer                              │
│  inventory/file.go       (FileClient 接口 + 实现)            │
│  inventory/file_utils.go (查询构建/分页/搜索)                 │
│  inventory/common.go     (分页/排序基础设施)                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                   Ent ORM Layer                              │
│  ent/schema/file.go     (File Schema 定义)                   │
│  ent/schema/metadata.go (Metadata Schema 定义)               │
│  ent/schema/entity.go   (Entity Schema 定义)                 │
│  ent/schema/common.go   (CommonMixin 软删除)                 │
│  ent/file.go            (生成代码 - File 模型)               │
│  ent/metadata.go        (生成代码 - Metadata 模型)           │
└─────────────────────────────────────────────────────────────┘
```

**核心设计原则**：

1. **数据库是唯一真相来源**：目录层级完全由 `file_children` 字段决定，无冗余路径字段
2. **内存树结构是查询缓存**：`dbfs.File` 在导航过程中缓存遍历结果，通过 `Path[2]` 维护双视角路径
3. **元数据与文件解耦**：Metadata 作为独立实体，通过外键关联，支持批量 Upsert 和按需预加载
4. **操作全部在事务中**：Rename/Move/Delete 等操作使用 `inventory.WithTx` 确保原子性
5. **路径锁保证并发安全**：所有写操作通过 `acquireByPath` 获取路径锁，防止并发冲突
6. **能力校验隔离视图**：不同文件系统视图通过 Navigator 能力集控制可执行操作
7. **软删除 + 元数据恢复**：回收站文件通过元数据记录原始 URI，恢复时重命名回原名
