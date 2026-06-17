# Cloudreve 存储路由机制详解

本文档对照代码讲清：当一个请求（上传/下载/删除等）到达 Cloudreve 后，系统如何决定使用哪种存储后端（S3、本地盘、WebDAV/Remote 等），请求沿哪条路径转发到具体驱动，以及在出错时如何兜底。

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
PolicyTypeRemote = "remote"    // 远程从机（本质上走 HTTP RPC，可对接 WebDAV 等）
PolicyTypeObs    = "obs"       // 华为云 OBS
```

### 1.2 Group → StoragePolicy 的绑定关系

用户组（Group）与存储策略是一对多关系：每个用户组绑定一个默认存储策略。

- [inventory/policy.go#GetByGroup](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/inventory/policy.go#L145-L155) — 通过 Group 查询其关联的 StoragePolicy

### 1.3 Entity → StoragePolicy 的绑定关系

每个物理文件实体（Entity）通过 `storage_policy_entities` 外键字段绑定到创建它时的存储策略。

- [fs.go#DbEntity.PolicyID](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/fs/fs.go#L780-L782) — `e.model.StoragePolicyEntities` 即为 PolicyID

这意味着：同一文件的不同版本（Entity）可以存储在不同的后端策略上。

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
PUT /dav/path/to/file.ext
  → WebDAVAuth 中间件 (Basic Auth)
  → stripPrefix: 剥离 /dav 前缀，得到相对路径
  → fm.SharedAddressTranslation: 路径 → fs.File (ancestor) + fs.URI
  → DavAccountDisableSysFiles 检查: 以 "." 开头的文件是否被禁止
  → confirmLock: 确认/创建文件锁（WebDAV 锁机制）
  → request.SniffContentLength(c.Request): 从 HTTP 请求嗅探文件大小
  → 构造 fs.UploadRequest (Mode=ModeOverwrite)
  → m.Update(ctx, fileData)                ← 和 Web 端上传复用
       │
       ├── fs.PrepareUpload(ctx, req)      // DBFS 创建/更新 Entity（走 getPreferredPolicy 匹配策略）
       ├── m.Upload(ctx, req, policy, session)
       │    └── GetStorageDriver(...)       // 策略→驱动分发
       │         ├── local.Driver.Put      // 本地盘: os.OpenFile + io.Copy
       │         ├── s3.Driver.Put         // S3: s3manager.Uploader.UploadWithContext
       │         ├── remote.Driver.Put     // 远程从机: uploadClient.Upload (HTTP RPC)
       │         └── ... (cos/oss/obs/ks3/qiniu/upyun/onedrive)
       └── m.CompleteUpload(ctx, session)
```

**关键点**：WebDAV PUT 的核心上传执行 (`manager.Upload` → `driver.Handler.Put`) 与 Web 端 REST API 上传 **完全走同一条代码路径**，策略匹配、驱动分发、加密、失败清理机制全部复用。差异仅在于：
- WebDAV 使用 `ModeOverwrite` 覆盖已有文件
- WebDAV 不通过上传会话凭证（Token），直接将 HTTP Body 流交给驱动
- WebDAV 自带 WebDAV 协议级锁（`confirmLock`），与文件管理器内部锁并存

### 3.4 WebDAV GET → 文件管理层的完整路径

