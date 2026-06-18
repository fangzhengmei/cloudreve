# VFS 虚拟路径解析、跨驱动权限叠加与节点迁移数据流分析

## 一、核心架构概览

Cloudreve VFS（虚拟文件系统）采用 URI 统一标识 + 多导航器（Navigator）模式，实现了四种文件系统视图的统一访问：

| 文件系统类型 | URI Scheme | 对应导航器 | 主要用途 |
|-------------|------------|-----------|---------|
| `my` | `cloudreve://{uid}@my/path` | [MyNavigator](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/my_navigator.go) | 用户私有文件 |
| `share` | `cloudreve://{share_id}:{pwd}@share/path` | [ShareNavigator](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go) | 分享文件访问 |
| `trash` | `cloudreve://{uid}@trash/path` | [TrashNavigator](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/trash_navigator.go) | 回收站 |
| `shared_with_me` | `cloudreve://{uid}@shared_with_me/path` | [SharedWithMeNavigator](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/sharewithme_navigator.go) | 分享给我的 |

---

## 二、路径解析流程

### 2.1 URI 结构定义

在 [uri.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/uri.go) 中定义了完整的 URI 解析逻辑：

```
cloudreve://{user_id}:{password}@{fs_type}/{path}?{query_params}
         └─────┬─────┘  └──┬──┘   └──┬──┘   └──┬──┘
               │             │         │         └── 查询参数（搜索、过滤等）
               │             │         └──────────── 文件系统类型
               │             └────────────────────── 密码（可选，分享链接用）
               └──────────────────────────────────── 用户ID/分享ID
```

### 2.2 URI 解析核心方法

