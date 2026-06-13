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
| `LoadFileMetadata` | 预加载全部元数据 |
| `LoadFilePublicMetadata` | 仅预加载公开元数据（`is_public=true`） |
| `LoadFileShare` | 预加载分享信息 |
| `LoadFileUser` | 预加载所有者信息 |
| `LoadFileDirectLink` | 预加载直链信息 |

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

### 3.6 递归搜索

[递归搜索](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L355-L522) 实现了按层级深度优先搜索：

1. 从目标文件夹出发，逐层展开所有子文件夹
2. 在当前层的所有文件夹中搜索匹配的文件
3. 当前层搜索完毕后，展开下一层文件夹继续搜索
4. 受 `MaxRecursiveSearchedFolder` 限制，防止过度递归
5. 分页 Token 编码了当前层级和内部分页 Token：`{level}|{innerToken}`

---

## 四、重命名一致性

### 4.1 重命名流程

[DBFS.Rename](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L150-L248) 完整流程：

```
1. 获取导航器（需 NavigatorCapabilityRenameFile + NavigatorCapabilityLockFile）
2. 预加载元数据（LoadFileMetadata=true）
3. 通过路径定位目标文件
4. 权限校验（owner only）
5. 根目录不可重命名
6. 验证新文件名（validateFileName）
7. 文件类型需验证扩展名（validateExtension）和文件名正则（validateFileNameRegexp）
8. 获取文件锁
9. 在事务中执行 fc.Rename()
10. 处理扩展名变更后的缩略图元数据
11. 提交事务
12. 发出文件重命名事件
13. 更新内存中的 File 树结构
14. 处理全文索引差异
```

### 4.2 文件名验证

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

### 4.3 数据库级一致性保证

**唯一约束**：`UNIQUE(file_children, name)` 确保同一目录下不存在同名文件。

**冲突处理**：Rename 调用 `fc.Rename()` 时，若违反唯一约束会返回 `ent.IsConstraintError`，上层转换为 `fs.ErrFileExisted`。

### 4.4 扩展名变更时的元数据清理

当文件重命名导致扩展名变更时（[manage.go#L219-L224](file:///d:/fz/0601-1/solo-dogfeeding/code/49-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L219-L224)）：

```go
if target.Type() == types.FileTypeFile && !strings.EqualFold(filepath.Ext(newName), filepath.Ext(oldName)) {
    if err := fc.RemoveMetadata(ctx, target.Model, ThumbDisabledKey); err != nil {
        // 回滚事务
    }
}
```

扩展名改变后，原有的 `thumb:disabled` 标记会被移除，因为新文件类型可能需要重新生成缩略图。

### 4.5 内存树结构的一致性维护

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

### 4.6 软删除时的重命名

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

### 4.7 从回收站恢复时的重命名

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

### 4.8 全文索引一致性

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

### 4.9 复制时的元数据排除

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
