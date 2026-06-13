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
    data := &ogData{
        SiteName:    siteBasic.Name,
        Title:       siteBasic.Name,          // 默认：站点名
        Description: siteBasic.Description,    // 默认：站点描述
        ShareURL:    routes.MasterShareUrl(base, id, password).String(), // URL 中包含从路径提取的密码
        RedirectURL: routes.MasterShareLongUrl(id, password).String(),
        ImageURL:    pwa.MediumIcon,           // 默认：PWA 图标
    }

    shareID, err := dep.HashIDEncoder().Decode(id, hashid.ShareID)
    if err != nil {
        data.Description = ogStatusInvalidLink  // "Invalid Link" — ID 无法解码
        return renderOGHTML(data)
    }

    // 关键点：loadShareForOG → ShareInfoService.Get()
    // 密码错误不会返回 error，只会 unlocked=false
    // 只有分享过期、源文件失效、所有者被封、ID 不存在才会返回 error
    shareInfo, err := loadShareForOG(c, shareID, password)
    if err != nil {
        // ❌ 错误分支：仅针对 ID 无效 + 分享失效
        var appErr serializer.AppError
        if errors.As(err, &appErr) {
            data.Description = appErr.Msg  // 通常是 "Share link expired"
        } else {
            data.Description = ogStatusInvalidLink  // "Invalid Link"
        }
        return renderOGHTML(data)
    }

    // ✅ 正常分支：包括密码错误的锁定态（unlocked=false）也会走这里！
    data.Title = shareInfo.Name  // 标题始终是文件名（无论锁定与否）

    if shareInfo.SourceType != nil && *shareInfo.SourceType == types.FileTypeFolder {
        // 文件夹（无论是否锁定）：Description = "Folder"
        data.Description = "Folder"
    } else if shareInfo.Unlocked {
        // 🔓 解锁态文件：Description = 文件大小 + 真实缩略图
        data.Description = formatFileSize(shareInfo.Size)
        thumbnail, err := loadShareThumbnail(c, id, password, shareInfo)
        if err == nil {
            data.ImageURL = thumbnail  // 替换为真实文件缩略图
        }
    }
    // 🔒 锁定态文件：Description 保持为默认的站点描述，不显示大小，缩略图仍是 PWA 图标

    // 无论锁定与否，末尾都拼接所有者昵称
    data.Description += " · " + shareInfo.Owner.Nickname
    return renderOGHTML(data)
}
```

#### OG 预览四场景对比

| 场景 | Title | Description | ImageURL |
|------|-------|-------------|----------|
| **锁定态文件**（密码错误/未传） | 文件名 | `站点默认描述 · 所有者昵称` | PWA 图标 |
| **解锁态文件**（密码正确） | 文件名 | `12.34 MB · 所有者昵称` | 文件缩略图 |
| **文件夹**（无论是否锁定） | 文件夹名 | `Folder · 所有者昵称` | PWA 图标 |
| **分享失效/过期** | 站点名 | `"Share link expired"` 或 `"Invalid Link"` | PWA 图标 |

**之前的理解偏差纠正**：密码错误时**不会**进入错误分支展示错误描述。错误分支只处理「ID 解码失败 + 分享过期/失效」。密码错误时 OG 页面仍能看到文件名和所有者昵称，只是文件大小隐藏、用站点图标代替缩略图。

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

#### 端到端完整流程梳理：从文件管理入口到 File not found 错误

当所有者请求 `GET /share/:id?owner_extended=true` 且未传密码时，整个流程从 ShareInfoService 开始，经过文件管理系统的 7 层调用，最终密码错误被包装成 404。以下是逐层的详细状态和代码位置：

```
======================================================================
STEP 0: 入口 ShareInfoService.Get()  [service/share/visit.go:60-112]
======================================================================
上下文：
  - 当前用户 u = 分享所有者（已通过认证）
  - 请求参数：id = "<hashid>", owner_extended = true, password = ""（空）
  - 分享记录：已从数据库查到，share.Password = "abc123"

