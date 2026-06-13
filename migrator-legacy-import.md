# Cloudreve V3 → V4 旧数据迁移代码详解

---

## 证据级别说明

本文档中所有结论分为两类，严格区分：

- ✅ **[代码直接证实]**：通过阅读源代码可以直接验证的事实，包括函数调用、结构体定义、常量值、if 分支、循环逻辑等
- ⚠️ **[间接推断]**：基于字段/函数/常量的命名、注释、结构相似性做出的合理推测，但代码中没有直接的显式关联或注释

---

## 1. 整体架构与执行顺序

### 1.1 调用顺序 ✅[代码直接证实]

迁移命令 `cloudreve migrate --v3-conf <path>`（[cmd/migrate.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/cmd/migrate.go)）的内部执行顺序：

```
1. dependency.NewDependency()
   └─→ inventory.Init()
        ├─→ Schema.Create()               // 创建 V4 表结构
        ├─→ migrateDefaultSettings()      // 补齐缺失的默认设置
        ├─→ migrateDefaultStoragePolicy() // 补齐默认存储策略(ID=1)
        ├─→ migrateSysGroups()            // 补齐默认用户组
        ├─→ migrateOAuthClient()          // 补齐默认 OAuth 客户端
        └─→ applyPatches()                // 应用版本补丁(如 reset_secret_key)

2. migrator.NewMigrator()                 // 读取 V3 配置，连接 V3 数据库
                                            // 加载 migration_state.json

3. migrator.Migrate()                     // 执行 V3 → V4 数据迁移（见 1.2 步骤流水线）
```

**关键影响**：`migrateDefaultSettings()` **先于** `Migrator.Migrate()` 执行。因此，被 `noopMigrator` 标记的设置项不会"丢失"——它们中一部分已由 `migrateDefaultSettings()` 根据 V4 的 DefaultSettings 模板补齐了新的默认值，另一部分则是 V4 中已弃用的功能。

### 1.2 迁移步骤流水线 ✅[代码直接证实]

