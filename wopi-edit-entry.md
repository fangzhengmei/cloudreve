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
│  - WopiWriteAccess        (写入权限验证)        │
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

**示例**：`550e8400-e29b-41d4-a716-446655440000.abc123def456...`

**会话缓存**：
- 缓存键：`viewer_session_{sessionID}`
- 存储内容：`ViewerSessionCache` 结构体
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
```

### 2.3 写入权限控制

**核心代码**：[middleware/wopi.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/middleware/wopi.go#L16-L28)

`WopiWriteAccess` 中间件在执行写入操作前验证权限：

- 检查会话的 `Action` 字段是否为 `ActionEdit`
- 只读会话（`ActionPreview` / `ActionView`）返回 404 和 "read-only access" 错误

### 2.4 编辑权限判定

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

**锁节点结构**：
```go
type memLSNode struct {
    details      LockDetails           // 锁元数据
    token        string                // 锁令牌（空表示未显式锁定）
    refCount     int                   // 引用计数（自身+后代锁定数）
    expiry       time.Time             // 过期时间
    byExpiryIndex int                  // 在过期堆中的索引
    held         bool                  // 是否被 Confirm 持有
    childLocks   map[string]*memLSNode // 子锁关系
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

**关键实现细节**：
```go
// 锁创建时指定应用信息
app := lock.Application{
    Type:     string(fs.ApplicationViewer),
    ViewerID: viewerSession.ViewerID,
}
_, err = m.Lock(c, wopi.LockDuration, user, true, app, file.Uri(false), lockToken)
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
            break  // 堆顶未过期，后续都未过期
        }
        m.remove(m.byExpiry[0])  // 移除过期锁
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
│    └─ 验证文件具有上传能力                       │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│ 2. 锁验证（如有锁令牌）                          │
│    ├─ 从 X-WOPI-Lock 头部获取锁令牌             │
│    ├─ ConfirmLock 验证令牌有效性                │
│    │  ├─ 验证失败：尝试用该令牌创建新锁          │
│    │  │  ├─ 创建成功：解锁并返回空锁令牌        │
│    │  │  └─ 创建失败（冲突）：返回 409          │
│    │  └─ 验证成功：defer 释放锁                 │
│    └─ 保存 LockSession 供后续使用               │
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
│ 4. 执行文件更新                                  │
│    ├─ 创建 FileUpdateService                    │
│    ├─ 注入 LockSession 到上下文                 │
│    └─ 调用 PutContent 执行实际更新              │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│ 5. 错误处理与响应                                │
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

**核心代码**：[file.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/file.go#L305-L354)

```go
func (service *FileUpdateService) PutContent(c *gin.Context, ls fs.LockSession) (*FileResponse, error)
```

更新流程：
1. **内容长度嗅探**：通过 `request.SniffContentLength` 获取请求体大小
2. **大小限制检查**：不超过 `MaxOnlineEditSize` 配置
3. **构建上传请求**：`fs.UploadRequest` 包含文件内容和元数据
4. **注入锁会话**：将 `LockSession` 注入上下文供文件管理器使用
5. **执行更新**：调用 `m.Update(ctx, fileData)` 更新文件
6. **返回结果**：包含新版本号的文件信息

### 4.5 回写 API 端点

**核心代码**：[router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/routers/router.go#L215-L225)

```go
wopi := noAuth.Group("file/wopi", 
    middleware.HashID(hashid.FileID), 
    middleware.ViewerSessionValidation())
{
    wopi.GET(":id", controllers.CheckFileInfo)        // 获取文件元数据
    wopi.GET(":id/contents", controllers.GetFile)      // 获取文件内容
    wopi.POST(":id/contents", controllers.PutFile)     // 更新文件内容
    wopi.POST(":id", controllers.ModifyFile)           // 通用修改（锁、另存为）
}
```

---

## 5. 编辑冲突处理

### 5.1 冲突预防机制

#### 5.1.1 乐观锁通过文件锁实现

WOPI 采用 **悲观锁** 策略防止冲突：
1. 客户端在编辑前先获取文件锁（LOCK 操作）
2. 持有锁期间定期刷新（REFRESH_LOCK，每 10 分钟左右）
3. 回写时必须携带有效的锁令牌
4. 编辑完成后释放锁（UNLOCK）

#### 5.1.2 锁前置检查

**核心代码**：[viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L170-L210)

在 `PutContent` 中，即使客户端未携带锁令牌，系统也会尝试创建锁：
```go
lockToken := c.GetHeader(wopi.LockTokenHeader)
if lockToken != "" {
    // 验证现有锁
    release, ls, err := m.ConfirmLock(c, file, file.Uri(false), lockToken)
    if err != nil {
        // 验证失败，尝试创建新锁
        ls, err := m.Lock(c, wopi.LockDuration, user, true, app, file.Uri(false), lockToken)
        // ...
    }
}
```

### 5.2 冲突检测与响应

#### 5.2.1 锁冲突响应

**核心代码**：[viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go#L127-L139)

当锁冲突发生时，按照 WOPI 协议规范响应：

```go
var lockConflict lock.ConflictError
if errors.As(err, &lockConflict) {
    c.Status(http.StatusConflict)                    // 409 Conflict
    c.Header(wopi.LockTokenHeader, lockConflict[0].Token)  // 返回现有锁令牌
    return nil
}
```

**WOPI 协议要求**：
- HTTP 状态码：`409 Conflict`
- 响应头：`X-WOPI-Lock` 包含当前文件上的锁令牌
- 客户端收到 409 后应提示用户"文件正在被其他人编辑"

#### 5.2.2 一致性检查

**核心代码**：[dbfs/lock.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/fs/dbfs/lock.go#L220-L263)

在 `acquireByPath` 获取锁后，执行 `ensureConsistency` 检查：

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

这防止了 **TOCTOU**（Time Of Check, Time Of Use）攻击：在文件查询和锁获取之间，文件可能已被修改。

### 5.3 版本追踪

**核心代码**：[response.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/response.go#L196-L238)

每个文件响应都包含 `Version` 字段：
```go
type WopiFileInfo struct {
    BaseFileName string  // 文件名
    Version      string  // 版本号（EntityID 的 HashID 编码）
    Size         int64   // 文件大小
    // ...
}
```

**版本标识**：
- `Version` 字段是实体 ID 的 HashID 编码
- 每次成功更新后，`X-WOPI-ItemVersion` 响应头返回新版本
- 客户端可通过版本变化检测到其他编辑

### 5.4 锁持有者信息

**核心代码**：[lock/memlock.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/lock/memlock.go#L59-L68)

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

| 场景 | 检测点 | 响应 |
|-----|--------|------|
| 文件已被他人锁定 | LOCK/REFRESH_LOCK/PutContent 时 ConfirmLock 失败 | 409 + X-WOPI-Lock |
| 锁令牌不匹配 | LOCK/PutContent 时令牌与现有锁不一致 | 409 + X-WOPI-Lock |
| 文件在锁定间隙被修改 | acquireByPath 后的 ensureConsistency 检查 | fs.ErrModified |
| 文件大小超限 | PutContent 中检查 MaxOnlineEditSize | 413 Request Too Large |
| 文件已被删除 | PutContent 中 Get 返回 CodeNotFound | 404 Not Found |

---

## 6. 关键代码文件索引

| 文件路径 | 核心职责 | 关键函数/类型 |
|---------|---------|--------------|
| [pkg/wopi/wopi.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/wopi/wopi.go) | WOPI 协议常量与工具 | `GenerateWopiSrc`, `LockDuration` |
| [pkg/wopi/types.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/wopi/types.go) | 类型定义 | `SessionCache`, `WopiDiscovery` |
| [pkg/wopi/utf7.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/wopi/utf7.go) | UTF-7 编解码 | `UTF7Decode`, `UTF7Encode` |
| [middleware/wopi.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/middleware/wopi.go) | 鉴权中间件 | `ViewerSessionValidation`, `WopiWriteAccess` |
| [routers/controllers/wopi.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/routers/controllers/wopi.go) | WOPI 控制器 | `CheckFileInfo`, `GetFile`, `PutFile`, `ModifyFile` |
| [routers/router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/routers/router.go#L215-L225) | 路由定义 | WOPI 端点路由组 |
| [service/explorer/viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/viewer.go) | WOPI 业务逻辑 | `WopiService.Lock`, `PutContent`, `FileInfo` |
| [service/explorer/file.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/file.go#L305-L354) | 文件更新 | `FileUpdateService.PutContent` |
| [service/explorer/response.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/service/explorer/response.go#L196-L238) | 响应类型 | `WopiFileInfo` |
| [pkg/filemanager/lock/memlock.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/lock/memlock.go) | 内存锁实现 | `memLS`, `LockSystem` 接口 |
| [pkg/filemanager/fs/dbfs/lock.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/fs/dbfs/lock.go) | 文件系统锁封装 | `DBFS.Lock`, `ConfirmLock`, `ensureConsistency` |
| [pkg/filemanager/manager/viewer.go](file:///d:/fz/0601-1/solo-dogfeeding/code/50-Cloudreve/pkg/filemanager/manager/viewer.go) | 会话管理 | `CreateViewerSession`, `ViewerSessionCache` |

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
    |                  | 5. 返回文件元数据 |
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
    |                  |                   | 验证锁
    |                  |                   | 更新文件
    |                  | 13. 200 OK + X-WOPI-ItemVersion |
    |                  |<------------------|
    |                  |                   |
    |                  | 14. UNLOCK        |
    |                  |------------------>|
    |                  |<------------------|
```

---

*文档生成时间：2026-06-13*
*基于 Cloudreve v4 代码库分析*