执行流程：
  1. 第 75 行：inventory.IsValidShare(share) → ✅ 通过
     （检查时间过期、下载次数、所有者状态、源文件有效性）

  2. 第 83-87 行：密码校验（服务层）
     if s.Password == "" || ...  ← s.Password = ""
        && share.Edges.User.ID != u.ID  ← false（所有者本人）
     → 条件不满足，不进密码校验分支
     → unlocked = true （所有者豁免）

  3. 第 90 行：BuildShare(share, ..., unlocked=true, expired=false)
     → 返回解锁态详情，res.Password = "abc123"（真实密码可见）

  4. 第 93 行：进入 OwnerExtended 分支
     ┌───────────────────────────────────────────────────────────────┐
     │ 关键点：虽然第 3 步 res.Password 已有真实密码，               │
     │ 但第 98 行用的是请求参数 s.Password（空），不是 res.Password！│
     └───────────────────────────────────────────────────────────────┘

======================================================================
STEP 1: 构造 Share URI  [service/share/visit.go:97-101]
======================================================================
代码：
  97: shareUri, err := fs.NewUriFromString(fs.NewShareUri(res.ID, s.Password))
  98:                               ↑ res.ID = "<hashid>", s.Password = ""

调用 fs.NewShareUri(id, "")  [pkg/filemanager/fs/uri.go:357-362]:
  func NewShareUri(id, password string) string {
      if password != "" {
          // cloudreve://{id}:{password}@share
          return fmt.Sprintf("%s://%s:%s@%s", ...)
      }
      // 无密码分支 ← 走这里！
      return fmt.Sprintf("%s://%s@%s",
          constants.CloudreveScheme, id, constants.FileSystemShare)
  }

结果：
  uriStr = "cloudreve://<hashid>@share"
  ↳ URL 结构：scheme="cloudreve", userinfo="<hashid>", host="share"
  ↳ userinfo 中只有 username，没有 password
  ↳ 后续 path.Password() 将返回 ""

调用 fs.NewUriFromString(uriStr)  [pkg/filemanager/fs/uri.go:44-59]:
  → 解析成功，返回 *URI 对象

======================================================================
STEP 2: 创建 FileManager  [service/share/visit.go:94]
======================================================================
代码：
  94: m := manager.NewFileManager(dep, u)

调用 NewFileManager  [pkg/filemanager/manager/manager.go:152-171]:
  → 创建 DBFS 文件系统，绑定当前用户 u（所有者）
  → 内部 fs 字段 = dbfs.NewDatabaseFS(u, ...)

DBFS 初始化  [pkg/filemanager/fs/dbfs/dbfs.go:45-66]:
  type DBFS struct {
      user         *ent.User     ← = 所有者 u
      navigators   map[string]Navigator
      fileClient   inventory.FileClient
      shareClient  inventory.ShareClient  ← 用于后续查询分享
      ...
  }

======================================================================
STEP 3: 调用 manager.Get()  [service/share/visit.go:103]
======================================================================
代码：
  103: root, err := m.Get(c, shareUri)
       ↓
调用 manager.operation.Get()  [pkg/filemanager/manager/operation.go:51-53]:
  func (m *manager) Get(ctx context.Context, path *fs.URI, opts ...fs.Option) (fs.File, error) {
      return m.fs.Get(ctx, path, opts...)  ← 直接转发给 DBFS.Get
  }

======================================================================
STEP 4: DBFS.Get()  [pkg/filemanager/fs/dbfs/dbfs.go:371-407]
======================================================================
代码：
  func (f *DBFS) Get(ctx context.Context, path *fs.URI, opts ...fs.Option) (fs.File, error) {
      o := newDbfsOption()
      for _, opt := range opts {
          o.apply(opt)
      }

      // 获取对应导航器（根据 URI 类型选择不同的 Navigator）
      navigator, err := f.getNavigator(ctx, path, o.requiredCapabilities...)
      if err != nil {
          return nil, err  ← 这里不会失败
      }

      // ... 设置 context ...

      // 🔴 关键点：通过导航器获取文件
      target, err := f.getFileByPath(ctx, navigator, path)
      if err != nil {
          // 🔴 第一次包装：增加上下文前缀
          return nil, fmt.Errorf("failed to get target file: %w", err)
      }
      // ...
  }

调用 f.getNavigator()  [pkg/filemanager/fs/dbfs/dbfs.go:266-303]:
  根据 URI 的 host 字段判断类型：
  - host == "share" → 创建 shareNavigator
    return newShareNavigator(path, f.user, f.shareClient, f.l, f.hasher), nil

