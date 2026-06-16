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

### 3.2 下载/获取文件 URL 链路

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

### 3.3 删除链路

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

### 3.4 GetStorageDriver — 核心策略分发函数

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

---

## 6. 整体架构图

```
┌──────────────────────────────────────────────────────────────┐
│                     HTTP Layer (Gin Router)                  │
│  controllers/file.go → service/explorer/*.go                 │
└────────────────────────────┬─────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────┐
│                   FileManager (manager)                       │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐   │
│  │ FileOperation│  │UploadMgmt    │  │EntityManagement   │   │
│  │ Get/List/    │  │CreateSession │  │GetEntityUrls      │   │
│  │ Create/      │  │Upload/       │  │GetEntitySource    │   │
│  │ Rename/      │  │Complete/     │  │Thumbnail          │   │
│  │ Delete       │  │Cancel        │  │RecycleEntities    │   │
│  └──────┬──────┘  └──────┬───────┘  └────────┬──────────┘   │
│         │                │                    │              │
│         │    getPreferredPolicy()    getEntityPolicyDriver() │
│         │    (Group → Policy)        (Entity.PolicyID →      │
│         │                            Policy → Driver)        │
│         │                │                    │              │
│         └────────────────┼────────────────────┘              │
│                          ▼                                   │
│              GetStorageDriver(policy)                         │
│              ┌── CastStoragePolicyOnSlave() ──┐               │
│              │  (Slave模式策略类型转换)         │               │
│              └────────────────────────────────┘               │
│                          │                                   │
│           ┌──────────────┼──────────────┐                    │
│           ▼              ▼              ▼                    │
│     ┌──────────┐  ┌───────────┐  ┌──────────┐               │
│     │  DBFS    │  │  Driver   │  │  Entity  │               │
│     │(fs/dbfs) │  │ (Handler) │  │  Source  │               │
│     └──────────┘  └─────┬─────┘  └──────────┘               │
└─────────────────────────┼────────────────────────────────────┘
                          │
            ┌─────────────┼─────────────────┐
            ▼             ▼                 ▼
     ┌────────────┐ ┌──────────┐    ┌────────────┐
     │   local    │ │   s3     │    │  remote    │
     │ (本地盘)    │ │  cos/oss │    │ (WebDAV等) │
     │  obs/ks3   │ │  qiniu   │    │ (HTTP RPC) │
     │  upyun     │ │ onedrive │    │            │
     └────────────┘ └──────────┘    └────────────┘
```

---

## 7. 关键代码索引

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
| OnUploadFailed (上传失败清理) | pkg/filemanager/manager/upload.go | L369-L397 |
| UploadSentinelCheckTask (上传哨兵) | pkg/filemanager/manager/upload.go | L447-L548 |
| GetEntityUrls (批量获取下载URL) | pkg/filemanager/manager/entity.go | L205-L313 |
| EntitySource.Url (URL生成/代理决策) | pkg/filemanager/manager/entitysource/entitysource.go | L587-L668 |
| ShouldInternalProxy (内部代理决策) | pkg/filemanager/manager/entitysource/entitysource.go | L578-L585 |
| EntitySource.Serve (HTTP响应/反代) | pkg/filemanager/manager/entitysource/entitysource.go | L270-L506 |
| RecycleEntities (实体回收) | pkg/filemanager/manager/recycle.go | L191-L289 |
| ApplyProxyIfNeeded (自定义代理) | pkg/filemanager/driver/util.go | L12-L42 |
