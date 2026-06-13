# Cloudreve 存储策略与驱动抽象分析

## 1. 策略配置：数据模型与持久化

### 1.1 存储策略类型枚举

策略类型在 [types.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/inventory/types/types.go#L302-L312) 中定义了 10 种策略：

| 常量名 | 字符串值 | 说明 |
|---|---|---|
| `PolicyTypeLocal` | `local` | 本地存储 |
| `PolicyTypeQiniu` | `qiniu` | 七牛云 |
| `PolicyTypeUpyun` | `upyun` | 又拍云 |
| `PolicyTypeOss` | `oss` | 阿里云 OSS |
| `PolicyTypeCos` | `cos` | 腾讯云 COS |
| `PolicyTypeS3` | `s3` | S3 兼容存储 |
| `PolicyTypeKs3` | `ks3` | 金山云 KS3 |
| `PolicyTypeOd` | `onedrive` | 微软 OneDrive |
| `PolicyTypeRemote` | `remote` | 远程从节点 |
| `PolicyTypeObs` | `obs` | 华为云 OBS |

### 1.2 StoragePolicy 数据库 Schema

[policy.go (schema)](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/ent/schema/policy.go) 定义了策略的持久化字段，分为 **基础字段** 和 **扩展设置 JSON** 两部分：

#### 基础字段（关系型列）

```go
// [policy.go#L15-L40]
field.String("name")               // 策略名称
field.String("type")               // 策略类型（对应上面 10 种）
field.String("server").Optional()  // 存储服务器地址/Endpoint
field.String("bucket_name").Optional()  // Bucket/容器名
field.Bool("is_private").Optional()     // 是否为私有空间
field.Text("access_key").Optional()     // AccessKey / 用户名
field.Text("secret_key").Optional()     // SecretKey / 密码
field.Int64("max_size").Optional()      // 允许上传的最大单文件大小
field.String("dir_name_rule").Optional()// 目录命名规则模板
field.String("file_name_rule").Optional()// 文件命名规则模板
field.JSON("settings", &types.PolicySetting{}).Optional()  // 扩展设置 JSON
field.Int("node_id").Optional()         // 绑定的从节点 ID
```

#### 边缘关系（Edges）

```go
// [policy.go#L48-L58]
edge.To("groups", Group.Type)      // 被哪些用户组使用
edge.To("files", File.Type)        // 哪些文件使用此策略
edge.To("entities", Entity.Type)   // 哪些实体（版本/缩略图）使用此策略
edge.From("node", Node.Type)       // 关联的从节点
```

### 1.3 PolicySetting 扩展设置（JSON 列）

[PolicySetting](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/inventory/types/types.go#L42-L108) 是一个 JSON 结构，包含约 30 个可选字段，按功能分为：

| 类别 | 字段 | 说明 |
|---|---|---|
| **鉴权** | `Token` `OauthRedirect` `OdDriver` `Region` | 又拍云 Token、OneDrive OAuth、区域代码 |
| **Endpoint** | `ServerSideEndpoint` `S3ForcePathStyle` `UseCname` `SourceAuth` `QiniuUploadCdn` | 服务端/内网 Endpoint、S3 Path 风格、CNAME 开关 |
| **限流** | `TPSLimit` `TPSLimitBurst` | 每秒 API 请求数限制与突发量 |
| **上传** | `ChunkSize` `ChunkConcurrency` `Relay` `PreAllocate` `S3DeleteBatchSize` | 分片大小、并发数、服务端中转、预分配磁盘 |
| **类型过滤** | `FileType` `IsFileTypeDenyList` `NameRegexp` `IsNameRegexpDenyList` | 扩展名黑白名单、文件名正则 |
| **缩略图** | `ThumbExts` `ThumbSupportAllExts` `ThumbMaxSize` `ThumbGeneratorProxy` `NativeMediaProcessing` | 原生缩略图 API、本地代理生成 |
| **媒体元数据** | `MediaMetaExts` `MediaMetaGeneratorProxy` | 原生媒体 API、本地代理提取 |
| **代理下载** | `CustomProxy` `ProxyServer` `InternalProxy` `StreamSaver` | 自定义反代、内置代理、StreamSaver |
| **加密** | `Encryption` | 是否启用服务端加密 |

### 1.4 持久化与缓存层

[StoragePolicyClient](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/inventory/policy.go#L29-L45) 是策略的仓储接口，底层实现 `storagePolicyClient` 使用 **Ent ORM + KV 缓存**：

```go
// [policy.go#L58-L66]
type StoragePolicyClient interface {
    GetByGroup(ctx, group) (*ent.StoragePolicy, error)
    GetPolicyByID(ctx, id int) (*ent.StoragePolicy, error)
    UpdateAccessKey(ctx, policy, token string) error
    ListPolicyByType(ctx, t types.PolicyType) ([]*ent.StoragePolicy, error)
    ListPolicies(ctx, args) (*ListPolicyResult, error)
    Upsert(ctx, policy) (*ent.StoragePolicy, error)
    Delete(ctx, policy) error
}
```

#### 缓存策略

缓存键格式：`storage_policy_{id}`，通过 gob 序列化 `ent.StoragePolicy{}`：

- **GetPolicyByID**：先查 KV 缓存，命中直接返回；否则 DB 查询 → 写入缓存（TTL = -1，永不过期）
- **Upsert / UpdateAccessKey / Delete**：操作 DB 后 **主动删除缓存键**，保证一致性
- **GetByGroup**：走 eager loading（预加载 Node 边），不经过 KV 缓存
- **缓存穿透控制**：通过上下文 `SkipStoragePolicyCache{}` 标记可绕过缓存

#### Upsert 写入的特殊逻辑

```go
// [policy.go#L128-L130]
if policy.Type != types.PolicyTypeOd {
    updateQuery.SetAccessKey(policy.AccessKey)
}
```
OneDrive 策略的 AccessKey 是 OAuth 刷新后自动更新的，**更新策略时不覆盖此字段**。

---

## 2. 驱动初始化流程：从策略到 Handler

### 2.1 核心工厂函数 GetStorageDriver

驱动工厂定义在 [fs.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/manager/fs.go#L65-L90)，通过 **switch-case** 按策略类型分发到对应的构造函数：

```go
func (m *manager) GetStorageDriver(ctx context.Context, policy *ent.StoragePolicy) (driver.Handler, error) {
    switch policy.Type {
    case types.PolicyTypeLocal:    return local.New(policy, m.l, m.config), nil
    case types.PolicyTypeRemote:   return remote.New(ctx, policy, m.settings, m.config, m.l)
    case types.PolicyTypeOss:      return oss.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeCos:      return cos.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeS3:       return s3.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeKs3:      return ks3.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeObs:      return obs.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeQiniu:    return qiniu.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeUpyun:    return upyun.New(ctx, policy, m.settings, m.config, m.l, m.dep.MimeDetector(ctx))
    case types.PolicyTypeOd:       return onedrive.New(ctx, policy, m.settings, m.config, m.l, m.dep.CredManager())
    default:                       return nil, ErrUnknownPolicyType
    }
}
```

#### 构造函数依赖注入特征

| 策略类型 | 返回 error | 关键依赖 |
|---|---|---|
| `local` | 否 | Logger, Config |
| `remote` | 是 | Setting, Config, Logger |
| `oss/cos/s3/ks3/obs/qiniu/upyun` | 是 | Setting, Config, Logger, **MimeDetector** |
| `onedrive` | 是 | Setting, Config, Logger, **CredManager** |

> **设计要点**：对象存储驱动需要 MIME 探测器在上传时推断 Content-Type；OneDrive 需要凭证管理器处理 OAuth 刷新。

### 2.2 从节点策略类型转换 CastStoragePolicyOnSlave（含 Bug）

[fs.go#L31-L63](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/manager/fs.go#L31-L63) 在 **Slave 模式**下对策略做类型转换：

| 分支 | 策略类型 | 预期行为 | 当前状态 |
|---|---|---|---|
| 1 | `Remote` → `Local` | 当 Remote 策略的 NodeID 等于当前从节点 ID 时，转为 Local 直接访问本地磁盘 | ✅ 正常（return &policyCopy） |
| 2 | `Local` → `Remote` | Slave 上的 Local 策略转为 Remote，通过 Master 的 RPC 操作 | ✅ 正常（return &policyCopy） |
| 3 | `OSS` → `OSS` | 清空 `ServerSideEndpoint`，从节点只能用公网 Endpoint 访问 | **🐛 Bug**：创建副本后未 return，修改被丢弃 |

**OSS 分支 Bug 详情**：

```go
// [fs.go#L55-L60]
} else if policy.Type == types.PolicyTypeOss {
    policyCopy := *policy
    if policyCopy.Settings != nil {
        policyCopy.Settings.ServerSideEndpoint = ""  // 修改了副本
    }
    // ❌ 缺少: return &policyCopy
    //    执行继续落到第 62 行，返回原始 policy，上述清空操作完全失效
}
return policy  // ⚠️ 返回的是未修改的原始 policy
```

**影响**：在 Slave 节点上使用 OSS 策略时，`ServerSideEndpoint`（内网 Endpoint）未被清空，从节点可能尝试通过不可达的内网地址访问 OSS，导致请求失败或超时。

**修复方案**：在 `policyCopy.Settings.ServerSideEndpoint = ""` 之后添加 `return &policyCopy`。

### 2.3 典型驱动初始化示例

#### Local 驱动（最简单）

[local.go#L44-L59](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/driver/local/local.go#L44-L59)：
```go
type Driver struct {
    Policy     *ent.StoragePolicy
    httpClient request.Client
    l          logging.Logger
    config     conf.ConfigProvider
}

func New(p *ent.StoragePolicy, l logging.Logger, config conf.ConfigProvider) *Driver {
    return &Driver{
        Policy:     p,
        l:          l,
        httpClient: request.NewClient(config, request.WithLogger(l)),
        config:     config,
    }
}
```
无状态，不做任何网络调用，直接可用。

#### OSS 驱动（含客户端初始化）

[oss.go#L83-L101](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/driver/oss/oss.go#L83-L101)：
```go
func New(ctx context.Context, policy *ent.StoragePolicy, ...) (*Driver, error) {
    chunkSize := policy.Settings.ChunkSize
    if chunkSize == 0 { chunkSize = 25 << 20 } // 默认 25MB

    driver := &Driver{
        policy: policy, settings: settings, chunkSize: chunkSize,
        config: config, l: l, mime: mime,
        httpClient: request.NewClient(config, request.WithLogger(l)),
    }
    // 构造后立即调用 InitOSSClient 初始化 SDK 客户端
    return driver, driver.InitOSSClient(false)
}
```
`InitOSSClient` 从策略中提取 AccessKey / SecretKey / Endpoint / Region，构造阿里云 OSS SDK 客户端。

#### OneDrive 驱动（OAuth 依赖）

[onedrive.go#L55-L72](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/driver/onedrive/onedrive.go#L55-L72)：
```go
func New(ctx context.Context, policy *ent.StoragePolicy, ..., cred credmanager.CredManager) (*Driver, error) {
    chunkSize := policy.Settings.ChunkSize
    if chunkSize == 0 { chunkSize = 50 << 20 } // 默认 50MB

    c := NewClient(policy, request.NewClient(...), cred, l, settings, chunkSize)
    return &Driver{
        policy: policy, client: c, settings: settings,
        l: l, config: config, chunkSize: chunkSize,
    }, nil
}
```
通过 `CredManager` 获取和刷新 OAuth Access Token，不直接持有凭证。

### 2.4 驱动获取的完整调用链

```
上层业务 (Upload/Source/Delete)
    ↓
manager.GetStorageDriver(ctx, policy)
    ↓
manager.CastStoragePolicyOnSlave(ctx, policy)   // Slave 模式下类型转换
    ↓
switch policy.Type → 调用具体驱动 New()
    ↓
具体驱动构造：
  - 解析 PolicySetting 默认值（ChunkSize 等）
  - 初始化云厂商 SDK 客户端（可能返回 error）
  - 注入依赖 (Logger/MimeDetector/CredManager)
    ↓
返回 driver.Handler 接口实例
```

---

## 3. 驱动能力差异：接口与特性矩阵

### 3.1 Handler 接口定义

所有驱动实现 [Handler](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/driver/handler.go#L44-L87) 接口，共 12 个方法：

| 方法 | 功能 | 约束/备注 |
|---|---|---|
| `Put` | 服务端流式上传 | `ctx` 取消时清理临时文件；`ModeOverwrite` 控制覆盖 |
| `Delete` | 批量删除 | 返回删除失败的路径列表 + 最后一个错误 |
| `Open` | 打开物理文件为 `*os.File` | **仅** 支持 `InboundGet` 能力的驱动 |
| `LocalPath` | 获取本地绝对路径 | **仅** 支持 `InboundGet` 能力的驱动 |
| `Thumb` | 获取缩略图 URL | 不支持时返回 `"not implemented"` error |
| `Source` | 获取下载/外链 URL | 传入过期时间、下载标记、限速、显示名 |
| `Token` | 生成客户端上传凭证 | 含分片上传 URL 列表、完成 URL、回调配置 |
| `CancelToken` | 取消有状态上传会话 | 如 S3 AbortMultipartUpload |
| `CompleteUpload` | Sentinel 模式下完成校验 | 检查上传后文件大小一致性 |
| `List` | 递归列取物理对象 | 支持进度回调 `onProgress` |
| `Capabilities` | 返回能力描述对象 | 每次调用返回 `Capabilities` 结构体 |
| `MediaMeta` | 原生媒体元数据提取 | 图片 EXIF / 音视频信息等 |

### 3.2 Capabilities 能力描述对象

[Capabilities](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/driver/handler.go#L89-L111) 结构体：

```go
type Capabilities struct {
    StaticFeatures         *boolset.BooleanSet  // 静态布尔能力集
    MaxSourceExpire        time.Duration        // 外链最大过期时间
    MinSourceExpire        time.Duration        // 外链最小过期时间
    MediaMetaSupportedExts []string             // 支持原生元数据的扩展名
    MediaMetaProxy         bool                 // 是否用本地代理生成元数据
    ThumbSupportedExts     []string             // 支持原生缩略图的扩展名
    ThumbSupportAllExts    bool                 // 是否所有扩展名都支持缩略图
    ThumbMaxSize           int64                // 缩略图最大文件大小
    ThumbProxy             bool                 // 是否用本地代理生成缩略图
    BrowserRelayedDownload bool                 // 是否用 StreamSaver 中转下载
}
```

### 3.3 静态能力标志（HandlerCapability）

[handler.go#L13-L24](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/driver/handler.go#L13-L24) 定义了 3 个 `HandlerCapability`，通过 [BooleanSet](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/boolset/boolset.go) 位集存储：

| 能力标志 | 值 | 说明 |
|---|---|---|
| `HandlerCapabilityProxyRequired` | 0 | 需要 Cloudreve 代理才能获取文件内容 |
| `HandlerCapabilityInboundGet` | 1 | 可直接读取本地文件（支持 `Open`/`LocalPath`） |
| `HandlerCapabilityUploadSentinelRequired` | 2 | 无回调合规机制，需 Sentinel 定时任务校验上传完成 |

BooleanSet 是紧凑的字节位图实现，按 `flag/8` 定位字节、`flag%8` 定位位。

### 3.4 各驱动能力声明对比

| 驱动 | ProxyRequired | InboundGet | UploadSentinel | 备注 |
|---|---|---|---|---|
| **local** | ✅ | ✅ | ❌ | 唯一具备 InboundGet 的驱动 |
| **s3** | ❌ | ❌ | ✅ | 无合规回调机制 |
| **onedrive** | ❌ | ❌ | ✅ | 无合规回调机制 |
| **oss/cos/obs** | ❌ | ❌ | ❌ | 支持上传回调 (Callback) |
| **qiniu/upyun/ks3** | ❌ | ❌ | ❌ | 各有回调机制 |
| **remote** | ❌ | ❌ | ❌ | 委托给从节点处理 |

#### 声明方式示例

Local 驱动在 `init()` 中批量设置：
```go
// [local.go#L36-L41]
func init() {
    boolset.Sets(map[driver.HandlerCapability]bool{
        driver.HandlerCapabilityProxyRequired: true,
        driver.HandlerCapabilityInboundGet:    true,
    }, capabilities.StaticFeatures)
}
```

S3 / OneDrive 在 `init()` 中开启 Sentinel：
```go
// [s3.go#L68-L72]
func init() {
    boolset.Sets(map[driver.HandlerCapability]bool{
        driver.HandlerCapabilityUploadSentinelRequired: true,
    }, features)
}
```

### 3.5 Sentinel 能力的业务影响

当驱动具备 `UploadSentinelRequired` 时，[CreateUploadSession](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/manager/upload.go#L120-L134) 会：

1. 创建 `UploadSentinelCheckTask` 定时任务
2. 将任务投递到 `EntityRecycleQueue`
3. 将 `SentinelTaskID` 记录到 UploadSession
4. 过期后若未收到回调 → 任务清理占位文件

具备回调机制的驱动（OSS/COS/Qiniu/Upyun/OBS/Ks3）**不创建** Sentinel，依赖云厂商回调通知完成。

### 3.6 动态能力差异（Capabilities 各字段填充策略）

| 能力字段 | local | oss | s3 | onedrive | remote |
|---|---|---|---|---|---|
| `StaticFeatures` | Proxy+Inbound | 空集 | Sentinel | Sentinel | 空集 |
| `MediaMetaProxy` | `true` | `Settings.MediaMetaGeneratorProxy` | `Settings.MediaMetaGeneratorProxy` | `Settings.MediaMetaGeneratorProxy` | — |
| `ThumbProxy` | `true` | `Settings.ThumbGeneratorProxy` | `Settings.ThumbGeneratorProxy` | `Settings.ThumbGeneratorProxy` | — |
| `ThumbSupportedExts` | 空 | `Settings.ThumbExts` | 空 | `Settings.ThumbExts` | — |
| `ThumbSupportAllExts` | false | `Settings.ThumbSupportAllExts` | false | `Settings.ThumbSupportAllExts` | — |
| `MaxSourceExpire` | — | 7天 (V4 Sign 限制) | 7天 | — | — |
| `BrowserRelayedDownload` | false | false | false | `Settings.StreamSaver` | — |
| `MediaMetaSupportedExts` | 空 | `Settings.MediaMetaExts` (需 NativeMediaProcessing=true) | 空 | 空 | — |

> **设计意图**：能力不是硬编码的，**大量能力由管理员在 PolicySetting 中配置**，同一驱动类型的不同策略实例可以表现出完全不同的能力组合。

---

## 4. 错误语义：错误类型、错误码与传播

### 4.1 AppError 统一错误结构

[error.go#L16-L67](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/serializer/error.go#L16-L67) 定义了整个应用的标准错误类型：

```go
type AppError struct {
    Code     int     // 业务错误码
    Msg      string  // 面向用户的可读消息
    RawError error   // 底层原始错误（Go error 链）
}
```

**关键方法**：
- `Error()`：优先拼接 `Msg + ": " + RawError.Error()`
- `ErrCode()`：递归解包 RawError 中的 AppError，返回**最内层**错误码
- `Unwrap()`：支持 `errors.As` / `errors.Is` 链式匹配
- `WithError(raw)`：派生新 AppError，替换 RawError 保留 Code/Msg

### 4.2 错误码体系

错误码约定见 [error.go#L69-L284](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/serializer/error.go#L69-L284)：

| 码段 | 含义 | 示例 |
|---|---|---|
| `2xx` | 兼容 HTTP 状态 | `203` 未完全成功 |
| `4xx` | 兼容 HTTP 状态 | `401` 未登录、`403` 无权限、`404` 未找到、`409` 冲突 |
| `400xx` | **客户端错误**（用户/前端） | `40002` 上传失败、`40004` 对象存在、`40006` 策略不允许、`40055` 元数据不一致 |
| `500xx` | **服务端错误**（系统/内部） | `50001` DB 错误、`50005` 内部设置、`50007` 回调失败、`50010` 节点离线 |

#### 存储/驱动相关的关键错误码

| 错误码 | 常量名 | 触发场景 |
|---|---|---|
| `40002` | `CodeUploadFailed` | 上传过程底层 IO 错误 |
| `40004` | `CodeObjectExist` | 目标路径对象已存在（非覆盖模式） |
| `40006` | `CodePolicyNotAllowed` | 策略类型不允许该操作（如非 Local 走 ConfirmUploadSession） |
| `40011` | `CodeUploadSessionExpired` | 上传会话过期 |
| `40012` | `CodeInvalidChunkIndex` | 分片序号超出范围 |
| `40035` | `CodePolicyNotExist` | 策略 ID 在 DB 中不存在 |
| `40049` | `CodeFileTooLarge` | 文件超过策略 MaxSize 或用户组容量 |
| `40050` | `CodeFileTypeNotAllowed` | 扩展名命中黑名单 |
| `40054` | `CodeConflictUploadOngoing` | 同路径已有上传进行中 |
| `40055` | `CodeMetaMismatch` | Sentinel 校验时文件大小与声明不一致 |
| `40057` | `CodePolicyChanged` | 用户组策略在操作期间变更 |
| `50005` | `CodeInternalSetting` | 策略类型未知 (`ErrUnknownPolicyType`) |
| `50007` | `CodeCallbackError` | 从节点 → 主节点回调失败 |
| `50011` | `CodeQueryMetaFailed` | 查询物理对象元信息失败 |

### 4.3 fs 层预定义错误

[fs.go#L28-L45](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/fs/fs.go#L28-L45) 在更高层定义了带业务语义的 AppError 常量：

```go
var (
    ErrDirectLinkInvalid    = serializer.NewError(serializer.CodeNotFound, "Direct link invalid", nil)
    ErrUnknownPolicyType    = serializer.NewError(serializer.CodeInternalSetting, "Unknown policy type", nil)
    ErrFileExisted          = serializer.NewError(serializer.CodeObjectExist, "Object existed", nil)
    ErrInsufficientCapacity = serializer.NewError(serializer.CodeInsufficientCapacity, "Insufficient capacity", nil)
    ErrMetaMismatch         = ... // 等 15+ 种
)
```

驱动层可以直接返回这些 `AppError`（如 S3 的 `Put` 返回 `fs.ErrFileExisted`），也可以返回原始 Go error，由上层包装。

### 4.4 错误传播路径示例

#### 场景一：上传文件冲突（S3 → S3）

```
s3.Put(ctx, file)
    ├─ handler.Meta(ctx, savePath)  // 检查是否存在
    │   └─ 存在 → return nil (无 err)
    └─ 存在检测通过 → return fs.ErrFileExisted  // AppError{Code:40004}
        ↓
manager.Upload(...)
    └─ 直接返回（AppError 不包装）
        ↓
Service/Controller 层
    └─ serializer.Err(ctx, err)
        ├─ errors.As 识别为 AppError
        ├─ 取 Code=40004, Msg="Object existed"
        └─ 构造 JSON Response 返回前端
```

#### 场景二：未知策略类型（Manager 层）

```
manager.GetStorageDriver(ctx, policy)
    └─ switch default → return nil, ErrUnknownPolicyType
        // AppError{Code:50005, Msg:"Unknown policy type", RawError:nil}
        ↓
CreateUploadSession
    ├─ err != nil → m.OnUploadFailed(...)  // 回滚
    └─ return nil, err
        ↓
Controller → serializer.Err(ctx, err) → Response{Code:50005, Msg:"Unknown policy type"}
```

#### 场景三：Sentinel 校验失败（OneDrive）

```
onedrive.CompleteUpload(ctx, session)
    ├─ client.Meta(...) → 获取实际文件大小
    └─ res.Size != session.Props.Size
        └─ return serializer.NewError(
               CodeMetaMismatch,
               fmt.Sprintf("File size not match, expected: %d, actual: %d", ...),
               nil
           )
        ↓
manager.CompleteUpload → 向上传播
    ↓
回调接口 → 返回错误响应给云厂商 / 记录失败
```

### 4.5 AggregateError 批量错误

对于批量删除、批量移动等操作，使用 [AggregateError](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/serializer/error.go#L384-L449) 聚合多个子错误：

```go
type AggregateError struct {
    errs map[string]error  // key=操作对象ID, value=该操作的错误
}
```

- 单错误时直接透出原始错误消息
- 多错误时聚合为 `CodeBatchOperationNotFullyCompleted (40081)`
- 通过 `Expand(ctx)` 展开为 `map[string]Response` 返回给前端

### 4.6 错误响应包装规则

[ErrWithDetails](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/serializer/error.go#L340-L376) 是最终返回前端的错误构造函数，执行以下步骤：

1. 尝试 `errors.As(err, &appError)` 解包内层 AppError
2. 若找到，**使用内层 AppError 的 Code/Msg**（支持多层嵌套时取最底层业务码）
3. 特殊类型处理：
   - `CodeLockConflict` → 附加锁冲突详情
   - `CodeBatchOperationNotFullyCompleted` → 展开聚合错误详情
4. **非 Release 模式**才把底层错误字符串放入 `Response.Error` 字段
5. 附加 `CorrelationID` 用于链路追踪

> **生产环境安全提示**：Release 模式下 `Response.Error` 字段为空，只暴露 `Code` 和 `Msg`，避免泄露堆栈、路径、凭证等敏感信息。

### 4.7 Delete 方法的特殊错误语义

Handler 接口的 `Delete` 有独特的双返回值约定：
```go
Delete(ctx context.Context, files ...string) ([]string, error)
```

- 第一个返回值：**删除失败的文件路径列表**（部分失败场景）
- 第二个返回值：**最后一个**底层错误（仅用于日志/调试）

S3 驱动的实现具有代表性：单文件删除忽略 `NoSuchKey`（幂等），批量删除时汇总 S3 返回的每个对象错误到 `failed` 列表。上层根据 `len(failed) > 0` 判断是否完全成功。

---

## 5. 附录：文件索引

| 组件 | 文件路径 |
|---|---|
| 策略 Schema | [ent/schema/policy.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/ent/schema/policy.go) |
| 策略类型常量 | [inventory/types/types.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/inventory/types/types.go#L302-L312) |
| PolicySetting | [inventory/types/types.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/inventory/types/types.go#L42-L108) |
| 策略仓储接口 | [inventory/policy.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/inventory/policy.go) |
| Handler 接口定义 | [pkg/filemanager/driver/handler.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/driver/handler.go) |
| 驱动工厂 | [pkg/filemanager/manager/fs.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/manager/fs.go) |
| 上传会话创建 | [pkg/filemanager/manager/upload.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/manager/upload.go) |
| Local 驱动 | [pkg/filemanager/driver/local/local.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/driver/local/local.go) |
| OSS 驱动 | [pkg/filemanager/driver/oss/oss.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/driver/oss/oss.go) |
| S3 驱动 | [pkg/filemanager/driver/s3/s3.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/driver/s3/s3.go) |
| OneDrive 驱动 | [pkg/filemanager/driver/onedrive/onedrive.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/driver/onedrive/onedrive.go) |
| Remote 驱动 | [pkg/filemanager/driver/remote/remote.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/driver/remote/remote.go) |
| BooleanSet 位集 | [pkg/boolset/boolset.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/boolset/boolset.go) |
| 错误体系 | [pkg/serializer/error.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/serializer/error.go) |
| FS 层错误常量 | [pkg/filemanager/fs/fs.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/pkg/filemanager/fs/fs.go#L28-L45) |
| 依赖注入容器 | [application/dependency/dependency.go](file:///d:/fz/0601-1/solo-dogfeeding/code/42-Cloudreve/application/dependency/dependency.go) |