shareNavigator 初始化  [pkg/filemanager/fs/dbfs/share_navigator.go:98-111]:
  type shareNavigator struct {
      share       *ent.Share     ← 初始为 nil，Root() 时才查询
      sharePath   *fs.URI        ← = "cloudreve://<hashid>@share"
      user        *ent.User      ← = 所有者 u
      shareClient inventory.ShareClient
      hasher      hashid.Encoder
      l           logging.Logger
      shareRoot   fs.File        ← 初始为 nil，Root() 时才设置
  }

======================================================================
STEP 5: DBFS.getFileByPath()  [pkg/filemanager/fs/dbfs/dbfs.go:684-701]
======================================================================
代码：
  func (f *DBFS) getFileByPath(ctx context.Context, navigator Navigator, path *fs.URI) (*File, error) {
      defer duration.ToLog(ctx, f.l, "Get object by path from dbfs", "path", path)()
      file, err := navigator.To(ctx, path)  ← 🔴 调用 shareNavigator.To
      if err != nil {
          return file, err  ← 透传错误，不包装
      }
      return file, nil
  }

======================================================================
STEP 6: shareNavigator.To()  [pkg/filemanager/fs/dbfs/share_navigator.go:181-218]
======================================================================
代码：
  func (n *shareNavigator) To(ctx context.Context, path *fs.URI) (fs.File, error) {
      var (
          currentFsFile = n.shareRoot  ← 初始为 nil
          paths         = lo.DropWhile(lo.WithoutEmpty(strings.Split(path.Path(), "/")),
                              func(s string) bool { return s == "/" || s == "" })
      )

      // 🔴 关键点：如果 shareRoot 为空，先调用 Root() 初始化
      if currentFsFile == nil {
          shareRoot, err := n.Root(ctx, path)  ← 🔴 进入密码校验
          if err != nil {
              return nil, err  ← 透传错误
          }
          currentFsFile = shareRoot
          // ... 缓存 shareRoot ...
      }

      // ... 遍历路径查找文件（如果是文件夹内的文件）...
      // 本场景中 path 就是根路径，直接返回 shareRoot
  }

======================================================================
STEP 7: shareNavigator.Root() - 密码校验失败点  [share_navigator.go:114-179]
======================================================================
代码：
  func (n *shareNavigator) Root(ctx context.Context, path *fs.URI) (fs.File, error) {
      // 从 URI 中提取 shareID 并解码
      shareID := path.Name()  ← = "<hashid>"
      id, err := n.hasher.Decode(shareID, hashid.ShareID)  ← 解码为数据库 ID
      if err != nil {
          return nil, ErrShareNotFound
      }

      // 查询分享记录（第二次查询数据库！第一次在 visit.go 第 67 行）
      share, err := n.shareClient.GetByID(ctx, id, share.DefaultPreloads...)
      if err != nil {
          return nil, ErrShareNotFound
      }

      // 🔴 第一次校验：IsValidShare（分享本身有效性）
      if err := inventory.IsValidShare(share); err != nil {
          return nil, ErrShareNotFound
      }
      → ✅ 通过（时间、下载次数、所有者、源文件都没问题）

      // 🔴 🔴 🔴 第二次校验：密码校验（所有者也不豁免！）
      // 代码第 130-131 行：
      if share.Password != "" && share.Password != path.Password() {
          //   share.Password    = "abc123"  （数据库中的真实密码）
          //   path.Password()   = ""        （URI 中没有密码，来自 s.Password = ""）
          //   条件成立！密码不匹配！
          return nil, ErrShareIncorrectPassword
          //   ↳ Code:  40069 CodeIncorrectPassword
          //   ↳ Msg:   "Incorrect share password"
      }

      // （不会执行到这里）
      ...
  }

⚠️  关键点：
  - 这里完全没有检查 n.user（当前用户）是否是所有者
  - 即使 n.user == share.Edges.User（所有者本人），只要 URI 中没带密码，就会失败
  - 而 visit.go 第 85 行的服务层密码校验是豁免所有者的
  - 两层校验逻辑不一致！

