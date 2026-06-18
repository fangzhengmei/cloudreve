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

### 3.2 导航器能力集定义

在 [navigator.go#L80-L149](pkg/filemanager/fs/dbfs/navigator.go#L80-L149) 中定义了四种导航器的能力集：

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

权限叠加发生在 **导航器初始化** 和 **目录遍历** 两个阶段，叠加的结果是“导航器能力集”与“文件系统语义”的交集：

```
1. 导航器初始化时定义基础能力集
   [navigator.go#L111-L149](pkg/filemanager/fs/dbfs/navigator.go#L111-L149)
         │
         ▼
2. 根目录创建时绑定能力集
   [my_navigator.go#L111](pkg/filemanager/fs/dbfs/my_navigator.go#L111)
   root.CapabilitiesBs = n.Capabilities(false).Capability
         │
         ▼
3. 子文件创建时从父目录继承
   [file.go#L329-L353](pkg/filemanager/fs/dbfs/file.go#L329-L353)
   ├─► newFile(parent, model) 时
   └─► f.CapabilitiesBs = parent.CapabilitiesBs
         │
         ▼
4. 操作前校验权限
   [dbfs.go#L763-L769](pkg/filemanager/fs/dbfs/dbfs.go#L763-L769)
   capabilities := res.Capabilities(false).Capability
   for _, capability := range requiredCapabilities {
       if !capabilities.Enabled(int(capability)) {
           return ErrNotSupportedAction
       }
   }
```

> 关键点：`getNavigator()` 在创建导航器时会传入 `requiredCapabilities`，导航器工厂内部用这些能力位与目标文件能力集做“与”运算，从而把“当前用户所在视图是否允许该动作”叠加到最终结果上。能力集不是动态计算的，而是根目录一次性写入、子节点继承的位图，叠加发生在校验时的 `Enabled()` 逐位判断中。

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

### 3.5 所有权检查

在 [manage.go](pkg/filemanager/fs/dbfs/manage.go) 中，每个修改操作前都会检查所有权：

```go
// 通用所有权检查模式
if _, ok := ctx.Value(ByPassOwnerCheckCtxKey{}).(bool); !ok && target.Owner().ID != f.user.ID {
    return ErrOwnerOnly
}
```

可通过 `WithBypassOwnerCheck()` 绕过所有权检查（供系统内部调用）。

### 3.6 存储驱动权限关系

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

### 4.5 失败回滚与清理

迁移/上传失败时由 [OnUploadFailed()](pkg/filemanager/manager/upload.go#L369-L397) 处理：

- **有状态（主节点）**：释放锁、删除新建占位文件、或回滚版本控制（`VersionControl(..., true)`）。
- **无状态（从节点）**：直接调用驱动 `Delete` 删除已落盘的物理文件。

占位实体超时未完成时，由 `UploadSentinelCheckTask`（[manager/upload.go#L501-L548](pkg/filemanager/manager/upload.go#L501-L548)）兜底清理：删除占位 Entity 的物理源文件并取消上传凭证。

### 4.6 涉及的关键组件

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

### 6.2 注意事项

1. **路径转义**：`PathEscape()` 与标准库 `url.PathEscape` 不同，需注意前端一致性。
   [uri.go#L371-L444](pkg/filemanager/fs/uri.go#L371-L444)

2. **权限继承**：子文件从父目录继承 CapabilitiesBs，修改父目录权限需递归更新。

3. **符号链接**：分享目录通过符号链接实现，递归翻译可能产生循环引用。

4. **迁移任务尚未接通**：`RelocateTaskType` 仅有数据结构与类型注册，无工厂、无调用点；真正的跨驱动搬运由 `SlaveUploadTask` + 无状态上传链路承担。二次开发若要实现“存储策略切换”功能，应复用 4.2 的三段式入库，而非直接接 `PrepareRelocateRes`（其字段如 `ParentFiles`/`PrimaryEntityParentFiles` 的处理逻辑尚未实现）。

5. **主从权限不绕过**：跨节点 RPC 仍会在主节点重新叠加导航器能力与所有权检查，从节点只负责物理传输，不持有业务权限决策。

6. **版本保留与清理**：`CompleteUpload` 通过 `CapEntities` 按所有者版本保留设置裁剪旧实体，裁剪产生的 `StorageDiff` 会触发物理文件回收任务（`ExplicitEntityRecycleTask`），迁移时需注意旧实体回收的异步性。
