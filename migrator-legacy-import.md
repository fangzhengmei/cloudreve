# Cloudreve V3 → V4 旧数据迁移代码详解

## 1. 整体架构

迁移代码位于 [application/migrator](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator) 目录，职责是将 Cloudreve V3 的数据（基于 GORM + 原始 SQL 表）导入 V4 的 Ent ORM 数据库。

入口命令定义在 [cmd/migrate.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/cmd/migrate.go)，通过 `cloudreve migrate --v3-conf <path>` 触发。

### 1.1 迁移步骤流水线

定义在 [migrator.go#L44-L63](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L44-L63)，严格按序执行：

| Step 常量 | 值 | 说明 |
|---|---|---|
| `StepInitial` | 0 | 初始状态 |
| `StepSchema` | 1 | 创建 V4 表结构 |
| `StepSettings` | 2 | 迁移系统设置 |
| `StepNode` | 3 | 迁移节点 |
| `StepPolicy` | 4 | 迁移存储策略 |
| `StepGroup` | 5 | 迁移用户组 |
| `StepUser` | 6 | 迁移用户 |
| `StepFolders` | 7 | 迁移文件夹 |
| `StepFolderParent` | 8 | 补设文件夹父级关系 |
| `StepFile` | 9 | 迁移文件 |
| `StepShare` | 10 | 迁移分享 |
| `StepDirectLink` | 11 | 迁移直链 |
| `Step_CommunityPlaceholder1` | 12 | 占位 |
| `Step_CommunityPlaceholder2` | 13 | 占位 |
| `StepAvatar` | 14 | 迁移头像文件 |
| `StepWebdav` | 15 | 迁移 WebDAV 账户 |
| `StepCompleted` | 16 | 完成 |

---

## 2. 旧字段映射（V3 → V4）

### 2.1 User 用户

**V3 模型**：[model/user.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/user.go)

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID(int(u.ID))` | 原样保留 V3 ID |
| `Email` | `Email` | 直传 |
| `Nick` | `Nick` | 直传 |
| `Password` | `Password` | 直传 |
| `Status` (int) | `Status` (enum) | 枚举映射：`Active→StatusActive`, `NotActivicated→StatusInactive`, `Baned→StatusManualBanned`, `OveruseBaned→StatusSysBanned` |
| `GroupID` | `GroupID` | 直传 |
| `Storage` (uint64) | `Storage` (int64) | 类型转换 |
| `TwoFactor` | `TwoFactorSecret` | 非空时设置，字段名变更 |
| `Avatar` | `Avatar` | 非空时设置 |
| — | `Settings` | 新增，固定值 `{VersionRetention: true, VersionRetentionMax: 10}` |

**未迁移字段**：`Options`/`OptionsSerialized`（用户个性化配置如 `profile_off`, `preferred_theme`）、`Authn`（WebAuthn 凭据）、`PicInfo`。

### 2.2 Folder 文件夹

**V3 模型**：[model/folder.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/folder.go)

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID(int(f.ID))` | 原样保留 |
| `Name` | `Name` | 根目录时置空 `""` |
| `ParentID` (*uint) | `ParentID` | 分两步：StepFolders 时跳过，StepFolderParent 时补设 |
| `OwnerID` | `OwnerID` | 直传 |
| `CreatedAt`/`UpdatedAt` | `CreatedAt`/`UpdatedAt` | 直传 |
| — | `Type` | 固定为 `FileTypeFolder` |

**注意**：V4 中文件夹和文件统一为 `File` 实体，通过 `Type` 字段区分。`ParentID == nil` 表示根目录（Name 置空）；`ParentID == 0` 跳过（脏数据）。

### 2.3 File 文件

**V3 模型**：[model/file.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/file.go)

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID(int(f.ID) + LastFolderID)` | **ID 偏移**：文件 ID = V3 ID + 最大文件夹 ID，避免与文件夹 ID 冲突 |
| `Name` | `Name` | 如存在冲突重命名则用 `FileConflictRename` 映射的新名 |
| `SourceName` | → Entity `Source` | 拆分为 Entity 实体（版本实体 + 可能的缩略图实体） |
| `UserID` | `OwnerID` | 字段名变更 |
| `Size` | `Size` | 直传 |
| `FolderID` | `FileChildren` | V4 文件夹作为父级，字段名变更 |
| `PolicyID` | `StoragePoliciesID` | 字段名变更 |
| `Metadata` | → Entity | `thumb_status=exist` 时额外创建缩略图 Entity |
| — | `PrimaryEntity` | 主版本 Entity 的 ID |
| — | `Type` | 固定为 `FileTypeFile` |

**未迁移字段**：`PicInfo`、`UploadSessionID`、`Position`。

### 2.4 Policy 存储策略

**V3 模型**：[model/policy.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/policy.go)

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `Name` | `Name` | 直传 |
| `Type` | `Type` | 直传（字符串策略类型如 "local", "onedrive" 等） |
| `Server` | `Server` | 直传 |
| `BucketName` | `BucketName` | 直传 |
| `IsPrivate` | `IsPrivate` | 直传 |
| `BaseURL` | → `Settings.CustomProxy` + `Settings.ProxyServer` | 非空时设置自定义代理 |
| `AccessKey` | `AccessKey` | 直传 |
| `SecretKey` | `SecretKey` | 直传 |
| `MaxSize` | `MaxSize` | 类型转换 uint64→int64 |
| `DirNameRule` | `DirNameRule` | 直传（缺随机元素时强制覆盖默认值） |
| `FileNameRule` | `FileNameRule` | 直传（缺随机元素时强制覆盖默认值） |
| `Options` (JSON) | → `Settings` (PolicySetting) | 逐字段映射见下表 |

**PolicyOption → PolicySetting 映射**：

| V3 PolicyOption 字段 | V4 PolicySetting 字段 |
|---|---|
| `Token` | `Token` |
| `FileType` | `FileType` |
| `OauthRedirect` | `OauthRedirect` |
| `OdDriver` | `OdDriver` |
| `Region` | `Region` |
| `ServerSideEndpoint` | `ServerSideEndpoint` |
| `ChunkSize` | `ChunkSize` |
| `TPSLimit` | `TPSLimit` |
| `TPSLimitBurst` | `TPSLimitBurst` |
| `S3ForcePathStyle` | `S3ForcePathStyle` |
| `ThumbExts` | `ThumbExts` |
| `OdProxy` | → `CustomProxy=true` + `ProxyServer` |

**特殊逻辑**：
- OneDrive 策略自动设置 `ThumbSupportAllExts=true`
- COS/OSS/又拍云/七牛/远程策略根据类型硬编码 `ThumbExts`
- COS 策略强制 `ChunkSize=25MB`
- 缩略图代理设置从 V3 的 `thumb_proxy_enabled` + `thumb_proxy_policy` 系统设置读取
- 远程策略（`PolicyTypeRemote`）自动创建一个 Slave Node

**未迁移字段**：`AutoRename`、`IsOriginLinkEnable`、`OptionsSerialized.MimeType`、`OptionsSerialized.PlaceholderWithSize`。

### 2.5 Group 用户组

**V3 模型**：[model/group.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/group.go)

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `Name` | `Name` | 直传 |
| `Policies` (JSON) | `StoragePoliciesID` | 取过滤后的策略列表第一个 |
| `MaxStorage` | `MaxStorage` | uint64→int64 |
| `ShareEnabled` | → `Permissions[GroupPermissionShare]` | 布尔值转为权限位 |
| `WebDAVEnabled` | → `Permissions[GroupPermissionWebDAV]` | 布尔值转为权限位 |
| `SpeedLimit` | `SpeedLimit` | 直传 |
| `Options` (JSON) | → `Settings` (GroupSetting) | 见下表 |

**GroupOption → GroupSetting 映射**：

| V3 GroupOption 字段 | V4 GroupSetting 字段 |
|---|---|
| `CompressSize` | `CompressSize` |
| `DecompressSize` | `DecompressSize` |
| `Aria2Options` | `RemoteDownloadOptions` |
| `SourceBatchSize` | `SourceBatchSize` |
| `RedirectedSource` | `RedirectedSource` |
| `Aria2BatchSize` | `Aria2BatchSize` |
| — | `MaxWalkedFiles=100000` | 硬编码默认值 |
| — | `TrashRetention=7*24*3600` | 硬编码 7 天 |

**权限映射**：

| V3 来源 | V4 权限 |
|---|---|
| `group.ID == 1` | `GroupPermissionIsAdmin` |
| `group.ID == 3` | `GroupPermissionIsAnonymous` |
| `opts.ShareDownload` | `GroupPermissionShareDownload` |
| `group.WebDAVEnabled` | `GroupPermissionWebDAV` |
| `opts.ArchiveDownload` | `GroupPermissionArchiveDownload` |
| `opts.ArchiveTask` | `GroupPermissionArchiveTask` |
| `opts.WebDAVProxy` | `GroupPermissionWebDAVProxy` |
| `opts.Aria2` | `GroupPermissionRemoteDownload` |
| `opts.AdvanceDelete` | `GroupPermissionAdvanceDelete` |
| `group.ShareEnabled` | `GroupPermissionShare` |
| `opts.RedirectedSource` | `GroupPermissionRedirectedSource` |

**未迁移字段**：`GroupOption.OneTimeDownload`。

### 2.6 Node 节点

**V3 模型**：[model/node.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/node.go)

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `Type` (ModelType) | `Type` (node.Type) | `SlaveNodeType→TypeSlave`, `MasterNodeType→TypeMaster` |
| `Status` (NodeStatus) | `Status` | `NodeActive→StatusActive`, 其他→`StatusSuspended` |
| `Name` | `Name` | 直传 |
| `Server` | `Server` | 直传 |
| `SlaveKey` | `SlaveKey` | 直传 |
| `Aria2Enabled` | → `Capabilities[NodeCapabilityRemoteDownload]` | 布尔→能力位 |
| `Aria2Options` (JSON) | → `Settings.Aria2Setting` | 逐字段映射 |
| `Rank` | `Weight` | 字段名变更 |

**Aria2Option → Aria2Setting 映射**：

| V3 Aria2Option | V4 Aria2Setting |
|---|---|
| `Server` | `Server` |
| `Token` | `Token` |
| `Options` (JSON string) | `Options` (map[string]any) |
| `TempPath` | `TempPath` |

Master 节点额外赋予 `NodeCapabilityExtractArchive` + `NodeCapabilityCreateArchive` 能力。

**未迁移字段**：`MasterKey`、`Aria2Option.Interval`、`Aria2Option.Timeout`。

### 2.7 Setting 系统设置

**V3 模型**：[model/setting.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/setting.go)

迁移逻辑在 [settings.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/settings.go) 的 `migrators` 映射表中定义，分为三种处理：

1. **noopMigrator（丢弃）**：以下 V3 设置在 V4 中不再需要，直接跳过：
   - `siteKeywords`, `over_used_template`, `download_timeout`, `preview_timeout`, `doc_preview_timeout`
   - `slave_node_retry`, `slave_ping_interval`, `slave_recover_interval`, `slave_transfer_timeout`
   - `onedrive_monitor_timeout`, `onedrive_source_timeout`, `share_download_session_timeout`, `onedrive_callback_check`
   - `mail_activation_template`, `mail_reset_pwd_template`
   - `appid`, `appkey`（QQ 互联）
   - `wechat_*`（微信支付系列）
   - `hot_share_num`, `defaultTheme`, `theme_options`
   - `max_worker_num`, `max_parallel_transfer`, `secret_key`
   - `avatar_size_m`, `avatar_size_s`
   - `home_view_method`, `share_view_method`, `cron_recycle_upload_session`
   - `captcha_TCaptcha_*`, `initial_files`, `office_preview_service`
   - `phone_required`, `phone_enabled`
   - `custom_payment_*`

2. **字段名变更**：
   | V3 设置名 | V4 设置名 | 说明 |
   |---|---|---|
   | `thumb_file_suffix` | `thumb_entity_suffix` | 重命名 |
   | `wopi_session_timeout` | `viewer_session_timeout` | 重命名 |

3. **值转换**：
   | V3 设置名 | 转换逻辑 |
   |---|---|
   | `captcha_type` | `tcaptcha` → `normal`，其他值保留 |
   | `thumb_max_src_size` | 一对多：值同时写入 `thumb_music_cover_max_size`, `thumb_libreoffice_max_size`, `thumb_ffmpeg_max_size`, `thumb_vips_max_size`, `thumb_builtin_max_size` |

4. **直传**：未在 `migrators` 表中的设置，name/value 原样写入 V4。

5. **新增设置**：`hash_id_salt` 从 V3 配置文件 `System.HashIDSalt` 读取后插入。

### 2.8 Share 分享

**V3 模型**：[model/share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/share.go)

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `Password` | `Password` | 非空时设置 |
| `IsDir` | → `FileID` 偏移计算 | 文件时 `sourceId = SourceID + LastFolderID`，目录时 `sourceId = SourceID` |
| `UserID` | `UserID` | 直传 |
| `SourceID` | `FileID` | 根据 IsDir 决定是否加偏移 |
| `Views` | `Views` | 直传 |
| `Downloads` | `Downloads` | 直传 |
| `RemainDownloads` | `RemainDownloads` | ≥0 时设置 |
| `Expires` | `Expires` | 非空时设置 |

**未迁移字段**：`PreviewEnabled`、`SourceName`。

### 2.9 DirectLink 直链（SourceLink）

**V3 模型**：[model/source_link.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/source_link.go)

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `FileID` | `FileID` | 加 `LastFolderID` 偏移 |
| `Name` | `Name` | 直传 |
| `Downloads` | `Downloads` | 直传 |
| — | `Speed` | 固定为 0 |

### 2.10 WebDAV 账户

**V3 模型**：[model/webdav.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/webdav.go)

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `Name` | `Name` | 直传 |
| `Password` | `Password` | 直传 |
| `UserID` | `OwnerID` | 字段名变更 |
| `Root` | `URI` | 格式化为 `"cloudreve://my" + Root` |
| `Readonly` | → `Options[DavAccountReadOnly]` | 布尔→选项位 |
| `UseProxy` | → `Options[DavAccountProxy]` | 布尔→选项位 |
| — | `Props` | 空结构体 `DavAccountProps{}` |

### 2.11 Avatar 头像

头像迁移在 [avatars.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/avatars.go)，不涉及数据库实体，而是文件复制：

| V3 路径 | V4 路径 |
|---|---|
| `{avatar_path}/avatar_{uid}_2.png` | `{data_path}/avatar/avatar_{uid}.png` |

`avatar_path` 来自 V3 设置 `avatar_path`，通过 `util.RelativePath()` 解析。

---

## 3. Entity 实体创建

V4 引入了 `Entity` 概念，代表存储后端中的实际数据对象。每个 V3 文件在迁移时会创建 1~2 个 Entity：

### 3.1 insertEntity 函数

定义在 [file.go#L160-L189](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/file.go#L160-L189)，逻辑如下：

```
输入: source(存储路径), entityType, policyID, createdBy, size
entityKey = policyID + "+" + source

1. 查找 EntitySources 缓存中是否已存在同 key 的 Entity
   - 如果存在：更新该 Entity 的 reference_count +1，返回
   - 如果更新失败：降级为创建新 Entity
2. 创建新 Entity：
   - Source = source
   - Type = entityType
   - Size = size
   - StoragePolicyEntities = policyID
   - CreatedBy = createdBy
   - ReferenceCount = 1
3. 将新 Entity ID 写入 EntitySources 缓存
```

### 3.2 文件迁移时的 Entity 创建流程

对每个 V3 文件：

1. **缩略图 Entity**（条件创建）：
   - 条件：`metadata["thumb_status"] == "exist"`
   - Source = `SourceName + ThumbSuffix`
   - Type = `EntityTypeThumbnail`
   - Size：本地策略时尝试 `os.Stat` 获取文件大小，否则为 0

2. **版本 Entity**（必定创建）：
   - Source = `SourceName`
   - Type = `EntityTypeVersion`
   - Size = `f.Size`

3. **File 实体**：
   - `PrimaryEntity` = 版本 Entity ID
   - `AddEntities` = [版本 Entity, 缩略图 Entity(如有)]

---

## 4. 幂等处理

### 4.1 步骤级幂等

[migrator.go#L180-L309](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L180-L309) 中，`Migrate()` 方法通过 `if m.state.Step <= StepXxx` 检查确保：
- 已完成的步骤不会重复执行
- 当前步骤失败后重试时，从当前步骤继续

### 4.2 批次级幂等（offset 机制）

大数据量表（User、Folder、File、Share、DirectLink、Webdav）采用分批迁移 + offset 记录：
- 每批 1000 条，批次完成后 `saveState()` 持久化 offset
- 恢复时从上次保存的 offset 开始查询，跳过已处理的数据

### 4.3 Entity 去重

`EntitySources` 缓存（`map[string]int`，key = `policyID+source`）确保相同存储路径的 Entity 不会重复创建，而是增加引用计数。

### 4.4 文件冲突处理

[file.go#L129-L139](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/file.go#L129-L139) 中：
- 如果创建文件时遇到约束错误（`ent.IsConstraintError`），记录到 `FileConflictRename` 映射
- 重命名为 `{原ID}_{原名}`，然后 **跳出当前批次**（`continue out`），下一批重试时使用新名称
- 如果重命名后仍冲突，直接报错要求手动处理

### 4.5 关联检查

文件/文件夹/分享/直链/WebDAV 迁移时，会检查关联的 User、Folder、Policy 是否已存在于 state 缓存中，不存在则跳过该条记录并打印 Warning。

### 4.6 PostgreSQL 序列重置

每个有批量迁移的实体完成后，如果是 PostgreSQL 数据库，执行 `SELECT SETVAL(...)` 重置自增序列，确保后续插入不会 ID 冲突。

---

## 5. 中断恢复机制

### 5.1 State 状态文件

定义在 [migrator.go#L21-L41](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L21-L41)，持久化为 `migration_state.json`，存放在 V3 配置文件同目录。

State 结构体关键字段：

| 字段 | 用途 |
|---|---|
| `Step` | 当前迁移步骤编号 |
| `UserOffset` / `FolderOffset` / `FileOffset` / `ShareOffset` / `DirectLinkOffset` / `WebdavOffset` / `FolderParentOffset` | 各批次迁移的当前偏移量 |
| `PolicyIDs` | 已迁移的策略 ID 集合 |
| `LocalPolicyIDs` | 已迁移的本地策略 ID 集合 |
| `UserIDs` | 已迁移的用户 ID 集合 |
| `FolderIDs` | 已迁移的文件夹 ID 集合 |
| `EntitySources` | Entity 去重缓存，key="policyID+source", value=Entity ID |
| `LastFolderID` | 最大文件夹 ID，用于文件 ID 偏移 |
| `FileConflictRename` | 文件冲突重命名映射 |
| `ThumbSuffix` | V3 缩略图文件后缀 |
| `V3AvatarPath` | V3 头像存储路径 |

### 5.2 恢复流程

1. **启动时**（[migrator.go#L90-L133](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L90-L133)）：
   - 检查 `migration_state.json` 是否存在
   - 存在则加载 state，打印恢复信息（步骤名、offset）
   - 不存在则从 `StepInitial` 开始

2. **执行中**：
   - 每个步骤完成后调用 `updateStep()` 更新 Step 并保存 state
   - 批量步骤中每批完成后调用 `saveState()` 保存 offset
   - 用户/文件夹迁移错误时先 `saveState()` 再返回 error

3. **失败时**：
   - 命令行提示用户可用相同命令重试，将从上次保存点继续
   - `--force-reset` 标志可删除 state 文件从头开始

### 5.3 saveState / loadState

- `saveState`：JSON 序列化 State → 写入文件
- `loadState`：读取文件 → JSON 反序列化到 State
- `updateStep`：更新 Step 值 + saveState

### 5.4 注意事项

- **非精确幂等**：offset 机制基于查询偏移量而非主键，如果迁移过程中 V3 数据有增删，恢复后可能跳过或重复部分数据
- **事务保护**：每批数据在事务中处理，事务失败会回滚当前批次，但 state 的 offset 可能在部分成功时未更新（设计意图是重新处理该批）
- **User/Folder 错误时主动 saveState**：确保错误发生时已迁移的 ID 集合被保存，恢复后不再重复插入
- **文件冲突导致整个批次重试**：`continue out` 跳出内层循环，offset 未增加，下次从同一批开始，但 `FileConflictRename` 已记录冲突文件的新名称

---

## 6. V3 配置初始化

V3 数据库连接通过 [conf/conf.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/conf/conf.go) 和 [model/init.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/init.go) 初始化：

1. `conf.Init()` 读取 V3 的 INI 配置文件，映射到 `DatabaseConfig`、`SystemConfig` 等结构体
2. `model.Init()` 根据数据库类型（sqlite/mysql/postgres/mssql）创建 GORM 连接
3. 兼容 `sqlite3`→`sqlite`，`mariadb`→`mysql`

---

## 7. 未迁移的 V3 数据

以下 V3 模型存在但无对应迁移逻辑：

| V3 模型 | 文件 | 说明 |
|---|---|---|
| `Tag` | [model/tag.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/tag.go) | 用户自定义标签（文件分类/目录直达） |
| `Task` | [model/task.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/task.go) | 异步任务记录 |
| `User.Authn` | model/user.go | WebAuthn 凭据 |
| `User.Options` | model/user.go | 用户个性化配置 |
| `File.PicInfo` | model/file.go | 图片信息 |
| `Policy.AutoRename` | model/policy.go | 自动重命名 |
| `Policy.IsOriginLinkEnable` | model/policy.go | 原始链接启用 |
| `Share.PreviewEnabled` | model/share.go | 预览启用 |
| `Share.SourceName` | model/share.go | 搜索用字段 |
| `Node.MasterKey` | model/node.go | 从→主通信密钥 |

---

## 8. 关键设计决策总结

1. **文件夹与文件统一模型**：V4 将 V3 的 Folder 和 File 统一为 `File` 实体，通过 `Type` 区分
2. **文件 ID 偏移**：文件 ID = V3 ID + LastFolderID，避免文件夹和文件 ID 空间冲突
3. **Entity 解耦**：V4 将存储实体（物理文件引用）从 File 元数据中拆出，支持多版本和引用计数
4. **两阶段文件夹迁移**：先创建文件夹（无父级），再批量设置父级关系，避免外键约束冲突
5. **设置迁移表驱动**：通过 `migrators` map 实现声明式的设置转换规则
6. **远程策略自动建节点**：V3 远程存储策略在 V4 中需要关联 Slave Node，迁移时自动创建