======================================================================
STEP 8: 错误向上传递与包装
======================================================================

  层级 7: shareNavigator.Root()
    → 返回 ErrShareIncorrectPassword (40069, "Incorrect share password")

  层级 6: shareNavigator.To() 第 185 行
    → return nil, err  （透传，不包装）

  层级 5: DBFS.getFileByPath() 第 700 行
    → return file, err  （透传，不包装）

  层级 4: DBFS.Get() 第 406 行  ← 🔴 第一次包装
    → return nil, fmt.Errorf("failed to get target file: %w", err)
    现在错误链：
      外层："failed to get target file"
      内层：ErrShareIncorrectPassword (40069, "Incorrect share password")

  层级 3: manager.operation.Get() 第 52 行
    → return m.fs.Get(ctx, path, opts...)  （透传，不包装）

  层级 2: ShareInfoService.Get() 第 103 行
    → root, err := m.Get(c, shareUri)  （收到第一次包装后的错误）

  层级 1: ShareInfoService.Get() 第 105 行  ← 🔴 第二次包装（完全掩盖）
    → return nil, serializer.NewError(serializer.CodeNotFound, "File not found", err)
    现在：
      Code:  404 CodeNotFound  （错误码被篡改！）
      Msg:   "File not found"   （错误信息被覆盖！）
      Error: 第一次包装后的错误链（作为内部 cause，前端不可见）

======================================================================
STEP 9: 最终响应
======================================================================

  HTTP/1.1 200 OK
  Content-Type: application/json

  {
    "code": 404,
    "msg": "File not found",
    "data": null
  }

  ⚠️ 用户感知：
    - 第一层分享详情显示 Unlocked=true，Password="abc123"（密码可见）
    - 但 SourceUri 缺失，提示"文件找不到"
    - 完全意识不到是因为自己没传 password 参数导致的
```

**关键代码位置汇总表**：

| 步骤 | 操作 | 文件 | 行号 |
|-----|------|------|------|
| STEP 0 | ShareInfoService 入口，所有者密码豁免 | [visit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/visit.go#L83-L87) | 83-87 |
| STEP 1 | 构造无密码的 Share URI | [uri.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/uri.go#L357-L362) | 357-362 |
| STEP 2 | 创建 FileManager + DBFS | [manager.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/manager/manager.go#L152-L171) | 152-171 |
| STEP 3 | manager.Get 转发 | [operation.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/manager/operation.go#L51-L53) | 51-53 |
| STEP 4 | DBFS.Get 调用 getFileByPath | [dbfs.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L371-L407) | 371-407 |
| STEP 5 | getFileByPath 调用 navigator.To | [dbfs.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L684-L701) | 684-701 |
| STEP 6 | shareNavigator.To 触发 Root 初始化 | [share_navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go#L181-L218) | 181-218 |
| STEP 7 | shareNavigator.Root 密码校验失败（所有者不豁免） | [share_navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go#L130-L131) | 130-131 |
| STEP 8 | DBFS.Get 第一次包装错误 | [dbfs.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L404-L406) | 406 |
| STEP 8 | visit.go 第二次包装（掩盖为 404） | [visit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/visit.go#L103-L105) | 105 |

**流程中的三处设计缺陷**：

1. **不一致的密码豁免**：第 85 行服务层豁免所有者密码，第 130 行文件系统层不豁免
2. **密码来源不一致**：第 90 行 `res.Password` 已有真实密码，但第 98 行仍用空的 `s.Password` 构造 URI
3. **过度的错误掩盖**：密码错误应该返回 40069 提示用户输入密码，而不是 404 说文件不存在

---

#### 深度分析：从密码错误到 "File not found" 的完整包装链路

当 `owner_extended=true` 遇到密码错误时，错误信息经过 **5 层调用栈、2 次包装**，最终的密码错误信息被完全掩盖。以下是逐层追踪的细节：

```
所有者请求 GET /share/:id?owner_extended=true  (不传 password)
  │
  │  第一层：ShareInfoService.Get()
  │  ├─ 第 75 行：IsValidShare() → ✅ 通过
  │  ├─ 第 85 行：密码校验 → 所有者豁免 → unlocked=true
  │  ├─ 第 90 行：BuildShare() → ✅ 返回解锁态详情（含真实密码）
  │  │
  │  └─ 第 93 行：OwnerExtended 分支
  │     │
  │     ├─ 98: NewShareUri(res.ID, s.Password) → "cloudreve://<hashid>@share"  ← 无密码
  │     │
  │     └─ 103: m.Get(c, shareUri)
  │             │
  │             │  第二层：manager.operation.Get()  [operation.go:51-53]
  │             │  └─ return m.fs.Get(ctx, path, opts...)  → 直接透传
  │             │             │
  │             │             │  第三层：DBFS.Get()  [dbfs.go:371-407]
  │             │             │  ├─ 378: f.getNavigator(...) → 创建 shareNavigator ✅
  │             │             │  ├─ 404: f.getFileByPath(ctx, navigator, path)
  │             │             │  │   │
  │             │             │  │   │  第四层：DBFS.getFileByPath()  [dbfs.go:684-701]
  │             │             │  │   └─ 685: file, err := navigator.To(ctx, path)
  │             │             │  │            │
  │             │             │  │            │  第五层：share_navigator.To()  [share_navigator.go:181-218]
  │             │             │  │            ├─ 182-189: shareRoot == nil → 调用 Root()
  │             │             │  │            │   │
  │             │             │  │            │   │  第六层：share_navigator.Root()  [share_navigator.go:114-179]
  │             │             │  │            │   ├─ 118: GetByHashID() → ✅ 查到 share
  │             │             │  │            │   ├─ 123: IsValidShare() → ✅ 通过
  │             │             │  │            │   │
  │             │             │  │            │   ├─ 🔴 130-131: 密码校验失败
  │             │             │  │            │   │       share.Password = "abc123"
  │             │             │  │            │   │       path.Password() = ""
  │             │             │  │            │   │       不匹配！
  │             │             │  │            │   │
  │             │             │  │            │   └─ 返回 ErrShareIncorrectPassword
  │             │             │  │            │       Code:  40069 CodeIncorrectPassword
  │             │             │  │            │       Msg:   "Incorrect share password"
  │             │             │  │            │       Error: nil
  │             │             │  │            │
  │             │             │  │            └─ 185: return nil, err  → 直接返回，不包装
  │             │             │  │
  │             │             │  └─ 700: return file, err  → 直接返回，不包装
  │             │             │
  │             │             ├─ 🔴 第一次包装（增加上下文前缀）：
  │             │             │  406: return nil, fmt.Errorf("failed to get target file: %w", err)
  │             │             │       外层："failed to get target file"
  │             │             │       内层：ErrShareIncorrectPassword (40069, "Incorrect share password")
  │             │             │
  │             │             └─ 返回包装后的错误 ↑
  │             │
  │             └─ 返回透传的错误 ↑
  │
  └─ 🔴 第二次包装（完全掩盖原始错误）：
     105: return nil, serializer.NewError(serializer.CodeNotFound, "File not found", err)
          Code:  404 CodeNotFound
          Msg:   "File not found"
          Error: 第一次包装后的错误链（作为内部 cause）

  └─ 🔚 最终接口响应：
          { "code": 404, "msg": "File not found" }
          ⚠️ 用户完全看不到任何与密码相关的提示！
