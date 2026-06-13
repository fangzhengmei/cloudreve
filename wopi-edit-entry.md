# Cloudreve WOPI 在线编辑代码分析

本文档深入分析 Cloudreve 项目中 WOPI (Web Application Open Platform Interface) 在线编辑功能的实现，重点关注协议鉴权、文件锁、回写路径和编辑冲突处理四个核心模块。

## 目录

- [1. 整体架构概述](#1-整体架构概述)
- [2. 协议鉴权机制](#2-协议鉴权机制)
- [3. 文件锁实现](#3-文件锁实现)
- [4. 文件回写路径](#4-文件回写路径)
- [5. 编辑冲突处理](#5-编辑冲突处理)
- [6. 关键代码文件索引](#6-关键代码文件索引)

---

## 1. 整体架构概述

Cloudreve 的 WOPI 实现在分层架构中跨越多个模块：

```
┌─────────────────────────────────────────────────┐
│              WOPI 客户端 (Office Online)        │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│              路由层 (routers/router.go)         │
│  - /file/wopi/:id          (CheckFileInfo)      │
│  - /file/wopi/:id/contents (GetFile/PutFile)    │
│  - /file/wopi/:id          (ModifyFile)         │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│            中间件层 (middleware/wopi.go)        │
│  - ViewerSessionValidation (访问令牌验证)       │
│  - WopiWriteAccess        (未使用!)             │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│           控制器层 (controllers/wopi.go)        │
│  - CheckFileInfo   - GetFile                    │
│  - PutFile         - ModifyFile                 │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│          服务层 (service/explorer/viewer.go)    │
│  - WopiService.FileInfo()                       │
│  - WopiService.Lock()/Unlock()/RefreshLock()    │
│  - WopiService.PutContent()                     │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│       领域层 (pkg/filemanager/*)                │
│  - lock/memlock.go     (内存锁实现)             │
│  - fs/dbfs/lock.go     (文件系统锁封装)         │
│  - fs/dbfs/dbfs.go     (实体创建+版本校验)      │
│  - manager/viewer.go   (会话管理)               │
└─────────────────────────────────────────────────┘
```

---

## 2. 协议鉴权机制

### 2.1 会话创建流程

**核心代码**：[manager/viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/manager/viewer.go#L46-L84)

当用户请求在线编辑时，系统首先创建一个 Viewer 会话：

```go
func (m *manager) CreateViewerSession(ctx context.Context, uri *fs.URI, version string, viewer *types.Viewer) (*ViewerSession, error)
```

**鉴权令牌结构**：
- 格式：`sessionID.randomToken`（由 `.` 分隔的两部分）
- `sessionID`：UUID v4 生成的会话唯一标识
- `randomToken`：128 位加密安全随机字符串

**会话缓存** (`ViewerSessionCache`)：
- 缓存键：`viewer_session_{sessionID}`
- 存储内容：ID、Uri、UserID、FileID、ViewerID、Version、Token
- **不包含** `Action` 字段（区分编辑/预览的动作类型）
- 过期时间：由配置 `ViewerSessionTTL` 控制（默认 3600 秒）

### 2.2 请求验证中间件

**核心代码**：[middleware/wopi.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/middleware/wopi.go#L30-L94)

`ViewerSessionValidation` 中间件对每个 WOPI 请求进行验证：

```
1. 解析 access_token
   ├─ 从 URL 查询参数获取 access_token
   ├─ 按 "." 分割为 sessionID 和 token 两部分
   └─ 格式验证：必须为两部分，否则返回 403

2. 会话有效性验证
   ├─ 从 KV 存储按 sessionID 查找会话缓存
   ├─ 不存在则返回 403
   └─ 提取会话中的 UserID、FileID、ViewerID

3. 用户上下文重建
   ├─ 根据 UserID 从数据库加载用户信息
   └─ 将用户信息注入请求上下文

4. 文件归属验证
   ├─ 从 URL 路径解析 fileID（HashID 解码）
   ├─ 与会话中的 FileID 对比
   └─ 不匹配则返回 403

5. 查看器可用性验证
   ├─ 检查 ViewerID 对应的查看器是否存在且未禁用
   └─ 不存在则返回 500

6. 会话写入上下文
   └─ util.WithValue(c, manager.ViewerSessionCacheCtx{}, &session)
      注意：写入的 key 是 manager.ViewerSessionCacheCtx{}，不是 wopi.WopiSessionCtx
```

### 2.3 写入权限控制（三层机制详解）

> **⚠️ 重要修正**：写入权限控制分为三层，各层功能、调用顺序和生效方式均不同。

#### 调用顺序总览

```
WOPI 请求到达
    │
    ▼
┌─ 中间件层 ──────────────────────────────────────────┐
│ ① ViewerSessionValidation（总是生效）               │
│   - 验证 access_token → 会话 → 用户 → 文件归属    │
│   - 写入上下文：manager.ViewerSessionCacheCtx{}     │
│                                                     │
│ ② WopiWriteAccess（未接入！即使接入也无法工作）     │
│   - 读取上下文：wopi.WopiSessionCtx ("wopi_session")│
│   - 期望类型：*wopi.SessionCache（含 Action 字段） │
│   - 实际类型：*manager.ViewerSessionCache（无 Action）│
│   - key 不同 + 类型不同 → MustGet 会 panic         │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─ 服务层（CheckFileInfo 接口）──────────────────────┐
│ ③ 客户端只读提示                                    │
│   - 计算 canEdit → 设置 ReadOnly/UserCanWrite      │
│   - 仅告知客户端，无服务端强制拦截                  │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─ 服务层（PutContent 接口）─────────────────────────┐
│ ④ 服务端上传能力校验                                │
│   - m.Get() 时传入 NavigatorCapabilityUploadFile   │
│   - getNavigator() 检查文件系统是否支持上传能力     │
│   - 不支持则返回 ErrNotSupportedAction（强制拦截）  │
└─────────────────────────────────────────────────────┘
```

#### 第一层：客户端只读提示（CheckFileInfo 接口）

**核心代码**：[viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L334-L335)

在 `FileInfo` 接口中，`canEdit` 条件：
```go
canEdit := file.PrimaryEntityID() == targetEntity.ID()  // 必须是最新版本
       && file.OwnerID() == user.ID                      // 必须是文件所有者
       && uri.FileSystem() == constants.FileSystemMy     // 必须在"我的文件"中
```

满足条件时，WOPI 响应中的字段设置：
- `ReadOnly: false`
- `UserCanWrite: true`
- `UserCanReview: true`

**功能定位**：
- 仅用于**告知 WOPI 客户端**是否显示编辑界面
- 不是服务端的强制拦截，客户端可忽略此提示
- 调用时机：WOPI 客户端初始化时调用 `CheckFileInfo`

#### 第二层：服务端上传能力校验（PutContent 接口）

**核心代码**：
- 调用点：[viewer.go L165](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L165)
- 校验实现：[dbfs.go L763-L769](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L763-L769)

在 `PutContent` 接口中，获取文件时指定上传能力要求：
```go
file, err := m.Get(c, uri, 
    dbfs.WithRequiredCapabilities(dbfs.NavigatorCapabilityUploadFile), 
    dbfs.WithNotRoot())
```

`getNavigator()` 中执行实际校验：
```go
capabilities := res.Capabilities(false).Capability
for _, capability := range requiredCapabilities {
    if !capabilities.Enabled(int(capability)) {
        return nil, fs.ErrNotSupportedAction.WithError(
            fmt.Errorf("action %q is not supported under current fs", capability))
    }
}
```

**功能定位**：
- 服务端**强制拦截**，不具备上传能力则写入失败
- 检查**文件系统级别**的能力（如回收站、共享目录可能没有上传能力）
- 不检查用户级别的编辑权限（只要有上传能力就允许写入）
- 调用时机：每次 `PutContent` 实际写入时

#### 第三层：未接入的写入校验（WopiWriteAccess 中间件）

**核心代码**：
- 定义：[middleware/wopi.go L16-L28](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/middleware/wopi.go#L16-L28)
- 路由：[router.go L215-L225](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/routers/router.go#L215-L225)
- 会话缓存定义：[manager/viewer.go L23-L31](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/manager/viewer.go#L23-L31)
- WOPI 会话定义：[wopi/types.go L59-L64](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/wopi/types.go#L59-L64)

**问题一：路由未接入**
```go
// 实际路由配置
wopi := noAuth.Group("file/wopi",
    middleware.HashID(hashid.FileID),
    middleware.ViewerSessionValidation())  // 只有会话验证，没有 WopiWriteAccess!
```

**问题二：上下文 key 不匹配**

| 中间件 | 读取的上下文 key | 实际写入的上下文 key |
|-------|-----------------|-------------------|
| `WopiWriteAccess` | `wopi.WopiSessionCtx` (字符串 `"wopi_session"`) | — |
| `ViewerSessionValidation` | — | `manager.ViewerSessionCacheCtx{}` (空结构体) |

**问题三：类型不匹配**

| 中间件 | 期望的类型 | 实际存储的类型 |
|-------|-----------|--------------|
| `WopiWriteAccess` | `*wopi.SessionCache`（含 `Action ActonType` 字段） | `*manager.ViewerSessionCache`（**无** `Action` 字段） |

**结论**：
- 即使将 `WopiWriteAccess` 加入路由，`MustGet` 也会因为 key 不匹配而 **panic**
- 即使 key 碰巧匹配，类型断言 `.(*wopi.SessionCache)` 也会 **panic**
- 这是遗留的设计缺陷，`WopiWriteAccess` 是完全的"死代码"

---

## 3. 文件锁实现

### 3.1 锁系统接口

**核心代码**：[lock/memlock.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/lock/memlock.go#L28-L33)

```go
type LockSystem interface {
    Create(now time.Time, details ...LockDetails) ([]string, error)
    Unlock(now time.Time, tokens ...string) error
    Confirm(now time.Time, requests LockInfo) (func(), string, error)
    Refresh(now time.Time, duration time.Duration, token string) (LockDetails, error)
}
```

### 3.2 内存锁数据结构

**核心代码**：[lock/memlock.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/lock/memlock.go#L77-L402)

`memLS` 是基于内存的锁实现，使用以下数据结构：

```go
type memLS struct {
    mu        sync.Mutex                     // 全局互斥锁
    byName    map[string]map[string]*memLSNode  // ns -> path -> node
    byToken   map[string]*memLSNode          // token -> node
    byExpiry  byExpiry                       // 过期堆（优先队列）
    gen       uint64                         // 生成计数器
}
```

### 3.3 锁持续时间

**核心代码**：[wopi.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/wopi/wopi.go#L57)

```go
LockDuration = time.Duration(30) * time.Minute
```

WOPI 锁的有效期为 **30 分钟**，客户端需要定期刷新以维持锁。

### 3.4 WOPI 锁操作

WOPI 协议通过 `X-WOPI-Override` 头部指定操作类型：

| 操作类型 | HTTP 方法 | X-WOPI-Override | 说明 |
|---------|----------|----------------|------|
| LOCK | POST | LOCK | 创建或刷新锁 |
| UNLOCK | POST | UNLOCK | 释放锁 |
| REFRESH_LOCK | POST | REFRESH_LOCK | 刷新锁有效期 |

#### 3.4.1 LOCK 操作

**核心代码**：[viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L102-L156)

```
LOCK 流程：
1. 解析文件 URI，获取文件对象
2. 从 X-WOPI-Lock 头部获取客户端提供的锁令牌
3. 调用 ConfirmLock 检查是否已被相同令牌锁定
   ├─ 已锁定：刷新锁，返回 200 和锁令牌
   └─ 未锁定：尝试创建新锁
4. 创建锁失败时的冲突处理
   └─ 返回 409 Conflict，X-WOPI-Lock 头部携带现有锁令牌
5. 创建锁成功：返回 200，X-WOPI-Lock 头部携带锁令牌
```

#### 3.4.2 REFRESH_LOCK 操作

**核心代码**：[viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L65-L100)

```
REFRESH_LOCK 流程：
1. 确认文件存在且可读
2. 调用 ConfirmLock 验证锁令牌
   ├─ 验证失败：返回 409，X-WOPI-Lock 为空字符串
   └─ 验证成功：释放临时持有，刷新锁有效期
3. 返回 200，X-WOPI-Lock 头部携带锁令牌
```

#### 3.4.3 UNLOCK 操作

**核心代码**：[viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L46-L63)

```
UNLOCK 流程：
1. 调用 Unlock 方法释放锁
   ├─ 成功：返回 200
   └─ 失败（未锁定或令牌不匹配）：返回 409，X-WOPI-Lock 为空
```

### 3.5 锁冲突检测

**核心代码**：[lock/memlock.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/lock/memlock.go#L188-L230)

`canCreate` 方法执行三级冲突检测：

```
冲突检测逻辑：
1. 目标节点已被锁定 → 冲突
2. 请求非零深度锁，且目标节点存在后代锁 → 冲突
3. 祖先节点被非零深度锁锁定 → 冲突
```

### 3.6 锁过期清理

**核心代码**：[lock/memlock.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/lock/memlock.go#L262-L269)

每次锁操作前调用 `collectExpiredNodes`，通过最小堆（`byExpiry`）高效清理过期锁：
```go
func (m *memLS) collectExpiredNodes(now time.Time) {
    for len(m.byExpiry) > 0 {
        if now.Before(m.byExpiry[0].expiry) {
            break
        }
        m.remove(m.byExpiry[0])
    }
}
```

---

## 4. 文件回写路径

### 4.1 回写入口

**核心代码**：[controllers/wopi.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/routers/controllers/wopi.go#L36-L43)

两种回写模式：

| 模式 | 路由 | 说明 |
|-----|------|------|
| 覆盖更新 | POST /file/wopi/:id/contents | 覆盖现有文件 |
| 另存为新文件 | POST /file/wopi/:id | X-WOPI-Override: PUT_RELATIVE |

### 4.2 回写处理流程

**核心代码**：[viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L158-L266)

```
PutContent 处理流程：
┌─────────────────────────────────────────────────┐
│ 1. 准备文件上下文                                │
│    ├─ 解析 URI，获取文件对象                     │
│    └─ 验证文件具有上传能力（第二层校验）         │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│ 2. 锁验证（仅当携带锁令牌时）                     │
│    ├─ if lockToken != "" {                      │
│    │  ├─ ConfirmLock 验证令牌有效性             │
│    │  │  ├─ 验证失败：尝试用该令牌创建新锁        │
│    │  │  │  ├─ 创建成功：解锁并返回空锁令牌      │
│    │  │  │  └─ 创建失败（冲突）：返回 409        │
│    │  │  └─ 验证成功：defer 释放锁               │
│    │  └─ 保存 LockSession 供后续使用             │
│    └─ } else {                                   │
│       └─ lockSession = nil（不进行任何锁检查）  │
│    }                                             │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│ 3. PUT_RELATIVE 模式处理（另存为）               │
│    ├─ 从 X-WOPI-SuggestedTarget 获取目标文件名  │
│    ├─ UTF-7 解码文件名                          │
│    ├─ 如仅含扩展名（以.开头），拼入原文件名     │
│    └─ 构建新文件 URI                            │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│ 4. 构建上传请求                                  │
│    ├─ FileUpdateService.Uri = fileUri           │
│    ├─ ⚠️ FileUpdateService.Previous = ""（空）  │
│    └─ PreviousVersion 未传入，底层版本校验不触发 │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│ 5. 执行文件更新                                  │
│    ├─ 创建 FileUpdateService                    │
│    ├─ 注入 LockSession（可能为 nil）到上下文     │
│    ├─ 调用 PutContent → m.Update                │
│    └─ 底层 CreateEntity 跳过版本校验            │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│ 6. 错误处理与响应                                │
│    ├─ CodeFileTooLarge → 413 Request Too Large  │
│    ├─ CodeNotFound → 404 Not Found              │
│    ├─ 成功：返回 200，X-WOPI-ItemVersion 头部   │
│    └─ PUT_RELATIVE：返回新文件名 JSON           │
└─────────────────────────────────────────────────┘
```

### 4.3 UTF-7 文件名解码

**核心代码**：[utf7.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/wopi/utf7.go)

WOPI 协议使用 **Modified UTF-7** 编码传输非 ASCII 文件名。主要特点：
- 使用 `&` 而非 `+` 作为 Base64 移位标记
- 使用 `,` 而非 `/` 作为 Base64 字符
- `&` 字符编码为 `&-`
- 解码入口：`wopi.UTF7Decode()`

### 4.4 实际文件更新

**核心代码**：[file.go L305-L354](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/file.go#L305-L354)

```go
func (service *FileUpdateService) PutContent(c *gin.Context, ls fs.LockSession) (*FileResponse, error)
```

**关键细节**：`FileUpdateService` 结构体定义：
```go
type FileUpdateService struct {
    Uri      string `form:"uri" binding:"required"`
    Previous string `form:"previous"`  // ← 旧版本标识，WOPI 路径未设置此字段
}
```

更新流程：
1. **内容长度嗅探**：通过 `request.SniffContentLength` 获取请求体大小
2. **大小限制检查**：不超过 `MaxOnlineEditSize` 配置
3. **构建上传请求**：`fs.UploadRequest` 中 `PreviousVersion` 为空（WOPI 未传参）
4. **注入锁会话**：将 `LockSession`（可能为 `nil`）注入上下文
5. **执行更新**：调用 `m.Update(ctx, fileData)` 更新文件
6. **返回结果**：包含新版本号的文件信息

### 4.5 回写 API 端点

**核心代码**：[router.go L215-L225](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/routers/router.go#L215-L225)

```go
wopi := noAuth.Group("file/wopi", 
    middleware.HashID(hashid.FileID), 
    middleware.ViewerSessionValidation())
{
    wopi.GET(":id", controllers.CheckFileInfo)
    wopi.GET(":id/contents", controllers.GetFile)
    wopi.POST(":id/contents", controllers.PutFile)
    wopi.POST(":id", controllers.ModifyFile)
}
```

### 4.6 历史版本读取与回写一致性

> **⚠️ 关键发现**：打开历史版本后的编辑存在严重的一致性问题 —— 读取的是历史版本，但回写的是最新版本。

#### 4.6.1 读取时锁定的版本

**核心代码**：
- 会话创建：[manager/viewer.go L46-L70](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/manager/viewer.go#L46-L70)
- 实体查找：[fs.go L711-L734](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/fs/fs.go#L711-L734)
- GetFile：[viewer.go L280-L284](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L280-L284)

**版本锁定机制**：

```go
// 会话创建时，version 参数被保存到 ViewerSessionCache
sessionCache := &ViewerSessionCache{
    ID:       sessionID,
    Uri:      file.Uri(false).String(),
    UserID:   m.user.ID,
    ViewerID: viewer.ID,
    FileID:   file.ID(),
    Version:  version,  // ← 版本号被持久化到会话缓存
    Token:    fmt.Sprintf("%s.%s", sessionID, token),
}

// FindDesiredEntity 根据 version 查找目标实体
func FindDesiredEntity(file File, version string, hasher hashid.Encoder, entityType *types.EntityType) (bool, Entity) {
    if version == "" {
        // version 为空时返回主实体（最新版本）
        return true, file.PrimaryEntity()
    }
    // version 非空时，解码并查找特定的历史版本实体
    requestedVersion, err := hasher.Decode(version, hashid.EntityID)
    for _, entity := range file.Entities() {
        if entity.ID() == requestedVersion && (entityType == nil || *entityType == entity.Type()) {
            return true, entity
        }
    }
    // ...
}
```

**读取路径**：
```
GetFile (viewer.go L268)
    ↓
fs.FindDesiredEntity(file, viewerSession.Version, ...)  (L281)
    ↓
根据 session.Version 找到历史版本实体 targetEntity
    ↓
m.GetEntitySource(c, targetEntity.ID(), fs.WithEntity(targetEntity))  (L290)
    ↓
返回历史版本的文件内容
```

**结论**：读取时确实锁定了会话创建时指定的历史版本，返回的是该版本的实体内容。

#### 4.6.2 回写时数据落在哪里

**核心代码**：
- PutContent：[viewer.go L212-L234](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L212-L234)
- FileUpdateService：[file.go L305-L308](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/file.go#L305-L308)

**回写路径分析**：

```go
// PutContent 中使用的是 viewerSession.Uri（文件 URI），不是版本特定的 URI
fileUri := viewerSession.Uri  // L212

// 创建 FileUpdateService 时只设置了 Uri，Previous 为空
subService := FileUpdateService{
    Uri: fileUri,  // L233
    // ❌ 没有设置 Previous，也没有使用 viewerSession.Version
}

// FileUpdateService 定义
type FileUpdateService struct {
    Uri      string `form:"uri" binding:"required"`
    Previous string `form:"previous"`  // ← WOPI 路径下始终为空
}
```

**回写流向**：
```
PutContent (viewer.go L158)
    ↓
fileUri = viewerSession.Uri  ← 文件 URI，指向文件本身，不是特定版本
    ↓
FileUpdateService{Uri: fileUri, Previous: ""}
    ↓
PutContent → m.Update (file.go L348)
    ↓
CreateEntity → 创建新的主实体（最新版本）
    ↓
✅ 写入到文件的最新版本（覆盖当前主实体）
❌ **不会写入到读取的历史版本**
```

**关键代码证据** - `canEdit` 条件：
```go
// viewer.go L334
canEdit := file.PrimaryEntityID() == targetEntity.ID()  // 必须是最新版本
       && file.OwnerID() == user.ID                      // 必须是文件所有者
       && uri.FileSystem() == constants.FileSystemMy     // 必须在"我的文件"中
```

**一致性问题总结**：

| 阶段 | 操作的版本 | 说明 |
|-----|-----------|------|
| 会话创建 | `s.Version`（可能是历史版本） | 保存到 `ViewerSessionCache.Version` |
| GetFile 读取 | `viewerSession.Version` 指定的历史版本 | 通过 `FindDesiredEntity` 找到历史实体 |
| CheckFileInfo | 检查 `PrimaryEntityID() == targetEntity.ID()` | 历史版本时 `canEdit = false`，客户端只读 |
| PutContent 回写 | **文件的最新版本**（主实体） | 使用 `viewerSession.Uri`，不区分版本 |

**问题**：
1. 如果打开的是历史版本，`canEdit = false`，客户端会显示只读
2. 但如果客户端绕过这个限制（直接调用 PutContent API），回写会成功
3. 回写会**覆盖最新版本**，而不是修改历史版本
4. 用户编辑的是历史版本的内容，但保存后最新版本被覆盖，导致"打开历史版本编辑后内容不一致"

#### 4.6.3 查看/编辑操作类型是否记录在会话中

> **⚠️ 关键发现**：操作类型（查看/编辑）**只在链接生成阶段使用**，没有记录在会话状态中。

**核心代码**：
- 会话缓存定义：[manager/viewer.go L23-L31](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/manager/viewer.go#L23-L31)
- 请求参数：[viewer.go L369-L373](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L369-L373)
- WOPI 链接生成：[wopi.go L60-L85](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/wopi/wopi.go#L60-L85)

**分析**：

```go
// 1. 请求参数中有 PreferredAction 字段
type CreateViewerSessionService struct {
    Uri             string               `json:"uri" form:"uri" binding:"required"`
    Version         string               `json:"version" form:"version"`
    ViewerID        string               `json:"viewer_id" form:"viewer_id" binding:"required"`
    PreferredAction types.ViewerAction   `json:"preferred_action" form:"preferred_action" binding:"required"`
    // ↑ 有这个字段，但只在生成链接时使用
}

// 2. 会话缓存中没有 Action 字段
type ViewerSessionCache struct {
    ID       string
    Uri      string
    UserID   int
    FileID   int
    ViewerID string
    Version  string
    Token    string
    // ❌ 没有 Action 字段！
}

// 3. 调用 CreateViewerSession 时没有传递 action
// viewer.go L409
viewerSession, err := m.CreateViewerSession(c, uri, s.Version, targetViewer)
// ↑ 只传了 version，没有传 s.PreferredAction

// 4. PreferredAction 只在生成 WOPI 链接时使用
// viewer.go L417
wopiSrc, err := wopi.GenerateWopiSrc(c, s.PreferredAction, targetViewer, viewerSession)
// ↑ 用于选择 WOPI 服务器的 URL 模板（embedview 或 edit）

// 5. GenerateWopiSrc 中 action 的用途
func GenerateWopiSrc(ctx context.Context, action types.ViewerAction, ...) (*url.URL, error) {
    // 根据 action 选择可用的 WOPI 操作 URL
    availableActions, ok := viewer.WopiActions[viewerSession.File.Ext()]
    fallbackOrder := []types.ViewerAction{action, types.ViewerActionView, types.ViewerActionEdit}
    for _, a := range fallbackOrder {
        if src, ok = availableActions[a]; ok {
            break
        }
    }
    // 生成 WOPI 客户端 URL，不会修改会话
}
```

**操作类型的生命周期**：
```
用户请求创建会话（携带 preferred_action=edit）
    ↓
CreateViewerSessionService 接收参数
    ↓
m.CreateViewerSession() → 创建会话（不保存 action）
    ↓
wopi.GenerateWopiSrc(preferred_action, ...) → 选择 WOPI URL 模板
    ↓
返回 WOPI src URL 给前端
    ↓
前端跳转到 WOPI 客户端（URL 决定是预览还是编辑模式）
    ↓
后续所有 WOPI 请求（CheckFileInfo/GetFile/PutFile）
    ↓
⚠️ 会话中没有 action 信息，无法区分是查看还是编辑操作
```

**影响**：
1. 中间件 `WopiWriteAccess` 期望从会话中读取 `Action` 字段来区分编辑操作，但会话中根本没有这个字段
2. 服务端无法通过会话判断当前是"查看"还是"编辑"模式
3. 即使前端生成的是"预览"链接，用户仍然可以通过直接调用 API 进行写入
4. 无法基于操作类型进行细粒度的权限控制或审计

---

## 5. 编辑冲突处理

### 5.1 冲突预防机制

#### 5.1.1 悲观锁策略（客户端可选）

WOPI 采用 **悲观锁** 策略防止冲突，但锁是**客户端可选**的：

```
客户端正确流程（带锁）：
1. 编辑前先获取文件锁（LOCK 操作）
2. 持有锁期间定期刷新（REFRESH_LOCK，每 10 分钟左右）
3. 回写时携带有效的锁令牌
4. 编辑完成后释放锁（UNLOCK）

实际可能的流程（不带锁）：
1. 直接获取文件内容（GetFile）
2. 编辑后直接回写（PutFile），不携带锁令牌
3. 服务端不进行任何冲突检查，直接覆盖
```

#### 5.1.2 锁前置检查（仅当携带锁令牌时）

**核心代码**：[viewer.go L170-L210](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L170-L210)

```go
lockToken := c.GetHeader(wopi.LockTokenHeader)
if lockToken != "" {
    // 只有当 lockToken 不为空时才验证锁
    release, ls, err := m.ConfirmLock(c, file, file.Uri(false), lockToken)
    if err != nil {
        ls, err := m.Lock(c, wopi.LockDuration, user, true, app, file.Uri(false), lockToken)
        // ...
    }
}
// 如果 lockToken 为空，跳过所有锁检查，直接更新
```

### 5.2 冲突检测与响应

#### 5.2.1 锁冲突响应

**核心代码**：[viewer.go L127-L139](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L127-L139)

当锁冲突发生时，按照 WOPI 协议规范响应：

```go
var lockConflict lock.ConflictError
if errors.As(err, &lockConflict) {
    c.Status(http.StatusConflict)
    c.Header(wopi.LockTokenHeader, lockConflict[0].Token)
    return nil
}
```

**注意**：只有当客户端携带锁令牌时才可能触发此响应。

#### 5.2.2 一致性检查（仅在获取锁后执行）

**核心代码**：[dbfs/lock.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/fs/dbfs/lock.go#L220-L263)

在 `acquireByPath` 获取锁**之后**，执行 `ensureConsistency` 检查：

```go
func (f *DBFS) ensureConsistency(ctx context.Context, files ...*File) error {
    // 重新从数据库查询文件及其所有祖先
    // 检查以下字段是否与获取锁前一致：
    //   - Name        （文件名）
    //   - FileChildren（子文件数）
    //   - OwnerID     （所有者ID）
    //   - Type        （文件类型）
}
```

**关键限制**：
- 只在 `acquireByPath` 中调用，即只有获取锁时才执行
- 如果不携带锁令牌，跳过锁获取，`ensureConsistency` 也不会执行
- 只检查元数据一致性，不检查内容版本

### 5.3 版本与冲突校验的生效层级

> **⚠️ 重要修正**：底层 DBFS 已有旧版本校验能力，但 WOPI 回写路径**尚未接入**。

#### 5.3.1 底层版本校验机制（DBFS CreateEntity）

**核心代码**：[dbfs.go L251-L267](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L251-L267)

```go
func (f *DBFS) CreateEntity(ctx context.Context, ...) (fs.Entity, error) {
    o := newDbfsOption()
    for _, opt := range opts {
        o.apply(opt)
    }

    // 如果上传者指定了之前的最新版本 ID (etag)，检查是否仍然有效
    if o.previousVersion != "" {
        entityId, err := f.hasher.Decode(o.previousVersion, hashid.EntityID)
        if err != nil {
            return nil, serializer.NewError(serializer.CodeParamErr, "Unknown version ID", err)
        }

        entities, err := file.(*File).Model.Edges.EntitiesOrErr()
        if err != nil || entities == nil {
            return nil, fmt.Errorf("create entity: previous entities not load")
        }

        // 如果最新实体与上传者指定的不一致，说明编辑期间文件已被他人修改
        if e := file.PrimaryEntity(); e == nil || e.ID() != entityId {
            return nil, fs.ErrStaleVersion
        }
    }
    // ... 后续创建新实体
}
```

**校验逻辑**：
1. 解码 `previousVersion`（HashID 编码的实体 ID）
2. 获取文件的当前主实体
3. 比较当前主实体 ID 与传入的版本 ID
4. 不一致则返回 `fs.ErrStaleVersion`（版本过期错误）

**参数传递链**：
```
调用方传入 PreviousVersion
    → UploadProps.PreviousVersion 字段
    → WithPreviousVersion(req.Props.PreviousVersion) 选项
    → dbfsOption.previousVersion 字段
    → CreateEntity 中 o.previousVersion != "" 检查
    → 对比 PrimaryEntity().ID() 与解码后的 entityId
```

#### 5.3.2 WOPI 回写路径为何未接入版本校验

**调用链追踪**：

```
WopiService.PutContent (viewer.go L158)
    ↓
FileUpdateService{Uri: fileUri} (viewer.go L232-L234)
    ↓  ⚠️ Previous 字段为空
FileUpdateService.PutContent (file.go L311)
    ↓  PreviousVersion: service.Previous (空字符串)
UploadProps.PreviousVersion = "" (file.go L332)
    ↓
m.Update → fs.PrepareUpload → CreateEntity
    ↓  WithPreviousVersion("")  → previousVersion = ""
CreateEntity 中 o.previousVersion != "" → false
    ↓  ⚠️ 版本校验被跳过
直接创建新实体，不检查版本是否过期
```

**根本原因**：

1. **WOPI 协议层未传递版本号**：
   - WOPI 协议通过 `X-WOPI-Lock` 锁令牌而非版本号进行并发控制
   - 客户端回写时不携带 `PreviousVersion` 之类的版本标识
   - `WopiService.PutContent` 中没有读取任何版本相关头部

2. **PutContent 未设置 PreviousVersion**：
   ```go
   // viewer.go L232-L234 — WOPI 路径创建 FileUpdateService
   subService := FileUpdateService{
       Uri: fileUri,
       // ❌ 没有设置 Previous!
   }
   ```

3. **设计思路依赖锁机制**：
   - 预期 WOPI 客户端通过 LOCK/REFRESH_LOCK 机制保证独占编辑
   - 认为持有锁就保证了不会有并发写入
   - 因此没有额外接入版本校验

**问题**：
- 由于锁是可选的（客户端可不携带锁令牌直接写入），版本校验也随之失效
- 存在"无锁 + 无版本校验"的双重缺失，可能导致数据静默覆盖
- 底层已有完整的版本校验能力，只需在 WOPI 路径中传入 `PreviousVersion` 即可激活

#### 5.3.3 冲突校验的完整生效层级

| 层级 | 校验机制 | 生效条件 | 是否接入 WOPI | 说明 |
|-----|---------|---------|--------------|------|
| 1. WOPI 协议层 | 锁令牌验证 | 客户端携带 `X-WOPI-Lock` 头部 | ⚠️ 可选接入 | 客户端可跳过 |
| 2. DBFS 锁层 | `ensureConsistency` 检查 | 获取锁成功后自动执行 | ⚠️ 间接接入 | 依赖上层锁机制 |
| 3. DBFS 实体层 | `PreviousVersion` 版本校验 | 调用方传入 `previousVersion` 选项 | ❌ **未接入** | 底层有能力，WOPI 未传参 |
| 4. 数据库层 | 数据库事务 | 总是执行 | ✅ 总是接入 | 保证原子性，但不检查业务冲突 |

**各层功能对比**：

| 机制 | 防止的问题 | 检查内容 | 触发方式 | WOPI 状态 |
|-----|-----------|---------|---------|----------|
| 锁令牌 | 并发编辑冲突 | 文件是否被他人锁定 | `X-WOPI-Lock` 头部 | 可选 |
| ensureConsistency | TOCTOU 攻击 | 文件名/子文件数/所有者/类型 | 获取锁后自动检查 | 间接（依赖锁） |
| **PreviousVersion** | **版本过期覆盖** | **主实体 ID 是否匹配** | **调用时传入版本号** | **未接入** |
| 数据库事务 | 数据不一致 | ACID 特性 | 总是执行 | 已接入 |

### 5.4 锁持有者信息

**核心代码**：[lock/memlock.go L59-L68](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/lock/memlock.go#L59-L68)

每个锁记录持有者信息：
```go
type Owner struct {
    Application Application `json:"application"`
}

type Application struct {
    Type     string `json:"type"`      // "viewer"
    ViewerID string `json:"viewer_id"` // 查看器 ID
}
```

这支持在冲突时向用户显示"文件正在被 [应用名称] 编辑"。

### 5.5 冲突场景总结

| 场景 | 检测点 | 响应 | 生效条件 |
|-----|--------|------|---------|
| 文件已被他人锁定 | LOCK/REFRESH_LOCK 时 ConfirmLock 失败 | 409 + X-WOPI-Lock | 客户端携带锁令牌 |
| 锁令牌不匹配 | PutContent 时令牌与现有锁不一致 | 409 + X-WOPI-Lock | 客户端携带锁令牌 |
| 文件在锁定间隙被修改 | acquireByPath 后的 ensureConsistency 检查 | fs.ErrModified | 客户端携带锁令牌 |
| **版本过期覆盖** | **CreateEntity 中 PreviousVersion 检查** | **fs.ErrStaleVersion** | **❌ WOPI 未传参** |
| 并发无锁写入 | ❌ 无检测 | 后写入者覆盖先写入者 | 客户端不携带锁令牌 |
| 文件大小超限 | PutContent 中检查 MaxOnlineEditSize | 413 Request Too Large | 总是生效 |
| 文件已被删除 | PutContent 中 Get 返回 CodeNotFound | 404 Not Found | 总是生效 |

---

## 6. 关键代码文件索引

| 文件路径 | 核心职责 | 关键函数/类型 |
|---------|---------|--------------|
| [pkg/wopi/wopi.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/wopi/wopi.go) | WOPI 协议常量与工具 | `GenerateWopiSrc`, `LockDuration` |
| [pkg/wopi/types.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/wopi/types.go) | WOPI 类型定义 | `SessionCache`（含 Action 字段）, `WopiSessionCtx` |
| [pkg/wopi/utf7.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/wopi/utf7.go) | UTF-7 编解码 | `UTF7Decode`, `UTF7Encode` |
| [middleware/wopi.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/middleware/wopi.go) | 鉴权中间件 | `ViewerSessionValidation`, `WopiWriteAccess`（未接入） |
| [routers/router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/routers/router.go#L215-L225) | 路由定义 | WOPI 端点路由组 |
| [routers/controllers/wopi.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/routers/controllers/wopi.go) | WOPI 控制器 | `CheckFileInfo`, `GetFile`, `PutFile`, `ModifyFile` |
| [service/explorer/viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go) | WOPI 业务逻辑 | `WopiService.Lock`, `PutContent`, `FileInfo` |
| [service/explorer/file.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/file.go#L305-L354) | 文件更新 | `FileUpdateService.PutContent`, `Previous` 字段 |
| [service/explorer/response.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/response.go#L196-L238) | 响应类型 | `WopiFileInfo` |
| [pkg/filemanager/lock/memlock.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/lock/memlock.go) | 内存锁实现 | `memLS`, `LockSystem` 接口 |
| [pkg/filemanager/fs/dbfs/lock.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/fs/dbfs/lock.go) | 文件系统锁封装 | `DBFS.Lock`, `ConfirmLock`, `ensureConsistency` |
| [pkg/filemanager/fs/dbfs/dbfs.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L251-L267) | 实体创建+版本校验 | `CreateEntity`, `PreviousVersion` 版本校验 |
| [pkg/filemanager/fs/dbfs/options.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/fs/dbfs/options.go#L112-L117) | DBFS 选项 | `WithPreviousVersion` |
| [pkg/filemanager/manager/viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/manager/viewer.go) | 会话管理 | `CreateViewerSession`, `ViewerSessionCache`（无 Action） |

---

## 7. 时序图：完整编辑流程

```
用户客户端         WOPI 客户端        Cloudreve 服务器
    |                  |                   |
    | 1. 请求在线编辑  |                   |
    |----------------->|                   |
    |                  | 2. CreateViewerSession |
    |                  |------------------>|
    |                  | 3. 返回 access_token + WOPI src |
    |                  |<------------------|
    |                  |                   |
    |                  | 4. CheckFileInfo (access_token) |
    |                  |------------------>|
    |                  |                   | 验证会话
    |                  |                   | 计算 canEdit → ReadOnly/UserCanWrite
    |                  | 5. 返回文件元数据  |    （客户端只读提示）
    |                  |<------------------|
    |                  |                   |
    |                  | 6. GetFile        |
    |                  |------------------>|
    |                  | 7. 返回文件内容   |
    |                  |<------------------|
    |                  |                   |
    | 8. 用户编辑文件  |                   |
    |<---------------->|                   |
    |                  |                   |
    |                  | 9. LOCK (X-WOPI-Lock: token) |
    |                  |------------------>|
    |                  | 10. 200 OK + X-WOPI-Lock |
    |                  |<------------------|
    |                  |                   |
    |                  | 11. 定期 REFRESH_LOCK |
    |                  |------------------>|
    |                  |<------------------|
    |                  |                   |
    |                  | 12. PUT /contents (X-WOPI-Lock: token) |
    |                  |------------------>|
    |                  |                   | 上传能力校验（强制）
    |                  |                   | 锁验证（如携带，可选）
    |                  |                   | ⚠️ 版本校验未接入
    |                  |                   | 更新文件
    |                  | 13. 200 OK + X-WOPI-ItemVersion |
    |                  |<------------------|
    |                  |                   |
    |                  | 14. UNLOCK        |
    |                  |------------------>|
    |                  |<------------------|
```

---

## 8. 关键修正总结

### 修正 1：写入权限限制的调用顺序

写入权限限制分为三层，按调用顺序：

| 层级 | 位置 | 功能 | 强制性 |
|-----|------|------|-------|
| ① 客户端只读提示 | `FileInfo` 接口 → `canEdit` | 告知客户端是否显示编辑界面 | 否，客户端可忽略 |
| ② 服务端上传能力校验 | `PutContent` → `m.Get(WithRequiredCapabilities(UploadFile))` | 检查文件系统是否支持上传 | 是，不支持则失败 |
| ③ 未接入的写入校验 | `WopiWriteAccess` 中间件 | 检查会话 Action 是否为 Edit | 未接入，且 key/类型不匹配会 panic |

**`WopiWriteAccess` 无法工作的三重原因**：
- 路由未使用此中间件
- 读取的上下文 key (`wopi.WopiSessionCtx`) 与写入的 key (`manager.ViewerSessionCacheCtx{}`) 不同
- 期望的类型 (`*wopi.SessionCache`) 与实际类型 (`*manager.ViewerSessionCache`) 不同，且后者无 `Action` 字段

### 修正 2：版本与冲突校验的生效层级

底层 DBFS 已有完整的版本校验能力，但 WOPI 回写路径尚未接入：

| 校验机制 | 底层能力 | WOPI 接入状态 |
|---------|---------|--------------|
| 锁令牌验证 | ✅ 完整 | ⚠️ 可选（客户端可不携带） |
| ensureConsistency | ✅ 完整 | ⚠️ 间接（依赖锁） |
| **PreviousVersion 版本校验** | **✅ 完整** | **❌ 未接入**（`FileUpdateService.Previous` 为空） |
| 数据库事务 | ✅ 完整 | ✅ 已接入 |

**接入方法**：在 [viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L232-L234) 的 `PutContent` 中，将文件的当前版本号传入 `FileUpdateService.Previous`，即可激活底层版本校验。

---

*文档生成时间：2026-06-13*
*基于 Cloudreve v4 代码库分析*
*最后修正：2026-06-13（详细说明写入权限三层调用顺序；分层解释版本校验底层能力与 WOPI 未接入现状）*
