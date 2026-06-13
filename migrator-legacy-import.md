# Cloudreve V3 → V4 旧数据迁移代码详解

## 1. 整体架构

迁移代码位于 [application/migrator](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator) 目录，职责是将 Cloudreve V3 的数据（基于 GORM + 原始 SQL 表）导入 V4 的 Ent ORM 数据库。

入口命令定义在 [cmd/migrate.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/cmd/migrate.go)，通过 `cloudreve migrate --v3-conf <path>` 触发。

### 1.1 迁移步骤流水线

定义在 [migrator.go#L44-L63](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L44-L63)，严格按序执行：

| Step 常量 | 值 | 说明 | 类型 | 事务粒度 |
|---|---|---|---|---|
| `StepInitial` | 0 | 初始状态 | — | — |
| `StepSchema` | 1 | 创建 V4 表结构 | 一次性 | Ent 内置幂等 |
| `StepSettings` | 2 | 迁移系统设置 | 一次性 | 单事务 |
| `StepNode` | 3 | 迁移节点 | 一次性 | 无事务 |
| `StepPolicy` | 4 | 迁移存储策略 | 一次性 | 单事务 |
| `StepGroup` | 5 | 迁移用户组 | 一次性 | 无事务 |
| `StepUser` | 6 | 迁移用户 | 批量 | 每批一事务 |
| `StepFolders` | 7 | 迁移文件夹 | 批量 | 每批一事务 |
| `StepFolderParent` | 8 | 补设文件夹父级关系 | 批量 | 每批一事务 |
| `StepFile` | 9 | 迁移文件 | 批量 | 每批一事务 |
| `StepShare` | 10 | 迁移分享 | 批量 | 每批一事务 |
| `StepDirectLink` | 11 | 迁移直链 | 批量 | 每批一事务 |
| `Step_CommunityPlaceholder1` | 12 | Pro 占位：礼品码 | 占位（Pro） | — |
| `Step_CommunityPlaceholder2` | 13 | Pro 占位：容量包 | 占位（Pro） | — |
| `StepAvatar` | 14 | 迁移头像文件 | 一次性 | 文件操作，无事务 |
| `StepWebdav` | 15 | 迁移 WebDAV 账户 | 批量 | 每批一事务 |
| `StepCompleted` | 16 | 完成 | — | — |

---

## 2. 旧字段映射（V3 → V4）

### 2.1 User 用户

**V3 模型**：[model/user.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/user.go)
**迁移函数**：[user.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/user.go)

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

**未迁移字段**：`Options`/`OptionsSerialized`（用户个性化配置如 `profile_off`, `preferred_theme`）、`Authn`（WebAuthn 凭据）。

### 2.2 Folder 文件夹

**V3 模型**：[model/folder.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/folder.go)
**迁移函数**：[folders.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/folders.go)

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
**迁移函数**：[file.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/file.go)

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
**迁移函数**：[policy.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/policy.go)

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `Name` | `Name` | 直传 |
| `Type` | `Type` | 直传（字符串策略类型） |
| `Server` | `Server` | 直传 |
| `BucketName` | `BucketName` | 直传 |
| `IsPrivate` | `IsPrivate` | 直传 |
| `BaseURL` | → `Settings.CustomProxy` + `Settings.ProxyServer` | 非空时设置自定义代理 |
| `AccessKey` | `AccessKey` | 直传 |
| `SecretKey` | `SecretKey` | 直传 |
| `MaxSize` | `MaxSize` | uint64→int64 |
| `DirNameRule` | `DirNameRule` | 直传（缺随机元素时强制覆盖默认值） |
| `FileNameRule` | `FileNameRule` | 直传（缺随机元素时强制覆盖默认值） |
| `Options` (JSON) | → `Settings` (PolicySetting) | 逐字段映射 |

**PolicyOption → PolicySetting 映射**：

| V3 PolicyOption | V4 PolicySetting |
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
- 缩略图代理设置从 V3 系统设置 `thumb_proxy_enabled` + `thumb_proxy_policy` 读取
- 远程策略（`PolicyTypeRemote`）自动创建一个 Slave Node

**未迁移字段**：`AutoRename`、`IsOriginLinkEnable`、`OptionsSerialized.MimeType`、`OptionsSerialized.PlaceholderWithSize`。

### 2.5 Group 用户组

**V3 模型**：[model/group.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/group.go)
**迁移函数**：[group.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/group.go)

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `Name` | `Name` | 直传 |
| `Policies` (JSON) | `StoragePoliciesID` | 取过滤后的策略列表第一个 |
| `MaxStorage` | `MaxStorage` | uint64→int64 |
| `ShareEnabled` | → `Permissions[GroupPermissionShare]` | 布尔→权限位 |
| `WebDAVEnabled` | → `Permissions[GroupPermissionWebDAV]` | 布尔→权限位 |
| `SpeedLimit` | `SpeedLimit` | 直传 |
| `Options` (JSON) | → `Settings` (GroupSetting) | 逐字段映射 |

**权限映射表**：

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
**迁移函数**：[node.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/node.go)

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

Master 节点额外赋予 `NodeCapabilityExtractArchive` + `NodeCapabilityCreateArchive` 能力。

**未迁移字段**：`MasterKey`、`Aria2Option.Interval`、`Aria2Option.Timeout`。

### 2.7 Setting 系统设置

**V3 模型**：[model/setting.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/setting.go)
**迁移函数**：[settings.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/settings.go)

迁移逻辑通过 `migrators` map 表驱动，分为四类处理：

1. **noopMigrator（丢弃）**：约 40 项 V3 设置直接跳过，不写入 V4
2. **字段名变更**：`thumb_file_suffix`→`thumb_entity_suffix`，`wopi_session_timeout`→`viewer_session_timeout`
3. **值转换**：`captcha_type` 将 `tcaptcha` 转 `normal`；`thumb_max_src_size` 一对多拆分为 5 个缩略图设置
4. **直传**：未在 migrators 表中的设置，name/value 原样写入 V4

**新增设置**：`hash_id_salt` 从 V3 配置文件 `System.HashIDSalt` 读取后插入。

### 2.8 Share 分享

**V3 模型**：[model/share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/share.go)
**迁移函数**：[share.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/share.go)

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
**迁移函数**：[directlink.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/directlink.go)

| V3 字段 | V4 字段 | 映射说明 |
|---|---|---|
| `gorm.Model.ID` | `RawID` | 原样保留 |
| `FileID` | `FileID` | 加 `LastFolderID` 偏移 |
| `Name` | `Name` | 直传 |
| `Downloads` | `Downloads` | 直传 |
| — | `Speed` | 固定为 0 |

### 2.10 WebDAV 账户

**V3 模型**：[model/webdav.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/webdav.go)
**迁移函数**：[webdav.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/webdav.go)

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

---

## 3. Entity 实体创建

V4 引入了 `Entity` 概念，代表存储后端中的实际数据对象。每个 V3 文件在迁移时会创建 1~2 个 Entity。

### 3.1 insertEntity 函数

定义在 [file.go#L160-L189](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/file.go#L160-L189)，去重逻辑：

```
entityKey = strconv.Itoa(policyID) + "+" + source

1. 在 EntitySources 缓存中查找
   - 找到：reference_count +1，返回现有 Entity
   - 更新失败：降级为创建新 Entity
2. 创建新 Entity（Source / Type / Size / StoragePolicyEntities / CreatedBy / ReferenceCount=1）
3. 写入 EntitySources 缓存
```

### 3.2 文件迁移时的 Entity 创建流程

对每个 V3 文件：
1. **缩略图 Entity**（条件：metadata 中 `thumb_status == "exist"`）
2. **版本 Entity**（必定创建，Type = `EntityTypeVersion`）
3. **File 实体**关联两个 Entity，`PrimaryEntity` 指向版本 Entity

---

## 4. 恢复语义深度分析

### 4.1 一次性步骤 vs 批量步骤：重试行为核心差异

迁移步骤分为两大类，重试时的行为有本质区别。

#### 4.1.1 一次性步骤

一次性步骤是指**整个步骤作为一个逻辑单元**，要么整体完成，要么整体失败后重试整个步骤。

| 步骤 | 事务保护 | 重试起点 | 重试是否安全 | 关键风险点 |
|---|---|---|---|---|
| StepSchema | Ent 内置 | 重新执行 | ✅ 安全 | Schema.Create 是幂等的（CREATE TABLE IF NOT EXISTS） |
| StepSettings | 单事务 | 重新执行整步 | ⚠️ 有条件安全 | 事务提交前失败→回滚安全；提交后失败→主键冲突 |
| StepPolicy | 单事务 | 重新执行整步 | ⚠️ 有条件安全 | 同上；含远程策略时会同时创建 Node，也在事务内 |
| StepNode | 无事务 | 重新执行整步 | ❌ 不安全 | 逐个创建 RawID，中途失败后重试 → 第一条就主键冲突 |
| StepGroup | 无事务 | 重新执行整步 | ❌ 不安全 | 同上，逐个创建 RawID |
| StepAvatar | 文件操作 | 重新复制全部 | ✅ 覆盖式安全 | 文件复制会覆盖，不会冲突 |

**无事务一次性步骤的致命问题**：
以 `migrateNode` 为例（[node.go#L22-L86](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/node.go#L22-L86)）：
- 循环逐个 `Create().SetRawID()`，不使用事务
- 假设迁移到第 3 个节点失败，前 2 个已写入数据库
- `state.Step` 仍为 `StepNode`（因为 `updateStep` 在整步完成后才调用）
- 重试时从第 1 个节点开始执行，第 1 个就会因主键冲突报错
- 用户看到的错误是"创建节点失败"，但实际原因是之前部分成功的残留

**有事务一次性步骤的风险窗口**：
以 `migrateSettings` 为例（[settings.go#L182-L210](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/settings.go#L182-L210)）：
```
事务提交成功 → updateStep(StepNode) 失败
   ↓
数据已写入 V4，但 Step 仍为 StepSettings
   ↓
重试时重新执行 migrateSettings
   ↓
name 唯一约束冲突（hash_id_salt 或其他设置）
```
这个时间窗口很小但存在。

#### 4.1.2 批量步骤

批量步骤是指**数据分批处理**，每批独立事务，失败后从最后一个成功批次的末尾继续。

| 步骤 | 错误时 saveState? | 每批事务 | 重试起点 | 数据一致性 |
|---|---|---|---|---|
| StepUser | ✅ 是 | 每批一事务 | UserOffset | 精确（按 offset） |
| StepFolders | ✅ 是 | 每批一事务 | FolderOffset | 精确（按 offset） |
| StepFolderParent | ❌ 否 | 每批一事务 | FolderParentOffset | 精确（按 offset） |
| StepFile | ❌ 否 | 每批一事务 | FileOffset | 精确（按 offset） |
| StepShare | ❌ 否 | 每批一事务 | ShareOffset | 精确（按 offset） |
| StepDirectLink | ❌ 否 | 每批一事务 | DirectLinkOffset | 精确（按 offset） |
| StepWebdav | ❌ 否 | 每批一事务 | WebdavOffset | 精确（按 offset） |

**批量步骤的标准执行序列**（以 migrateUser 为例）：

```
for {
    查询 V3 数据（offset, batchSize=1000）
    开始事务
    循环创建 1000 个用户（SetRawID）
         ↓ 出错 → Rollback → return error
    提交事务
    offset += 1000
    state.UserOffset = offset
    saveState()  ← 状态持久化点
}
整步完成 → updateStep(StepFolders)
```

**关键保证**：
- 每批事务原子性：批次内要么全成功要么全失败
- 状态持久化在事务提交之后：不会出现"状态已前进但数据未写入"
- 失败时当前批次回滚 + offset 不更新：恢复后从同一批重新开始，不会重复也不会丢数据

**User/Folders 的额外 saveState**：
在 [migrator.go#L231-L233](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L231-L233) 和 [migrator.go#L243-L245](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L243-L245)：
```go
if err := m.migrateUser(); err != nil {
    m.saveState()  // 错误返回前再保存一次
    return err
}
```
这是为了确保 `UserIDs`/`FolderIDs` 集合等内存状态也被持久化（虽然 offset 已经在每批后保存了）。其他批量步骤没有这个额外 saveState，因为它们的 state 变化主要就是 offset。

#### 4.1.3 对比总结

| 维度 | 一次性步骤（有事务） | 一次性步骤（无事务） | 批量步骤 |
|---|---|---|---|
| 原子性 | 全步原子 | 逐条写入 | 每批原子 |
| 失败后残留 | 无（回滚） | 已创建的实体残留 | 已提交批次保留 |
| 重试安全性 | 大部分安全（提交后窗口除外） | 完全不安全 | 安全 |
| 恢复精度 | 整步重做 | 整步重做但必失败 | 精确到批次 |
| 状态更新时机 | 整步完成后 | 整步完成后 | 每批完成后 |

---

### 4.2 force-reset 的实际影响范围

`--force-reset` 标志定义在 [cmd/migrate.go#L22](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/cmd/migrate.go#L22)，实现非常简单（[cmd/migrate.go#L47-L53](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/cmd/migrate.go#L47-L53)）：

```go
if forceReset && util.Exists(stateFilePath) {
    logger.Info("Force resetting migration state. Will start from the beginning.")
    if err := os.Remove(stateFilePath); err != nil {
        // ...
    }
}
```

#### 4.2.1 force-reset 只做一件事

**删除 `migration_state.json` 文件**，仅此而已。

#### 4.2.2 force-reset 不会做的事

| 资源 | 是否被清理 | 说明 |
|---|---|---|
| V4 数据库表结构 | ❌ 保留 | Schema 完全不动 |
| V4 数据库中的数据 | ❌ 全部保留 | 已迁移的用户、文件、设置等全部保留 |
| 已复制的头像文件 | ❌ 保留 | 文件系统操作不回滚 |
| V3 数据库 | ❌ 完全不碰 | 迁移是只读 V3 |
| V3 配置文件 | ❌ 保留 | 只读 |

#### 4.2.3 force-reset 后重新运行各步骤的行为

假设之前已经跑了一部分，现在 force-reset 后重新运行：

| 步骤 | 结果 | 原因 |
|---|---|---|
| StepSchema | ✅ 成功 | Ent Schema.Create 幂等（表已存在就跳过） |
| StepSettings | ❌ 失败 | `hash_id_salt` 或其他设置的 `name` 唯一索引冲突 |
| StepNode | ❌ 失败 | RawID 主键冲突（第一个节点就报错） |
| StepPolicy | 分情况 | 之前整步未完成（事务未提交）→ 安全；之前已完成 → ID 冲突 |
| StepGroup | ❌ 失败 | RawID 主键冲突（第一个组就报错） |
| StepUser | ❌ 失败 | RawID 主键冲突 |
| StepFolders | ❌ 失败 | RawID 主键冲突 |
| 后续步骤 | ❌ 失败 | 依赖前面步骤的数据，且 RawID 冲突 |

> ⚠️ **重要结论**：`--force-reset` 的命名有误导性。它不是"重置迁移"，只是"重置状态文件"。如果任何写入步骤已经部分或全部完成，force-reset 后重新运行**必然失败**。必须手动清空 V4 数据库（或重建数据库）才能真正从头开始迁移。

#### 4.2.4 force-reset 的正确使用场景

只有一种场景是安全的：
- 迁移还没开始或只跑了 `StepSchema`（Schema 幂等）
- 想重新从 `StepSettings` 开始（但 V4 settings 表必须是空的）

其他所有场景下，force-reset 后重试都会失败。

---

### 4.3 状态持久化的完整时序

State 保存发生在以下时机：

| 时机 | 保存内容 | 对应代码 |
|---|---|---|
| 每步成功后 | Step 前进 + 当前 step 的 state | `updateStep()` |
| 批量步骤每批成功后 | offset + 各 ID 集合 | `saveState()` |
| User/Folders 步骤错误时 | 当前内存中的 state | `m.saveState()` 后 return |
| 构造函数中加载 | 从文件读取 state | `loadState()` |

**状态文件位置**：与 V3 配置文件同目录，文件名为 `migration_state.json`。

---

## 5. 占位字段与未接入迁移范围

迁移代码中有多层占位设计，核心目的是保证社区版和 Pro 版的 `state` 文件格式兼容，以及未来扩展时不打乱步骤顺序。

### 5.1 Step 占位

在 [migrator.go#L57-L58](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/migrator.go#L57-L58) 定义了两个步骤占位：

| 占位常量 | 值 | 推测对应 Pro 功能 | 社区版行为 |
|---|---|---|---|
| `Step_CommunityPlaceholder1` | 12 | 礼品码（Gift Code）迁移 | 在 Migrate() 中无代码，自动跳过 |
| `Step_CommunityPlaceholder2` | 13 | 容量包（Storage Pack）迁移 | 在 Migrate() 中无代码，自动跳过 |

**工作原理**：
- 社区版的 `Migrate()` 中，`StepDirectLink` (11) 之后直接到 `StepAvatar` (14)
- 因为 `if m.state.Step <= StepXxx` 的判断方式，步骤编号 12 和 13 自然被跳过
- Pro 版可以在这两个槽位插入自己的迁移函数，而不需要改步骤编号
- 这样社区版和 Pro 版的 state 文件中 Step 数字含义一致，可以互换

### 5.2 State 字段占位

State 结构体中有两个 offset 字段在社区版完全未使用：

| State 字段 | 类型 | 推测对应 | 对应步骤 |
|---|---|---|---|
| `GiftCodeOffset` | int | 礼品码迁移批次偏移 | Step_CommunityPlaceholder1 |
| `StoragePackOffset` | int | 容量包迁移批次偏移 | Step_CommunityPlaceholder2 |

**证据链**：
- V4 inventory 中有 `CreateStoragePackArgs` 类型（[inventory/user.go#L131-L136](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/inventory/user.go#L131-L136)）
- V4 错误码中有 `CodeInvalidGiftCode = 40065`（[pkg/serializer/error.go#L211](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/pkg/serializer/error.go#L211)）
- 但 V3 migrator 的 model 目录中**没有** gift_code 或 storage_pack 模型定义
- 说明 Pro 版的 V3 有这些表，社区版 V3 没有

### 5.3 权限/能力位占位

除了迁移步骤外，V4 的权限和能力位枚举中也有 Community 占位：

**用户组权限占位**（[types.go#L254-L260](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/inventory/types/types.go#L254-L260)）：

```
GroupPermission_CommunityPlaceholder1  // 位置 8
GroupPermission_CommunityPlaceholder2  // 位置 10
GroupPermission_CommunityPlaceholder3  // 位置 13
GroupPermission_CommunityPlaceholder4  // 位置 14
```

**节点能力位占位**（[types.go#L271](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/inventory/types/types.go#L271)）：

```
NodeCapability_CommunityPlaceholder   // 位置 4
```

这些占位与迁移不直接相关，但体现了同一设计思路：在社区版代码中预留槽位，让 Pro 版可以填充对应功能而不打乱枚举编号。

### 5.4 社区版未接入迁移的完整清单

#### 5.4.1 V3 有模型但完全未迁移

| V3 模型 | 文件 | 说明 |
|---|---|---|
| `Tag` | [model/tag.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/tag.go) | 用户自定义标签（文件分类/目录直达） |
| `Task` | [model/task.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/task.go) | 异步任务记录 |

#### 5.4.2 部分字段未迁移

| V3 来源 | 未迁移字段 | 原因推测 |
|---|---|---|
| User | `Authn` | V4 有独立的 Passkey 表，但迁移未做映射 |
| User | `Options`/`OptionsSerialized` | V4 用户设置体系可能已重构 |
| File | `PicInfo` | 元数据字段，可能已整合到 Entity/Metadata |
| File | `UploadSessionID` | 上传会话，迁移时已失效 |
| Policy | `AutoRename` | V4 策略体系可能有变化 |
| Policy | `IsOriginLinkEnable` | 功能已变更 |
| Policy | `OptionsSerialized.MimeType` | 已废弃或整合 |
| Policy | `OptionsSerialized.PlaceholderWithSize` | 已废弃或整合 |
| Group | `OptionsSerialized.OneTimeDownload` | 功能已变更 |
| Share | `PreviewEnabled` | V4 分享体系可能有变化 |
| Share | `SourceName` | 搜索用字段，可能由 File.Name 替代 |
| Node | `MasterKey` | 主从通信架构变化 |
| Node | `Aria2OptionsSerialized.Interval` | 配置体系变化 |
| Node | `Aria2OptionsSerialized.Timeout` | 配置体系变化 |

#### 5.4.3 被丢弃的系统设置（noopMigrator）

通过 `noopMigrator` 直接丢弃的 40+ 项设置，分类：

| 类别 | 设置项 |
|---|---|
| 站点外观 | `siteKeywords`、`defaultTheme`、`theme_options`、`home_view_method`、`share_view_method` |
| 会话超时 | `download_timeout`、`preview_timeout`、`doc_preview_timeout`、`share_download_session_timeout`、`over_used_template` |
| 从节点管理 | `slave_node_retry`、`slave_ping_interval`、`slave_recover_interval`、`slave_transfer_timeout` |
| OneDrive | `onedrive_monitor_timeout`、`onedrive_source_timeout`、`onedrive_callback_check` |
| 邮件 | `mail_activation_template`、`mail_reset_pwd_template` |
| 第三方登录/支付 | `appid`、`appkey`（QQ互联）、`wechat_*`（微信支付，6项）、`custom_payment_*`（3项）、`captcha_TCaptcha_*`（4项） |
| 其他功能 | `hot_share_num`、`max_worker_num`、`max_parallel_transfer`、`secret_key`（V4 有单独的 secret_key 迁移逻辑）、`avatar_size_m`、`avatar_size_s`、`cron_recycle_upload_session`、`initial_files`、`office_preview_service`、`phone_required`、`phone_enabled` |

---

## 6. 幂等处理机制汇总

### 6.1 多层次幂等设计

| 层级 | 机制 | 覆盖范围 |
|---|---|---|
| 步骤级 | `Step` 编号 + `if step <=` 判断 | 整步骤不重复执行 |
| 批次级 | `offset` 偏移量记录 | 批量步骤不重复处理已成功批次 |
| 实体级 | `EntitySources` 缓存（policyID+source 唯一键） | 相同存储路径的 Entity 不重复创建，引用计数+1 |
| 冲突处理 | `FileConflictRename` 映射 | 同名文件自动重命名后重试 |
| 关联校验 | `UserIDs`/`FolderIDs`/`PolicyIDs` 集合校验 | 跳过关联缺失的记录 |

### 6.2 幂等性的局限

1. **V3 数据变更风险**：offset 基于查询偏移量，而非主键 ID。如果迁移过程中 V3 数据库有增删操作，恢复后可能重复或遗漏数据。
2. **无事务一次性步骤**：Node 和 Group 迁移中途失败后无法安全重试。
3. **事务提交与状态保存的时间差**：理论上存在"事务已提交但 state 未写入"的窗口（极小概率）。
4. **force-reset 假象**：只清状态不清数据，导致重试必败。

---

## 7. V3 配置初始化

V3 数据库连接通过 [conf/conf.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/conf/conf.go) 和 [model/init.go](file:///d:/fz/0601-1/solo-dogfeeding/code/48-Cloudreve/application/migrator/model/init.go) 初始化：

1. `conf.Init()` 读取 V3 的 INI 配置文件，映射到 `DatabaseConfig`、`SystemConfig` 等结构体
2. `model.Init()` 根据数据库类型创建 GORM 连接
3. 兼容 `sqlite3`→`sqlite`，`mariadb`→`mysql`

---

## 8. 关键设计决策总结

1. **文件夹与文件统一模型**：V4 将 V3 的 Folder 和 File 统一为 `File` 实体，通过 `Type` 区分
2. **文件 ID 偏移**：文件 ID = V3 ID + LastFolderID，避免文件夹和文件 ID 空间冲突
3. **Entity 解耦**：V4 将存储实体（物理文件引用）从 File 元数据中拆出，支持多版本和引用计数
4. **两阶段文件夹迁移**：先创建文件夹（无父级），再批量设置父级关系，避免外键约束冲突
5. **设置迁移表驱动**：通过 `migrators` map 实现声明式的设置转换规则
6. **远程策略自动建节点**：V3 远程存储策略在 V4 中需要关联 Slave Node，迁移时自动创建
7. **步骤槽位预留**：通过 CommunityPlaceholder 占位步骤，确保社区版与 Pro 版迁移状态兼容
8. **状态文件外置**：状态文件与 V3 配置文件同目录，不依赖 V4 数据库，便于独立管理
9. **批量步骤事务保证**：每批独立事务，offset 在事务提交后更新，确保恢复时不丢不重
10. **部分步骤缺少事务保护**：Node 和 Group 迁移为无事务逐条写入，失败后无法安全重试