```

**每一层错误处理的代码位置**：

| 层级 | 文件 | 行号 | 处理方式 |
|-----|------|------|---------|
| 1 错误原点 | [share_navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go#L130-L131) | 130-131 | 返回 `ErrShareIncorrectPassword` (40069) |
| 2 透传 | [share_navigator.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/share_navigator.go#L184-L185) | 184-185 | `return nil, err` 不包装 |
| 3 透传 | [dbfs.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L684-L700) | 685,700 | `return file, err` 不包装 |
| 4 第一次包装 | [dbfs.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/fs/dbfs/dbfs.go#L404-L406) | 406 | `fmt.Errorf("failed to get target file: %w", err)` |
| 5 透传 | [operation.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/pkg/filemanager/manager/operation.go#L51-L53) | 52 | `return m.fs.Get(ctx, path, opts...)` 不包装 |
| 6 第二次包装 | [visit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/visit.go#L103-L105) | 105 | `serializer.NewError(CodeNotFound, "File not found", err)` → 错误码改为 404 |

**关键问题点**：
1. **错误码被篡改**：`CodeIncorrectPassword (40069)` → `CodeNotFound (404)`，前端无法区分是真的文件不存在还是密码错误
2. **错误信息被覆盖**：`"Incorrect share password"` → `"File not found"`，用户看不到密码错误提示
3. **内部 cause 无用**：虽然用 `%w` 保留了错误链，但 `serializer.NewError` 只会把 `Msg` 字段返回给前端，内部错误在响应中不可见
4. **不一致的密码豁免**：同一请求中第 85 行所有者免密，第 130 行又要求密码

---

### 9.3 失效判断（IsValidShare）和 过期判断（IsShareExpired）的区别，以及 Expired 标记的实际可达性

两个函数的包含关系：`IsValidShare = IsShareExpired + 所有者状态检查 + 源文件状态检查`

位置：[inventory/share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/inventory/share.go#L227-L258)

#### 函数定义对比

```go
// IsShareExpired：仅检查分享自身的两个过期属性（纯数据判断，不涉及外部关联）
func IsShareExpired(share *ent.Share) error {
    if (share.Expires != nil && share.Expires.Before(time.Now())) ||
        (share.RemainDownloads != nil && *share.RemainDownloads <= 0) {
        return ErrShareLinkExpired   // "share link expired"
    }
    return nil
}

