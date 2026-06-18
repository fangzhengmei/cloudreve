# VFS 虚拟路径解析、跨驱动权限叠加与节点迁移数据流分析

## 一、核心架构概览

Cloudreve VFS（虚拟文件系统）采用 URI 统一标识 + 多导航器（Navigator）模式，实现了四种文件系统视图的统一访问：

| 文件系统类型 | URI Scheme | 对应导航器 | 主要用途 |
|-------------|------------|-----------|---------|
| `my` | `cloudreve://{uid}@my/path` | [my_navigator.go](pkg/filemanager/fs/dbfs/my_navigator.go) | 用户私有文件 |
| `share` | `cloudreve://{share_id}:{pwd}@share/path` | [share_navigator.go](pkg/filemanager/fs/dbfs/share_navigator.go) | 分享文件访问 |
| `trash` | `cloudreve://{uid}@trash/path` | [trash_navigator.go](pkg/filemanager/fs/dbfs/trash_navigator.go) | 回收站 |
| `shared_with_me` | `cloudreve://{uid}@shared_with_me/path` | [sharewithme_navigator.go](pkg/filemanager/fs/dbfs/sharewithme_navigator.go) | 分享给我的 |

---

## 二、路径解析流程

### 2.1 URI 结构定义

在 [uri.go](pkg/filemanager/fs/uri.go) 中定义了完整的 URI 解析逻辑：

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
| `NewUriFromString()` | 从字符串解析 URI | [uri.go#L44-L59](pkg/filemanager/fs/uri.go#L44-L59) |
| `FileSystem()` | 提取文件系统类型 | [uri.go#L214-L216](pkg/filemanager/fs/uri.go#L214-L216) |
| `ID()` | 提取用户/分享ID | [uri.go#L132-L141](pkg/filemanager/fs/uri.go#L132-L141) |
| `Elements()` | 拆分路径元素 | [uri.go#L123-L130](pkg/filemanager/fs/uri.go#L123-L130) |
| `Path()` | 获取规范化路径 | [uri.go#L143-L150](pkg/filemanager/fs/uri.go#L143-L150) |

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
     │    [my_navigator.go#L74-L112](pkg/filemanager/fs/dbfs/my_navigator.go#L74-L112)
     │    │
     │    ├─► 校验用户ID哈希
     │    ├─► 检查用户状态
     │    └─► 从 DB 加载根目录
     │
     ├─► 2. 逐级遍历路径元素
     │    [my_navigator.go#L114-L126](pkg/filemanager/fs/dbfs/my_navigator.go#L114-L126)
     │    │
     │    └─► [baseNavigator.walkNext()] ──► 查找子文件
     │         [navigator.go#L177-L203](pkg/filemanager/fs/dbfs/navigator.go#L177-L203)
     │         │
     │         ├─► 检查缓存（File.Children）
     │         ├─► DB 查询 GetChildFile()
     │         └─► 构建 File 对象并缓存
     │
     └─► 3. 返回目标 File 对象
          │
          ├─► 双路径视图：Path[0]=所有者视图, Path[1]=访问者视图
          │    [file.go#L177-L199](pkg/filemanager/fs/dbfs/file.go#L177-L199)
          └─► 权限集继承：CapabilitiesBs 从父目录继承
```

### 2.4 双路径视图设计

每个 [File](pkg/filemanager/fs/dbfs/file.go#L44-L57) 对象维护两个 URI 视图：

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

在 [dbfs.go#L308-L369](pkg/filemanager/fs/dbfs/dbfs.go#L308-L369) 中实现了 `SharedAddressTranslation()`：

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
     │    [uri.go#L204-L212](pkg/filemanager/fs/uri.go#L204-L212)
     │
     └─► 递归翻译直到找到真实文件
```

---

## 三、跨驱动权限叠加机制

### 3.1 权限表示：BooleanSet

[boolset.BooleanSet](pkg/boolset/boolset.go) 是权限的底层存储结构，采用位图存储：

```go
type BooleanSet []byte  // 每个字节存储8个权限位

func (b *BooleanSet) Enabled(flag int) bool {
    if flag >= len(*b)*8 {
        return false
    }
    return (*b)[flag/8] & (1 << uint(flag%8)) != 0
}
```

### 3.2 能力位枚举与占位符

能力位定义在 [navigator.go#L80-L108](pkg/filemanager/fs/dbfs/navigator.go#L80-L108)。**枚举并非连续**，其中插入了 9 个 `NavigatorCapability_CommunityPlacehodler1~9` 占位符，用于保持位序稳定、预留扩展位。这意味着 `BooleanSet` 的字节数组必须覆盖到最高位（`ModifyProps`），中间的占位位恒为 0：

```go
const (
    NavigatorCapabilityCreateFile NavigatorCapability = iota   // 0
    NavigatorCapabilityRenameFile                               // 1
    NavigatorCapability_CommunityPlacehodler1                   // 2 (占位)
    NavigatorCapability_CommunityPlacehodler2                   // 3 (占位)
    NavigatorCapability_CommunityPlacehodler3                   // 4 (占位)
    NavigatorCapability_CommunityPlacehodler4                   // 5 (占位)
    NavigatorCapabilityUploadFile                               // 6
    NavigatorCapabilityDownloadFile                             // 7
    NavigatorCapabilityUpdateMetadata                           // 8
    NavigatorCapabilityListChildren                             // 9
    NavigatorCapabilityGenerateThumb                            // 10
    NavigatorCapability_CommunityPlacehodler5                   // 11 (占位)
    NavigatorCapability_CommunityPlacehodler6                   // 12 (占位)
    NavigatorCapability_CommunityPlacehodler7                   // 13 (占位)
    NavigatorCapabilityDeleteFile                               // 14
    NavigatorCapabilityLockFile                                 // 15
    NavigatorCapabilitySoftDelete                               // 16
    NavigatorCapabilityRestore                                  // 17
    NavigatorCapabilityShare                                    // 18
    NavigatorCapabilityInfo                                     // 19
    NavigatorCapabilityVersionControl                           // 20
    NavigatorCapability_CommunityPlacehodler8                   // 21 (占位)
    NavigatorCapability_CommunityPlacehodler9                   // 22 (占位)
    NavigatorCapabilityEnterFolder                             // 23
    NavigatorCapabilityModifyProps                              // 24
)
```

### 3.3 四种导航器能力集（逐项校准）

在 [navigator.go#L110-L150](pkg/filemanager/fs/dbfs/navigator.go#L110-L150) 的 `init()` 中注册。下表为逐项核对结果（✓=开启）：

| 能力位 | MyNavigator | ShareNavigator | TrashNavigator | SharedWithMeNavigator |
|--------|:---:|:---:|:---:|:---:|
| CreateFile | ✓ | | | |
| RenameFile | ✓ | | | |
| UploadFile | ✓ | | | |
| DownloadFile | ✓ | ✓ | | ✓ |
| UpdateMetadata | ✓ | | | |
| ListChildren | ✓ | ✓ | ✓ | ✓ |
| GenerateThumb | ✓ | ✓ | | |
| DeleteFile | ✓ | | ✓ | |
| LockFile | ✓ | ✓ | ✓ | |
| SoftDelete | ✓ | | | |
| Restore | | | ✓ | |
| Share | ✓ | | | |
| Info | ✓ | ✓ | ✓ | |
| VersionControl | ✓ | ✓ | | |
| EnterFolder | ✓ | ✓ | | ✓ |
| ModifyProps | ✓ | ✓ | | |
| **合计** | **15** | **8** | **5** | **3** |

对应源码（精简标注）：

```go
// MyNavigator - 15 项（用户私有目录，最全）
boolset.Sets(map[NavigatorCapability]bool{
    NavigatorCapabilityCreateFile: true, NavigatorCapabilityRenameFile: true,
    NavigatorCapabilityUploadFile: true, NavigatorCapabilityDownloadFile: true,
    NavigatorCapabilityUpdateMetadata: true, NavigatorCapabilityListChildren: true,
    NavigatorCapabilityGenerateThumb: true, NavigatorCapabilityDeleteFile: true,
    NavigatorCapabilityLockFile: true, NavigatorCapabilitySoftDelete: true,
    NavigatorCapabilityShare: true, NavigatorCapabilityInfo: true,
    NavigatorCapabilityVersionControl: true, NavigatorCapabilityEnterFolder: true,
    NavigatorCapabilityModifyProps: true,
}, myNavigatorCapability)

// ShareNavigator - 8 项（含 LockFile / EnterFolder / ModifyProps，非纯只读）
boolset.Sets(map[NavigatorCapability]bool{
    NavigatorCapabilityDownloadFile: true, NavigatorCapabilityListChildren: true,
    NavigatorCapabilityGenerateThumb: true, NavigatorCapabilityLockFile: true,
    NavigatorCapabilityInfo: true, NavigatorCapabilityVersionControl: true,
    NavigatorCapabilityEnterFolder: true, NavigatorCapabilityModifyProps: true,
}, shareNavigatorCapability)

// TrashNavigator - 5 项（含 LockFile / Info）
boolset.Sets(map[NavigatorCapability]bool{
    NavigatorCapabilityListChildren: true, NavigatorCapabilityDeleteFile: true,
    NavigatorCapabilityLockFile: true, NavigatorCapabilityRestore: true,
    NavigatorCapabilityInfo: true,
}, trashNavigatorCapability)

// SharedWithMeNavigator - 3 项（含 EnterFolder）
boolset.Sets(map[NavigatorCapability]bool{
    NavigatorCapabilityListChildren: true, NavigatorCapabilityDownloadFile: true,
    NavigatorCapabilityEnterFolder: true,
}, sharedWithMeNavigatorCapability)
```

> 校准说明：之前文档误记 ShareNavigator 为 5 项（漏 LockFile/EnterFolder/ModifyProps）、TrashNavigator 为 3 项（漏 LockFile/Info）、SharedWithMeNavigator 为 2 项（漏 EnterFolder）、MyNavigator 为 17 项。实际分别为 8/5/3/15 项。ShareNavigator 并非“纯只读”——它支持 LockFile（对分享文件加锁）、ModifyProps（修改视图设置），但不允许任何写入类操作（Create/Rename/Upload/Delete/SoftDelete/Share）。

### 3.4 权限叠加流程（校准：子集判定而非与运算）

权限叠加发生在 **导航器初始化** 与 **目录遍历** 两个阶段：

```
1. 导航器初始化时定义基础能力集
   [navigator.go#L110-L150](pkg/filemanager/fs/dbfs/navigator.go#L110-L150)
         │
         ▼
2. 根目录创建时绑定能力集
   [my_navigator.go#L111](pkg/filemanager/fs/dbfs/my_navigator.go#L111)
   root.CapabilitiesBs = n.Capabilities(false).Capability
         │
         ▼
3. 子文件创建时从父目录逐级继承（原样拷贝）
   [file.go#L329-L353](pkg/filemanager/fs/dbfs/file.go#L329-L353)
   ├─► newFile(parent, model) 时
   └─► f.CapabilitiesBs = parent.CapabilitiesBs
         │
         ▼
4. getNavigator 选择导航器时做“子集判定”
   [dbfs.go#L763-L769](pkg/filemanager/fs/dbfs/dbfs.go#L763-L769)
   capabilities := res.Capabilities(false).Capability
   for _, capability := range requiredCapabilities {
       if !capabilities.Enabled(int(capability)) {
           return ErrNotSupportedAction
       }
   }
```

> **校准关键点**：之前文档称“导航器工厂用能力位与目标文件能力集做‘与’运算”，这是不准确的。实际机制是：
> 1. 每个导航器把**自己的能力集**盖写到根目录的 `CapabilitiesBs`，子节点原样继承，整棵 File 树共享同一份能力集；
> 2. `getNavigator()` 收到调用方传入的 `requiredCapabilities`（如上传时传 `NavigatorCapabilityUploadFile`+`NavigatorCapabilityLockFile`），逐一用 `Enabled()` 判断这些**必需位**是否都在导航器能力集中——这是“**必需位 ⊆ 已启用位**”的子集判定，不是两个独立集合的按位与；
> 3. 同一逻辑文件在不同导航器下会重建出**不同的 File 对象**（各自盖写各自的能力集），所有者视图与访问者视图的能力集不会出现在同一对象上做运算，因此不存在“交集”——访问者视图的能力完全由其导航器决定。

### 3.5 跨驱动权限叠加的特殊场景

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
文件权限 = ShareNavigator 能力集（覆盖所有者视图能力）
     │
     ▼
最终权限仅包含：下载、列表、查看信息等
```

> 注意：分享场景下 `shareNavigatorCapability` 是“覆盖”而非“与”运算——因为源文件在所有者视图下虽具备全部能力，但分享视图下访问者的能力完全由分享导航器决定，写入类能力位被全部置零。

**场景2：管理员访问用户文件**

```go
// 管理员权限检查
[my_navigator.go#L95-L97](pkg/filemanager/fs/dbfs/my_navigator.go#L95-L97)
if targetUser.Status != user.StatusActive && 
   !n.user.Edges.Group.Permissions.Enabled(int(types.GroupPermissionIsAdmin)) {
    return ErrPermissionDenied
}
```

### 3.6 所有权检查

在 [manage.go](pkg/filemanager/fs/dbfs/manage.go) 中，每个修改操作前都会检查所有权：

```go
// 通用所有权检查模式
if _, ok := ctx.Value(ByPassOwnerCheckCtxKey{}).(bool); !ok && target.Owner().ID != f.user.ID {
    return ErrOwnerOnly
}
```

可通过 `WithBypassOwnerCheck()` 绕过所有权检查（供系统内部调用）。

### 3.7 存储驱动权限关系

跨驱动权限不仅取决于导航器能力，还取决于“用户所在用户组能否使用目标存储策略”。这一层关系在 [inventory/policy.go](inventory/policy.go#L145-L155) 中体现：

```go
// GetByGroup：按用户组关联查可用存储策略
func (c *storagePolicyClient) GetByGroup(ctx context.Context, group *ent.Group) (*ent.StoragePolicy, error) {
    res, err := withStoragePolicyEagerLoading(ctx, c.client.Group.QueryStoragePolicies(group)).WithNode().First(ctx)
    // ...
}
```

权限关系链：

```
用户 → 用户组(Group) → 关联存储策略(StoragePolicy) → 存储驱动(Driver) → 物理存储
```

- **用户组绑定策略**：管理员在后台为每个用户组分配可用的存储策略（一对多），见 [dbfs.go#L669-L682](pkg/filemanager/fs/dbfs/dbfs.go#L669-L682) 的 `getPreferredPolicy()`，它取所有者用户组关联的首个策略。
- **策略决定驱动**：`StoragePolicy.Type` 决定使用哪个 driver.Handler（Local/Remote/COS/S3/OneDrive 等），见 [manager/fs.go](pkg/filemanager/manager/fs.go) 的 `GetStorageDriver()`。
- **主从策略翻译**：当请求落到从节点时，主节点的 `Remote` 策略需要被翻译为从节点的 `Local` 策略（反之亦然），由 `CastStoragePolicyOnSlave()` 完成。这一步是“跨驱动”的关键，保证同一逻辑文件在主从两侧用不同的物理驱动落盘。
- **策略设置约束**：`Policy.Settings.Relay` 控制是否中转上传；非中转且非本地策略时，`ConfirmUploadSession()` 会直接拒绝（`CodePolicyNotAllowed`），见 [manager/upload.go#L168-L170](pkg/filemanager/manager/upload.go#L168-L170)。

---

## 四、节点迁移（Relocate）数据流

节点迁移是指将文件实体（Entity）从一个存储策略（Storage Policy）/ 节点迁移到另一个的过程。**需要特别说明**：当前仓库中 `RelocateTaskType` 相关的数据结构已经定义并被注册为可恢复任务类型，但其任务工厂尚未注册、`PrepareRelocateRes`/`RelocateEntity` 在业务代码中没有任何调用点。也就是说，专门的“迁移任务”目前是脚手架状态。

真正在运行的“跨驱动 / 跨节点数据搬运”由两条路径实现：
1. **跨节点实体搬运**：`SlaveUploadTask` + `manager.Update()` 的无状态（stateless）分支；
2. **外部资源搬运入库**：`RemoteDownloadTask` 的传输阶段（复用上传链路）。

下面分别说明触发点、执行流程、实体入库与存储驱动权限关系。

### 4.1 脚手架：专用的 Relocate 数据结构（未接通）

在 [fs.go#L380-L392](pkg/filemanager/fs/fs.go#L380-L392) 中定义：

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

- 任务类型常量：[queue/task.go#L104](pkg/queue/task.go#L104) `RelocateTaskType = "relocate"`
- 可恢复任务注册：[application/dependency/dependency.go#L683](application/dependency/dependency.go#L683) 将其与归档/解压/导入一并声明为可恢复任务类型。
- 任务类型允许列表：[service/explorer/workflows.go#L336](service/explorer/workflows.go#L336) 允许用户查询这些任务类型。
- 数据库参数：[inventory/file.go#L136-L141](inventory/file.go#L136-L141) `RelocateEntityParameter`。

> 全仓搜索 `PrepareRelocate` / `RelocateEntity` / `relocateEntity` 仅命中上述定义处和本文档，没有任何 `queue.RegisterResumableTaskFactory(RelocateTaskType, ...)` 调用，也没有任何业务方法构造 `PrepareRelocateRes`。因此本节只描述数据结构语义与预期流程，不作为“正在运行”的流程。

**预期流程（据数据结构推断）**：

```
1. 锁定目标文件  → ApplicationRelocate（fs.go#L695）
2. 收集待迁移实体 → 遍历文件及版本，构建 RelocateEntity 列表
3. 生成新存储路径 → generateSavePath()（dbfs.go#L788-L803）
4. 数据传输       → 源驱动读 + 目标驱动写（可选加解密）
5. DB 事务        → RelocateEntityParameter 更新 Entity.Source / StoragePolicyID / 父链
6. 清理旧数据     → 通知源驱动删除旧文件、更新容量统计、释放锁
```

### 4.2 实际链路一：跨节点实体搬运（SlaveUploadTask）

#### 4.2.1 触发点

当从节点（slave）需要把本地已有文件上传到主节点（master）时，主节点会创建一个 `SlaveUploadTask`。任务定义见 [pkg/filemanager/workflows/upload.go](pkg/filemanager/workflows/upload.go#L39-L47)，由 `NewSlaveUploadTask` 构造（[upload.go#L50-L67](pkg/filemanager/workflows/upload.go#L50-L67)）。

任务状态 `SlaveUploadTaskState` 携带：
- `Files []SlaveUploadEntity`：待搬运文件列表（含源路径 `Src`、目标 `Uri`、大小、索引）；
- `MaxParallel`：并发度；
- `Transferred map[int]interface{}`：已传输索引（断点续传）；
- `UserID`：目标用户ID。

#### 4.2.2 执行流程（Do 方法）

[upload.go#L69-L223](pkg/filemanager/workflows/upload.go#L69-L223)：

```
SlaveUploadTask.Do(ctx)
     │
     ├─► 1. 获取主节点（必须为 master）  [upload.go#L74-L82]
     │      np.Get(...) → node；!IsMaster() 直接报错
     │
     ├─► 2. 反序列化状态，恢复断点       [upload.go#L87-L95]
     │      已在 Transferred 中的文件直接跳过
     │
     ├─► 3. 并发 worker 池搬运            [upload.go#L97-L200]
     │      对每个 SlaveUploadEntity：
     │        ├─► os.Open(本地源文件 Src)
     │        ├─► 构造 fs.UploadRequest{Uri, Size, File, Seeker}
     │        ├─► fm.Update(ctx, req,
     │        │        fs.WithNode(t.node),
     │        │        fs.WithStatelessUserID(t.state.UserID),
     │        │        fs.WithNoEntityType())   [upload.go#L165]
     │        └─► 成功后 state.Transferred[index] = nil（持久化断点）
     │
     ├─► 4. 序列化新状态回写 PrivateState [upload.go#L205-L211]
     │
     └─► 5. 全部失败 → StatusError；否则 StatusCompleted
```

#### 4.2.3 实体入库（manager.Update → updateStateless）

`fm.Update()` 在 [manager/upload.go#L326-L367](pkg/filemanager/manager/upload.go#L326-L367)，无状态分支走 [updateStateless()](pkg/filemanager/manager/upload.go#L400-L436)：

```
updateStateless(ctx, req, o)
     │
     ├─► A. PrepareUpload（RPC 到主节点）
     │      o.Node.PrepareUpload(StatelessPrepareUploadService{UploadRequest, UserID})
     │      └─► 主节点 service/node/rpc.go StatelessPrepareUpload()
     │           └─► fm.PrepareUpload() → DBFS.PrepareUpload()
     │                ├─► 选择/校验存储策略 getPreferredPolicy()
     │                ├─► generateSavePath() 生成物理路径
     │                ├─► 创建占位 File + 占位 Entity（带 UploadSessionID）
     │                └─► 返回 UploadSession{Policy, FileID, EntityID, LockToken}
     │
     ├─► B. Upload（本节点落盘到存储驱动）
     │      m.Upload(ctx, req, res.Session.Policy, res.Session)
     │      └─► GetStorageDriver(CastStoragePolicyOnSlave(policy)).Put(ctx, req)
     │           └─► 在从节点用本地/对应驱动写入物理文件
     │
     └─► C. CompleteUpload（RPC 到主节点完成入库）
            o.Node.CompleteUpload(StatelessCompleteUploadService{Session, UserID})
            └─► 主节点 StatelessCompleteUpload() → fm.CompleteUpload()
                 └─► DBFS.CompleteUpload()
                      ├─► fc.UpgradePlaceholder()   把占位 Entity 升级为正式版本
                      ├─► fc.RemoveMetadata()        清理 upload session 标记
                      ├─► fc.UpsertMetadata()        写入用户自定义元数据
                      ├─► fc.CapEntities()           版本保留策略裁剪（产生 StorageDiff）
                      └─► CommitWithStorageDiff()    提交事务 + 应用容量差值
```

> 这是真正的“实体入库”三段式：**占位创建 → 物理落盘 → 升级提交**。占位阶段已经在 DB 写入 File/Entity 记录（状态为占位），落盘阶段写入物理对象，升级阶段把 Entity 标记为正式并按版本保留策略裁剪旧实体。

#### 4.2.4 主节点 RPC 入口

主节点暴露的无状态接口见 [service/node/rpc.go](service/node/rpc.go)：

- `StatelessPrepareUpload()`（[rpc.go#L46-L64](service/node/rpc.go#L46-L64)）：用 `UserID` 取登录用户，注入 `UserCtx`，调用 `fm.PrepareUpload`。
- `StatelessCompleteUpload()`（[rpc.go#L70-L81](service/node/rpc.go#L70-L81)）：同上，调用 `fm.CompleteUpload`。
- `StatelessOnUploadFailed()`（[rpc.go#L87-L99](service/node/rpc.go#L87-L99)）：失败回滚（删占位 / 回滚版本控制 / 删物理文件）。
- `StatelessCreateFile()`（[rpc.go#L103-L120](service/node/rpc.go#L103-L120)）：直接在主节点创建目录/占位文件。

从节点 HTTP 入口见 [routers/controllers/slave.go](routers/controllers/slave.go)（`SlaveUpload` / `SlaveGetUploadSession` / `SlaveDeleteUploadSession` / `SlaveServeEntity` 等），使用 HMAC 鉴权（[remote/client.go#L57-L80](pkg/filemanager/driver/remote/client.go#L57-L80)）。

### 4.3 实际链路二：外部资源搬运入库（RemoteDownloadTask）

`RemoteDownloadTask` 在把外部 URL 下载到本地后，复用 `fm.Update()` 把临时文件作为新版本/新实体入库，其传输阶段进度类型常量 `ProgressTypeRelocateTransferCount = "relocate"`（[remote_download.go#L71](pkg/filemanager/workflows/remote_download.go#L71)）也印证了“迁移”语义被复用在下载入库链路上。

### 4.4 存储驱动权限关系（迁移相关）

迁移链路中，存储策略与驱动的权限关系如下：

| 环节 | 代码位置 | 作用 |
|------|---------|------|
| 用户组 → 策略 | [inventory/policy.go#L145-L155](inventory/policy.go#L145-L155) `GetByGroup` | 用户组关联可用策略，决定目标驱动 |
| 父目录 → 策略 | [dbfs.go#L669-L682](pkg/filemanager/fs/dbfs/dbfs.go#L669-L682) `getPreferredPolicy` | 上传/迁移时取所有者用户组的首个策略 |
| 主从策略翻译 | [manager/fs.go](pkg/filemanager/manager/fs.go) `CastStoragePolicyOnSlave` | Remote↔Local 互译，保证主从用各自物理驱动 |
| 策略 → 驱动 | [manager/fs.go](pkg/filemanager/manager/fs.go) `GetStorageDriver` | 按 `Policy.Type` 实例化 driver.Handler |
| 中转约束 | [manager/upload.go#L168-L170](pkg/filemanager/manager/upload.go#L168-L170) | 非本地且非中转策略拒绝直传 |
| 容量校验 | [dbfs/upload.go#L64-L67](pkg/filemanager/fs/dbfs/upload.go#L64-L67) `validateUserCapacity` | 迁移/上传前校验所有者剩余容量 |
| 文件大小校验 | [dbfs/upload.go#L52-L62](pkg/filemanager/fs/dbfs/upload.go#L52-L62) `validateFileSize` | 按策略最大单文件大小校验 |

> 权限叠加在迁移链路的体现：跨节点搬运时，主节点 `PrepareUpload` 内部会重新走一遍 `getNavigator` + 能力校验（`NavigatorCapabilityUploadFile` + `NavigatorCapabilityLockFile`，见 [dbfs/upload.go#L74](pkg/filemanager/fs/dbfs/upload.go#L74)）和所有权检查（[dbfs/upload.go#L105-L107](pkg/filemanager/fs/dbfs/upload.go#L105-L107)）。也就是说，即使数据来自从节点的 RPC，主节点仍会以“当前用户视图”重新叠加导航器能力，不会因为跨节点而绕过权限。

### 4.5 压缩任务产物上传触发

`CreateArchiveTask`（压缩）与 `ExtractArchiveTask`（解压）在产物入库时均复用 `fm.Update()`，但**触发分支依节点角色而异**——这是此前文档遗漏的一条关键迁移触发链路。分支选择的根因在于 `NewFileManager()` 的初始化逻辑：当节点处于 SlaveMode 或传入 user 为 nil 时，返回的 manager 恒为 stateless（[manager.go#L152-L156](pkg/filemanager/manager/manager.go#L152-L156)、[manager.go#L173-L184](pkg/filemanager/manager/manager.go#L173-L184)），其 `m.stateless` 为 true，`fm.Update()` 会自动路由到 `updateStateless()`（[manager/upload.go#L345-L347](pkg/filemanager/manager/upload.go#L345-L347)）。

#### 4.5.1 CreateArchiveTask 产物上传

`CreateArchiveTask.Do()` 根据所分配节点是否为 master 走两条不同路径（[archive.go#L137-L167](pkg/filemanager/workflows/archive.go#L137-L167)）：

**主节点路径**（3 阶段，全部本地完成）：

```
Phase: NotStarted    → initializeTempFolder()    准备临时目录
Phase: CompressFiles → createArchiveFile()      本地压缩生成 zip
Phase: UploadArchive → uploadArchive()          ← 产物上传触发点
    └─► fm.Update(ctx, fileData)                有状态服务端上传
         （archive.go#L471，不传 WithStatelessUserID）
```

**从节点路径**（4 阶段，压缩与上传分离到从节点）：

```
Phase: NotStarted → listEntitiesAndSendToSlave()
    └─► DryRun 收集实体列表 + 策略 → 创建 SlaveCreateArchiveTask 到从节点
Phase: AwaitSlaveCompressing → awaitSlaveCompressing()
    └─► 轮询从节点压缩任务，取回 ZipFilePath
Phase: CreateAndAwaitSlaveUploading → createAndAwaitSlaveUploading()
    └─► 创建 SlaveUploadTask 到从节点（携带 zip 临时路径）   ← 产物上传触发点
         从节点 SlaveUploadTask.Do() → fm.Update(WithNode, WithStatelessUserID, WithNoEntityType)
Phase: CompleteUpload → completeUpload()
    └─► 空操作！直接返回 StatusCompleted
         （archive.go#L368-L370）
```

> **校准关键点**：从节点路径中，主节点的 `completeUpload()` 是**空操作**（仅 `return task.StatusCompleted, nil`）——因为从节点的无状态上传已经通过 RPC 在主节点完成了 `CompleteUpload`（占位升级 + 版本裁剪 + 事务提交），主节点无需再做任何 DB 操作。这与"主节点需要在 completeUpload 阶段做实体入库"的理解不同。

#### 4.5.2 ExtractArchiveTask 产物上传

解压任务对**每个解压出的文件**逐一触发 `fm.Update()`，同样依节点角色分两条路径：

| 节点角色 | 触发方法 | fm.Update 调用 | 代码位置 |
|---------|---------|---------------|---------|
| 主节点 | `masterExtractArchive()` | `fm.Update(ctx, fileData, WithNoEntityType)` | [extract.go#L429](pkg/filemanager/workflows/extract.go#L429) |
| 从节点 | `SlaveExtractArchiveTask.Do()` | `fm.Update(ctx, fileData, WithNode, WithStatelessUserID, WithNoEntityType)` | [extract.go#L803](pkg/filemanager/workflows/extract.go#L803) |

目录创建走 `fm.Create()`（主节点 [extract.go#L398](pkg/filemanager/workflows/extract.go#L398) / 从节点 [extract.go#L772](pkg/filemanager/workflows/extract.go#L772)），同样区分有无 `WithStatelessUserID`。

#### 4.5.3 各任务 fm.Update 分支汇总

| 调用场景 | 节点角色 | manager 类型 | 选项 | 走哪条分支 |
|---------|---------|------------|------|-----------|
| 归档产物上传 | master | stateful | 无 | `Update()` stateful |
| 归档产物上传 | slave | stateless | `WithNode`+`WithStatelessUserID`+`WithNoEntityType` | `updateStateless()` |
| 解压文件上传 | master | stateful | `WithNoEntityType` | `Update()` stateful |
| 解压文件上传 | slave | stateless | `WithNode`+`WithStatelessUserID`+`WithNoEntityType` | `updateStateless()` |
| 远程下载入库 | master | stateful | `WithNoEntityType` | `Update()` stateful |
| 远程下载入库 | slave | stateless | 创建 `SlaveUploadTask` | `updateStateless()` |

> 同一个 `fm.Update()` 调用，在主节点（stateful manager）走三段本地调用（PrepareUpload → Upload → CompleteUpload），在从节点（stateless manager）走三段 RPC（PrepareUpload RPC → 本地 Put → CompleteUpload RPC）。分支选择不取决于调用方传了什么选项，而取决于 **manager 实例本身的 `stateless` 标志**——选项（`WithNode`/`WithStatelessUserID`）只是为 RPC 提供必要参数。

> **注意：客户端分片上传不在此表**——客户端上传确认走 `m.Upload()`（[UploadManagement 接口](pkg/filemanager/manager/upload.go#L188-L221)）+ `m.CompleteUpload()`（[upload.go#L289-L324](pkg/filemanager/manager/upload.go#L289-L324)），分片组合直接驱动 `d.Put()` / `d.CompleteUpload()`，**不经过 `fm.Update()`（[FileOperation 接口](pkg/filemanager/manager/upload.go#L326-L367)）编排的三段式**。本表仅汇总服务端任务（归档/解压/远程下载）的 `fm.Update()` 分支。

### 4.6 无状态搬运与客户端上传确认的对比

仓库中存在三种"数据入库"链路，它们的触发方、凭证模型、Session 管理、失败回滚方式各有不同。此前文档将"无状态搬运"与"客户端上传确认"混为一谈，下表逐项校准：

| 维度 | 客户端上传确认 | 无状态搬运（stateless） | 有状态服务端上传（stateful） |
|------|-------------|-------------------|---------------------|
| **触发方** | 外部客户端（HTTP API） | 从节点任务（SlaveUploadTask / SlaveExtractArchiveTask） | 主节点任务（archive / extract / remote_download on master） |
| **入口方法** | 会话创建：`CreateUploadSessionService.Create` → `m.CreateUploadSession`；分片写入：`UploadService.LocalUpload` → `m.Upload()`（每片，[upload.go#L188](service/explorer/upload.go#L188)）；最后一片：`m.CompleteUpload()`（[upload.go#L202](service/explorer/upload.go#L202)） | `fm.Update(WithStatelessUserID, WithNode)` → 内部调 `updateStateless()` | `fm.Update()`（无 stateless 选项） |
| **manager 类型** | 主节点：stateful（`NewFileManager(dep, user)`）；从节点分片接收 `SlaveUpload`：stateless（`NewFileManager(dep, nil)`，[service/explorer/upload.go#L150](service/explorer/upload.go#L150)） | stateless（`NewFileManager(dep, nil)`） | stateful（`NewFileManager(dep, user)`） |
| **凭证生成** | 生成 `UploadCredential`（S3 presigned URL / OSS token），返回给客户端 | 无外部凭证，从节点直接 driver.Put | 无外部凭证，主节点直接 driver.Put |
| **Session 缓存** | KV（`UploadSessionCachePrefix`，TTL = UploadSessionTTL）+ 可选哨兵任务 | RPC 响应内存传递，**不入 KV** | 进程内传递，**不入 KV** |
| **分片确认** | `ConfirmUploadSession()` 校验分片偏移、锁令牌、策略中转约束 | 无分片（整文件 Put） | 无分片（整文件 Put） |
| **中转约束** | `ConfirmUploadSession`：非本地 + 非中转 → `CodePolicyNotAllowed`（[upload.go#L168-L170](pkg/filemanager/manager/upload.go#L168-L170)） | 无此约束（从节点本地 Put，不经客户端中转） | 无此约束（主节点本地 Put） |
| **物理写入** | 非中转：客户端直传存储驱动（Cloudreve 不调 `d.Put`）；中转：`m.Upload()` 每片调 `d.Put()`（[upload.go#L188](service/explorer/upload.go#L188)），主→从经 remote driver 转发到从节点 `SlaveUpload` | 从节点 `CastStoragePolicyOnSlave` 后 `m.Upload()` → `d.Put()`（整文件） | 主节点 `m.Upload()` → `d.Put()`（整文件） |
| **完成入库** | 主节点 `m.CompleteUpload()`：`d.CompleteUpload()` + `m.fs.CompleteUpload()`（占位升级+版本裁剪）；从节点 stateless manager 的 `m.fs` 为 nil，**仅做 `d.CompleteUpload()`，跳过 DBFS**（[upload.go#L303-L307](pkg/filemanager/manager/upload.go#L303-L307) `if m.fs != nil`） | RPC `CompleteUpload` → 主节点 `StatelessCompleteUpload()` → `fm.CompleteUpload()`（DBFS 完成） | 本地 `CompleteUpload`：`d.CompleteUpload()` + `m.fs.CompleteUpload()` |
| **哨兵清理** | `UploadSentinelCheckTask`（COS/S3 超时兜底，[upload.go#L501-L548](pkg/filemanager/manager/upload.go#L501-L548)） | **无**（无外部客户端超时风险） | **无** |
| **失败回滚** | `OnUploadFailed`：释放锁 / 删占位文件 / 版本回滚 | RPC `OnUploadFailed` → 主节点回滚 + 从节点 driver.Delete | `OnUploadFailed`：释放锁 / 删占位文件 / 版本回滚 |
| **返回值** | `fs.File`（返回给客户端） | `nil, nil`（从节点不需要 File 对象） | `fs.File` |
| **后续任务** | 主节点 `onNewEntityUploaded`：媒体元数据 + 全文索引；从节点 stateless 跳过（[upload.go#L439](pkg/filemanager/manager/upload.go#L439)） | **跳过**（[upload.go#L439](pkg/filemanager/manager/upload.go#L439) `if !m.stateless`） | `onNewEntityUploaded`：媒体元数据 + 全文索引 |

> **核心差异总结**：
> 1. **客户端上传确认**是"先发凭证、客户端自行上传、再回调确认"的异步两段式（CreateUploadSession → 客户端传 → CompleteUpload），需要 KV 缓存 Session 和哨兵兜底超时；分片写入走 `m.Upload()`（UploadManagement 接口），**不经过 `fm.Update()`**。
> 2. **无状态搬运**是"从节点本地读文件 → RPC 到主节点建占位 → 本地 Put → RPC 完成入库"的同步三段式，Session 不入 KV、无哨兵、无客户端凭证，但**主节点仍完整执行能力校验与所有权检查**（见 4.4 节权限叠加说明）；DBFS 完成通过显式 RPC（`o.Node.CompleteUpload`）触发。
> 3. **有状态服务端上传**是无状态搬运的"本地版"——同样的三段式但全部在主节点进程内完成，返回 File 对象并触发后续媒体/索引任务。
> 4. **关键区分：`m.Upload()` vs `m.Update()`**——`m.Upload()`（[UploadManagement 接口](pkg/filemanager/manager/upload.go#L188-L221)）仅做物理写入 `d.Put()`，是三种链路共用的底层原语；`m.Update()`（[FileOperation 接口](pkg/filemanager/manager/upload.go#L326-L367)）编排完整三段式（PrepareUpload → Upload → CompleteUpload），仅服务端任务（归档/解压/远程下载）调用。客户端上传确认走的是 `m.Upload()` + `m.CompleteUpload()` 分片组合，不走 `m.Update()`。
> 5. 三者的 DB 层实体入库逻辑（占位创建 → 升级提交 → 版本裁剪 → StorageDiff）完全一致，差异仅在于**物理写入由谁执行**（客户端 / 从节点 / 主节点）和**DB 操作经由什么通道**（HTTP 回调 / RPC / 本地调用）。但客户端上传在从节点（stateless）的 `CompleteUpload` **不做 DBFS 完成**（`m.fs` 为 nil），DBFS 完成始终在主节点执行。

#### 4.6.1 客户端上传确认：四种场景的完整责任链路

"客户端上传确认"并非单一链路——根据存储策略的**节点归属**（主/从）和**中转设置**（Relay=true/false），实际分为四种场景，各节点的责任截然不同：

##### 场景 A：主节点本地策略（Local policy）

```
客户端 ←→ 主节点（同时负责物理写入 + DBFS）

[主节点 CreateUploadSession] → m.fs.PrepareUpload() 建占位 + KV 存 Session + 可选哨兵任务
      │
      ├─► 非中转：d.Token() 生成凭证 → 客户端直传存储（S3/COS 等）→ 云厂商 HTTP 回调到主节点
      │     ProcessCallback(c) → m.CompleteUpload() → d.CompleteUpload() + m.fs.CompleteUpload()
      │
      └─► 中转：不生成凭证，客户端分片 POST 到主节点 FileUpload 接口
            LocalUpload → ConfirmUploadSession → processChunkUpload
                  ├─► 每片：m.Upload() → Local driver.Put()
                  └─► 最后一片：m.CompleteUpload() → m.fs.CompleteUpload() + onNewEntityUploaded
```

**责任分配**：
| 责任 | 承担方 | 代码位置 |
|------|--------|---------|
| 占位创建 | 主节点 stateful manager | [upload.go#L81](pkg/filemanager/manager/upload.go#L81) |
| 物理写入 | 非中转=客户端/云厂商；中转=主节点 Local driver | [upload.go#L98](pkg/filemanager/manager/upload.go#L98) vs [L188](pkg/filemanager/manager/upload.go#L188) |
| DBFS 完成入库 | 主节点 stateful manager | 回调 [callback/upload.go#L54](service/callback/upload.go#L54) 或 本地 [upload.go#L304](pkg/filemanager/manager/upload.go#L304) |
| 媒体元数据 + 全文索引 | 主节点 `onNewEntityUploaded` | [upload.go#L438-L444](pkg/filemanager/manager/upload.go#L438-L444) |
| 失败回滚 | 主节点 `OnUploadFailed` 解锁+删占位+版本回滚 | [upload.go#L371-L386](pkg/filemanager/manager/upload.go#L371-L386) |

##### 场景 B：从节点 Remote 策略（非中转 Relay=false）

**这是最容易混淆的场景**——存在**两个同 ID 的 Session** 分别存活于主/从节点的 KV 中，DBFS 完成通过从节点 Local driver 的 **HMAC 回调** 触发：

```
[主节点 CreateUploadSession]
   ├─► m.fs.PrepareUpload() 建占位
   ├─► 主节点 KV 存 Session（ID=XYZ）
   └─► Remote driver.Token()  [remote.go#L139-L158]
         ├─► 设置 session.Callback = 主节点回调 URL（PolicyTypeRemote）
         ├─► uploadClient.CreateUploadSession() → HTTP PUT 到从节点 SlaveGetUploadSession
         │      从节点 [slave.go#L88-L107]：
         │        NewFileManager(dep, nil) → stateless
         │        m.CreateUploadSession(WithUploadSession(&service.Session)) // 复用主节点传来的 Session（含 Callback URL）
         │        从节点 KV 存 Session（ID=XYZ，与主节点同 ID）
         └─► 返回上传凭证（指向从节点 SlaveUpload 端点的签名 URL）

客户端分片 POST 到从节点 /upload/:sessionId  → SlaveUpload [service/explorer/upload.go#L135-L154]
      │
      ├─► 从节点 KV 取 Session（ID=XYZ）
      ├─► stateless manager = NewFileManager(dep, nil)
      └─► processChunkUpload
            ├─► 每片：m.Upload() → CastStoragePolicyOnSlave(Remote→Local) → Local driver.Put()
            └─► 最后一片：m.CompleteUpload() [upload.go#L289-L324]
                  ├─► GetStorageDriver(Local)
                  ├─► d.CompleteUpload(session) ← **关键触发点！**
                  │      Local driver [local.go#L255-L294] 看到 session.Callback != ""
                  │      → 发送 HTTP POST（HMAC 签名，SlaveKey 加密）到主节点回调 URL
                  │
                  ├─► [主节点侧] 回调路由 /callback/remote/:sessionID/:key
                  │      middleware.UseUploadSession → 从**主节点自己的 KV** 取 Session（ID=XYZ）
                  │      ProcessCallback [service/callback/upload.go#L45-L59]
                  │        NewFileManager(dep, user) → stateful（带 user！）
                  │        m.CompleteUpload(c, uploadSession)
                  │          ├─► d.CompleteUpload → Remote driver = no-op [remote.go#L166-L168]
                  │          ├─► m.fs.CompleteUpload → **占位升级+版本裁剪+事务提交！**
                  │          ├─► m.onNewEntityUploaded → 媒体元数据 + 全文索引
                  │          └─► 主节点 KV 删除 Session（ID=XYZ）
                  │      返回 200 OK 给从节点
                  │
                  ├─► [回到从节点侧] 回调成功
                  ├─► m.fs == nil → 跳过 DBFS（从节点 stateless 无 fs）
                  └─► 从节点 KV 删除 Session（ID=XYZ）
```

**责任分配（非中转 Remote 策略）**：
| 责任 | 承担方 | 触发方式 | 代码位置 |
|------|--------|---------|---------|
| 占位创建 | **主节点** | stateful PrepareUpload | [upload.go#L81](pkg/filemanager/manager/upload.go#L81) |
| Session 入 KV | **主 + 从** 各存一份（同 ID） | 双方各调 kv.Set | [upload.go#L136-L140](pkg/filemanager/manager/upload.go#L136-L140) × 2 |
| 物理写入分片 | **从节点** | 客户端直传 SlaveUpload | [upload.go#L188](pkg/filemanager/manager/upload.go#L188) |
| DBFS 完成入库 | **主节点** | 从节点 Local driver HMAC 回调触发 | [local.go#L255-L294](pkg/filemanager/driver/local/local.go#L255-L294) → [callback/upload.go#L54](service/callback/upload.go#L54) |
| 媒体元数据 + 全文索引 | **主节点** | 回调内 onNewEntityUploaded | [upload.go#L438-L444](pkg/filemanager/manager/upload.go#L438-L444) |
| 失败-主节点清理 | **主节点** | 哨兵任务兜底（超时未回调） | [upload.go#L447-L548](pkg/filemanager/manager/upload.go#L447-L548) |
| 失败-从节点清理 | **从节点** | processChunkUpload 出错 → OnUploadFailed → Local driver.Delete | [upload.go#L387-L395](pkg/filemanager/manager/upload.go#L387-L395) |
| 失败-回调失败 | **从节点** | d.CompleteUpload 返回错误 → 整个 CompleteUpload 失败 | [local.go#L279-L291](pkg/filemanager/driver/local/local.go#L279-L291) |

##### 场景 C：从节点 Remote 策略（中转 Relay=true）

中转模式下**无回调机制**——主节点作为代理亲自将数据搬运到从节点，并在本地完成所有 DBFS：

```
[主节点 CreateUploadSession]
   ├─► Relay=true → unrelayed = false
   ├─► 跳过 d.Token()，不生成客户端凭证
   └─► 主节点 KV 存 Session

客户端分片 POST 到主节点 FileUpload → LocalUpload → processChunkUpload
      ├─► 每片：m.Upload() → d.Put(Remote driver)
      │              Remote driver.Put [remote.go#L78-L82] → remoteClient.Upload [client.go#L95-L132]
      │                ├─► 从节点创建 **内部临时 Session**（新 ID，无 Callback URL！）
      │                ├─► 分片传输到从节点 SlaveUpload
      │                │      从节点 processChunkUpload → 最后一片 → m.CompleteUpload
      │                │            ├─► d.CompleteUpload：session.Callback == "" → **无回调！直接返回**
      │                │            └─► m.fs == nil → 跳过 DBFS（本来就不需要，因为是内部临时 Session）
      │                └─► 全部分片成功后 remoteClient.Upload 返回
      └─► 最后一片：m.CompleteUpload()  **在主节点本地执行！**
                  ├─► d.CompleteUpload(Remote) = no-op
                  ├─► m.fs.CompleteUpload → **占位升级+版本裁剪+事务提交**
                  ├─► m.onNewEntityUploaded → 媒体 + 索引
                  └─► 主节点 KV 删除 Session
```

**责任分配（中转 Remote 策略）**：
| 责任 | 承担方 | 关键区别 |
|------|--------|---------|
| 占位创建 | 主节点 | 仅主节点 Session，从节点临时 Session 无占位 |
| 物理写入 | 主节点转发 → 从节点 | 从节点有**两个** Session：内部临时（无 Callback）+ 不存在（因为主节点做 DBFS） |
| DBFS 完成入库 | 主节点 | **本地调用**，无需回调 |
| 媒体元数据 + 全文索引 | 主节点 | 本地 onNewEntityUploaded |

##### 场景 D：云存储直传（S3/COS/OSS/Obs/Upyun/OneDrive）

与场景 B 结构类似，但"回调"由**第三方云存储厂商**发起：

```
[主节点 CreateUploadSession]
   ├─► S3/COS/OSS driver.Token → 生成 presigned multipart URL
   │      同时在 multipart 初始化参数中设置 Callback = 主节点 /callback/s3|cos|oss/:sessionID
   │      [s3.go#L342-L345] / [oss.go#L497-L504]
   └─► 返回凭证给客户端

客户端 → 按 presigned URL 直传云存储分片
      → 所有分片完成，云存储完成 multipart 合并
      → 云存储**主动 HTTP POST 回调**到主节点设置的 Callback URL

[主节点回调路由] /callback/{policy_type}/:sessionID/:key
      middleware.UseUploadSession → 从 KV 取 Session
      各策略特有的签名校验（OSS CallbackValidate、QiniuCallbackValidate、UpyunCallbackAuth 等）
      ProcessCallback → m.CompleteUpload → DBFS 完成 + 媒体/索引 + KV 删除
```

---

#### 4.6.2 无状态搬运（updateStateless）：完整责任链路

与客户端上传确认的"回调驱动"不同，无状态搬运完全由**显式 RPC** 驱动，Session 从**不入 KV**，从始至终只做物理搬运：

```
从节点 updateStateless(ctx, req, o) [upload.go#L400-L436]
   │
   ├─► A. PrepareUpload 阶段 → 显式 RPC 到主节点
   │      o.Node.PrepareUpload(StatelessPrepareUploadService{UploadRequest, UserID})
   │      → 主节点 [rpc.go#L46-L64]：
   │          通过 UserID 取 LoginUser，注入 UserCtx
   │          fm = NewFileManager(dep, user) → stateful（带 user！）
   │          fm.PrepareUpload → DBFS.PrepareUpload
   │              ├─► getPreferredPolicy() 选策略（用户组→策略）
   │              ├─► 能力校验：getNavigator + UploadFile+LockFile 子集判定
   │              ├─► 所有权检查
   │              ├─► generateSavePath() 生成物理路径
   │              └─► 创建占位 File + Entity（UploadSessionID、LockToken）
   │          返回 UploadSession{Policy, FileID, EntityID, LockToken, ...}
   │      **重要：这个 Session 是 RPC 响应，主/从双方都不 kv.Set()！**
   │
   ├─► B. Upload 阶段 → 从节点本地落盘
   │      m = stateless manager（m.fs == nil，m.stateless == true）
   │      m.Upload(ctx, req, CastStoragePolicyOnSlave(policy), session) [upload.go#L188-L221]
   │          ├─► GetStorageDriver → Local driver（CastStoragePolicyOnSlave 把 Remote 翻成 Local）
   │          ├─► 可选加密包装 cryptor
   │          └─► d.Put(ctx, req) → 整文件落盘（无分片！）
   │      失败：o.Node.OnUploadFailed RPC → 主节点解锁+删占位/回滚版本，从节点 driver.Delete
   │
   └─► C. CompleteUpload 阶段 → 显式 RPC 到主节点
          o.Node.CompleteUpload(StatelessCompleteUploadService{Session, UserID})
          → 主节点 [rpc.go#L70-L81]：
              通过 UserID 取 LoginUser，注入 UserCtx
              fm = NewFileManager(dep, user) → stateful
              fm.CompleteUpload(ctx, s.UploadSession) [upload.go#L289-L324]
                  ├─► d.CompleteUpload → driver 特定（Local=空，S3=合并分片等）
                  ├─► m.fs != nil → m.fs.CompleteUpload → **占位升级+版本裁剪+事务提交！**
                  ├─► session.SentinelTaskID → 取消哨兵（如有）
                  ├─► m.onNewEntityUploaded → **媒体元数据 + 全文索引**
                  │     （此处 !m.stateless = true → **会执行！** 与客户端上传从节点场景不同）
                  └─► _ = m.kv.Delete → 尝试删除 KV 但 Session 从未存入过 → 空操作 no-op
          返回 fs.File 给从节点 → 但从节点 updateStateless 直接 return nil, nil 丢弃

      ✓ 完成：从节点物理文件存在 + 主节点 DB 实体升级提交
```

**责任分配（无状态搬运）**：
| 责任 | 承担方 | 触发方式 | 代码位置 |
|------|--------|---------|---------|
| 占位创建 | 主节点 | `o.Node.PrepareUpload` 显式 RPC | [rpc.go#L46-L64](service/node/rpc.go#L46-L64) |
| Session 存储 | **不存** | RPC 内存传递，不入 KV | 对比 [upload.go#L136](pkg/filemanager/manager/upload.go#L136) 未被调用 |
| 能力校验+所有权检查 | 主节点 | RPC 内 stateful manager 执行 | [rpc.go#L55-L56](service/node/rpc.go#L55-L56) |
| 物理写入 | 从节点 | `m.Upload()` → Local driver.Put（整文件无分片） | [upload.go#L188](pkg/filemanager/manager/upload.go#L188) |
| DBFS 完成入库 | 主节点 | `o.Node.CompleteUpload` 显式 RPC | [rpc.go#L70-L80](service/node/rpc.go#L70-L80) |
| 媒体元数据 + 全文索引 | 主节点 | RPC 内 stateful onNewEntityUploaded | [upload.go#L438-L444](pkg/filemanager/manager/upload.go#L438-L444) |
| 失败-主节点清理 | 主节点 | `o.Node.OnUploadFailed` 显式 RPC | [rpc.go#L87-L98](service/node/rpc.go#L87-L98) |
| 失败-从节点清理 | 从节点 | updateStateless 内 OnUploadFailed → driver.Delete | [upload.go#L412-L417](pkg/filemanager/manager/upload.go#L412-L417) |
| 哨兵超时兜底 | **无** | 无外部客户端参与 → 无需 | 对比 [upload.go#L121-L134](pkg/filemanager/manager/upload.go#L121-L134) |

---

#### 4.6.3 关键责任差异：逐项对比

上述分析揭示了几个此前文档未充分说明的核心差异：

| 对比维度 | 客户端上传确认（非中转 Remote） | 客户端上传确认（中转 Remote） | 无状态搬运（stateless） |
|---------|-------------------------------|-----------------------------|----------------------|
| **Session 数量与位置** | 主+从 KV 各一份，**同 ID** | 仅主节点 KV（从节点内部临时 Session 独立 ID） | **不入 KV**，RPC 内存传递 |
| **主从通信方向** | 从→主（**回调驱动**，Local driver HMAC POST） | 无回调，主→从单向推数据 | 从→主（**RPC 驱动**，3 次显式调用） |
| **触发 DBFS 完成的信号** | 从节点 CompleteUpload 中 **Local driver 发现 Callback URL 非空** 时发起 HMAC 回调 | 主节点 processChunkUpload 最后一片 **本地 CompleteUpload 调用** | 从节点 updateStateless 中 **显式 `o.Node.CompleteUpload` RPC** |
| **从节点 CompleteUpload 中的 m.fs** | nil（stateless），DBFS 跳过 | 不涉及（中转场景 CompleteUpload 在主节点执行） | nil（stateless），DBFS 跳过 |
| **DBFS 执行的 manager 类型** | 主节点 stateful（回调内 NewFileManager(dep, user)） | 主节点 stateful（LocalUpload 原 manager） | 主节点 stateful（RPC 内 NewFileManager(dep, user)） |
| **物理写入是否分片** | 是，由客户端分片，从节点逐片接收 | 是，客户端→主节点→从节点 双层分片 | **否**，整文件 Put |
| **onNewEntityUploaded 执行方** | 主节点（回调内 stateful manager） | 主节点（本地 CompleteUpload 内） | 主节点（RPC 内 stateful manager） |
| **从节点 onNewEntityUploaded** | 跳过（m.stateless=true） | 跳过（从节点临时 Session 的 CompleteUpload 不涉及） | 跳过（m.stateless=true） |
| **凭证与权限模型** | 主→从：上传 URL HMAC 签名；从→主：回调 SlaveKey HMAC | 主→从：Remote client 的 SlaveKey HMAC | 全程 Node RPC，SlaveKey HMAC |
| **占位实体与文件大小** | 主节点 PrepareUpload 根据客户端请求 Props 建占位 | 同左 | 主节点 PrepareUpload 根据从节点 UploadRequest Props 建占位 |
| **回调机制类型** | Local driver.CompleteUpload 主动 HTTP POST（Cloudreve 内部机制） | **无回调** | **无回调**，显式 RPC 三段式 |
| **哨兵任务** | 主节点创建 UploadSentinelCheckTask（超时未收到回调时清理） | 主节点可选创建 | **无** |

---

#### 4.6.4 常见混淆点澄清

**混淆点 1：从节点的两种"CompleteUpload"路径**

从节点上存在两种完全不同的 CompleteUpload 调用，不可混淆：

| 场景 | 调用位置 | 触发方 | Session 中的 Callback | DBFS 完成地点 |
|------|---------|--------|---------------------|-------------|
| 接收客户端分片（非中转 Remote） | [service/explorer/upload.go#L202](service/explorer/upload.go#L202) processChunkUpload 最后一片 | 外部客户端 HTTP 请求 | **非空**（指向主节点回调 URL）→ Local driver HMAC POST 回调到主节点 | 主节点（回调内） |
| 执行无状态搬运 updateStateless | [pkg/filemanager/manager/upload.go#L421](pkg/filemanager/manager/upload.go#L421) 显式 RPC | 从节点内部任务代码 | Session 无 Callback 字段概念 | 主节点（RPC 内） |

**混淆点 2：回调机制的三种不同触发者**

客户端上传确认链路中的"回调"并非统一机制：

1. **云存储厂商发起**（S3/COS/OSS/OD）：厂商完成 multipart 后按初始化时设置的 Callback URL 发起 HTTP POST
2. **从节点 Local driver 发起**（非中转 Remote 策略）：`CompleteUpload` 中检测到 `session.Callback != ""` [local.go#L256](pkg/filemanager/driver/local/local.go#L256)，主动用 SlaveKey HMAC 签名 POST 到主节点
3. **客户端主动确认**（中转/本地策略）：客户端发送最后一片后，主节点在 `processChunkUpload` 内直接调用 `m.CompleteUpload()`，不涉及网络回调

而**无状态搬运不使用任何回调机制**——全部通过从节点主动发起的 `o.Node.XXX` 显式 RPC 三段式完成。

**混淆点 3：中转模式下从节点的"幽灵 CompleteUpload"**

中转模式下，从节点确实会在接收完主节点转发的分片后执行 CompleteUpload（内部临时 Session 的），但这个 CompleteUpload：
- Session 的 Callback 为空 → 不会回调主节点
- manager 是 stateless → 不会做 DBFS
- 本质上只是"从节点本地物理写入完成确认"，不产生任何持久化副作用
- 真正的 DBFS 完成在**主节点侧** processChunkUpload 的最后一片内完成

**混淆点 4：stateless manager 含义的双重语境**

"stateless" 这个词在两条链路中含义不同：
- **客户端上传确认（SlaveUpload）**：manager 是 stateless，指它没有 `m.fs`、无法做 DBFS，但 Session 是从 KV 里取出来的**有状态**对象
- **无状态搬运（updateStateless）**：manager 是 stateless，整个链路 Session 从不入 KV，完全**无状态**，所有状态通过 RPC 参数显式传递

---

### 4.7 失败回滚与清理

迁移/上传失败时由 [OnUploadFailed()](pkg/filemanager/manager/upload.go#L369-L397) 处理：

- **有状态（主节点）**：释放锁、删除新建占位文件、或回滚版本控制（`VersionControl(..., true)`）。
- **无状态（从节点）**：直接调用驱动 `Delete` 删除已落盘的物理文件。

占位实体超时未完成时，由 `UploadSentinelCheckTask`（[manager/upload.go#L501-L548](pkg/filemanager/manager/upload.go#L501-L548)）兜底清理：删除占位 Entity 的物理源文件并取消上传凭证。

### 4.8 涉及的关键组件

| 组件 | 职责 | 文件 |
|------|------|------|
| `StoragePolicy` | 存储策略配置，含驱动类型、路径规则、中转/加密设置 | [ent/schema/policy.go](ent/schema/policy.go) |
| `driver.Handler` | 各存储驱动具体实现（Local/Remote/COS/S3/OneDrive…） | [pkg/filemanager/driver/](pkg/filemanager/driver/) |
| `Entity` | 文件实体（物理存储实例，含 Source 路径、策略ID、UploadSessionID） | [ent/schema/entity.go](ent/schema/entity.go) |
| `File` | 逻辑文件元数据，可挂多个 Entity（版本/缩略图） | [ent/schema/file.go](ent/schema/file.go) |
| `LockSystem` | 分布式锁，迁移/上传期间防并发修改 | [pkg/filemanager/lock/](pkg/filemanager/lock/) |
| `SlaveUploadTask` | 跨节点搬运任务（实际运行的迁移载体） | [pkg/filemanager/workflows/upload.go](pkg/filemanager/workflows/upload.go) |
| `Node` / `remote.Client` | 主从 RPC 通信与鉴权 | [pkg/cluster/node.go](pkg/cluster/node.go)、[pkg/filemanager/driver/remote/client.go](pkg/filemanager/driver/remote/client.go) |

---

## 五、关键代码索引

### 5.1 路径解析核心

| 功能 | 文件位置 |
|------|---------|
| URI 解析 | [uri.go](pkg/filemanager/fs/uri.go) |
| 导航器接口 | [navigator.go#L38-L58](pkg/filemanager/fs/dbfs/navigator.go#L38-L58) |
| MyNavigator.To() | [my_navigator.go#L74-L126](pkg/filemanager/fs/dbfs/my_navigator.go#L74-L126) |
| ShareNavigator.To() | [share_navigator.go#L181-L218](pkg/filemanager/fs/dbfs/share_navigator.go#L181-L218) |
| 基础路径遍历 | [navigator.go#L177-L203](pkg/filemanager/fs/dbfs/navigator.go#L177-L203) |
| File 双路径视图 | [file.go#L177-L199](pkg/filemanager/fs/dbfs/file.go#L177-L199) |

### 5.2 权限系统核心

| 功能 | 文件位置 |
|------|---------|
| BooleanSet 实现 | [boolset.go](pkg/boolset/boolset.go) |
| 导航器能力集定义 | [navigator.go#L80-L149](pkg/filemanager/fs/dbfs/navigator.go#L80-L149) |
| 权限校验 | [dbfs.go#L763-L769](pkg/filemanager/fs/dbfs/dbfs.go#L763-L769) |
| 所有权检查 | [manage.go#L60-L62](pkg/filemanager/fs/dbfs/manage.go#L60-L62) |
| 用户组→策略 | [inventory/policy.go#L145-L155](inventory/policy.go#L145-L155) |
| 主从策略翻译 | [manager/fs.go](pkg/filemanager/manager/fs.go) `CastStoragePolicyOnSlave` |

### 5.3 节点迁移核心

| 功能 | 文件位置 |
|------|---------|
| 专用迁移数据结构（脚手架） | [fs.go#L380-L392](pkg/filemanager/fs/fs.go#L380-L392) |
| 专用迁移参数（脚手架） | [inventory/file.go#L136-L141](inventory/file.go#L136-L141) |
| 迁移任务类型 | [queue/task.go#L104](pkg/queue/task.go#L104) |
| 可恢复任务声明 | [application/dependency/dependency.go#L683](application/dependency/dependency.go#L683) |
| 存储路径生成 | [dbfs.go#L788-L803](pkg/filemanager/fs/dbfs/dbfs.go#L788-L803) |
| 存储策略获取 | [dbfs.go#L669-L682](pkg/filemanager/fs/dbfs/dbfs.go#L669-L682) |
| 跨节点搬运任务 | [pkg/filemanager/workflows/upload.go](pkg/filemanager/workflows/upload.go) |
| 压缩任务（产物上传触发） | [pkg/filemanager/workflows/archive.go](pkg/filemanager/workflows/archive.go) |
| 解压任务（产物上传触发） | [pkg/filemanager/workflows/extract.go](pkg/filemanager/workflows/extract.go) |
| 远程下载入库任务 | [pkg/filemanager/workflows/remote_download.go](pkg/filemanager/workflows/remote_download.go) |
| manager stateless 判定 | [manager.go#L152-L184](pkg/filemanager/manager/manager.go#L152-L184) |
| 客户端上传会话创建 | [manager/upload.go#L50-L147](pkg/filemanager/manager/upload.go#L50-L147) |
| 客户端分片确认 | [manager/upload.go#L149-L182](pkg/filemanager/manager/upload.go#L149-L182) |
| 无状态上传入库 | [manager/upload.go#L400-L436](pkg/filemanager/manager/upload.go#L400-L436) |
| 占位实体创建/升级 | [dbfs/upload.go#L72-L260](pkg/filemanager/fs/dbfs/upload.go#L72-L260)、[dbfs/upload.go#L262-L366](pkg/filemanager/fs/dbfs/upload.go#L262-L366) |
| 主节点 RPC 入口 | [service/node/rpc.go](service/node/rpc.go) |
| 失败回滚/哨兵清理 | [manager/upload.go#L369-L397](pkg/filemanager/manager/upload.go#L369-L397)、[manager/upload.go#L501-L548](pkg/filemanager/manager/upload.go#L501-L548) |

---

## 六、设计亮点与注意事项

### 6.1 设计亮点

1. **URI 统一标识**：四种文件系统通过 URI Scheme 区分，接口统一。
2. **双路径视图**：所有者视图与访问者视图分离，支持分享链接的路径透明。
3. **位图权限存储**：BooleanSet 高效存储权限，位运算快速校验。
4. **导航器模式**：新增文件系统类型只需实现 Navigator 接口。
5. **上下文缓存**：File 对象缓存子节点，减少 DB 查询。
6. **占位-落盘-升级三段式入库**：跨节点场景下主节点先建占位、从节点落盘、主节点升级提交，兼顾事务原子性与物理传输解耦。
7. **manager stateless 自动路由**：`NewFileManager(dep, u)` 在 SlaveMode 或 `u == nil` 时返回 stateless manager（`m.stateless = true`），使同一 `fm.Update()` 调用在主节点走本地三段、在从节点自动走 RPC 三段，调用方代码无需感知节点角色差异。

### 6.2 注意事项

1. **路径转义**：`PathEscape()` 与标准库 `url.PathEscape` 不同，需注意前端一致性。
   [uri.go#L371-L444](pkg/filemanager/fs/uri.go#L371-L444)

2. **权限继承**：子文件从父目录继承 CapabilitiesBs，修改父目录权限需递归更新。

3. **符号链接**：分享目录通过符号链接实现，递归翻译可能产生循环引用。

4. **迁移任务尚未接通**：`RelocateTaskType` 仅有数据结构与类型注册，无工厂、无调用点；真正的跨驱动搬运由 `SlaveUploadTask` + 无状态上传链路承担。二次开发若要实现“存储策略切换”功能，应复用 4.2 的三段式入库，而非直接接 `PrepareRelocateRes`（其字段如 `ParentFiles`/`PrimaryEntityParentFiles` 的处理逻辑尚未实现）。

5. **主从权限不绕过**：跨节点 RPC 仍会在主节点重新叠加导航器能力与所有权检查，从节点只负责物理传输，不持有业务权限决策。

6. **版本保留与清理**：`CompleteUpload` 通过 `CapEntities` 按所有者版本保留设置裁剪旧实体，裁剪产生的 `StorageDiff` 会触发物理文件回收任务（`ExplicitEntityRecycleTask`），迁移时需注意旧实体回收的异步性。

7. **三种上传链路不可混用**：客户端上传确认（`CreateUploadSession` + `ConfirmUploadSession` + `CompleteUpload`）、无状态搬运（`updateStateless`）、有状态服务端上传（stateful `Update`）是三条独立链路（详见 4.6 节）。客户端上传确认有 KV 缓存、凭证生成、哨兵兜底；后两者无 KV、无凭证、无哨兵。二次开发若需在服务端搬运文件，应通过 `fm.Update()` 走 stateful 或 stateless 分支（由 `NewFileManager` 的 user 参数决定），**不要**混用 `CreateUploadSession`（那会生成客户端凭证并入 KV，服务端任务无法消费）。

8. **归档从节点路径的 completeUpload 是空操作**：`CreateArchiveTask` 在从节点路径下，压缩和上传均委托给从节点的 Slave 任务完成，主节点的 `completeUpload()` 阶段仅返回 `StatusCompleted` 不做任何 DB 操作——因为从节点的无状态上传已通过 RPC 在主节点完成了实体入库。二次开发若扩展归档任务，不应在 `completeUpload` 阶段重复执行实体入库逻辑。

9. **`m.Upload()` 与 `m.Update()` 接口区分**：二者属于不同接口，不可混淆。`m.Upload()` 属 [UploadManagement 接口](pkg/filemanager/manager/upload.go#L188-L221)，仅做物理写入 `d.Put(ctx, req)`，不创建占位、不完成入库，是客户端分片上传、无状态搬运、有状态服务端上传三条链路共用的底层原语；`m.Update()` 属 [FileOperation 接口](pkg/filemanager/manager/upload.go#L326-L367)，编排完整三段式（PrepareUpload → Upload → CompleteUpload），仅服务端任务（归档/解压/远程下载）调用。客户端上传确认（`UploadService.LocalUpload` → `processChunkUpload`）全程调 `m.Upload()` + `m.CompleteUpload()`，**不调 `m.Update()`**。