[webdav.go#L334-L372](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/webdav/webdav.go#L334-L372) — `handleGetHeadPost`

```
GET /dav/path/to/file.ext
  → WebDAVAuth 中间件
  → stripPrefix
  → fm.SharedAddressTranslation: 路径 → target (fs.File)
  → target.Type() 必须是 FileTypeFile
  → fm.GetEntitySource(c, target.PrimaryEntityID())  ← 和 Web 端下载复用
       │
       ├── fs.GetEntity(ctx, entityID)              // 从 DB 取 Entity
       ├── getEntityPolicyDriver(ctx, entity, nil)  // Entity.PolicyID → Policy → Driver
       └── entitysource.NewEntitySource(...)
  → es.Apply(WithSpeedLimit(user.Group.SpeedLimit))
  → 决策: ShouldInternalProxy() 或 (DavAccountProxy 且 GroupPermissionWebDAVProxy)
       │
       ├── true  → es.Serve(c.Writer, c.Request)     // 反代/流式响应
       │     ├── IsLocal()
       │     │    └── local.Driver.Open + io.Copy    // 本地盘：直接读文件
       │     └── !IsLocal()
       │          ├── handler.Source(...)             // S3/remote: 取预签名 URL
       │          └── httputil.ReverseProxy           // Cloudreve 反向代理到后端
       └── false → es.Url(...) + c.Redirect(302, src.Url)
                  └── s3.Driver.Source → Presign   // 302 跳转到 S3 预签名 URL
```

**WebDAV GET 与 Web 端 GET 的复用关系**：
- `GetEntitySource`、`getEntityPolicyDriver`、`NewEntitySource`、`EntitySource.Serve` 完全复用
- WebDAV 额外增加了一层代理决策：当 DAV 账号开启了 `DavAccountProxy` 且用户组有 `GroupPermissionWebDAVProxy` 权限时，强制走 Cloudreve 内部代理（某些 WebDAV 客户端不支持 302 跳转）

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

## 5. 回退兜底机制

### 5.1 策略未找到时的兜底

[manager/fs.go#L92-L117](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/fs.go#L98-L100)

```go
if policyID == 0 {
    policy = &ent.StoragePolicy{Type: types.PolicyTypeLocal, Settings: &types.PolicySetting{}}
}
```

Entity 没有绑定策略时，默认回退到空本地策略。

### 5.2 上传失败时的清理 — OnUploadFailed

[manager/upload.go#L369-L397](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/upload.go#L369-L397)

上传失败后分两种模式处理：

**Master 模式**：
1. 释放文件锁 (`Unlock`)
2. 若新建了占位文件 → 删除占位文件
3. 若为更新已有文件 → 回滚版本控制（删除新版本 Entity）

**Slave 模式**：
1. 获取驱动并删除已上传的物理文件
2. 日志记录失败

### 5.3 上传哨兵 — UploadSentinelCheckTask

[manager/upload.go#L447-L499](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/upload.go#L447-L499)

对于不支持合规回调的存储策略（设置了 `HandlerCapabilityUploadSentinelRequired`），系统在上传会话创建后同时创建一个延迟执行的哨兵任务：

1. 任务在 `上传会话过期时间 + 5分钟` 后执行
2. 执行时检查上传会话是否已通过回调完成
3. 若未完成 → 删除占位 Entity 的物理文件 + 取消上传 Token
4. 若已完成 → 任务自动标记为 completed

### 5.4 实体回收 — RecycleEntities

[manager/recycle.go#L191-L289](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/recycle.go#L191-L289)

Entity 回收是异步批量执行的，包含以下容错：

1. **按策略分组**：同一策略的 Entity 一起删除，避免频繁切换驱动
2. **批量分片**：每 100 个 Entity 一批，避免单次请求过大
3. **部分失败隔离**：使用 `AggregateError` 收集每个 Entity 的独立错误，单个删除失败不影响其他
4. **force 模式**：强制模式下即使物理文件删除失败，仍从数据库中移除 Entity 记录
5. ** unlink-only 处理**：标记为 `UnlinkOnly` 的 Entity 只解除 DB 记录关联，不删除物理文件

### 5.5 下载 URL 缓存与重试

[manager/entity.go#L260-L303](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/entity.go#L260-L303) — 下载 URL 有 KV 缓存，缓存有效期比 URL 实际过期时间短一个 `EntityUrlCacheMargin`。

[entitysource/entitysource.go#L710-L726](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/entitysource/entitysource.go#L710-L726) — 非本地文件读取时也有 URL 缓存，过期前 1 分钟刷新。

### 5.6 自定义代理 — ApplyProxyIfNeeded

[driver/util.go#L12-L42](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/util.go#L12-L42)

当策略配置了 `CustomProxy = true` 时，下载 URL 会被改写为代理服务器地址，保留原始路径和查询参数。这是一个灵活的中间层代理方案。

### 5.7 WebDAV 特有的兜底与错误处理

#### 5.7.1 锁的自动释放

[webdav.go#L97-L189](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/webdav/webdav.go#L97-L189) — `confirmLock`

WebDAV 所有写操作（PUT/DELETE/MKCOL/COPY/MOVE/LOCK/PROPPATCH）在执行前都会通过 `confirmLock` 获取锁，并通过 `defer release()` 保证在函数返回时自动释放：

```go
release, ls, status, err := confirmLock(c, fm, user, ancestor, nil, uri, nil)
if err != nil {
    return status, err
}
defer release()
```

- 若请求不带 `If` 头 → 创建临时锁（自动随请求结束释放）
- 若请求带 `If` 头 → 遍历所有条件 token 逐一确认，第一个匹配成功的生效
- 所有 token 均失败 → 返回 412 Precondition Failed
- 目标被他人锁定 → 返回 423 Locked

#### 5.7.2 LOCK 失败的自动回滚

[webdav.go#L454-L467](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/webdav/webdav.go#L454-L467)

`handleLock` 中若 LOCK 操作创建了文件占位符但后续操作失败，defer 块自动释放锁：

```go
defer func() {
    if retErr != nil {
        _ = fm.Unlock(c, token)
    }
}()
```

#### 5.7.3 错误码到 HTTP 状态码的映射

[webdav.go#L729-L760](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/webdav/webdav.go#L729-L760) — `purposeStatusCodeFromError`

所有 WebDAV 方法的错误统一通过此函数转换为符合 WebDAV 规范的 HTTP 状态码：

| 错误类型 | HTTP 状态码 |
|---------|------------|
| `ent.IsNotFound` / `CodeNotFound` / `CodeParentNotExist` / `CodeEntityNotExist` | 404 Not Found |
| `lock.ErrNoSuchLock` | 409 Conflict |
| `CodeNoPermissionErr` | 403 Forbidden |
| `CodeLockConflict` | 423 Locked |
| `CodeObjectExist` | 405 Method Not Allowed |
| 其他 | 500 Internal Server Error |

同时支持展开 `AggregateError`，递归地取第一个子错误的映射结果。

#### 5.7.4 WebDAV 上传失败的级联兜底

WebDAV PUT 调用 `manager.Update`，而 `Update` 内部在任何一步失败时都会调用 `OnUploadFailed`：

[manager/upload.go#L349-L364](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/manager/upload.go#L349-L364)

```go
if err := m.Upload(ctx, req, uploadSession.Policy, uploadSession); err != nil {
    m.OnUploadFailed(ctx, uploadSession)   // 锁释放 + 占位文件删除 + 版本回滚
    return nil, fmt.Errorf("failed to upload new entity: %w", err)
}
file, err := m.CompleteUpload(ctx, uploadSession)
if err != nil {
    m.OnUploadFailed(ctx, uploadSession)   // 完成阶段失败同样清理
    return nil, fmt.Errorf("failed to complete update: %w", err)
}
```

因此 WebDAV PUT 无论在 PrepareUpload / Upload / CompleteUpload 哪一步失败，都会触发：
1. 释放 WebDAV 层的锁（`defer release()` in handlePut）
2. 释放 FileManager 内部锁
3. 删除新建的占位文件 / 回滚版本控制（OnUploadFailed）
4. 由 purposeStatusCodeFromError 将错误转成合适的 HTTP 状态码返回给客户端

---

## 6. 三种核心驱动的 PUT/GET/DELETE 实现对比

以下从实现角度对比 local（本地盘）、s3（S3 兼容）、remote（远程从机 / WebDAV RPC）三种驱动在核心方法上的差异。它们均实现同一 `driver.Handler` 接口。

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
| CompleteUpload | [local/local.go#L255-L295](file:///d:/fz/0601-2/solo-dogfeeding/code/13-Cloudreve/pkg/filemanager/driver/local/local.go#L255-L295) — Slave 侧回调 Master（当作为 remote 策略的影子驱动时） | 空实现 | 空实现 |
| CancelToken | 空实现 | 未实现（S3 MultipartUpload 自动过期） | `DeleteUploadSession` RPC 到从机 |

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