// IsValidShare：全面检查分享是否可访问（含外部关联对象）
func IsValidShare(share *ent.Share) error {
    // 步骤 1：先检查数据层面过期
    if err := IsShareExpired(share); err != nil {
        return err                  // ErrShareLinkExpired
    }
    // 步骤 2：检查关联所有者状态
    owner, err := share.Edges.UserOrErr()
    if err != nil || owner.Status != user.StatusActive {
        return ErrOwnerInactive      // "owner is inactive"
    }
    // 步骤 3：检查关联源文件状态
    file, err := share.Edges.FileOrErr()
    if err != nil || file.FileChildren == 0 || file.OwnerID != owner.ID {
        return ErrSourceFileInvalid  // "source file is deleted"
    }
    return nil
}
```

#### 关键代码路径：单个分享详情中 `Expired` 字段实际不可达

在 `BuildShare()` 中，`Expired` 字段定义为：
```go
Expired: inventory.IsShareExpired(s) != nil || expired
```

但顺着单个分享详情接口的代码路径 [service/share/visit.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/visit.go#L60-L112) 追溯：

```
GET /share/:id （单个分享详情）
    │
    ├─ 第 67 行：shareClient.GetByID() → 查到 share 记录
    │
    ├─ 第 75 行：IsValidShare(share)
    │   │
    │   └─ 内部先调用 IsShareExpired()
    │       ├─ 如果 IsShareExpired 返回错误（时间过期或下载次数耗尽）
    │       │   └─ IsValidShare 立即返回 ErrShareLinkExpired
    │       │       └─ 第 76 行：return CodeNotFound, "Share link expired"
    │       │           ⚠️ BuildShare() 根本不会被调用！接口直接返回 404
    │       │
    │       └─ 如果 IsShareExpired 通过（返回 nil）
    │           └─ 继续检查所有者状态、源文件状态...
    │
    ├─ 第 90 行：BuildShare(share, ..., unlocked, expired=false)
    │   │
    │   └─ 此时 IsShareExpired(s) 已经为 nil（否则走不到这里）
    │      expired 参数为 false
    │      → Expired = false || false = false  恒为 false
    │
    └─ 🔚 结论：在单个分享详情接口的 200 成功响应中，
               Expired 字段永远是 false！没有任何情况能让它为 true。
