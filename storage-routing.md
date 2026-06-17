# Cloudreve 存储路由机制详解

本文档对照代码讲清：当一个请求（上传/下载/删除等）到达 Cloudreve 后，系统如何决定使用哪种存储后端（S3、本地盘、Remote 从机等），请求沿哪条路径转发到具体驱动，以及在出错时如何兜底。

> **易混概念澄清**：WebDAV 与 remote 是两个完全不同层次的概念，分属协议入口层和存储驱动层，详见 [第 1.4 节](#14-两层易混概念辨析-webdav入口协议-vs-remote从机存储策略)。

---

## 1. 核心概念与数据模型

### 1.1 StoragePolicy（存储策略）

存储策略是路由的核心实体，对应数据库表 `storage_policies`，由 ent schema 定义：

- [schema/policy.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/ent/schema/policy.go#L15-L39) — 字段包括 `name`、`type`、`server`、`bucket_name`、`access_key`、`secret_key`、`max_size`、`dir_name_rule`、`file_name_rule`、`settings`（JSON）、`node_id`
- `type` 字段是策略类型字符串，定义在 [types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/inventory/types/types.go#L301-L311)：

```
PolicyTypeLocal  = "local"     // 本地磁盘
PolicyTypeQiniu  = "qiniu"     // 七牛
PolicyTypeUpyun  = "upyun"     // 又拍云
PolicyTypeOss    = "oss"       // 阿里云 OSS
PolicyTypeCos    = "cos"       // 腾讯云 COS
PolicyTypeS3     = "s3"        // S3 兼容
PolicyTypeKs3    = "ks3"       // 金山云 KS3
PolicyTypeOd     = "onedrive"  // OneDrive
PolicyTypeRemote = "remote"    // 远程从机（Master ↔ Slave 间 HTTP RPC）
PolicyTypeObs    = "obs"       // 华为云 OBS
```

### 1.2 Group → StoragePolicy 的绑定关系

用户组（Group）与存储策略是一对多关系：每个用户组绑定一个默认存储策略。

- [inventory/policy.go#GetByGroup](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/inventory/policy.go#L145-L155) — 通过 Group 查询其关联的 StoragePolicy

### 1.3 Entity → StoragePolicy 的绑定关系

每个物理文件实体（Entity）通过 `storage_policy_entities` 外键字段绑定到创建它时的存储策略。

- [fs.go#DbEntity.PolicyID](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/fs/fs.go#L780-L782) — `e.model.StoragePolicyEntities` 即为 PolicyID

这意味着：同一文件的不同版本（Entity）可以存储在不同的后端策略上。

### 1.4 两层易混概念辨析：WebDAV（入口协议） vs Remote（从机存储策略）

这是两个完全不同层次的概念，切勿混淆：

| 维度 | WebDAV | Remote（从机策略） |
|------|--------|-------------------|
| **所处层次** | 协议入口层（最上层，面向客户端） | 存储驱动层（底层，面向存储后端） |
| **代码位置** | `pkg/webdav/` + `middleware/auth.go#WebDAVAuth` | `pkg/filemanager/driver/remote/` |
| **面向对象** | 终端用户的 WebDAV 客户端（Finder/资源管理器等） | Master 节点 ↔ Slave 节点的内部 RPC |
| **核心作用** | 把 WebDAV 协议请求（PUT/GET/LOCK/MKCOL…）翻译成 FileManager 的内部调用 | 作为 `driver.Handler` 的一个实现，通过 HTTP RPC 调用 Slave 节点的存储能力 |
| **鉴权方式** | HTTP Basic Auth → DAV 账号 → 用户 | HMAC 签名（SlaveKey）+ 节点 ID |
| **是否有状态** | 有用户上下文（关联 Group / 权限校验） | Master 侧有状态，Slave 侧 stateless |
| **与策略的关系** | 不关心具体存储策略，全部交给 FileManager | 本身就是一种策略类型（`PolicyTypeRemote`），与 local/s3 平级 |
| **挂载点 / 入口** | `/dav/*` 路由 | `GetStorageDriver` 中 switch 分支的一个 case |

**一句话总结**：WebDAV 是"门"，客户端从这扇门进来；remote 是"快递员"，Master 节点派它去 Slave 节点取货/送货。两扇门（Web 端 REST API 和 WebDAV）都通往同一个大厅（FileManager），大厅里有多个快递员（local/s3/remote/oss/cos/...）。

**完整的三层架构**（自上而下）：

```
┌─────────────────────────────────────────────────────────┐
│  协议入口层                                              │
│  ┌──────────────┐   ┌──────────────┐                    │
│  │  Web API     │   │   WebDAV     │                    │
│  │  /api/v4/*   │   │   /dav/*     │                    │
│  │ Session Auth │   │  Basic Auth  │                    │
│  └──────┬───────┘   └──────┬───────┘                    │
└─────────┼──────────────────┼────────────────────────────┘
          │                  │
          ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│  FileManager 层（业务逻辑层）                            │
│  · DBFS（数据库文件系统）                                │
│  · 策略匹配（getPreferredPolicy / getEntityPolicyDriver）│
│  · 驱动分发（GetStorageDriver）                         │
│  · 版本控制 / 锁 / 回收 / 缩略图 / 加密                 │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  存储驱动层（driver.Handler 实现）                      │
│  ┌────────┐ ┌──────┐ ┌────────┐ ┌──────┐ ┌──────────┐   │
│  │ local  │ │ s3   │ │ remote │ │ oss  │ │ onedrive │   │
│  │ (本地盘)│ │ (S3) │ │ (从机) │ │ (OSS)│ │ (OneDrive)│  │
│  └────────┘ └──────┘ └───┬────┘ └──────┘ └──────────┘   │
│                           │                              │
│                      HTTP RPC                           │
│                           │                              │
│                  ┌────────▼───────┐                     │
│                  │  Slave 节点    │                     │
│                  │  local 驱动    │                     │
│                  └────────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

---

## 2. 策略匹配：请求如何命中具体存储驱动

### 2.1 上传场景 — 由用户组决定策略

上传时的策略匹配入口在 `DBFS.getPreferredPolicy`：

[dbfs.go#L668-L682](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L668-L682)

```go
func (f *DBFS) getPreferredPolicy(ctx context.Context, file *File) (*ent.StoragePolicy, error) {
    ownerGroup := file.Owner().Edges.Group
    if ownerGroup == nil {
        return nil, fmt.Errorf("owner group not loaded")
    }
    sc, _ := inventory.InheritTx(ctx, f.storagePolicyClient)
    groupPolicy, err := sc.GetByGroup(ctx, ownerGroup)
    ...
    return groupPolicy, nil
}
```

**匹配逻辑**：取目标父目录的 Owner → Owner 的 Group → Group 关联的 StoragePolicy。

此方法在以下场景被调用：
- [upload.go#L124](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/fs/dbfs/upload.go#L124) — `PrepareUpload` 中确定上传策略
- [manage.go#L125](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L125) — `Create` 文件时获取策略做扩展名校验
- [manage.go#L180](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/fs/dbfs/manage.go#L180) — `Rename` 文件时做扩展名校验

**特殊场景**：用户可在创建上传会话时通过 `policy_id` 指定 PreferredStoragePolicy，覆盖默认策略。见 [upload.go#L126-L127](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/fs/dbfs/upload.go#L126-L127) — 导入物理文件时直接按指定策略 ID 查询。

### 2.2 下载/读取场景 — 由 Entity 已有绑定决定策略

下载或读取文件时，策略由 Entity 自身的 `PolicyID` 字段决定，不需要重新匹配：

[entity.go#L92-L117](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/fs.go#L92-L117)

```go
func (m *manager) getEntityPolicyDriver(cxt context.Context, e fs.Entity, policyOverwrite *ent.StoragePolicy) (*ent.StoragePolicy, driver.Handler, error) {
    policyID := e.PolicyID()
    if policyID == 0 {
        policy = &ent.StoragePolicy{Type: types.PolicyTypeLocal, Settings: &types.PolicySetting{}}
    } else {
        if policyOverwrite != nil && policyOverwrite.ID == policyID {
            policy = policyOverwrite
        } else {
            policy, err = m.policyClient.GetPolicyByID(cxt, e.PolicyID())
        }
    }
    d, err := m.GetStorageDriver(cxt, policy)
    return policy, d, nil
}
```

**关键点**：`PolicyID == 0` 时兜底为本地策略（内存构造的空 Policy），不会报错。

### 2.3 从机模式下的策略转换 — CastStoragePolicyOnSlave

在分布式部署中，Master 节点的 "remote" 策略在 Slave 节点上应被视为 "local"，反之亦然：

[fs.go#L31-L63](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/fs.go#L31-L63)

```go
func (m *manager) CastStoragePolicyOnSlave(ctx context.Context, policy *ent.StoragePolicy) *ent.StoragePolicy {
    if !m.stateless { return policy }
    nodeId := cluster.NodeIdFromContext(ctx)
    if policy.Type == types.PolicyTypeRemote {
        if nodeId != policy.NodeID { return policy }
        policyCopy.Type = types.PolicyTypeLocal  // 远程策略 → 本地策略
        return &policyCopy
    } else if policy.Type == types.PolicyTypeLocal {
        policyCopy.Type = types.PolicyTypeRemote  // 本地策略 → 远程策略
        policyCopy.NodeID = nodeId
        return &policyCopy
    } else if policy.Type == types.PolicyTypeOss {
        policyCopy.Settings.ServerSideEndpoint = ""  // OSS 清除内网端点
    }
    return policy
}
```

---

## 3. 转发路径：请求从入口到存储驱动的完整调用链路

### 3.1 上传链路

```
HTTP Request
  → routers/controllers/file.go#CreateUploadSession
    → service/explorer/upload.go#CreateUploadSessionService.Create
      → manager.NewFileManager(dep, user)
      → manager.CreateUploadSession(ctx, req)
        → manager.upload.go#L50-L147
          ├── fs.PrepareUpload(ctx, req)          // DBFS 创建占位文件/Entity
          │   └── dbfs.upload.go#L72-L260
          │       └── getPreferredPolicy(ctx, ancestor)  // 按 Group 匹配策略
          ├── GetStorageDriver(ctx, CastStoragePolicyOnSlave(ctx, policy))  // 策略→驱动
          │   └── manager.fs.go#L65-L89  // switch policy.Type 分发
          └── d.Token(ctx, uploadSession, req)     // 获取上传凭证（预签名URL等）
```

**关键代码位置**：
- [manager/upload.go#L87](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/upload.go#L87) — `GetStorageDriver` + `CastStoragePolicyOnSlave`
- [manager/fs.go#L65-L89](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/fs.go#L65-L89) — 类型到驱动的 switch 分发

### 3.2 WebDAV 入口链路（PUT / GET / MKCOL / LOCK 等）

WebDAV 是独立于 REST API 的另一个入口，使用 Basic Auth 鉴权，挂载在 `/dav` 路径下。

**路由注册**：
- [router.go#L1336-L1350](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/routers/router.go#L1336-L1350) — `initWebDAV` 注册 `Any("/*path", webdav.ServeHTTP)` 及 PROPFIND/MKCOL/LOCK 等特殊方法

**WebDAV 鉴权中间件**：
- [middleware/auth.go#L104-L174](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/middleware/auth.go#L104-L174) — `WebDAVAuth`

鉴权流程：
1. 从 HTTP Basic Auth 提取用户名/密码
2. `userClient.GetActiveByDavAccount(c, username, password)` 按 DAV 账号查用户
3. 验证用户组是否启用了 WebDAV 权限（`GroupPermissionWebDAV`）
4. 若账号只读，则拦截 PUT/DELETE/MKCOL/COPY/MOVE/LOCK/UNLOCK 等写操作
5. `SetUserCtxByUser(c, expectedUser)` 将用户注入上下文

**核心分发器 ServeHTTP**：
- [webdav.go#L55-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/webdav/webdav.go#L55-L95)

```go
func ServeHTTP(c *gin.Context) {
    fm := manager.NewFileManager(dep, u)
    defer fm.Recycle()

    switch c.Request.Method {
    case "OPTIONS":  status, err = handleOptions(c, u, fm)
    case "GET", "HEAD", "POST":  status, err = handleGetHeadPost(c, u, fm)
    case "DELETE":  status, err = handleDelete(c, u, fm)
    case "PUT":     status, err = handlePut(c, u, fm)
    case "MKCOL":   status, err = handleMkcol(c, u, fm)
    case "COPY", "MOVE":  status, err = handleCopyMove(c, u, fm)
    case "LOCK":    status, err = handleLock(c, u, fm)
    case "UNLOCK":  status, err = handleUnlock(c, u, fm)
    case "PROPFIND":status, err = handlePropfind(c, u, fm)
    case "PROPPATCH": status, err = handleProppatch(c, u, fm)
    }
}
```

所有 WebDAV 方法共用同一个 `FileManager` 实例，与 Web 端 REST API 完全复用同一套 FileManager。

### 3.3 WebDAV PUT → 文件管理层的完整路径

[webdav.go#L227-L285](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/webdav/webdav.go#L227-L285) — `handlePut`

```
╔══════════════════════════════════════════════════════════════════╗
║  WebDAV 协议入口层（pkg/webdav/）                                 ║
║  职责：协议翻译、鉴权、锁管理、路径转换                            ║
╠══════════════════════════════════════════════════════════════════╣
║  PUT /dav/path/to/file.ext                                       ║
║    → WebDAVAuth 中间件 (Basic Auth → DAV 账号 → 用户)           ║
║    → stripPrefix: 剥离 /dav 前缀，得到相对路径                   ║
║    → fm.SharedAddressTranslation: 路径 → fs.File + fs.URI       ║
║    → DavAccountDisableSysFiles 检查                              ║
║    → confirmLock: 确认/创建 WebDAV 协议级锁                     ║
║    → request.SniffContentLength: 嗅探文件大小                    ║
║    → 构造 fs.UploadRequest (Mode=ModeOverwrite)                  ║
╚═════════════════════════╤════════════════════════════════════════╝
                          │
          ──── WebDAV / FileManager 边界 ────
                          │
╔═════════════════════════▼════════════════════════════════════════╗
║  FileManager 层（pkg/filemanager/manager/ + fs/dbfs/）           ║
║  职责：策略匹配、版本控制、加密、回收、实体管理                     ║
╠══════════════════════════════════════════════════════════════════╣
║  m.Update(ctx, fileData)                                         ║
║    ├── fs.PrepareUpload(ctx, req)                                ║
║    │    └── getPreferredPolicy(ctx, ancestor)  // Group→Policy  ║
║    ├── m.Upload(ctx, req, policy, session)                       ║
║    │    └── GetStorageDriver(...)  // 策略→驱动分发              ║
║    │         ┌───────────────────────────────────────────┐      ║
║    │         │  存储驱动层（driver.Handler 实现）        │      ║
║    │         │  · local.Driver.Put  → 本地磁盘            │      ║
║    │         │  · s3.Driver.Put     → S3 兼容存储         │      ║
║    │         │  · remote.Driver.Put → Slave 节点 HTTP RPC│      ║
║    │         │  · cos/oss/obs/ks3/qiniu/upyun/onedrive   │      ║
║    │         └───────────────────────────────────────────┘      ║
║    └── m.CompleteUpload(ctx, session)                           ║
╚══════════════════════════════════════════════════════════════════╝
```

**两层边界点**：WebDAV 层在 `m.Update()` 调用处交出控制权，之后的策略匹配、驱动分发、版本控制完全由 FileManager 处理，WebDAV 不感知也不干预。

**关键点**：
- WebDAV PUT 的核心上传执行 (`manager.Upload` → `driver.Handler.Put`) 与 Web 端 REST API 上传 **完全走同一条代码路径**，策略匹配、驱动分发、加密、失败清理机制全部复用
- 差异仅三点：WebDAV 用 `ModeOverwrite` 覆盖、直接传 HTTP Body 流、带 WebDAV 协议级锁（`confirmLock`）
- WebDAV 协议锁（`fs.Lock` / DBFS lock 表）与文件管理器内部锁（上传锁、目录锁）是两套独立的锁机制，WebDAV 写操作会同时持有两把

### 3.3.1 Remote 策略的跨节点上传链路（Master → Slave）

当用户组绑定的存储策略类型是 `remote` 时，`GetStorageDriver` 返回 `remote.Driver`，上传请求会跨节点转发到 Slave 节点。

**Master 侧（remote.Driver.Put）**：
[client.go#L95-L139](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/remote/client.go#L95-L139) — `remoteClient.Upload`

```
remote.Driver.Put(file)
  ├── 构造 UploadSession（生成 UUID、设置过期时间）
  ├── CreateUploadSession(ctx, session, overwrite)
  │    └── POST /api/v4/slave/upload/session  → Slave 节点
  │         └── HMAC 签名 (SlaveKey)
  ├── chunk.NewChunkGroup(file, ...)  // 分片 + 重试
  └── for each chunk:
       └── uploadChunk(...)  // PUT /api/v4/slave/upload/{id}?chunk=n
```

**Slave 侧（从机处理上传）**：

1. 创建上传会话：[slave.go#L88-L107](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/service/explorer/slave.go#L88-L107) — `SlaveCreateUploadSessionService.Create`
   - `manager.NewFileManager(dep, nil)` → 因为 `u == nil`，创建 **stateless FileManager**
   - `m.CreateUploadSession(c, req, fs.WithUploadSession(&service.Session))`
   - stateless 模式下 FS 为 nil，上传会话仅存于 KV 缓存

2. 接收分片上传：[upload.go#L135-L154](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/service/explorer/upload.go#L135-L154) — `UploadService.SlaveUpload`
   - 从 KV 取上传会话
   - `manager.NewFileManager(dep, nil)` → stateless
   - `processChunkUpload` → 分片写入本地文件
   - 内部走 local 驱动的 Put 逻辑

3. 上传完成回调：上传完成后，Slave 节点回调 Master 的 `MasterSlaveCallbackUrl`
   - Master 收到回调 → `CompleteUpload` → 标记 Entity 为有效状态

**Master ↔ Slave 的边界**：
- Master 侧的 `remote.Driver.Put` 通过 `remoteClient` 发起 HTTP 请求
- Slave 侧的 `SlaveUpload` / `SlaveGetUploadSession` 等 controller 接收请求
- 通信鉴权：HMAC 签名 + SlaveKey + 节点 ID
- Slave 侧使用 **stateless FileManager**（无用户、无 DBFS），直接操作本地文件

### 3.4 WebDAV GET → 文件管理层的完整路径

[webdav.go#L334-L372](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/webdav/webdav.go#L334-L372) — `handleGetHeadPost`

```
╔══════════════════════════════════════════════════════════════════╗
║  WebDAV 协议入口层（pkg/webdav/）                                 ║
║  职责：协议翻译、鉴权、Range 透传、强制代理决策                    ║
╠══════════════════════════════════════════════════════════════════╣
║  GET /dav/path/to/file.ext                                       ║
║    → WebDAVAuth 中间件                                          ║
║    → stripPrefix                                                ║
║    → fm.SharedAddressTranslation: 路径 → target (fs.File)       ║
║    → target.Type() 必须是 FileTypeFile                          ║
║    → fm.GetEntitySource(c, target.PrimaryEntityID())            ║
╚═════════════════════════╤════════════════════════════════════════╝
                          │
          ──── WebDAV / FileManager 边界 ────
                          │
╔═════════════════════════▼════════════════════════════════════════╗
║  FileManager 层 + EntitySource 层                                ║
╠══════════════════════════════════════════════════════════════════╣
║    ├── fs.GetEntity(ctx, entityID)        // 从 DB 取 Entity    ║
║    ├── getEntityPolicyDriver(ctx, entity, nil)                  ║
║    │    └── GetStorageDriver(policy)    // Entity.PolicyID→驱动 ║
║    ├── entitysource.NewEntitySource(...)                        ║
║    ├── es.Apply(WithSpeedLimit(...))                            ║
║    └── 决策: ShouldInternalProxy() 或 DavAccountProxy           ║
║         │                                                       ║
║         ├── true  → es.Serve(c.Writer, c.Request)               ║
║         │     ├── IsLocal()                                     ║
║         │     │    └── local.Driver.Open + http.ServeContent    ║
║         │     └── !IsLocal()                                    ║
║         │          ├── handler.Source(...)  // 取预签名 URL     ║
║         │          └── httputil.ReverseProxy  // 反向代理        ║
║         └── false → es.Url(...) + c.Redirect(302, src.Url)     ║
║                   └── s3/remote.Driver.Source → 预签名 URL     ║
╚══════════════════════════════════════════════════════════════════╝
```

**WebDAV 层的特殊决策**：当 DAV 账号开启了 `DavAccountProxy` 且用户组有 `GroupPermissionWebDAVProxy` 权限时，强制走 Cloudreve 内部代理。这是因为某些 WebDAV 客户端（如旧版 macOS Finder）不支持 302 重定向下载。

**WebDAV GET 与 Web 端 GET 的复用关系**：
- `GetEntitySource`、`getEntityPolicyDriver`、`NewEntitySource`、`EntitySource.Serve` 完全复用
- 差异：WebDAV 额外增加了一层 `DavAccountProxy` 强制代理决策

### 3.4.1 Remote 策略的跨节点下载链路（Master → Slave）

当 Entity 绑定的策略是 `remote` 类型时，下载链路会跨节点转发到 Slave 节点。

**Master 侧（remote.Driver.Source）**：
[remote.go#L111-L136](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/remote/remote.go#L111-L136) — `Driver.Source`

```
remote.Driver.Source(ctx, e, args)
  ├── routes.SlaveFileContentUrl(...)  // 组装从机下载 URL
  │    └─ /api/v4/slave/file/content/{src}/{name}
  ├── auth.SignURI(...)  // HMAC 签名 URL
  └── 返回签名后的 Slave URL
```

**Slave 侧（从机提供下载）**：
[slave.go#L41-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/service/explorer/slave.go#L41-L76) — `EntityDownloadService.SlaveServe`

```
SlaveServe(c)
  ├── base64 解码 src → 本地文件路径
  ├── local.NewLocalFileEntity(types.EntityTypeVersion, path)  // 构造本地实体
  ├── m.GetEntitySource(c, 0, fs.WithEntity(entity))
  │    └── stateless FileManager + local 驱动
  └── entitySource.Serve(c.Writer, c.Request, ...)
       └── local.Driver.Open + http.ServeContent  // 直接读本地文件
```

**Master → Slave 的下载边界**：
- Master 侧 `EntitySource.Serve` 决定走反代还是 302
  - 反代模式：Master 作为反向代理，从 Slave 拉取数据再转发给客户端（增加延迟，隐藏 Slave 地址）
  - 302 模式：Master 返回签名的 Slave URL，客户端直接访问 Slave（速度快，暴露 Slave 地址）
- Slave 侧始终走 local 驱动直接读取本地文件
- 两种模式下 Master 与 Slave 的鉴权均为 HMAC URL 签名

### 3.5 下载/获取文件 URL 链路

```
HTTP Request
  → controllers/file.go#FileURL
    → service/explorer/file.go#FileURLService.Get
      → manager.GetEntityUrls(ctx, urlReq, ...)
        → manager.entity.go#L205-L313
          ├── fs.Get(ctx, arg.URI, ...)            // DBFS 获取文件对象
          ├── FindDesiredEntity(file, ...)          // 定位目标 Entity
          ├── getEntityPolicyDriver(ctx, target, nil)  // Entity.PolicyID → Policy → Driver
          │   ├── policyClient.GetPolicyByID(ctx, e.PolicyID())  // 查策略
          │   └── GetStorageDriver(ctx, policy)                  // switch 分发
          └── entitysource.NewEntitySource(target, d, policy, ...)
              └── entitySource.Url(ctx, ...)         // 生成下载URL或内部代理URL
```

**内部代理 vs 直接外链的决策** 在 [entitysource/entitysource.go#L578-L585](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/entitysource/entitysource.go#L578-L585)：

```go
func (f *entitySource) ShouldInternalProxy(opts ...EntitySourceOption) bool {
    handlerCapability := f.handler.Capabilities()
    return f.e.ID() == 0
        || handlerCapability.StaticFeatures.Enabled(int(driver.HandlerCapabilityProxyRequired))
        || (f.policy.Settings.InternalProxy || f.e.Encrypted()) && !f.o.NoInternalProxy
}
```

四种情况走内部代理：
1. Entity ID 为 0（空实体）
2. 驱动声明了 `HandlerCapabilityProxyRequired`
3. 策略配置了 `InternalProxy = true`
4. Entity 被加密

### 3.6 删除链路

```
HTTP Request
  → controllers/file.go#Delete
    → service/explorer/file.go#DeleteFileService.Delete
      → manager.Delete(ctx, uris, ...)
        → manager.operation.go#L160-L197
          ├── fs.Delete(ctx, path, ...)              // DBFS 软删除/硬删除
          └── newExplicitEntityRecycleTask(...)       // 创建异步回收任务
              → manager.recycle.go#ExplicitEntityRecycleTask.Do
                → manager.RecycleEntities(ctx, false, entityIDs...)
                  → manager.recycle.go#L191-L289
                    ├── fs.StaleEntities(ctx, entityIDs...)  // 获取过期 Entity
                    ├── lo.GroupBy(entities, PolicyID)       // 按 PolicyID 分组
                    └── for each group:
                        ├── getEntityPolicyDriver(ctx, chunk[0], nil)  // 取驱动
                        └── d.Delete(ctx, toBeDeletedSrc...)           // 批量删除物理文件
```

**关键点**：回收时按 PolicyID 分组批量删除，同一策略的文件一起处理。

### 3.7 WebDAV DELETE / MKCOL / COPY / MOVE / LOCK 等操作的复用

WebDAV 的其余写操作全部通过 FileManager 间接到达 DBFS，**没有自己的存储逻辑**：

| WebDAV 方法 | FileManager 调用 | 存储层影响 |
|------------|------------------|-----------|
| `DELETE` | `fm.Delete(ctx, []*fs.URI{uri})` | 同 Web 端删除，触发异步实体回收 |
| `MKCOL` | `fm.Create(ctx, uri, FileTypeFolder, ...)` | DBFS 创建文件夹记录（无物理存储操作） |
| `COPY` | `fm.MoveOrCopy(ctx, src, dst, true)` | 同 Web 端复制，Entity 级深拷贝 |
| `MOVE` | `fm.MoveOrCopy(ctx, src, dst, false)` + `fm.Rename` | 同 Web 端移动，DB 记录重定位 |
| `LOCK` / `UNLOCK` | `fm.Lock` / `fm.Unlock` / `fm.ConfirmLock` | WebDAV 协议级锁（DBFS lock 表） |
| `PROPFIND` | `fm.Walk` + `allprop`/`props` | 纯 DB 查询，不触碰物理存储 |
| `PROPPATCH` | `patch` → 元数据写入 | DBFS metadata 表 |

### 3.8 GetStorageDriver — 核心策略分发函数

[manager/fs.go#L65-L89](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/fs.go#L65-L89)

```go
func (m *manager) GetStorageDriver(ctx context.Context, policy *ent.StoragePolicy) (driver.Handler, error) {
    switch policy.Type {
    case types.PolicyTypeLocal:   return local.New(policy, m.l, m.config), nil
    case types.PolicyTypeRemote:  return remote.New(ctx, policy, m.settings, m.config, m.l)
    case types.PolicyTypeOss:     return oss.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeCos:     return cos.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeS3:      return s3.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeKs3:     return ks3.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeObs:     return obs.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeQiniu:   return qiniu.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeUpyun:   return upyun.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeOd:      return onedrive.New(ctx, policy, m.settings, m.config, m.l, m.dep.CredManager())
    default:                      return nil, ErrUnknownPolicyType
    }
}
```

所有驱动均实现 [driver.Handler](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/handler.go#L44-L87) 接口，提供统一的 `Put`/`Delete`/`Source`/`Token`/`List`/`Capabilities` 等方法。

---

## 4. 驱动 Handler 接口与能力声明

[driver/handler.go#L44-L87](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/handler.go#L44-L87)

每个驱动通过 `Capabilities()` 声明自身能力：

| 能力常量 | 含义 | 影响路由行为 |
|---------|------|-------------|
| `HandlerCapabilityProxyRequired` | 必须通过 Cloudreve 代理获取内容 | 下载时走内部代理 URL |
| `HandlerCapabilityInboundGet` | 支持直接获取文件 `*os.File` | `IsLocal()` 返回 true，Serve 时直接读文件 |
| `HandlerCapabilityUploadSentinelRequired` | 不支持合规回调，需哨兵监控 | 上传后创建 Sentinel 定时任务兜底清理 |

`Capabilities` 结构体还包含：
- `MaxSourceExpire` / `MinSourceExpire` — 源 URL 有效期约束
- `ThumbSupportedExts` / `ThumbProxy` — 缩略图能力
- `MediaMetaSupportedExts` / `MediaMetaProxy` — 媒体元数据能力
- `BrowserRelayedDownload` — 是否需要浏览器中继下载

---

## 5. 回退兜底机制（按分层整理）

回退兜底机制分布在三层中，各司其职，层层递进。

### 5.1 协议入口层兜底（WebDAV 特有）

WebDAV 层在协议层面有自己的错误处理和资源释放机制，与 FileManager 层的机制独立但协同。

#### 5.1.1 锁的获取与释放 — confirmLock

[webdav.go#L97-L189](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/webdav/webdav.go#L97-L189) — `confirmLock`

WebDAV 所有写操作（PUT/DELETE/MKCOL/COPY/MOVE/LOCK/PROPPATCH）在进入 FileManager 之前，先通过 `confirmLock` 获取/确认锁，并通过 `defer release()` 保证在函数返回时自动释放。该函数同时支持 src 和 dst 两个资源的锁（用于 COPY/MOVE）。

**分支一：请求不带 `If` 头（客户端未预先创建锁）**

此时系统创建**临时锁**，用于防止与其他客户端的锁冲突，请求结束时自动释放：

```go
if hdr == "" {
    srcToken, dstToken := "", ""
    ap := fs.LockApp(fs.ApplicationDAV)
    if src != nil {
        ls, err = fm.Lock(ctx, -1, user, true, ap, src, "")
        srcToken = ls.LastToken()
        ctx = fs.LockSessionToContext(ctx, ls)  // 锁会话注入上下文，后续 FileManager 操作可感知
    }
    if dst != nil {
        ls, err = fm.Lock(ctx, -1, user, true, ap, dst, "")
        dstToken = ls.LastToken()
        ctx = fs.LockSessionToContext(ctx, ls)
    }
    return func() {  // defer release()
        if dstToken != "" { _ = fm.Unlock(ctx, dstToken) }
        if srcToken != "" { _ = fm.Unlock(ctx, srcToken) }
    }, ls, 0, nil
}
```

关键细节：
- **会话在 src 与 dst 间累积**：`ctx` 在锁定 src 后通过 `fs.LockSessionToContext` 注入，第二个 `fm.Lock(ctx, ...)` 复用同一个 session 对象，因此返回的 `ls` 同时包含 src 和 dst 两个 token
- **`release()` 真正删除临时锁**：闭包调用 `fm.Unlock(ctx, token)` → `f.ls.Unlock` → 从锁存储中移除节点，临时锁在请求结束后彻底消失
- src 锁失败时直接返回；dst 锁失败时会先 `fm.Unlock` 释放已获取的 src 锁
- 返回的 `release()` 按 dst → src 逆序释放

**分支二：请求带 `If` 头（客户端已预先创建锁，需验证复用）**

`If` 头按 WebDAV 规范解析为 ifLists 的**析取（OR 语义）**——只要任意一个 ifList 验证通过即可：

```go
ih, ok := parseIfHeader(hdr)
for _, l := range ih.lists {
    if src != nil {
        releaseSrc, ls, err = fm.ConfirmLock(c, srcAnc, src, tokens...)  // 传入该 ifList 的所有 token
        if errors.Is(err, lock.ErrConfirmationFailed) {
            continue  // 当前 ifList 失败，尝试下一个
        }
    }
    if dst != nil {
        releaseDst, ls, err = fm.ConfirmLock(c, dstAnc, dst, tokens...)
        if errors.Is(err, lock.ErrConfirmationFailed) {
            continue
        }
    }
    return func() { releaseDst(); releaseSrc() }, ls, 0, nil  // 第一个成功的 ifList 生效
}
return nil, nil, http.StatusPreconditionFailed, ErrLocked  // 所有 ifList 均失败 → 412
```

关键细节：
- `fm.ConfirmLock` 一次性接收一个 ifList 内的所有 token，**不是逐一确认**，而是整体校验该 ifList 是否匹配
- `ErrConfirmationFailed` 时 `continue` 尝试下一个 ifList（OR 语义）
- 全部 ifList 失败时返回 **412 Precondition Failed**（遵循 RFC 4918 §10.4.1）
- **`release()` 释放 hold 而非删除锁**：`ConfirmLock` 内部调用 `f.ls.Confirm` → `memLS.Confirm` → `m.hold(n)` 将锁节点标记为 held（暂停过期回收），返回的 `release()` 调用 `m.unhold(n)` 恢复 held 标记并重新挂回过期堆。客户端持有的锁**不会被删除**，但 `release()` 本身**不是空操作**——它恢复了锁的正常过期流程（[memlock.go#L99-L116](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/lock/memlock.go#L99-L116)、[memlock.go#L347-L365](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/lock/memlock.go#L347-L365)）
- **会话不跨 src/dst 累积**：与分支一不同，分支二对 src 和 dst 均传 `c`（未注入会话的原始 context），两次 `ConfirmLock` 各自创建独立的 session；返回的 `ls` 是最后一次（dst）的 session，src 的 session 仅被 `releaseSrc` 闭包捕获用于释放 hold

#### 5.1.1a 三层锁复用链路 — 临时锁、ConfirmLock 与 DBFS 跳过重复加锁

上面两个分支描述的是 `confirmLock` 自身的锁获取逻辑。但锁会话真正的复用发生在 **confirmLock 返回之后**——调用方将 `ls` 重新注入 context，使 FileManager 内部的 DBFS 操作能跳过重复加锁。三层关系如下：

**第一层：confirmLock 创建/确认锁，产出 `ls`**

- 分支一（临时锁）：`fm.Lock` → `acquireByPath` → `f.ls.Create` 在锁存储中**新建**锁节点，token 记入 `session.Tokens[lKey]`
- 分支二（ConfirmLock 复用）：`fm.ConfirmLock` → `f.ls.Confirm` 在锁存储中**查找并 hold** 已有锁节点，token 记入 `session.Tokens[lKey]`

两条路径都把“该 URI 已被当前会话锁定”这一事实写入 `session.Tokens` 映射表。

**第二层：调用方重新注入 `ls`，打通 FileManager 感知**

`confirmLock` 返回后，**调用方**（而非 confirmLock 自身）负责将会话注入请求级 context：

```go
// handlePut, handleMkcol, handleMove 等
release, ls, status, err := confirmLock(c, fm, user, ancestor, nil, uri, nil)
defer release()
ctx := fs.LockSessionToContext(c, ls)   // ← 关键：将 ls 注入新 ctx
// ...
res, err := m.Update(ctx, fileData)      // FileManager 操作使用带会话的 ctx
```

此后所有 FileManager 操作（`Update`、`Create`、`MoveOrCopy`、`Rename` 等）接收的 `ctx` 均携带 `ls`。

**第三层：DBFS 内部跳过重复加锁**

当 FileManager 操作内部再次调用 `Lock` 或 `ConfirmLock` 锁定**同一 URI** 时，DBFS 层先检查 session：

```go
// dbfs/lock.go ConfirmLock (L46-L48)
if _, ok := session.Tokens[lKey]; ok {
    return func() {}, session, nil   // 跳过，返回空 release
}

// dbfs/lock.go acquireByPath (L131-L133)
if _, ok := session.Tokens[lKey]; ok {
    continue   // 跳过，不加入 lockDetails
}
```

- `ConfirmLock` 命中已锁 key → 返回 `func() {}` 空操作 release，**不调用** `f.ls.Confirm`
- `acquireByPath`（Lock 底层）命中已锁 key → `continue` 跳过，**不调用** `f.ls.Create`

这就是 FileManager 在 WebDAV 写操作链路中不会对同一资源重复加锁的根本原因——不是靠“避免死锁”的模糊描述，而是靠 `session.Tokens[lKey]` 的精确查表跳过。

**TokenStack 嵌套作用域**

`LockSessionFromCtx`（[dbfs/lock.go#L267-L280](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/fs/dbfs/lock.go#L267-L280)）每次调用都会向 `TokenStack` 压入一个新空帧：

```go
l.TokenStack = append(l.TokenStack, make([]string, 0))
```

每次 `Lock`/`ConfirmLock` 将 `lKey` 追加到**当前栈顶帧**；`DBFS.Release` 只弹出栈顶帧并解锁该帧内的 token。这使得 FileManager 嵌套操作（如 MoveOrCopy 内部再调 Rename）各自拥有独立的锁作用域，内层 Release 不会误释放外层锁。

#### 5.1.2 LOCK 失败的自动回滚

[webdav.go#L454-L467](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/webdav/webdav.go#L454-L467)

`handleLock` 中若 LOCK 操作创建了文件占位符但后续操作失败，defer 块自动释放锁：

```go
defer func() {
    if retErr != nil {
        _ = fm.Unlock(c, token)
    }
}()
```

#### 5.1.3 错误码到 HTTP 状态码的映射

[webdav.go#L729-L760](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/webdav/webdav.go#L729-L760) — `purposeStatusCodeFromError`

所有 WebDAV 方法的错误在返回给客户端前，统一通过此函数转换为符合 WebDAV 规范的 HTTP 状态码：

| 错误类型 | HTTP 状态码 |
|---------|------------|
| `ent.IsNotFound` / `CodeNotFound` / `CodeParentNotExist` / `CodeEntityNotExist` | 404 Not Found |
| `lock.ErrNoSuchLock` | 409 Conflict |
| `CodeNoPermissionErr` | 403 Forbidden |
| `CodeLockConflict` | 423 Locked |
| `CodeObjectExist` | 405 Method Not Allowed |
| 其他 | 500 Internal Server Error |

同时支持展开 `AggregateError`，递归地取第一个子错误的映射结果。

#### 5.1.4 上传失败的级联兜底

WebDAV PUT 调用 `manager.Update`，而 `Update` 内部在任何一步失败时都会调用 `OnUploadFailed`。

[manager/upload.go#L349-L364](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/upload.go#L349-L364)

因此 WebDAV PUT 无论在 PrepareUpload / Upload / CompleteUpload 哪一步失败，都会依次触发四层清理：
1. **WebDAV 层**：`defer release()` 释放协议级锁
2. **FileManager 层**：`OnUploadFailed` 释放内部文件锁
3. **FileManager 层**：删除新建的占位文件 / 回滚版本控制
4. **WebDAV 层**：`purposeStatusCodeFromError` 将错误转成合适的 HTTP 状态码返回

> **分层边界提示**：第 1、4 步在 WebDAV 层（pkg/webdav/），第 2、3 步在 FileManager 层（pkg/filemanager/manager/），两层通过 `m.Update()` 的返回值传递错误，但各自独立管理自己的资源。

### 5.2 FileManager 层兜底

FileManager 层的兜底不区分入口（Web API 还是 WebDAV），对所有入口一视同仁。

#### 5.2.1 策略未找到时的兜底

[manager/fs.go#L92-L117](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/fs.go#L98-L100)

```go
if policyID == 0 {
    policy = &ent.StoragePolicy{Type: types.PolicyTypeLocal, Settings: &types.PolicySetting{}}
}
```

Entity 没有绑定策略时，默认回退到空本地策略，避免因策略缺失导致整个操作失败。

#### 5.2.2 上传失败时的清理 — OnUploadFailed

[manager/upload.go#L369-L397](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/upload.go#L369-L397)

上传失败后分两种模式处理：

**Master 模式**：
1. 释放文件锁 (`Unlock`)
2. 若新建了占位文件 → 删除占位文件
3. 若为更新已有文件 → 回滚版本控制（删除新版本 Entity）

**Slave 模式**：
1. 获取驱动并删除已上传的物理文件
2. 日志记录失败

#### 5.2.3 上传哨兵 — UploadSentinelCheckTask

[manager/upload.go#L447-L499](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/upload.go#L447-L499)

对于不支持合规回调的存储策略（声明了 `HandlerCapabilityUploadSentinelRequired`，如 S3），系统在上传会话创建后同时创建一个延迟执行的哨兵任务：

1. 任务在 `上传会话过期时间 + 5分钟` 后执行
2. 执行时检查上传会话是否已通过回调完成
3. 若未完成 → 删除占位 Entity 的物理文件 + 取消上传 Token
4. 若已完成 → 任务自动标记为 completed

这是对"客户端上传中断、没有回调通知"场景的兜底。

#### 5.2.4 实体回收 — RecycleEntities

[manager/recycle.go#L191-L289](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/recycle.go#L191-L289)

Entity 回收是异步批量执行的，包含以下容错：

1. **按策略分组**：同一策略的 Entity 一起删除，避免频繁切换驱动
2. **批量分片**：每 100 个 Entity 一批，避免单次请求过大
3. **部分失败隔离**：使用 `AggregateError` 收集每个 Entity 的独立错误，单个删除失败不影响其他
4. **force 模式**：强制模式下即使物理文件删除失败，仍从数据库中移除 Entity 记录
5. **unlink-only 处理**：标记为 `UnlinkOnly` 的 Entity 只解除 DB 记录关联，不删除物理文件

#### 5.2.5 下载 URL 缓存与重试

[manager/entity.go#L260-L303](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/entity.go#L260-L303) — 下载 URL 有 KV 缓存，缓存有效期比 URL 实际过期时间短一个 `EntityUrlCacheMargin`。

[entitysource/entitysource.go#L710-L726](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/entitysource/entitysource.go#L710-L726) — 非本地文件读取时也有 URL 缓存，过期前 1 分钟刷新。

#### 5.2.6 自定义代理 — ApplyProxyIfNeeded

[driver/util.go#L12-L42](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/util.go#L12-L42)

当策略配置了 `CustomProxy = true` 时，下载 URL 会被改写为代理服务器地址，保留原始路径和查询参数。这是一个灵活的中间层代理方案。

### 5.3 存储驱动层兜底

各驱动在自身实现层面有独立的容错机制。

#### 5.3.1 Local 驱动

- **文件不存在检查**：Delete 时用 `util.Exists` 检查，不存在则跳过，不报错
- **目录自动创建**：Put 时 `prepareFileDirectory` 自动创建缺失的目录层级
- **预分配失败降级**：`Fallocate` 失败时继续正常写入，仅记录日志

#### 5.3.2 S3 驱动

- **不存在对象静默忽略**：Delete 时 `ErrCodeNoSuchKey` 错误被静默忽略，继续处理其他对象
- **批量删除自动分批**：超过 1000 个 key 时自动分多批调用 `DeleteObjects`
- **分片上传重试**：`s3manager.Uploader` 内置分片并发与失败重试
- **取消上传主动清理**：`CancelToken` 通过 `AbortMultipartUploadWithContext` 主动中止未完成的分片上传（[s3/s3.go#L468-L475](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/s3/s3.go#L468-L475)）；Put 流程内部失败时 `cancelUpload` 辅助函数亦会调用 `AbortMultipartUpload` 清理残留分片（[s3/s3.go#L477-L485](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/s3/s3.go#L477-L485)）
- **分片完成校验**：`CompleteUpload` 通过 HEAD Object（`Meta`）校验已上传文件大小是否与 `session.Props.Size` 一致，不匹配则返回 `CodeMetaMismatch` 错误（[s3/s3.go#L504-L523](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/s3/s3.go#L504-L523)）；仅当 `SentinelTaskID != 0`（哨兵模式）时才执行校验，否则直接返回 nil

#### 5.3.3 Remote 驱动（Master ↔ Slave 间）

- **固定间隔分片重试**：Master 侧 `remoteClient.Upload` 使用 `backoff.ConstantBackoff`（[backoff.go#L20-L46](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/chunk/backoff/backoff.go#L20-L46)），**固定休眠 5 秒**（`chunkRetrySleep = 5 * time.Second`，[client.go#L32](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/remote/client.go#L32)），最多重试 `ChunkRetryLimit` 次。注意是**固定间隔**而非指数退避；若错误是 `RetryableError` 且携带 HTTP `retry-after` 头，则改用 `retry-after` 指定的时间。
- **每分片独立重试**：重试发生在 `chunk.ChunkGroup.Process` 内部（[chunk.go#L73-L131](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/chunk/chunk.go#L73-L131)），切换到下一分片时调用 `backoff.Reset()` 重置计数器（[chunk.go#L159-L163](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/chunk/chunk.go#L159-L163)）。重试前提：错误非 `context.Canceled` 且（文件可 Seek 或临时缓冲可用）。
- **重试缓冲**：若启用 `UseChunkBuffer` 且文件不可 Seek，`omitErrorTeeReader` 将分片内容 tee 到临时文件，失败后从临时文件重试，避免数据丢失。
- **整批失败清理**：任一分片重试耗尽仍失败 → 调用 `DeleteUploadSession` 清理 Slave 侧上传会话（[client.go#L121-L129](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/remote/client.go#L121-L129)），返回错误。
- **上传会话缓存失效兜底**：Slave 侧上传会话过期返回 `CodeUploadSessionExpired`，Master 侧感知后整体失败
- **删除失败列表回传**：Slave 侧删除失败的文件路径通过响应体回传给 Master，Master 侧 `AggregateError` 汇总
- **HMAC 签名防篡改**：所有 Master ↔ Slave 通信均带 HMAC 签名，防止请求被篡改

---

## 6. 三种核心驱动的 PUT/GET/DELETE 实现对比

以下从实现角度对比 local（本地盘）、s3（S3 兼容）、remote（远程从机 / Master-Slave HTTP RPC）三种驱动在核心方法上的差异。它们均实现同一 `driver.Handler` 接口。

### 6.1 Put 方法对比

| 维度 | local.Driver.Put | s3.Driver.Put | remote.Driver.Put |
|------|------------------|---------------|-------------------|
| 代码位置 | [local/local.go#L126-L171](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/local/local.go#L126-L171) | [s3/s3.go#L194-L227](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/s3/s3.go#L194-L227) | [remote/remote.go#L78-L82](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/remote/remote.go#L78-L82) |
| 写入方式 | `os.OpenFile` + `io.Copy` 顺序写 | AWS SDK `s3manager.Uploader.UploadWithContext` 分片并发上传 | `uploadClient.Upload(ctx, file)` HTTP POST 到从机 |
| 覆盖检查 | 检查本地文件是否已存在 | `handler.Meta` HEAD Object 检查存在性 | 无（由从机侧 local 驱动负责） |
| 目录准备 | `prepareFileDirectory` → `os.MkdirAll` | 不需要（S3 无目录概念） | 不需要 |
| 预分配 | Policy.PreAllocate 开启时 `Fallocate` | 不支持 | 不支持 |
| Offset 支持 | 支持，`out.Seek(file.Offset, io.SeekStart)` 分片断点续传 | 不支持，每次传完整 Body | 不支持 |
| 加密位置 | Cloudreve 层加密（manager.Upload 在调用 Put 前包装 cryptor） | 同左 | 同左 |

### 6.2 Delete 方法对比

| 维度 | local.Driver.Delete | s3.Driver.Delete | remote.Driver.Delete |
|------|---------------------|------------------|----------------------|
| 代码位置 | [local/local.go#L175-L195](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/local/local.go#L175-L195) | [s3/s3.go#L231-L287](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/s3/s3.go#L231-L287) | [remote/remote.go#L86-L92](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/remote/remote.go#L86-L92) |
| 批量策略 | 逐个删除，无批量 | 单文件 → `DeleteObject`；多文件 → `DeleteObjects`（默认批次 1000） | 一次性批量 RPC 到从机 |
| 不存在处理 | `util.Exists` 检查，不存在则跳过 | `ErrCodeNoSuchKey` 被静默忽略 | 从机侧处理 |
| 失败策略 | 收集失败路径和最后一个错误，继续删其他 | 收集所有失败 key，继续删其他批次 | RPC 返回失败列表 |
| 缩略图清理 | 注释遗留，未实现 | 不清理（S3 无缩略图） | 从机侧处理 |

### 6.3 Source / Token 方法对比（下载凭证 + 上传凭证）

| 维度 | local.Driver | s3.Driver | remote.Driver |
|------|--------------|-----------|---------------|
| Source 实现 | 返回 `"not implemented"` 错误 | [s3/s3.go#L296-L332](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/s3/s3.go#L296-L332) — `GetObjectRequest.Presign` 预签名 URL，7 天 TTL，公有桶去签名 | [remote/remote.go#L111-L136](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/remote/remote.go#L111-L136) — 组装从机 `/api/v4/slave/file/content/...` URL，HMAC 签名 |
| Token 实现 | [local/local.go#L208-L235](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/local/local.go#L208-L235) — 本地创建占位文件，可选预分配，返回会话 | [s3/s3.go#L335-L411](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/s3/s3.go#L335-L411) — `CreateMultipartUpload` + 每个分片 `UploadPartRequest.Presign` + `CompleteMultipartUploadRequest.Presign` | [remote/remote.go#L139-L159](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/remote/remote.go#L139-L159) — 调用从机 `CreateUploadSession`，返回签名后的从机上传 URL |
| Capabilities | `ProxyRequired=true`, `InboundGet=true`（强制内部代理，支持直接读 *os.File） | `UploadSentinelRequired=true`（不支持回调，需哨兵监控） | 无 StaticFeatures（走默认路径） |
| CompleteUpload | [local/local.go#L255-L295](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/local/local.go#L255-L295) — Slave 侧回调 Master（当作为 remote 策略的影子驱动时） | [s3/s3.go#L504-L523](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/s3/s3.go#L504-L523) — HEAD Object 校验文件大小，不匹配返回 `CodeMetaMismatch`（仅哨兵模式 `SentinelTaskID != 0`） | 空实现 |
| CancelToken | 空实现 | [s3/s3.go#L468-L475](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/s3/s3.go#L468-L475) — `AbortMultipartUploadWithContext` 主动取消 | `DeleteUploadSession` RPC 到从机 |

### 6.4 EntitySource.Serve — 下载响应决策

[entitysource.go#L270-L506](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/entitysource/entitysource.go#L270-L506)

无论是 WebDAV GET 还是 Web 端下载，最终都统一通过 `EntitySource.Serve` 生成 HTTP 响应。内部按驱动能力分支：

```
EntitySource.Serve
  │
  ├── IsLocal() == true  (local 驱动: HandlerCapabilityInboundGet)
  │    ├── resetRequest()  // handler.Open → *os.File，验证存在性
  │    ├── http.ServeContent(w, r, ...)  // Go 标准库处理 Range/ETag/Content-Type
  │    └── 透明解密（如 Entity 已加密）
  │
  └── IsLocal() == false (s3/remote/cos/oss/... 驱动)
       ├── handler.Source(...) 获取预签名 URL
       ├── httputil.ReverseProxy 反向代理到后端
       │    ├── Director: 重写 Scheme/Host/Path，删除 Authorization
       │    ├── ModifyResponse: 清除后端 ETag，解密 Body
       │    └── ErrorHandler: 记录日志，502 Bad Gateway
       └── 支持 Range 请求透传
```

---

## 7. 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          HTTP Layer (Gin Router)                         │
│  ┌──────────────────────────────┐    ┌──────────────────────────────┐    │
│  │  Web API (/api/v4)            │    │  WebDAV (/dav)                │    │
│  │  controllers/file.go          │    │  webdav.ServeHTTP             │    │
│  │  → service/explorer/*.go      │    │  handlePut/GetHeadPost/...    │    │
│  │  (Session Auth + Scope)       │    │  (Basic Auth + DAV Permission)│    │
│  └───────────────┬───────────────┘    └──────────────┬───────────────┘    │
│                  │                                   │                    │
│                  └───────────────┬───────────────────┘                    │
│                                  ▼                                        │
│                       manager.NewFileManager(dep, user)                    │
└──────────────────────────────────┬────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                      FileManager (manager)                                │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  ┌──────────┐  │
│  │ FileOperation│  │UploadMgmt    │  │EntityManagement   │  │  Lock    │  │
│  │ Get/List/    │  │CreateSession │  │GetEntityUrls      │  │ConfirmLock│ │
│  │ Create/      │  │Upload/       │  │GetEntitySource    │  │Lock/     │  │
│  │ Rename/      │  │Complete/     │  │Thumbnail          │  │Unlock    │  │
│  │ Delete       │  │Cancel        │  │RecycleEntities    │  │          │  │
│  │ Walk         │  │Update        │  │                   │  │          │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬──────────┘  └────┬─────┘  │
│         │                │                    │                   │        │
│         │    getPreferredPolicy()    getEntityPolicyDriver()      │        │
│         │    (Group → Policy)        (Entity.PolicyID →           │        │
│         │                            Policy → Driver)             │        │
│         │                │                    │                   │        │
│         └────────────────┼────────────────────┴───────────────────┘        │
│                          ▼                                                  │
│              GetStorageDriver(policy)                                        │
│              ┌── CastStoragePolicyOnSlave() ──┐                             │
│              │  (Slave模式策略类型转换)         │                             │
│              └────────────────────────────────┘                             │
│                          │                                                  │
│           ┌──────────────┼──────────────┐                                   │
│           ▼              ▼              ▼                                   │
│     ┌──────────┐  ┌───────────┐  ┌──────────┐                              │
│     │  DBFS    │  │  Driver   │  │  Entity  │                              │
│     │(fs/dbfs) │  │ (Handler) │  │  Source  │                              │
│     └──────────┘  └─────┬─────┘  └──────────┘                              │
└─────────────────────────┼───────────────────────────────────────────────────┘
                          │
            ┌─────────────┼─────────────────┐
            ▼             ▼                 ▼
     ┌────────────┐ ┌──────────┐    ┌────────────┐
     │   local    │ │   s3     │    │  remote    │
     │ (本地盘)    │ │  cos/oss │    │ (Slave RPC)│
     │  obs/ks3   │ │  qiniu   │    │            │
     │  upyun     │ │ onedrive │    │            │
     └────────────┘ └──────────┘    └────────────┘
```

---

## 8. 关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 存储策略类型常量 | inventory/types/types.go | L301-L311 |
| 存储策略 Schema | ent/schema/policy.go | L15-L39 |
| Group→Policy 查询 | inventory/policy.go | L145-L155 |
| Policy 缓存查询 | inventory/policy.go | L158-L181 |
| Handler 接口定义 | pkg/filemanager/driver/handler.go | L44-L87 |
| Handler 能力常量 | pkg/filemanager/driver/handler.go | L14-L24 |
| GetStorageDriver (策略→驱动分发) | pkg/filemanager/manager/fs.go | L65-L89 |
| getEntityPolicyDriver (Entity→驱动) | pkg/filemanager/manager/fs.go | L92-L117 |
| CastStoragePolicyOnSlave (从机策略转换) | pkg/filemanager/manager/fs.go | L31-L63 |
| getPreferredPolicy (上传策略匹配) | pkg/filemanager/fs/dbfs/dbfs.go | L668-L682 |
| PrepareUpload (上传准备+策略绑定) | pkg/filemanager/fs/dbfs/upload.go | L72-L260 |
| CreateUploadSession (上传会话创建) | pkg/filemanager/manager/upload.go | L50-L147 |
| Upload (执行上传) | pkg/filemanager/manager/upload.go | L188-L221 |
| Update (WebDAV PUT 入口) | pkg/filemanager/manager/upload.go | L326-L367 |
| OnUploadFailed (上传失败清理) | pkg/filemanager/manager/upload.go | L369-L397 |
| UploadSentinelCheckTask (上传哨兵) | pkg/filemanager/manager/upload.go | L447-L548 |
| GetEntityUrls (批量获取下载URL) | pkg/filemanager/manager/entity.go | L205-L313 |
| GetEntitySource (下载 EntitySource 构造) | pkg/filemanager/manager/entity.go | L315-L346 |
| EntitySource.Url (URL生成/代理决策) | pkg/filemanager/manager/entitysource/entitysource.go | L587-L668 |
| ShouldInternalProxy (内部代理决策) | pkg/filemanager/manager/entitysource/entitysource.go | L578-L585 |
| EntitySource.Serve (HTTP响应/反代) | pkg/filemanager/manager/entitysource/entitysource.go | L270-L506 |
| RecycleEntities (实体回收) | pkg/filemanager/manager/recycle.go | L191-L289 |
| ApplyProxyIfNeeded (自定义代理) | pkg/filemanager/driver/util.go | L12-L42 |
| **WebDAV 入口** | | |
| initWebDAV (WebDAV 路由注册) | routers/router.go | L1336-L1350 |
| WebDAVAuth (WebDAV 鉴权中间件) | middleware/auth.go | L104-L174 |
| webdav.ServeHTTP (WebDAV 方法分发) | pkg/webdav/webdav.go | L55-L95 |
| handlePut (WebDAV PUT 处理) | pkg/webdav/webdav.go | L227-L285 |
| handleGetHeadPost (WebDAV GET/HEAD 处理) | pkg/webdav/webdav.go | L334-L372 |
| handleDelete (WebDAV DELETE 处理) | pkg/webdav/webdav.go | L563-L588 |
| handleCopyMove (WebDAV COPY/MOVE 处理) | pkg/webdav/webdav.go | L590-L689 |
| handleLock/handleUnlock (WebDAV 锁) | pkg/webdav/webdav.go | L374-L493 |
| handlePropfind (WebDAV 列目录) | pkg/webdav/webdav.go | L495-L561 |
| confirmLock (锁确认+自动释放) | pkg/webdav/webdav.go | L97-L189 |
| purposeStatusCodeFromError (错误码映射) | pkg/webdav/webdav.go | L729-L760 |
| **驱动实现** | | |
| local.Driver.Put (本地盘写入) | pkg/filemanager/driver/local/local.go | L126-L171 |
| local.Driver.Delete (本地盘删除) | pkg/filemanager/driver/local/local.go | L175-L195 |
| local.Driver.Token (本地盘上传凭证) | pkg/filemanager/driver/local/local.go | L208-L235 |
| local.Driver.CompleteUpload (从机回调) | pkg/filemanager/driver/local/local.go | L255-L295 |
| s3.Driver.Put (S3 写入) | pkg/filemanager/driver/s3/s3.go | L194-L227 |
| s3.Driver.Delete (S3 删除) | pkg/filemanager/driver/s3/s3.go | L231-L287 |
| s3.Driver.Source (S3 预签名下载 URL) | pkg/filemanager/driver/s3/s3.go | L296-L332 |
| s3.Driver.Token (S3 分片上传凭证) | pkg/filemanager/driver/s3/s3.go | L335-L411 |
| remote.Driver.Put (远程从机上传) | pkg/filemanager/driver/remote/remote.go | L78-L82 |
| remote.Driver.Delete (远程从机删除) | pkg/filemanager/driver/remote/remote.go | L86-L92 |
| remote.Driver.Source (远程从机下载 URL) | pkg/filemanager/driver/remote/remote.go | L111-L136 |
| remote.Driver.Token (远程从机上传凭证) | pkg/filemanager/driver/remote/remote.go | L139-L159 |
| remote.NewClient (remoteClient 构造) | pkg/filemanager/driver/remote/client.go | L58-L85 |
| remoteClient.Upload (从机分片上传) | pkg/filemanager/driver/remote/client.go | L95-L139 |
| **Slave 从机侧** | | |
| SlaveUpload (从机上传 controller) | routers/controllers/slave.go | L20-L31 |
| SlaveGetUploadSession (从机创建上传会话) | routers/controllers/slave.go | L33-L43 |
| SlaveServeEntity (从机下载文件) | routers/controllers/slave.go | L58-L66 |
| SlaveDelete (从机删除文件) | routers/controllers/slave.go | L93-L101 |
| SlaveCreateUploadSessionService.Create (从机上传会话服务) | service/explorer/slave.go | L88-L107 |
| EntityDownloadService.SlaveServe (从机下载服务) | service/explorer/slave.go | L41-L76 |
| UploadService.SlaveUpload (从机分片上传服务) | service/explorer/upload.go | L135-L154 |
| local.NewLocalFileEntity (构造本地文件实体) | pkg/filemanager/driver/local/entity.go | L15-L26 |
| newStatelessFileManager (无状态 FileManager) | pkg/filemanager/manager/manager.go | L173-L184 |
| NewFileManager (有/无状态分发) | pkg/filemanager/manager/manager.go | L152-L170 |