定义在 [migrator.go#L44-L63](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L44-L63)，在 `Migrate()` 函数中按序判断执行：

| Step 常量 | 值 | 对应函数 | 类型 | 事务粒度 |
|---|---|---|---|---|
| `StepInitial` | 0 | — | 初始状态 | — |
| `StepSchema` | 1 | `v4client.Schema.Create()` | 一次性 | Ent 内置幂等 |
| `StepSettings` | 2 | `migrateSettings()` | 一次性 | 单事务 |
| `StepNode` | 3 | `migrateNode()` | 一次性 | 无事务 |
| `StepPolicy` | 4 | `migratePolicy()` | 一次性 | 单事务 |
| `StepGroup` | 5 | `migrateGroup()` | 一次性 | 无事务 |
| `StepUser` | 6 | `migrateUser()` | 批量 | 每批一事务 |
| `StepFolders` | 7 | `migrateFolders()` | 批量 | 每批一事务 |
| `StepFolderParent` | 8 | `migrateFolderParent()` | 批量 | 每批一事务 |
| `StepFile` | 9 | `migrateFile()` | 批量 | 每批一事务 |
| `StepShare` | 10 | `migrateShare()` | 批量 | 每批一事务 |
| `StepDirectLink` | 11 | `migrateDirectLink()` | 批量 | 每批一事务 |
| `Step_CommunityPlaceholder1` | 12 | （无函数） | 占位 | — |
| `Step_CommunityPlaceholder2` | 13 | （无函数） | 占位 | — |
| `StepAvatar` | 14 | `migrateAvatars()` | 一次性（文件复制） | 无事务 |
| `StepWebdav` | 15 | `migrateWebdav()` | 批量 | 每批一事务 |
| `StepCompleted` | 16 | — | 完成 | — |

**占位步骤的实现方式** ✅[代码直接证实]：
- 在 `Migrate()` 中（[migrator.go#L281-L297](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L281-L297)），`StepDirectLink`(11) 完成后直接 `updateStep(StepAvatar)` 将 Step 设为 14
- 步骤 12 和 13 在 Migrate() 中完全没有对应的 `if m.state.Step <= StepXxx` 判断块
- 因此执行时 Step 从 11 跳到 14，12 和 13 自动跳过

---

## 2. 旧字段映射（V3 → V4）

### 2.1 User 用户

**V3 模型**：[model/user.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/user.go)
**迁移函数**：[user.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/user.go) ✅[代码直接证实]

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID(int(u.ID))` | 原样保留 |
| `Email` | `Email` | 直传 |
| `Nick` | `Nick` | 直传 |
| `Password` | `Password` | 直传 |
| `Status`(int) | `Status`(enum) | 枚举映射：`Active→StatusActive`, `NotActivicated→StatusInactive`, `Baned→StatusManualBanned`, `OveruseBaned→StatusSysBanned` |
| `GroupID` | `GroupID` | 直传 |
| `Storage`(uint64) | `Storage`(int64) | 类型转换 |
| `TwoFactor` | `TwoFactorSecret` | 非空时设置，字段名变更 |
| `Avatar` | `Avatar` | 非空时设置 |
| — | `Settings` | 固定值 `{VersionRetention: true, VersionRetentionMax: 10}` |

未迁移字段 ✅[代码直接证实]：`Options`/`OptionsSerialized`、`Authn`。

### 2.2 Folder 文件夹

**V3 模型**：[model/folder.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/folder.go)
**迁移函数**：[folders.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/folders.go) ✅[代码直接证实]

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID(int(f.ID))` | 原样保留 |
| `Name` | `Name` | 根目录时置空 `""` |
| `ParentID`(*uint) | `ParentID` | 分两步：StepFolders 时跳过，StepFolderParent 时补设 |
| `OwnerID` | `OwnerID` | 直传 |
| `CreatedAt`/`UpdatedAt` | `CreatedAt`/`UpdatedAt` | 直传 |
| — | `Type` | 固定为 `FileTypeFolder` |

V4 中文件夹和文件统一为 `File` 实体，通过 `Type` 字段区分。

### 2.3 File 文件

**V3 模型**：[model/file.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/file.go)
**迁移函数**：[file.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/file.go) ✅[代码直接证实]

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID(int(f.ID) + LastFolderID)` | **ID 偏移**：文件 ID = V3 ID + 最大文件夹 ID |
| `Name` | `Name` | 冲突时用 `FileConflictRename` 映射的新名 |
| `SourceName` | → Entity `Source` | 拆分为 Entity 实体 |
| `UserID` | `OwnerID` | 字段名变更 |
| `Size` | `Size` | 直传 |
| `FolderID` | `FileChildren` | V4 文件夹作为父级，字段名变更 |
| `PolicyID` | `StoragePoliciesID` | 字段名变更 |
| `Metadata` | → Entity | `thumb_status=exist` 时额外创建缩略图 Entity |
| — | `PrimaryEntity` | 主版本 Entity 的 ID |
| — | `Type` | 固定为 `FileTypeFile` |

未迁移字段 ✅[代码直接证实]：`PicInfo`、`UploadSessionID`、`Position`。

### 2.4 Policy 存储策略

**V3 模型**：[model/policy.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/policy.go)
**迁移函数**：[policy.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/policy.go) ✅[代码直接证实]

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `Name`/`Type`/`Server`/`BucketName` | 同名 | 直传 |
| `IsPrivate` | `IsPrivate` | 直传 |
| `BaseURL` | `Settings.CustomProxy=true`+`Settings.ProxyServer` | 非空时设置自定义代理 |
| `AccessKey`/`SecretKey` | 同名 | 直传 |
| `MaxSize` | `MaxSize` | uint64→int64 |
| `DirNameRule`/`FileNameRule` | 同名 | 缺随机元素时强制覆盖默认值 |
| `Options`(JSON) | → `Settings`(PolicySetting) | 逐字段映射 |

未迁移字段 ✅[代码直接证实]：`AutoRename`、`IsOriginLinkEnable`、`OptionsSerialized.MimeType`、`OptionsSerialized.PlaceholderWithSize`。

### 2.5 Group 用户组

**V3 模型**：[model/group.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/group.go)
**迁移函数**：[group.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/group.go) ✅[代码直接证实]

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `Name` | `Name` | 直传 |
| `Policies`(JSON) | `StoragePoliciesID` | 取过滤后列表第一个 |
| `MaxStorage` | `MaxStorage` | uint64→int64 |
| `ShareEnabled` | → `Permissions[GroupPermissionShare]` | 布尔→权限位 |
| `WebDAVEnabled` | → `Permissions[GroupPermissionWebDAV]` | 布尔→权限位 |
| `SpeedLimit` | `SpeedLimit` | 直传 |
| `Options`(JSON) | → `Settings`(GroupSetting) | 逐字段映射 |

未迁移字段 ✅[代码直接证实]：`GroupOption.OneTimeDownload`。

### 2.6 Node 节点

**V3 模型**：[model/node.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/node.go)
**迁移函数**：[node.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/node.go) ✅[代码直接证实]

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `Type`(ModelType) | `Type`(node.Type) | `SlaveNodeType→TypeSlave`, `MasterNodeType→TypeMaster` |
| `Status`(NodeStatus) | `Status` | `NodeActive→StatusActive`, 其他→`StatusSuspended` |
| `Name`/`Server`/`SlaveKey` | 同名 | 直传 |
| `Aria2Enabled` | → `Capabilities[NodeCapabilityRemoteDownload]` | 布尔→能力位 |
| `Aria2Options`(JSON) | → `Settings.Aria2Setting` | 逐字段映射 |
| `Rank` | `Weight` | 字段名变更 |

未迁移字段 ✅[代码直接证实]：`MasterKey`、`Aria2Option.Interval`、`Aria2Option.Timeout`。

### 2.7 Setting 系统设置

**V3 模型**：[model/setting.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/setting.go)
**迁移函数**：[settings.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/settings.go) ✅[代码直接证实]

迁移通过 `migrators` map 表驱动，共 46 个设置项有显式映射规则：

#### 2.7.1 noopMigrator（46 项中的 39 项）✅[代码直接证实]

`noopMigrator` 的语义是：**该设置项在 migrator 范围内不迁移**。具体命运分两类：

**A 类：V4 DefaultSettings 中有新默认值（migrateDefaultSettings 已补齐）**

| 设置名 | DefaultSettings 中的值 | 说明 |
|---|---|---|
| `defaultTheme` | `#1976d2` | V4 改了主题体系 |
| `theme_options` | 完整的多主题调色板 JSON | V4 改了主题体系 |
| `max_parallel_transfer` | `4` | V4 默认值 |
| `secret_key` | `RandStringRunesCrypto(256)` | 随机生成（同时 reset_secret_key patch 也会重置） |
| `mail_activation_template` | 基于 mailTemplateContents 重新生成的多语言 JSON | V4 重写了邮件模板体系 |
| `mail_reset_pwd_template` | （无，V4 叫 mail_reset_template） | V4 重命名并重新生成 |
| `captcha_type` | `normal` | V4 默认 |

> 注：`captcha_type` 不在 noopMigrator 中，是独立的 migrator，但 migrateDefaultSettings 也有它的默认值。

**B 类：完全弃用（V4 DefaultSettings 中无对应项）**

| 类别 | 被弃用的设置项 |
|---|---|
| 站点/分享 | `siteKeywords`、`over_used_template`、`hot_share_num` |
| 超时系列 | `download_timeout`、`preview_timeout`、`doc_preview_timeout`、`share_download_session_timeout` |
| 从节点管理 | `slave_node_retry`、`slave_ping_interval`、`slave_recover_interval`、`slave_transfer_timeout` |
| OneDrive | `onedrive_monitor_timeout`、`onedrive_source_timeout`、`onedrive_callback_check` |
| 第三方登录 | `appid`、`appkey`（QQ 互联） |
| 微信支付 | `wechat_enabled`、`wechat_appid`、`wechat_mchid`、`wechat_serial_no`、`wechat_api_key`、`wechat_pk_content` |
| 腾讯防水墙 | `captcha_TCaptcha_CaptchaAppId`、`captcha_TCaptcha_AppSecretKey`、`captcha_TCaptcha_SecretId`、`captcha_TCaptcha_SecretKey` |
| 自定义支付 | `custom_payment_enabled`、`custom_payment_endpoint`、`custom_payment_secret`、`custom_payment_name` |
| 头像大小 | `avatar_size_m`、`avatar_size_s` |
| 视图/计划任务 | `home_view_method`、`share_view_method`、`cron_recycle_upload_session` |
| 其他 | `initial_files`、`office_preview_service`、`phone_required`、`phone_enabled` |

#### 2.7.2 字段名变更（2 项）✅[代码直接证实]

| V3 设置名 | V4 设置名 |
|---|---|
| `thumb_file_suffix` | `thumb_entity_suffix` |
| `wopi_session_timeout` | `viewer_session_timeout` |

#### 2.7.3 值转换（2 项）✅[代码直接证实]

| V3 设置名 | 转换逻辑 |
|---|---|
| `captcha_type` | `tcaptcha` → `normal` |
| `thumb_max_src_size` | 一对多拆分为 5 个设置：`thumb_music_cover_max_size`、`thumb_libreoffice_max_size`、`thumb_ffmpeg_max_size`、`thumb_vips_max_size`、`thumb_builtin_max_size` |

#### 2.7.4 新增设置（1 项）✅[代码直接证实]

- `hash_id_salt`：从 V3 配置文件 `System.HashIDSalt` 读取后插入

#### 2.7.5 边效提取（2 项）✅[代码直接证实]

在迁移循环中同时将设置值提取到 state，供后续步骤使用：
- `thumb_file_suffix` → `state.ThumbSuffix`（文件迁移时拼接缩略图路径）
- `avatar_path` → `state.V3AvatarPath`（头像迁移时定位 V3 头像目录）

### 2.8 Share 分享

**V3 模型**：[model/share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/share.go)
**迁移函数**：[share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/share.go) ✅[代码直接证实]

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `Password` | `Password` | 非空时设置 |
| `IsDir` | → `FileID` 偏移计算 | 文件：`sourceId+LastFolderID`，目录：`sourceId` |
| `UserID` | `UserID` | 直传 |
| `SourceID` | `FileID` | 根据 IsDir 决定是否加偏移 |
| `Views`/`Downloads` | 同名 | 直传 |
| `RemainDownloads` | `RemainDownloads` | ≥0 时设置 |
| `Expires` | `Expires` | 非空时设置 |

未迁移字段 ✅[代码直接证实]：`PreviewEnabled`、`SourceName`。

### 2.9 DirectLink 直链（SourceLink）

**V3 模型**：[model/source_link.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/source_link.go)
**迁移函数**：[directlink.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/directlink.go) ✅[代码直接证实]

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `FileID` | `FileID` | 加 `LastFolderID` 偏移 |
| `Name`/`Downloads` | 同名 | 直传 |
| — | `Speed` | 固定为 0 |

### 2.10 WebDAV 账户

**V3 模型**：[model/webdav.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/webdav.go)
**迁移函数**：[webdav.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/webdav.go) ✅[代码直接证实]

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `Name`/`Password` | 同名 | 直传 |
| `UserID` | `OwnerID` | 字段名变更 |
| `Root` | `URI` | 格式化为 `"cloudreve://my" + Root` |
| `Readonly` | → `Options[DavAccountReadOnly]` | 布尔→选项位 |
| `UseProxy` | → `Options[DavAccountProxy]` | 布尔→选项位 |
| — | `Props` | 空结构体 `DavAccountProps{}` |

### 2.11 Avatar 头像

**迁移函数**：[avatars.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/avatars.go) ✅[代码直接证实]

文件复制：`{V3AvatarPath}/avatar_{uid}_2.png` → `{data_path}/avatar/avatar_{uid}.png`

---

## 3. Entity 实体创建

### 3.1 insertEntity 去重逻辑 ✅[代码直接证实]

定义在 [file.go#L160-L189](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/file.go#L160-L189)：

```
entityKey = strconv.Itoa(policyID) + "+" + source

1. 在 EntitySources(map[string]int) 缓存中查找 entityKey
   - 命中 → 尝试 Update 该 Entity：reference_count +1
   - Update 失败 → 降级为创建新 Entity
2. 未命中 → 创建新 Entity（ReferenceCount=1）
3. 将 Entity ID 写入 EntitySources 缓存
```

### 3.2 每个 V3 文件的 Entity 创建流程 ✅[代码直接证实]

1. **缩略图 Entity**（条件创建）：`metadata["thumb_status"] == "exist"` 时
   - Source = `SourceName + ThumbSuffix`
   - Type = `EntityTypeThumbnail`
2. **版本 Entity**（必定创建）：
   - Source = `SourceName`
   - Type = `EntityTypeVersion`
3. **File 实体**关联：
   - `PrimaryEntity` = 版本 Entity ID
   - `AddEntities` = [版本 Entity, 缩略图 Entity(如有)]

---

## 4. 恢复语义深度分析

### 4.1 一次性步骤 vs 批量步骤 ✅[代码直接证实]

| 维度 | 一次性（有事务） | 一次性（无事务） | 批量步骤 |
|---|---|---|---|
| 原子性 | 全步原子 | 逐条写入 | 每批原子 |
| 失败后残留 | 无（回滚） | 已创建的实体残留 | 已提交批次保留 |
| 重试安全性 | 有条件安全（提交后有小窗口） | ❌ 不安全 | ✅ 安全 |
| 恢复精度 | 整步重做 | 整步重做但必失败 | 精确到批次 |
| 状态更新时机 | 整步完成后 | 整步完成后 | 每批完成后 |

**一次性（有事务）步骤**：StepSettings、StepPolicy
**一次性（无事务）步骤**：StepNode、StepGroup、StepAvatar（文件操作）
**批量步骤**：StepUser、StepFolders、StepFolderParent、StepFile、StepShare、StepDirectLink、StepWebdav

### 4.2 无事务一次性步骤的致命问题 ✅[代码直接证实]

以 `migrateNode`（[node.go#L22-L86](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/node.go#L22-L86)）为例：
- 循环逐个 `Create().SetRawID()`，无事务包裹
- 假设迁移到第 3 个节点失败，前 2 个已写入数据库
- `state.Step` 仍为 `StepNode`（`updateStep` 在整步完成后才调用）
- 重试时从第 1 个节点开始执行，第 1 个就会因主键（RawID）唯一约束冲突报错
- 用户看到的错误是"创建节点失败"，但实际原因是之前部分成功的残留

**受影响步骤**：StepNode、StepGroup

### 4.3 批量步骤的标准执行序列 ✅[代码直接证实]

以 migrateUser 为例：

```
for {
    查询 V3 数据（offset, batchSize=1000）
    开始事务
    循环创建 1000 条
         ↓ 出错 → Rollback → return error（offset 不更新）
    提交事务
    offset += 1000
    state.UserOffset = offset
    saveState()  ← 持久化点
}
整步完成 → updateStep(下一步)
```

User 和 Folders 步骤额外在错误返回前 saveState 一次（[migrator.go#L231-L233](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L231-L233)、[L243-L245](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L243-L245)），确保 `UserIDs`/`FolderIDs` 集合持久化。

### 4.4 force-reset 的真实影响范围 ✅[代码直接证实]

**force-reset 的代码只有一个行为**（[cmd/migrate.go#L47-L53](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/cmd/migrate.go#L47-L53)）：

```go
if forceReset && util.Exists(stateFilePath) {
    os.Remove(stateFilePath)
}
```

即：**删除 `migration_state.json` 文件**，仅此而已。

**force-reset 不会做的事** ✅[代码直接证实]：
- ❌ 不清空 V4 数据库中的表结构或数据
- ❌ 不删除已复制到 V4 的头像文件
- ❌ 不回滚 Schema
- ❌ 不改动 V3 数据库或配置文件

**force-reset 后重新运行各步骤的后果** ✅[代码直接证实]：

| 步骤 | 结果 | 原因 |
|---|---|---|
| StepSchema | ✅ 成功 | Schema.Create 幂等 |
| StepSettings | ❌ 大概率失败 | `hash_id_salt` 或其他直传设置的 `name` 唯一索引冲突 |
| StepNode | ❌ 失败 | RawID 主键冲突（第一个节点即报错） |
| StepPolicy | 分情况 | 之前整步未完成（事务未提交）→ 安全；之前已完成 → ID 冲突 |
| StepGroup | ❌ 失败 | RawID 主键冲突 |
| 后续批量步骤 | ❌ 失败 | RawID 主键冲突 |

> ⚠️ **结论**：`--force-reset` 命名有误导性。它重置的是"状态文件"而非"迁移结果"。只要任何写入步骤已部分/全部完成，force-reset 后重试必然失败。必须**手动清空 V4 数据库**才能真正从头开始。

---

## 5. 占位字段与未接入迁移范围

### 5.1 Step 占位 ✅[代码直接证实]

**事实**：
- [migrator.go#L57-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L57-L58) 定义了两个常量：
  ```go
  Step_CommunityPlaceholder1 = 12
  Step_CommunityPlaceholder2 = 13
  ```
- 在 `Migrate()` 中，Step 11（StepDirectLink）完成后直接跳到 Step 14（StepAvatar），12 和 13 无对应代码块
- 因此社区版运行时这两个步骤编号自动被跳过

**关于常量名 "Community" 的含义** ⚠️[间接推断]：
- 常量名 `CommunityPlaceholder` 暗示这是"社区版的占位槽位"
- 推测 Pro 版在这两个槽位会插入自己的迁移函数，不改动步骤编号
- 这样社区版和 Pro 版的 state 文件中 Step 数字含义一致，可以互换

### 5.2 State 字段占位 ✅[代码直接证实]

**事实**：
- [migrator.go#L33](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L33)：`GiftCodeOffset int \`json:"gift_code_offset,omitempty"\``
- [migrator.go#L36](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L36)：`StoragePackOffset int \`json:"storage_pack_offset,omitempty"\``
- 在整个 migrator 目录中搜索这两个字段，**除定义外没有任何赋值或读取操作**（Grep 结果仅返回定义位置）
- 它们是 State 结构体中的"死字段"，在社区版迁移中完全不参与

### 5.3 settings.go 中的未使用结构体 ✅[代码直接证实]

**事实**：
- [settings.go#L19-L37](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/settings.go#L19-L37) 定义了两个结构体：
  ```go
  // PackProduct 容量包商品
  PackProduct struct {
      ID    int64  `json:"id"`
      Name  string `json:"name"`
      Size  uint64 `json:"size"`
      Time  int64  `json:"time"`
      Price int    `json:"price"`
      Score int    `json:"score"`
  }
  GroupProducts struct {
      ID        int64    `json:"id"`
      Name      string   `json:"name"`
      GroupID   uint     `json:"group_id"`
      Time      int64    `json:"time"`
      Price     int      `json:"price"`
      Score     int      `json:"score"`
      Des       []string `json:"des"`
      Highlight bool     `json:"highlight"`
  }
  ```
- 整个代码库中，这两个结构体只有定义，没有任何地方声明变量、解析 JSON 或作为函数参数（Grep 结果仅返回定义位置）
- `PackProduct` 的注释明确写着"容量包商品"
- 字段含义（Size=容量、Time=时长、Price=价格、Score=积分）符合"容量包商品"语义

### 5.4 占位步骤、State 字段、结构体之间的关联 ⚠️[间接推断]

**证据链（全部基于命名相似性，无直接代码关联）**：

| 占位资源 | 命名线索 | 推测对应 |
|---|---|---|
| Step_CommunityPlaceholder1 | 步骤占位 #1 | 礼品码迁移（对应 State.GiftCodeOffset） |
| Step_CommunityPlaceholder2 | 步骤占位 #2 | 容量包迁移（对应 State.StoragePackOffset） |
| State.GiftCodeOffset | JSON tag: `gift_code_offset` | 礼品码的批量迁移偏移量 |
| State.StoragePackOffset | JSON tag: `storage_pack_offset` | 容量包的批量迁移偏移量 |
| PackProduct 结构体 | 注释: "容量包商品"，字段 Size/Time/Price | 存储在某个 V3 设置中的容量包商品列表（JSON 数组） |
| GroupProducts 结构体 | GroupID、Des、Highlight | 存储在某个 V3 设置中的用户组商品列表 |

**反证/不确定性**：
- 虽然 V4 inventory 中有 `CreateStoragePackArgs` 和错误码 `CodeInvalidGiftCode = 40065`，但这只能证明 V4 本身有容量包和礼品码功能，不能证明它们和迁移占位的对应关系
- V3 migrator 的 model 目录中**没有** gift_code 或 storage_pack 数据表模型定义，所以如果 Pro 版 V3 有这些表，模型定义是在 Pro 版专属代码中
- PackProduct 和 GroupProducts 是在 `settings.go` 中定义的（而不是独立的 model 文件），这暗示它们可能是从某个系统设置（而非独立数据表）中解析出来的 JSON 结构

### 5.5 权限/能力位占位 ✅[代码直接证实]

**事实**：
- V4 inventory/types/types.go 中有 5 个带 `CommunityPlaceholder` 后缀的枚举值：
  ```go
  GroupPermission_CommunityPlaceholder1  // 位置 8
  GroupPermission_CommunityPlaceholder2  // 位置 10
  GroupPermission_CommunityPlaceholder3  // 位置 13
  GroupPermission_CommunityPlaceholder4  // 位置 14
  NodeCapability_CommunityPlaceholder    // 位置 4
  ```
- 这些枚举值在社区版代码中没有被任何逻辑引用
- 它们与迁移不直接相关，属于 V4 权限体系本身的预留设计

### 5.6 社区版完全未迁移的 V3 模型 ✅[代码直接证实]

| V3 模型 | 文件 | migrator 中有无对应迁移函数 |
|---|---|---|
| `Tag` | [model/tag.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/tag.go) | ❌ 无 |
| `Task` | [model/task.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/task.go) | ❌ 无 |

（Tag：用户自定义标签/文件分类/目录直达；Task：异步任务记录）

### 5.7 部分字段未迁移汇总 ✅[代码直接证实]

| V3 来源 | 未迁移字段 |
|---|---|
| User | `Authn`（WebAuthn 凭据）、`Options`/`OptionsSerialized`（个性化配置） |
| File | `PicInfo`、`UploadSessionID`、`Position` |
| Policy | `AutoRename`、`IsOriginLinkEnable`、`OptionsSerialized.MimeType`、`OptionsSerialized.PlaceholderWithSize` |
| Group | `OptionsSerialized.OneTimeDownload` |
| Share | `PreviewEnabled`、`SourceName` |
| Node | `MasterKey`、`Aria2OptionsSerialized.Interval`、`Aria2OptionsSerialized.Timeout` |

---

## 6. 幂等处理机制汇总

### 6.1 多层次幂等设计 ✅[代码直接证实]

| 层级 | 机制 | 覆盖范围 |
|---|---|---|
| 步骤级 | `Step` 编号 + `if step <=` 判断 | 已完成步骤不重复执行 |
| 批次级 | `offset` 偏移量记录 | 批量步骤不重复处理已成功批次 |
| 实体级 | `EntitySources` 缓存（policyID+source 唯一键） | 相同存储路径的 Entity 不重复创建，引用计数+1 |
| 冲突处理 | `FileConflictRename` 映射 | 同名文件自动重命名后重试 |
| 关联校验 | `UserIDs`/`FolderIDs`/`PolicyIDs` 集合校验 | 跳过关联缺失的记录 |
| 序列重置 | PostgreSQL 的 `SELECT SETVAL(...)` | 避免后续插入 ID 冲突 |

### 6.2 幂等性的局限 ⚠️[间接推断 + 代码证实混合]

| 局限 | 分析 |
|---|---|
| V3 数据变更风险 | ✅ 代码证实：offset 基于 `LIMIT 1000 OFFSET X`，而非主键范围。如果迁移过程中 V3 数据有增删，可能重复或遗漏。但迁移时应禁止写入 V3，这属于使用约定。 |
| 无事务一次性步骤 | ✅ 代码证实：Node/Group 中途失败无法安全重试。 |
| 事务提交与状态保存时间差 | ✅ 代码证实：理论上存在"事务已提交但 saveState 失败"窗口，但极小概率。 |
| force-reset 假象 | ✅ 代码证实：只清状态不清数据，重试必败。 |

---

## 7. V3 配置初始化 ✅[代码直接证实]

1. `conf.Init()`（[conf/conf.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/conf/conf.go)）读取 V3 的 INI 配置文件，映射到 `DatabaseConfig`、`SystemConfig` 等结构体
2. `model.Init()`（[model/init.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/init.go)）根据数据库类型创建 GORM 连接
3. 兼容：`sqlite3`→`sqlite`，`mariadb`→`mysql`

---

## 8. 关键设计决策总结 ✅[代码直接证实]

1. **文件夹与文件统一模型**：V4 将 V3 的 Folder 和 File 统一为 `File` 实体，通过 `Type` 区分
2. **文件 ID 偏移**：文件 ID = V3 ID + LastFolderID，避免文件夹和文件 ID 空间冲突
3. **Entity 解耦**：将存储实体（物理文件引用）从 File 元数据中拆出，支持多版本和引用计数
4. **两阶段文件夹迁移**：先创建文件夹（无父级），再批量设置父级关系，避免外键约束冲突
5. **设置迁移表驱动**：通过 `migrators` map 实现声明式的设置转换规则
6. **远程策略自动建节点**：V3 远程存储策略在 V4 中需要关联 Slave Node，迁移时自动创建
7. **步骤槽位预留**：通过 CommunityPlaceholder 常量和 State 死字段预留迁移步骤扩展点
8. **状态文件外置**：state 文件与 V3 配置文件同目录，不依赖 V4 数据库，便于独立管理
9. **V4 默认值体系前置**：migrateDefaultSettings 在 Migrator 前执行，部分"被丢弃"的设置实际由 V4 重新生成默认值
10. **部分步骤缺少事务保护**：Node 和 Group 迁移为无事务逐条写入，失败后无法安全重试