```

#### 不同场景的使用差异（修正版）

| 场景 | 判定函数 | 结果 | 是否能让前端看到 `Expired=true` |
|------|---------|------|------------------------------|
| **单个分享详情 GET /share/:id** | `IsValidShare`（拦截式） | 失败 → 直接 404 | ❌ 不可达。能成功返回时 Expired 恒为 false |
| **分享文件系统导航 share_navigator.Root()** | `IsValidShare`（拦截式） | 失败 → `ErrShareNotFound` | ❌ 同上 |
| **我的分享列表 / 用户主页分享列表** | `IsValidShare`（仅打标） | 结果作为 `expired` 参数传入 BuildShare | ✅ 可达。列表中过期/失效分享仍会展示，只是 Expired=true |
| **分享列表内部 `BuildShare` 计算** | `IsShareExpired \|\| expired` | 作为最终展示值 | ✅ 列表场景中两个条件任一成立即为 true |

#### 深度分析：为什么 Expired 字段在成功响应中几乎不会变为 true

全局搜索 `explorer.BuildShare(` 的所有调用点，仅有 **3 处**：

| # | 调用位置 | `expired` 参数值 | 说明 |
|---|---------|-----------------|------|
| 1 | [service/share/visit.go:90-91](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/visit.go#L90-L91) | `false` 硬编码 | 单个分享详情接口 |
| 2 | [service/share/response.go:39-40](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/response.go#L39-L40) | 传入的 `expired` 变量（来自 `IsValidShare` 结果） | 我的分享列表 |
| 3 | [service/share/response.go:39-40](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/response.go#L39-L40) | 传入的 `expired` 变量（来自 `IsValidShare` 结果） | 用户主页分享列表 |

**关键代码路径证明（单分享详情接口）**：

```
ShareInfoService.Get()  [visit.go:60-112]
  │
  ├─ 67: share, err := shareClient.GetByID(ctx, ...)  → 数据库查到 share 记录
  │    ↓
  ├─ 75: if err := inventory.IsValidShare(share); err != nil
  │    │
  │    └─ IsValidShare() 内部  [inventory/share.go:289-306]
  │        │
  │        ├─ if err := IsShareExpired(share); err != nil
  │        │   └─ 如果 IsShareExpired 返回错误
  │        │      └─ IsValidShare 立即 return err
  │        │          ↓
  │        └─ 76: return CodeNotFound, "Share link expired"
  │            ⚠️  BuildShare() 根本不会被调用！接口直接返回 404
  │
  └─ 90: 能走到这里 → IsShareExpired() 已为 nil（否则已经 404 了）
       BuildShare(share, ..., unlocked, false)
         │
         └─ Expired = (IsShareExpired(s) != nil) || false
                = (nil != nil) || false
                = false
```

**唯一可达路径：分享列表场景**：
```
BuildListShareResponse()  [response.go:21-36]
  │
  └─ for _, share := range res.Shares
         │
         ├─ expired := inventory.IsValidShare(share) != nil  → 不拦截，只打标
         │    ↓ 即使 true 也继续执行
         └─ BuildShare(share, ..., unlocked, expired)
                ↓
            Expired = (IsShareExpired(s) != nil) || expired
                    = 可能为 true！
```

**设计意图**：
- 单分享详情接口：失效的分享直接 404，不需要 `Expired` 标记
- 分享列表接口：失效的分享也要展示给用户（只是标记为灰色），所以需要 `Expired` 字段
- 这解释了为什么 `BuildShare` 同时接收 `IsShareExpired(s)` 和 `expired` 参数两个来源 — 列表场景下即使分享本身没过期（`IsShareExpired`=nil），如果所有者被封了（`IsValidShare` 失败），也应该标记为 `Expired=true`

列表场景的调用链：[service/share/response.go](file:///d:/fz/0601-1/solo-dogfeeding/code/43-Cloudreve/service/share/response.go#L21-L36)
```go
func BuildListShareResponse(res *inventory.ListShareResult, ...) *ListShareResponse {
    for _, share := range res.Shares {
        // 对每个分享调用 IsValidShare，结果转换为 bool 作为 expired 参数
        expired := inventory.IsValidShare(share) != nil
        // 不会因为 expired=true 就跳过，仍然加入返回列表
        infos = append(infos, *explorer.BuildShare(share, ..., unlocked, expired))
    }
}
```

#### 设计意图分析

1. **访问拦截用 IsValidShare（Fail-Closed）**：获取分享内容、浏览文件系统等真正需要访问资源的操作，必须通过完整校验 —— 源文件被删、所有者被封号都应该阻断访问。时间过期只是最表层的原因，其他失效原因同样不可访问。

2. **列表打标用 IsValidShare（Fail-Open 展示）**：在"我的分享"列表中，即使分享失效也要展示给用户看，只是标记为已过期。否则用户会疑惑"我创建的分享怎么不见了？"

3. **Expired 字段是展示专用**：它不承担访问控制功能（那是 `IsValidShare` 的事），只是告诉前端"这个分享在列表中是灰色失效状态"。单分享详情接口不需要这个标记，因为失效的分享在单查询时已被 404 拦截。

4. **错误码统一但语义不同**：`ShareInfoService.Get()` 中 `IsValidShare` 失败后统一返回 `"Share link expired"`（第 76 行），无论实际是时间过期、所有者被封还是源文件被删，对外都表现为"链接过期"，避免泄露「所有者被封号」「源文件被删除」等内部状态信息，减少信息泄露面。
