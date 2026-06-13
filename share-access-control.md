# Cloudreve 分享访问控制代码梳理

## 1. 核心文件索引

| 文件路径 | 功能说明 |
|---------|---------|
| [share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/ent/schema/share.go) | 分享数据模型 Schema 定义 |
| [share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/inventory/share.go) | 分享业务逻辑层（有效性校验、过期判断等） |
| [visit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/visit.go) | 分享访问服务（获取分享信息、密码校验、短链跳转） |
| [manage.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/manage.go) | 分享管理服务（创建、编辑、删除分享） |
| [share_navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go) | 分享文件系统导航器（权限边界核心） |
| [share_preview.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/middleware/share_preview.go) | 社交媒体爬虫 OG 预览中间件 |
| [share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/routers/controllers/share.go) | 分享 HTTP 控制器层 |
| [types.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/inventory/types/types.go) | 权限常量和 ShareProps 类型定义 |
| [navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go) | 各导航器能力（Capability）定义 |
| [file.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/file.go) | DBFS 文件对象（含 disableView 逻辑） |
| [response.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/explorer/response.go) | 分享响应构建（BuildShare） |

---

## 2. 分享数据模型

### 2.1 数据库字段

在 [ent/schema/share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/ent/schema/share.go#L17-L46) 中定义：

```go
type Share struct {
    ent.Schema
}

func (Share) Fields() []ent.Field {
    return []ent.Field{
        field.String("password").Optional(),           // 访问密码
        field.Int("views").Default(0),                  // 浏览次数
        field.Int("downloads").Default(0),              // 下载次数
        field.Time("expires").Nillable().Optional(),    // 过期时间
        field.Int("remain_downloads").Nillable().Optional(), // 剩余下载次数
        field.JSON("props", &types.ShareProps{}).Optional(), // 分享属性
    }
}
```

### 2.2 ShareProps 属性

在 [inventory/types/types.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/inventory/types/types.go#L223-L228) 中定义：

```go
type ShareProps struct {
    ShareView  bool `json:"share_view,omitempty"`   // 是否同步所有者的视图设置
    ShowReadMe bool `json:"show_read_me,omitempty"` // 是否自动展示 README 文件
}
```

### 2.3 用户组权限常量

在 [inventory/types/types.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/inventory/types/types.go#L245-L258) 中定义：

```go
const (
    GroupPermissionIsAdmin           = GroupPermission(iota)
    GroupPermissionIsAnonymous
    GroupPermissionShare             // 是否可以创建分享链接
    GroupPermissionShareDownload     // 是否可以访问（下载）分享链接
    // ...
)
```

---

## 3. 密码校验逻辑

### 3.1 获取分享信息时的密码校验

位置：[service/share/visit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/visit.go#L60-L112)

```go
func (s *ShareInfoService) Get(c *gin.Context) (*explorer.Share, error) {
    // ... 获取 share 对象 ...

    unlocked := true
    // 分享需要密码时校验
    if share.Password != "" && s.Password != share.Password && share.Edges.User.ID != u.ID {
        unlocked = false
    }
    // ...
}
```

**校验规则**：
1. 分享有设置密码（`share.Password != ""`）
2. 传入的密码与分享密码不匹配
3. 当前用户不是分享所有者

三个条件同时满足时，`unlocked = false`，即分享被锁定。

### 3.2 文件系统导航器层面的密码校验

位置：[pkg/filemanager/fs/dbfs/share_navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go#L114-L179)

```go
func (n *shareNavigator) Root(ctx context.Context, path *fs.URI) (*File, error) {
    // ... 获取 share ...

    // 检查密码
    if share.Password != "" && share.Password != path.Password() {
        return nil, ErrShareIncorrectPassword
    }
    // ...
}
```

**错误定义**：[pkg/filemanager/fs/dbfs/navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L26)
```go
ErrShareIncorrectPassword = serializer.NewError(
    serializer.CodeIncorrectPassword, 
    "Incorrect share password", 
    nil,
)
```

---

## 4. 过期处理逻辑

### 4.1 过期判断函数

位置：[inventory/share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/inventory/share.go#L227-L258)

```go
func IsValidShare(share *ent.Share) error {
    // 1. 检查分享是否过期
    if err := IsShareExpired(share); err != nil {
        return err
    }

    // 2. 检查所有者状态
    owner, err := share.Edges.UserOrErr()
    if err != nil || owner.Status != user.StatusActive {
        return ErrOwnerInactive
    }

    // 3. 检查源文件状态
    file, err := share.Edges.FileOrErr()
    if err != nil || file.FileChildren == 0 || file.OwnerID != owner.ID {
        return ErrSourceFileInvalid
    }

    return nil
}

func IsShareExpired(share *ent.Share) error {
    if (share.Expires != nil && share.Expires.Before(time.Now())) ||
        (share.RemainDownloads != nil && *share.RemainDownloads <= 0) {
        return ErrShareLinkExpired
    }
    return nil
}
```

### 4.2 过期触发的三个条件

| 条件 | 说明 | 错误 |
|------|------|------|
| 时间过期 | `share.Expires` 不为空且早于当前时间 | `ErrShareLinkExpired` |
| 下载次数耗尽 | `share.RemainDownloads` 不为空且 ≤ 0 | `ErrShareLinkExpired` |
| 所有者非活跃 | 所有者被删除或状态非 Active | `ErrOwnerInactive` |
| 源文件无效 | 源文件被删除（`FileChildren == 0`）或所有者变更 | `ErrSourceFileInvalid` |

### 4.3 下载次数递减

位置：[inventory/share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/inventory/share.go#L216-L225)

```go
func (c *shareClient) Downloaded(ctx context.Context, share *ent.Share) error {
    stm := c.client.Share.
        UpdateOneID(share.ID).
        AddDownloads(1)
    // 如果设置了剩余下载次数，则递减
    if share.RemainDownloads != nil && *share.RemainDownloads >= 0 {
        stm.AddRemainDownloads(-1)
    }
    _, err := stm.Save(ctx)
    return err
}
```

下载钩子触发位置：[pkg/filemanager/fs/dbfs/share_navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go#L302-L308)

```go
func (n *shareNavigator) ExecuteHook(ctx context.Context, hookType fs.HookType, file *File) error {
    switch hookType {
    case fs.HookTypeBeforeDownload:
        return n.shareClient.Downloaded(ctx, n.share)
    }
    return nil
}
```

---

## 5. 预览访问权限边界

### 5.1 社交媒体爬虫 OG 预览

位置：[middleware/share_preview.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/middleware/share_preview.go#L83-L103)

```go
func SharePreview(dep dependency.Dep) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 仅对社交媒体爬虫生效
        if !isSocialMediaBot(c.GetHeader("User-Agent")) {
            c.Next()
            return
        }
        // 提取分享 ID 和密码，渲染 OG 页面
        id, password := extractShareParams(c)
        html := renderShareOGPage(c, dep, id, password)
        // ...
    }
}
```

**支持的爬虫 UA**：`facebookexternalhit`、`facebot`、`twitterbot`、`linkedinbot`、`discordbot`、`telegrambot`、`slackbot`、`whatsapp`

### 5.2 OG 页面中的权限处理

位置：[middleware/share_preview.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/middleware/share_preview.go#L129-L179)

```go
func renderShareOGPage(c *gin.Context, dep dependency.Dep, id, password string) string {
    // ...
    shareInfo, err := loadShareForOG(c, shareID, password)
    if err != nil {
        // 密码错误或过期时，展示错误描述
        var appErr serializer.AppError
        if errors.As(err, &appErr) {
            data.Description = appErr.Msg
        } else {
            data.Description = ogStatusInvalidLink
        }
        return renderOGHTML(data)
    }

    // 解锁后才展示文件大小和缩略图
    if shareInfo.Unlocked {
        data.Description = formatFileSize(shareInfo.Size)
        thumbnail, err := loadShareThumbnail(c, id, password, shareInfo)
        // ...
    }
}
```

### 5.3 视图同步（disableView）控制

位置：[pkg/filemanager/fs/dbfs/share_navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go#L151)

```go
n.shareRoot.disableView = (share.Props == nil || !share.Props.ShareView) && n.user.ID != n.owner.ID
```

**disableView 触发条件**（同时满足）：
1. 分享未开启视图同步（`share.Props == nil` 或 `!share.Props.ShareView`）
2. 当前访问者不是分享所有者

**disableView 的影响**：在 [pkg/filemanager/fs/dbfs/file.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/file.go#L202-L213)

```go
func (f *File) View() *types.ExplorerView {
    // 如果所有者禁用了视图同步
    owner := f.Owner()
    if owner != nil && owner.Settings != nil && owner.Settings.DisableViewSync {
        return nil
    }

    // 如果导航器禁用了视图同步（分享场景）
    userRoot := f.UserRoot()
    if userRoot == nil || userRoot.disableView {
        return nil
    }
    // ... 返回视图设置
}
```

### 5.4 分享导航器 Capability（能力）

位置：[pkg/filemanager/fs/dbfs/navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/navigator.go#L128-L137)

```go
boolset.Sets(map[NavigatorCapability]bool{
    NavigatorCapabilityDownloadFile:   true,  // 下载文件
    NavigatorCapabilityListChildren:   true,  // 列出子文件
    NavigatorCapabilityGenerateThumb:  true,  // 生成缩略图
    NavigatorCapabilityLockFile:       true,  // 锁定文件
    NavigatorCapabilityInfo:           true,  // 获取信息
    NavigatorCapabilityVersionControl: true,  // 版本控制
    NavigatorCapabilityEnterFolder:    true,  // 进入文件夹
    NavigatorCapabilityModifyProps:    true,  // 修改属性
}, shareNavigatorCapability)
```

**注意**：分享导航器**不具备**以下能力（与我的文件对比）：
- 创建文件（`NavigatorCapabilityCreateFile`）
- 重命名文件（`NavigatorCapabilityRenameFile`）
- 上传文件（`NavigatorCapabilityUploadFile`）
- 更新元数据（`NavigatorCapabilityUpdateMetadata`）
- 删除文件（`NavigatorCapabilityDeleteFile`）
- 软删除（`NavigatorCapabilitySoftDelete`）
- 恢复文件（`NavigatorCapabilityRestore`）
- 再分享（`NavigatorCapabilityShare`）

---

## 6. 下载权限边界

### 6.1 用户组级别的下载权限校验

位置：[pkg/filemanager/fs/dbfs/share_navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go#L159-L173)

```go
if n.user.ID != n.owner.ID && !n.user.Edges.Group.Permissions.Enabled(int(types.GroupPermissionShareDownload)) {
    if inventory.IsAnonymousUser(n.user) {
        return nil, serializer.NewError(
            serializer.CodeAnonymouseAccessDenied,
            fmt.Sprintf("You don't have permission to access share links"),
            err,
        )
    }

    return nil, serializer.NewError(
        serializer.CodeNoPermissionErr,
        fmt.Sprintf("You don't have permission to access share links"),
        err,
    )
}
```

**校验逻辑**：
1. 访问者不是分享所有者
2. 访问者所在用户组未开启 `GroupPermissionShareDownload` 权限

满足以上两个条件时，直接拒绝访问。匿名用户返回 `CodeAnonymouseAccessDenied`，登录用户返回 `CodeNoPermissionErr`。

### 6.2 创建分享的用户组权限

位置：[service/share/manage.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/manage.go#L63-L100)

```go
func (service *ShareCreateService) Upsert(c *gin.Context, existed int) (string, error) {
    // 检查组权限：是否允许创建分享链接
    if !user.Edges.Group.Permissions.Enabled(int(types.GroupPermissionShare)) {
        return "", serializer.NewError(serializer.CodeGroupNotAllowed, "Group permission denied", nil)
    }
    // ...
}
```

### 6.3 下载前的钩子与计数

下载流程中，通过 `HookTypeBeforeDownload` 钩子触发下载计数：

1. 实体下载触发位置：[pkg/filemanager/manager/entity.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/manager/entity.go#L94-L97)
   ```go
   if err := m.fs.ExecuteNavigatorHooks(ctx, fs.HookTypeBeforeDownload, file); err != nil {
       m.l.Warning("Failed to execute navigator hooks: %s", err)
   }
   ```

2. 分享导航器执行钩子：[pkg/filemanager/fs/dbfs/share_navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go#L302-L308)
   - 调用 `shareClient.Downloaded()` 递增下载计数
   - 递减剩余下载次数（`remain_downloads`）

### 6.4 访问者视角的分享数据脱敏

位置：[service/explorer/response.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/explorer/response.go#L346-L383)

```go
func BuildShare(s *ent.Share, base *url.URL, hasher hashid.Encoder, requester *ent.User, owner *ent.User,
    name string, t types.FileType, unlocked bool, expired bool) *Share {
    
    res := Share{
        // 基础字段始终可见
        Name:              name,
        ID:                hashid.EncodeShareID(hasher, s.ID),
        Unlocked:          unlocked,
        Owner:             user.BuildUserRedacted(owner, redactLevel, hasher),
        Expired:           inventory.IsShareExpired(s) != nil || expired,
        Url:               BuildShareLink(s, hasher, base, unlocked),
        CreatedAt:         s.CreatedAt,
        Visited:           s.Views,
        SourceType:        util.ToPtr(t),
        PasswordProtected: s.Password != "",
    }

    // 解锁后才可见的字段
    if unlocked {
        res.RemainDownloads = s.RemainDownloads
        res.Downloaded = s.Downloads
        res.Expires = s.Expires
        res.Password = s.Password
        res.ShowReadMe = s.Props != nil && s.Props.ShowReadMe
        if t == types.FileTypeFile && s.Edges.File != nil {
            res.Size = s.Edges.File.Size
        }
    }

    // 仅所有者可见的字段
    if requester.ID == owner.ID {
        res.IsPrivate = s.Password != ""
        res.ShareView = s.Props != nil && s.Props.ShareView
    }

    return &res
}
```

---

## 7. 权限校验流程图

```
用户访问分享链接
        │
        ▼
┌─────────────────────────┐
│ 路由层：controllers/share │
│  GetShare()             │
└───────────┬─────────────┘
            │
            ▼
┌──────────────────────────────┐
│ 服务层：service/share/visit.go │
│  ShareInfoService.Get()      │
│  ├─ 1. 数据库查询 share      │
│  ├─ 2. IsValidShare() 校验   │
│  │   ├─ 时间过期?            │
│  │   ├─ 下载次数耗尽?        │
│  │   ├─ 所有者活跃?          │
│  │   └─ 源文件有效?          │
│  └─ 3. 密码校验             │
│      └─ 所有者免校验         │
└───────────┬──────────────────┘
            │
            ▼
┌──────────────────────────────────────┐
│ 文件系统层：share_navigator.go        │
│ shareNavigator.Root()                │
│  ├─ 1. IsValidShare() 校验           │
│  ├─ 2. 密码校验 (path.Password())    │
│  ├─ 3. 设置 disableView              │
│  │   └─ !ShareView && 非所有者 → true│
│  └─ 4. 组权限校验                    │
│      └─ GroupPermissionShareDownload │
└───────────┬──────────────────────────┘
            │
            ▼
┌──────────────────────────────────────┐
│ 下载时：HookTypeBeforeDownload       │
│  ├─ downloads++                      │
│  └─ remain_downloads-- (如果设置)    │
└──────────────────────────────────────┘
```

---

## 8. 总结：四层权限校验

| 层级 | 校验内容 | 位置 |
|------|---------|------|
| **第一层：分享有效性** | 过期时间、剩余下载次数、所有者状态、源文件状态 | `inventory.IsValidShare()` |
| **第二层：密码校验** | 分享密码匹配（所有者免校验） | `service/share/visit.go` 和 `share_navigator.Root()` |
| **第三层：用户组权限** | `GroupPermissionShareDownload`（下载/访问权限） | `share_navigator.Root()` |
| **第四层：导航器 Capability** | 文件系统操作能力边界（下载、列表、缩略图等） | `shareNavigatorCapability` |

---

## 9. 边界细节深度分析

### 9.1 密码错误时的锁定态展示

当访问者输入错误密码（或未输入密码）时，前端会收到 `unlocked=false` 的响应。此时 `BuildShare()` 对字段做了三层可见性控制：

位置：[service/explorer/response.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/explorer/response.go#L346-L414)

| 字段分类 | 字段名 | `unlocked=false`（锁定）时 | `unlocked=true`（解锁）时 |
|---------|--------|--------------------------|--------------------------|
| **始终可见** | `Name`、`ID`、`Unlocked`、`Owner`、`Expired`、`Url`、`CreatedAt`、`Visited`、`SourceType`、`PasswordProtected` | ✅ 可见 | ✅ 可见 |
| **解锁后可见** | `RemainDownloads`、`Downloaded`、`Expires`、`Password`、`ShowReadMe`、`Size` | ❌ 零值/空 | ✅ 真实值 |
| **仅所有者可见** | `IsPrivate`、`ShareView` | ❌ 零值（所有者除外） | ✅ 真实值（仅所有者） |

**URL 的差异化处理** 在 `BuildShareLink()` 中：
位置：[service/explorer/response.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/explorer/response.go#L500-L506)

```go
func BuildShareLink(s *ent.Share, hasher hashid.Encoder, base *url.URL, unlocked bool) string {
    shareId := hashid.EncodeShareID(hasher, s.ID)
    if unlocked {
        // 解锁状态：URL 中携带密码，形如 /s/{id}/{password}
        return routes.MasterShareUrl(base, shareId, s.Password).String()
    }
    // 锁定状态：URL 中不携带密码，形如 /s/{id}
    return routes.MasterShareUrl(base, shareId, "").String()
}
```

**锁定态前端表现总结**：
- 可以看到分享的文件名、所有者昵称、浏览次数
- 知道这个分享是有密码保护的（`PasswordProtected=true`）
- 不知道剩余下载次数、过期时间、文件大小
- 返回的分享 URL 不包含密码片段，需要用户手动输入密码后才能访问内容

---

### 9.2 所有者查看来源路径时的密码要求

所有者通过 `owner_extended=true` 参数获取分享对应的源文件路径（`SourceUri`）时，存在一个隐蔽的密码校验边界：

位置：[service/share/visit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/visit.go#L93-L109)

```go
if s.OwnerExtended && share.Edges.User.ID == u.ID {
    m := manager.NewFileManager(dep, u)
    defer m.Recycle()

    // 关键：用请求中传入的 s.Password 构造 share URI
    shareUri, err := fs.NewUriFromString(fs.NewShareUri(res.ID, s.Password))
    if err != nil {
        return nil, serializer.NewError(serializer.CodeInternalSetting, "Invalid share url", err)
    }

    // 通过 FileManager.Get → share_navigator.Root() 路径访问
    root, err := m.Get(c, shareUri)
    if err != nil {
        return nil, serializer.NewError(serializer.CodeNotFound, "File not found", err)
    }

    res.SourceUri = root.Uri(true).String()
}
```

**两层密码校验的差异**：

| 校验位置 | 所有者是否豁免 | 代码 |
|---------|--------------|------|
| `ShareInfoService.Get()` 第 85 行 | ✅ 所有者豁免，`unlocked` 始终为 true | `share.Edges.User.ID != u.ID` 作为判断条件之一 |
| `share_navigator.Root()` 第 130 行 | ❌ **所有者也需要密码** | `share.Password != "" && share.Password != path.Password()` |

**问题场景**：
如果分享设置了密码，但所有者请求 `owner_extended=true` 时没有在请求参数里传 `password`，则：
1. 第一层校验通过，`unlocked=true`，所有者能看到分享详情
2. 但构造 `shareUri` 时 `s.Password` 为空，URI 中不包含密码
3. 进入 `share_navigator.Root()` 时密码校验失败，返回 `ErrShareIncorrectPassword`
4. 最终接口返回 `"File not found"`（第 105 行被包装为 `CodeNotFound`）

**URI 密码注入逻辑** 在 [pkg/filemanager/fs/uri.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/uri.go#L357-L362)：

```go
func NewShareUri(id, password string) string {
    if password != "" {
        // cloudreve://{id}:{password}@share
        return fmt.Sprintf("%s://%s:%s@%s", constants.CloudreveScheme, id, password, constants.FileSystemShare)
    }
    // cloudreve://{id}@share  （无密码）
    return fmt.Sprintf("%s://%s@%s", constants.CloudreveScheme, id, constants.FileSystemShare)
}
```

---

### 9.3 失效判断（IsValidShare）和 过期判断（IsShareExpired）的区别

两个函数的包含关系：`IsValidShare = IsShareExpired + 所有者状态检查 + 源文件状态检查`

位置：[inventory/share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/inventory/share.go#L227-L258)

#### 函数定义对比

```go
// IsShareExpired：仅检查分享自身的两个过期属性
func IsShareExpired(share *ent.Share) error {
    if (share.Expires != nil && share.Expires.Before(time.Now())) ||
        (share.RemainDownloads != nil && *share.RemainDownloads <= 0) {
        return ErrShareLinkExpired   // "share link expired"
    }
    return nil
}

// IsValidShare：全面检查分享是否可访问
func IsValidShare(share *ent.Share) error {
    // 步骤 1：先检查过期
    if err := IsShareExpired(share); err != nil {
        return err                  // ErrShareLinkExpired
    }
    // 步骤 2：检查所有者状态
    owner, err := share.Edges.UserOrErr()
    if err != nil || owner.Status != user.StatusActive {
        return ErrOwnerInactive      // "owner is inactive"
    }
    // 步骤 3：检查源文件有效性
    file, err := share.Edges.FileOrErr()
    if err != nil || file.FileChildren == 0 || file.OwnerID != owner.ID {
        return ErrSourceFileInvalid  // "source file is deleted"
    }
    return nil
}
```

#### 不同场景的使用差异

| 调用位置 | 使用的函数 | 后果 |
|---------|----------|------|
| `ShareInfoService.Get()` 获取单个分享 | `IsValidShare` | 任何原因导致分享无效，直接返回 404 `"Share link expired"` |
| `share_navigator.Root()` 文件系统导航 | `IsValidShare` | 任何原因导致分享无效，返回 `ErrShareNotFound` |
| `BuildShare()` 构建响应的 `Expired` 字段 | `IsShareExpired \|\| expired` | 仅用于前端展示"已过期"标识，不会阻断访问 |
| `BuildListShareResponse()` 列表中的 `expired` 参数 | `IsValidShare`（结果传入 BuildShare 的 expired 参数） | 列表中分享的过期标识基于完整有效性 |

#### 设计意图分析

1. **访问拦截用 IsValidShare**：获取分享内容、浏览文件系统等真正需要访问分享资源的操作，必须通过完整校验 —— 源文件被删、所有者被封号都应该阻断访问

2. **展示标识分层处理**：
   - `Expired` 字段在 `BuildShare()` 中同时看 `IsShareExpired()` 和传入的 `expired` 参数
   - 在单分享查询时（`ShareInfoService.Get()`）传入 `expired=false`，所以 `Expired` 仅反映时间/下载次数耗尽
   - 在列表查询时（`BuildListShareResponse()`）传入 `IsValidShare()` 的结果，所以列表中的 `Expired` 包含所有失效原因

3. **错误码统一但语义不同**：`ShareInfoService.Get()` 中 `IsValidShare` 失败后统一返回 `"Share link expired"`（第 76 行），无论实际是时间过期、所有者被封还是源文件被删，对外都表现为"链接过期"，避免泄露内部状态信息