| 方法 | 功能 | 位置 |
|------|------|------|
| `NewUriFromString()` | 从字符串解析 URI | [uri.go#L44-L59](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/uri.go#L44-L59) |
| `FileSystem()` | 提取文件系统类型 | [uri.go#L214-L216](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/uri.go#L214-L216) |
| `ID()` | 提取用户/分享ID | [uri.go#L132-L141](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/uri.go#L132-L141) |
| `Elements()` | 拆分路径元素 | [uri.go#L123-L130](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/uri.go#L123-L130) |
| `Path()` | 获取规范化路径 | [uri.go#L143-L150](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/uri.go#L143-L150) |

### 2.3 路径解析完整流程

```
用户请求 URI
     │
     ▼
[DBFS.getNavigator()] ──► 根据 URI.Host 选择导航器
     │                          │
     │                          ├─► my → MyNavigator
     │                          ├─► share → ShareNavigator
     │                          ├─► trash → TrashNavigator
     │                          └─► shared_with_me → SharedWithMeNavigator
     │
     ▼
[Navigator.To()] ──────────► 解析路径到文件对象
     │
     ├─► 1. 初始化根目录（首次访问）
     │    [my_navigator.go#L74-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/my_navigator.go#L74-L112)
     │    │
     │    ├─► 校验用户ID哈希
     │    ├─► 检查用户状态
     │    └─► 从 DB 加载根目录
     │
     ├─► 2. 逐级遍历路径元素
     │    [my_navigator.go#L114-L126](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/my_navigator.go#L114-L126)
     │    │
     │    └─► [baseNavigator.walkNext()] ──► 查找子文件
     │         [navigator.go#L177-L203](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L177-L203)
     │         │
     │         ├─► 检查缓存（File.Children）
     │         ├─► DB 查询 GetChildFile()
     │         └─► 构建 File 对象并缓存
     │
     └─► 3. 返回目标 File 对象
          │
          ├─► 双路径视图：Path[0]=所有者视图, Path[1]=访问者视图
          │    [file.go#L177-L199](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L177-L199)
          └─► 权限集继承：CapabilitiesBs 从父目录继承
```

### 2.4 双路径视图设计

每个 [File](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L44-L57) 对象维护两个 URI 视图：

- `Path[pathIndexRoot]` (index=0)：**所有者视图**，用于实际存储操作
- `Path[pathIndexUser]` (index=1)：**访问者视图**，用于展示给当前用户

```go
const (
    pathIndexRoot = 0  // 所有者视图路径
    pathIndexUser = 1  // 访问者视图路径
)
```

**设计意图**：当用户访问他人分享的文件时，看到的是 `share://...` 路径，但实际存储路径是所有者的 `my://...` 路径。

### 2.5 符号链接与分享地址翻译

在 [dbfs.go#L308-L369](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L308-L369) 中实现了 `SharedAddressTranslation()`：

```
请求分享路径 URI
     │
     ▼
[SharedAddressTranslation()]
     │
     ├─► 检查是否为符号链接（IsSymbolic）
     │
     ├─► 从 Metadata 获取重定向 URI
     │    key: "sys:shared_redirect"
     │
     ├─► [URI.Rebase()] 进行路径重绑定
     │    [uri.go#L204-L212](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/uri.go#L204-L212)
     │
     └─► 递归翻译直到找到真实文件
```

---

## 三、跨驱动权限叠加机制

### 3.1 权限表示：BooleanSet

[boolset.BooleanSet](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/boolset/boolset.go) 是权限的底层存储结构，采用位图存储：

```go
type BooleanSet []byte  // 每个字节存储8个权限位

func (b *BooleanSet) Enabled(flag int) bool {
    if flag >= len(*b)*8 {
        return false
    }
    return (*b)[flag/8] & (1 << uint(flag%8)) != 0
}
```

### 3.2 导航器能力集定义

在 [navigator.go#L80-L149](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L80-L149) 中定义了四种导航器的能力集：

```go
// MyNavigator - 用户私有目录权限（最全）
boolset.Sets(map[NavigatorCapability]bool{
    NavigatorCapabilityCreateFile:     true,
    NavigatorCapabilityRenameFile:     true,
    NavigatorCapabilityUploadFile:     true,
    NavigatorCapabilityDownloadFile:   true,
    NavigatorCapabilityUpdateMetadata: true,
    NavigatorCapabilityListChildren:   true,
    NavigatorCapabilityDeleteFile:     true,
    NavigatorCapabilitySoftDelete:     true,
    NavigatorCapabilityShare:          true,
    NavigatorCapabilityVersionControl: true,
    // ... 共17项权限
}, myNavigatorCapability)

// ShareNavigator - 分享目录权限（只读为主）
boolset.Sets(map[NavigatorCapability]bool{
    NavigatorCapabilityDownloadFile:   true,
    NavigatorCapabilityListChildren:   true,
    NavigatorCapabilityGenerateThumb:  true,
    NavigatorCapabilityInfo:           true,
    NavigatorCapabilityVersionControl: true,
}, shareNavigatorCapability)

// TrashNavigator - 回收站权限
boolset.Sets(map[NavigatorCapability]bool{
    NavigatorCapabilityListChildren: true,
    NavigatorCapabilityDeleteFile:   true,
    NavigatorCapabilityRestore:      true,
}, trashNavigatorCapability)

// SharedWithMeNavigator - 分享给我权限
boolset.Sets(map[NavigatorCapability]bool{
    NavigatorCapabilityListChildren: true,
    NavigatorCapabilityDownloadFile: true,
}, sharedWithMeNavigatorCapability)
```

### 3.3 权限叠加流程

权限叠加发生在 **文件创建时** 和 **目录遍历时**：

```
1. 导航器初始化时定义基础能力集
   [navigator.go#L111-L149](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L111-L149)
         │
         ▼
2. 根目录创建时绑定能力集
   [my_navigator.go#L111](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/my_navigator.go#L111)
   root.CapabilitiesBs = n.Capabilities(false).Capability
         │
         ▼
3. 子文件创建时从父目录继承
   [file.go#L329-L353](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L329-L353)
   ├─► newFile(parent, model) 时
   └─► f.CapabilitiesBs = parent.CapabilitiesBs
         │
         ▼
4. 操作前校验权限
   [dbfs.go#L763-L769](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L763-L769)
   capabilities := res.Capabilities(false).Capability
   for _, capability := range requiredCapabilities {
       if !capabilities.Enabled(int(capability)) {
           return ErrNotSupportedAction
       }
   }
```

### 3.4 跨驱动权限叠加的特殊场景

**场景1：访问他人分享的文件**

```
访问者（用户A）请求分享链接
     │
     ▼
ShareNavigator 初始化
     │
     ├─► 校验分享密码
     ├─► 校验分享有效期
     └─► 应用 shareNavigatorCapability（只读权限）
     │
     ▼
文件权限 = ShareNavigator 能力集 ∩ 文件所有者权限
     │
     ▼
最终权限仅包含：下载、列表、查看信息等
```

**场景2：管理员访问用户文件**

```go
// 管理员权限检查
[my_navigator.go#L95-L97](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/my_navigator.go#L95-L97)
if targetUser.Status != user.StatusActive && 
   !n.user.Edges.Group.Permissions.Enabled(int(types.GroupPermissionIsAdmin)) {
    return ErrPermissionDenied
}
```

### 3.5 所有权检查

在 [manage.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/manage.go) 中，每个修改操作前都会检查所有权：

```go
// 通用所有权检查模式
if _, ok := ctx.Value(ByPassOwnerCheckCtxKey{}).(bool); !ok && target.Owner().ID != f.user.ID {
    return ErrOwnerOnly
}
```

可通过 `WithBypassOwnerCheck()` 绕过所有权检查（供系统内部调用）。

---

## 四、节点迁移（Relocate）数据流

节点迁移是指将文件实体（Entity）从一个存储策略（Storage Policy）迁移到另一个存储策略的过程。

### 4.1 核心数据结构

在 [fs.go#L380-L392](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/fs.go#L380-L392) 中定义：

```go
type PrepareRelocateRes struct {
    Entities  map[int]*RelocateEntity `json:"entities,omitempty"`
    LockToken string                  `json:"lock_token,omitempty"`
    Policy    *ent.StoragePolicy      `json:"policy,omitempty"`
}

type RelocateEntity struct {
    SrcEntity                *ent.Entity `json:"src_entity"`     // 源实体
    FileUri                  *URI        `json:"file_uri"`       // 所属文件URI
    NewSavePath              string      `json:"new_save_path"`  // 新存储路径
    ParentFiles              []int       `json:"parent_files"`   // 所有父目录ID（用于索引更新）
    PrimaryEntityParentFiles []int       `json:"primary_entity_parent_files"`
}
```

### 4.2 迁移任务类型

在 [queue/task.go#L104](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/queue/task.go#L104) 中定义：

```go
RelocateTaskType = "relocate"
```

### 4.3 迁移数据流（从代码分析的预期流程）

```
触发迁移请求
     │
     ▼
1. 锁定目标文件
   [LockByPath] 应用 ApplicationRelocate
   [dbfs.go#L695](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L695)
     │
     ▼
2. 收集待迁移实体
   ├─► 遍历文件及其所有版本
   ├─► 收集所有 Entity 对象
   └─► 构建 RelocateEntity 列表
     │
     ▼
3. 生成新存储路径
   [generateSavePath()]
   [dbfs.go#L788-L803](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L788-L803)
   │
   ├─► 应用目标策略的目录命名规则
   ├─► 应用目标策略的文件命名规则
   └─► 支持魔法变量替换：{random}、{timestamp}、{uid} 等
     │
     ▼
4. 数据传输（跨节点/跨驱动）
   │
   ├─► 源驱动：获取源文件内容
   ├─► 目标驱动：上传到新存储
   └─► 传输过程中可选加密/解密
     │
     ▼
5. 数据库操作（原子事务）
   [inventory.RelocateEntityParameter]
   [inventory/file.go#L136-L141](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/inventory/file.go#L136-L141)
   │
   ├─► 更新 Entity.Source 为新路径
   ├─► 更新 Entity.StoragePolicyID 为新策略
   ├─► 更新索引（ParentFiles 链）
   └─► 提交事务
     │
     ▼
6. 清理旧数据
   │
   ├─► 通知源驱动删除旧文件
   ├─► 更新存储使用统计
   └─► 释放锁
```

### 4.4 涉及的关键组件

| 组件 | 职责 | 文件 |
|------|------|------|
| `StoragePolicy` | 存储策略配置，包含驱动类型、路径规则 | [ent/schema/policy.go] |
| `driver.Handler` | 各存储驱动的具体实现 | [pkg/filemanager/driver/] |
| `Entity` | 文件实体（物理存储实例） | [ent/schema/entity.go] |
| `File` | 文件元数据（逻辑文件） | [ent/schema/file.go] |
| `LockSystem` | 分布式锁，防止并发修改 | [pkg/filemanager/lock/] |

### 4.5 存储策略切换时的权限检查

在迁移前，系统会验证用户对目标策略的使用权：

```go
// 获取用户组可用的存储策略
[dbfs.go#L669-L682](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L669-L682)
func (f *DBFS) getPreferredPolicy(ctx context.Context, file *File) (*ent.StoragePolicy, error) {
    ownerGroup := file.Owner().Edges.Group
    sc, _ := inventory.InheritTx(ctx, f.storagePolicyClient)
    groupPolicy, err := sc.GetByGroup(ctx, ownerGroup)
    // ...
}
```

---

## 五、关键代码索引

### 5.1 路径解析核心

| 功能 | 文件位置 |
|------|---------|
| URI 解析 | [uri.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/uri.go) |
| 导航器接口 | [navigator.go#L38-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L38-L58) |
| MyNavigator.To() | [my_navigator.go#L74-L126](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/my_navigator.go#L74-L126) |
| ShareNavigator.To() | [share_navigator.go#L181-L218](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go#L181-L218) |
| 基础路径遍历 | [navigator.go#L177-L203](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L177-L203) |
| File 双路径视图 | [file.go#L177-L199](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L177-L199) |

### 5.2 权限系统核心

| 功能 | 文件位置 |
|------|---------|
| BooleanSet 实现 | [boolset.go](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/boolset/boolset.go) |
| 导航器能力集定义 | [navigator.go#L80-L149](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L80-L149) |
| 权限校验 | [dbfs.go#L763-L769](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L763-L769) |
| 所有权检查 | [manage.go#L60-L62](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L60-L62) |

### 5.3 节点迁移核心

| 功能 | 文件位置 |
|------|---------|
| 迁移数据结构 | [fs.go#L380-L392](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/fs.go#L380-L392) |
| 迁移参数 | [inventory/file.go#L136-L141](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/inventory/file.go#L136-L141) |
| 存储路径生成 | [dbfs.go#L788-L803](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L788-L803) |
| 存储策略获取 | [dbfs.go#L669-L682](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L669-L682) |

---

## 六、设计亮点与注意事项

### 6.1 设计亮点

1. **URI 统一标识**：四种文件系统通过 URI Scheme 区分，接口统一
2. **双路径视图**：所有者视图与访问者视图分离，支持分享链接的路径透明
3. **位图权限存储**：BooleanSet 高效存储权限，位运算快速校验
4. **导航器模式**：新增文件系统类型只需实现 Navigator 接口
5. **上下文缓存**：File 对象缓存子节点，减少 DB 查询

### 6.2 注意事项

1. **路径转义**：`PathEscape()` 与标准库 `url.PathEscape` 不同，需注意前端一致性
   [uri.go#L371-L444](file:///d:/fz/0601-2/solo-dogfeeding/code/31-Cloudreve/pkg/filemanager/fs/uri.go#L371-L444)

2. **权限继承**：子文件从父目录继承 CapabilitiesBs，修改父目录权限需递归更新

3. **符号链接**：分享目录通过符号链接实现，递归翻译可能产生循环引用

4. **原子性**：节点迁移涉及跨驱动数据传输 + DB 事务，需考虑失败回滚策略
