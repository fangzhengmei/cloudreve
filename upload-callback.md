# 分块上传三段路径校验分析笔记

## 一、整体架构总览

分块上传分为三条核心路径，它们通过 `UploadSessionID` 和 `CallbackSecret` 作为关联纽带：

```
┌────────────────────────┐     ┌──────────────────────┐     ┌──────────────────────┐
│  ① 临时凭证签发        │────▶│  ② 客户端直传        │────▶│  ③ 服务端回调        │
│  (服务端 → 客户端)     │     │  (客户端 → 存储商)    │     │  (存储商 → 服务端)   │
└────────────────────────┘     └──────────────────────┘     └──────────────────────┘
         │                              │                              │
         ▼                              ▼                              ▼
  UploadSession存入KV           各存储凭证/预签名URL         签名校验+会话恢复+完成上传
```

**关联标识：**
- `UploadSessionID`：UUID v4，用于 KV 键值查找会话
- `CallbackSecret`：32位加密随机字符串，仅作为回调 URL 的路径参数之一，**提供路径不可枚举性（被动防御），不参与显式校验**
- KV 缓存 Key：`callback_{sessionID}`

> ⚠️ **重要澄清**：CallbackSecret 虽然存在于回调 URL 路径中，但服务端回调处理链中**从未读取 URL 中的 `:key` 参数，也从未与 KV 中存储的 `CallbackSecret` 做对比校验**。它的安全性完全依赖"无法枚举32位随机字符串"。详见下方"第五章"的详细证据链。

---

## 二、第一段路径：临时凭证签发

### 2.1 入口与核心流程

**入口函数：** [CreateUploadSession](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L50-L147)

```
请求验证 → 会话准备(PrepareUpload) → 驱动Token签发 → 哨兵任务(可选) → 存入KV缓存
```

### 2.2 会话准备 PrepareUpload

**实现位置：** [DBFS.PrepareUpload](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/fs/dbfs/upload.go#L72-L260)

核心校验与操作：

| 步骤 | 说明 | 异常处理 |
|------|------|---------|
| 导航器权限校验 | 验证目标路径的上传权限 | 返回 ErrNoPermissionErr |
| 符号链接检查 | 禁止向符号目录上传 | 返回 ErrSymbolicFolderFound |
| 文件存在性检查 | 新文件/新版本的合法校验 | 返回 ErrFileExisted / ErrPathNotExist |
| 所有权校验 | 必须是文件所有者（除非绕过） | 返回 ErrOwnerOnly |
| 分布式锁获取 | 锁定目标路径，有效期=TTL | 返回 ErrLockConflict |
| 存储策略选择 | 根据父目录确定存储策略 | - |
| 容量校验 | 用户配额检查 | 返回 ErrInsufficientCapacity |
| 占位文件创建 | 事务内创建 placeholder file/entity | 事务回滚 |
| CallbackSecret 生成 | `util.RandStringRunesCrypto(32)` 密码学随机 | - |

**关键会话字段构造（[dbfs/upload.go:232-252](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/fs/dbfs/upload.go#L232-L252)）：**

```go
session := &fs.UploadSession{
    Props:           &UploadProps{...},  // URI/Size/SavePath/ExpireAt等
    FileID:          fileId,             // 占位文件ID
    EntityID:        entityId,           // 占位实体ID
    UID:             f.user.ID,          // 上传者UID
    Policy:          policy,             // 存储策略（含AK/SK）
    CallbackSecret:  util.RandStringRunesCrypto(32),  // 回调密钥
    LockToken:       lockToken,          // 文件锁token
    NewFileCreated:  !fileExisted,       // 是否新建文件
    Importing:       req.ImportFrom != nil,
}
```

### 2.3 各存储驱动的 Token 签发

#### 2.3.1 凭证签发统一接口

定义于 [handler.go:69](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/handler.go#L69)：

```go
Token(ctx context.Context, uploadSession *fs.UploadSession, file *fs.UploadRequest) (*fs.UploadCredential, error)
```

#### 2.3.2 回调 URL 构造

所有驱动统一使用 [MasterSlaveCallbackUrl](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/cluster/routes/routes.go#L46-L49)：

```
{SiteURL}/api/v4/callback/{driverType}/{sessionID}/{callbackSecret}
```

回调 URL 同时嵌入了 sessionID 和 callbackSecret，存储商回调时会原样请求该 URL。

#### 2.3.3 各驱动签发方式对比

| 驱动类型 | 签发方式 | 客户端直传凭证 | 回调携带 | 代码位置 |
|---------|---------|--------------|---------|---------|
| **阿里云OSS** | 每分片预签名URL + CompleteURL | `UploadURLs[]` + `CompleteURL` + `Callback`(base64编码的回调策略) | 签名公钥URL + Authorization签名 + JSON Body | [oss.go:490-589](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/oss/oss.go#L490-L589) |
| **腾讯云COS** | 每分片预签名URL + CompleteURL | `UploadURLs[]` + `CompleteURL` | GET 请求（无签名，靠URL路径校验） | [cos.go:452-542](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/cos/cos.go#L452-L542) |
| **AWS S3** | 每分片预签名URL + CompleteURL | `UploadURLs[]` + `CompleteURL` | GET 请求（无签名） | [s3.go:335-411](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/s3/s3.go#L335-L411) |
| **七牛云** | upToken(上传凭证) + InitParts | `Credential`(upToken) + `UploadURLs[]` + `UploadID` | Authorization签名 + JSON Body | [qiniu.go:363-408](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/qiniu/qiniu.go#L363-L408) |
| **又拍云** | Policy(base64) + HMAC签名 | `UploadPolicy` + `Credential` | Content-MD5 + Date + Authorization签名 | [upyun.go:280-321](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/upyun/upyun.go#L280-L321) |
| **华为OBS/金山KS3** | 同S3模式 | 每分片预签名URL | GET / POST 请求 | 对应驱动 Token 方法 |
| **OneDrive** | Graph API uploadSession | `UploadURL` | 无独立回调（客户端完成后通知） | [onedrive.go:167](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/onedrive/onedrive.go#L167) |

#### 2.3.4 OSS 回调策略示例（[oss.go:503-508](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/oss/oss.go#L503-L508)）

```go
callbackPolicy := CallbackPolicy{
    CallbackURL:      uploadSession.Callback,  // 包含sessionID和secret
    CallbackBody:     `{"name":${x:fname},"source_name":${object},"size":${size},"pic_info":"${imageInfo.width},${imageInfo.height}"}`,
    CallbackBodyType: "application/json",
    CallbackSNI:      true,
}
// 序列化为 JSON 后 base64 编码，作为 complete 请求的 callback 参数
```

### 2.4 哨兵任务（UploadSentinelCheckTask）

**触发条件：** 驱动声明了 `HandlerCapabilityUploadSentinelRequired` 能力（当前 COS 声明了该能力，[cos.go:96-99](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/cos/cos.go#L96-L99)）

**核心逻辑（[manager/upload.go:450-548](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L450-L548)）：**

```
创建上传会话时排队 → 执行时间 = ExpireAt + 5分钟宽限期
                        │
                        ▼
            执行时检查任务状态 ──已完成──▶ 正常结束（回调已处理）
                  │
                 未完成
                  │
                  ▼
        删除物理文件 + CancelToken + 清理占位实体
```

**宽限期常量：** `uploadSentinelCheckMargin = 5 * time.Minute`（[manager/upload.go:460](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L460)）

### 2.5 KV 缓存写入

**位置：** [manager/upload.go:136-144](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L136-L144)

```go
err = m.kv.Set(
    UploadSessionCachePrefix + req.Props.UploadSessionID,  // "callback_" + UUID
    *uploadSession,
    max(1, int(req.Props.ExpireAt.Sub(time.Now()).Seconds())),  // TTL = 剩余有效期
)
```

### 2.6 过期窗口配置

| 参数 | 默认值 | 配置项 | 代码位置 |
|------|-------|-------|---------|
| 上传会话 TTL | 86400秒（24小时） | `upload_session_timeout` | [provider.go:691-693](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/setting/provider.go#L691-L693) |
| 哨兵任务宽限期 | 300秒（5分钟） | 硬编码常量 | [manager/upload.go:460](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L460) |
| OSS 公钥缓存 TTL | 604800秒（7天） | 硬编码常量 | [callback.go:60](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/oss/callback.go#L60) |

---

## 三、第二段路径：客户端直传

### 3.1 数据流向

客户端获取到 `UploadCredential` 后，不经过 Cloudreve 服务端，直接向存储商发送上传请求：

```
客户端
  │
  ├─ PUT {UploadURLs[i]}  ──────────────▶  OSS/S3/COS 存储节点（分片i）
  │      Header: 预签名信息已嵌入URL
  │      Body:   分片二进制数据
  │
  └─ POST {CompleteURL}  ──────────────▶  OSS/S3/COS 存储节点（合并分片）
         Header:  x-oss-callback / x-cos-meta-callback 等
         Body:    分片ETag列表等
```

### 3.2 各驱动客户端直传特点

| 驱动 | 分片上传方式 | 合并触发 | 回调触发时机 |
|-----|------------|---------|------------|
| OSS | 逐个 PUT 已签名的分片URL | 客户端调用 CompleteURL | Complete 请求完成后，OSS 主动回调 |
| COS/S3/KLS3 | 逐个 PUT 已签名的分片URL | 客户端调用 CompleteURL | 客户端主动 GET 回调URL（无存储商回调） |
| 七牛 | 分片直传到 upHost，带 upToken | 客户端调用七牛合并接口 | 七牛主动 POST 回调 |
| 又拍云 | POST 表单上传，带 Policy + Signature | 表单上传即完成（小文件） | 又拍云主动 POST 回调 |

**注意：** COS/S3/KLS3 等驱动没有存储商的主动回调机制，回调由客户端上传完成后主动调用，所以这些驱动需要**哨兵任务**兜底。

---

## 四、第三段路径：服务端回调

### 4.1 回调路由总览

路由定义于 [router.go:472-545](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/router.go#L472-L545)，统一格式：

```
POST|GET /api/v4/callback/{driverType}/{sessionID}/{callbackSecret}
```

### 4.2 中间件执行链（以 OSS 为例）

```
请求到达
  │
  ▼
① UseUploadSession(PolicyTypeOss)
  │  ├─ 从 URL Path 提取 sessionID
  │  ├─ 从 KV 获取 callback_{sessionID} → 不存在则 401（会话过期）
  │  ├─ 验证 Policy.Type 匹配（防止跨策略回调）
  │  └─ 通过 callbackSession.UID 恢复用户上下文
  │
  ▼
② OSSCallbackAuth()  [仅部分驱动有此层]
  │  └─ RSA 公钥签名验证
  │
  ▼
③ OSSCallbackValidate  [业务校验层]
  │  └─ 校验回调Body中的size与会话记录的size一致
  │
  ▼
④ ProcessCallback()
  │  ├─ 驱动层 CompleteUpload
  │  ├─ DBFS 层 CompleteUpload（实体转正、解锁等）
  │  ├─ 取消哨兵任务（若有）
  │  ├─ 提交媒体元数据/全文索引任务
  │  └─ 从 KV 删除会话
  │
  ▼
返回 200 OK
```

### 4.3 第一层：UseUploadSession 会话恢复

**实现位置：** [auth.go:178-217](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L178-L217)

核心校验（对照源码逐行分析）：

```go
// 1. sessionID 非空检查（从 URL Path 读 :sessionID）
sessionID := c.Param("sessionID")
if sessionID == "" { return CodeParamErr }

// 2. KV 会话存在性检查（已过期则不存在）
callbackSessionRaw, exist := dep.KV().Get("callback_" + sessionID)
if !exist { return CodeUploadSessionExpired }

// 3. 策略类型匹配检查（防止用OSS的会话ID去调COS回调）
callbackSession := callbackSessionRaw.(fs.UploadSession)
if callbackSession.Policy.Type != string(policyType) { return CodePolicyNotAllowed }

// 4. 恢复用户上下文
SetUserCtx(c, callbackSession.UID)
```

> ⚠️ **关键代码对照：** 路由定义中 URL 格式是 `{driverType}/{sessionID}/{key}`（见 [router.go:476-545](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/router.go#L476-L545)），第三个路径参数命名为 `:key` 即 CallbackSecret。但在整个 `uploadCallbackCheck` 函数中：
> - **完全没有** `c.Param("key")`
> - **完全没有** 从 `c.Param("key")` 读取 URL 中的 CallbackSecret
> - **完全没有** 与会话中存储的 `callbackSession.CallbackSecret` 做任何形式的对比校验
>
> 在后续所有中间件（OSSCallbackAuth / RemoteCallbackAuth / QiniuCallbackValidate / UpyunCallbackAuth）中，也没有任何一处代码读取或校验 URL 中的 `:key`。
>
> 代码证据：全局搜索 `c.Param(["']key["'])` 在 `middleware/` 目录下 **0 条匹配**；全局搜索 CallbackSecret 只出现在"生成 CallbackSecret"、"嵌入回调URL"和"返回给客户端"三类位置，**从未出现在"读取URL参数并比较"的代码位置**。

### 4.4 第二层：各驱动签名核验

#### 4.4.1 阿里云 OSS 签名核验

**中间件：** [OSSCallbackAuth](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L243-L259)
**算法实现：** [oss.VerifyCallbackSignature](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/oss/callback.go#L90-L123)

```
请求
  │
  ├─ Header: x-oss-pub-key-url (Base64编码的公钥下载地址)
  ├─ Header: Authorization (Base64编码的RSA签名)
  └─ Body:   JSON回调内容
  │
  ▼
① 公钥获取与校验
  │  ├─ 优先从 KV 缓存读取（7天TTL）
  │  ├─ 从 Header 解码公钥URL
  │  ├─ URL白名单校验：必须以 http(s)://gosspublic.alicdn.com/ 开头
  │  └─ 下载公钥并存入缓存
  │
  ▼
② 签名原文构造
  │ 待签名内容 = URL.Path + "\n" + Body
  │ MD5值 = md5(待签名内容)
  │
  ▼
③ RSA 验签
  └─ rsa.VerifyPKCS1v15(pubKey, crypto.MD5, MD5值, Authorization解码值)
```

**异常分支：**
- 公钥 URL 不在白名单 → `public key url invalid`
- Authorization 头缺失 → `no authorization field in Request header`
- PEM 解码失败 → `pubBlock not exist`
- RSA 验签失败 → 对应 crypto 错误

#### 4.4.2 七牛云签名核验

**中间件：** [QiniuCallbackValidate](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/controllers/callback.go#L34-L54)

```go
// 使用七牛 SDK 自带的验证方法
mac := qbox.NewMac(session.Policy.AccessKey, session.Policy.SecretKey)
ok, err := mac.VerifyCallback(c.Request)
```

七牛 SDK 内部逻辑：基于请求的 `Authorization` 头，使用 AK/SK 计算 HMAC-SHA1 签名并对比。

#### 4.4.3 又拍云签名核验

**中间件：** [UpyunCallbackAuth](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/controllers/callback.go#L80-L90)
**算法实现：** [upyun.ValidateCallback](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/upyun/upyun.go#L356-L384)

```
请求
  │
  ├─ Header: Content-Md5
  ├─ Header: Date
  ├─ Header: Authorization (格式: UPYUN AK:Signature)
  └─ Body:   表单数据
  │
  ▼
① Body MD5 校验
  │ 计算 body_md5 = hex(md5(Body))
  │ 对比 body_md5 == Content-Md5 头
  │
  ▼
② HMAC-SHA1 签名计算
  │ key = hex(md5(SecretKey))
  │ 待签名字符串 = "POST&" + URL.Path + "&" + Date + "&" + Content-Md5
  │ signature = "UPYUN " + AK + ":" + base64(hmac_sha1(key, 待签名字符串))
  │
  ▼
③ 对比 signature == Authorization 头
```

#### 4.4.4 远程（从机节点）签名核验

**中间件：** [RemoteCallbackAuth](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L220-L240)

使用从机的 `SlaveKey` 作为 HMAC 密钥，对整个请求进行签名校验。

#### 4.4.5 无独立签名校验的驱动

COS、S3、KS3、OBS、OneDrive：回调路径只经过 UseUploadSession，依赖 sessionID 的不可枚举性和 CallbackSecret 的随机性来保证安全。COS/S3 等额外有哨兵任务兜底。

### 4.5 第三层：业务校验 OSSCallbackValidate

**位置：** [callback.go:57-77](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/controllers/callback.go#L57-L77)

```go
// 解析回调Body
var callbackBody callback.UploadCallbackService
c.ShouldBindJSON(&callbackBody)

// 文件大小一致性校验
if uploadSession.Props.Size != callbackBody.Size {
    // 记录错误日志：expected vs actual
    c.JSON(401, "size mismatch")
    c.Abort()
}
```

> 其他驱动（如 COS）在 CompleteUpload 阶段也有类似的大小校验，见 [cos.go:550-569](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/cos/cos.go#L550-L569)。

### 4.6 第四层：ProcessCallback 完成上传

**位置：** [ProcessCallback](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/service/callback/upload.go#L45-L60) → [manager.CompleteUpload](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L289-L324)

执行步骤：

```
① d.CompleteUpload(ctx, session)        // 驱动层
  │  ├─ 七牛/又拍云/OSS：空实现（存储端已完成）
  │  └─ COS（带哨兵）：
  │     ├─ Head Object 获取实际文件大小
  │     └─ 校验 ContentLength == session.Props.Size
  │
② m.fs.CompleteUpload(ctx, session)     // DBFS 层
  │  ├─ Get 占位文件（含实体信息）
  │  ├─ ConfirmLock 确认文件锁有效 → 释放锁
  │  ├─ 实体类型转换（placeholder → version/其他）
  │  ├─ 版本保留策略检查
  │  ├─ 更新实体 Size/Source/LastModified
  │  └─ 提交文件修改事务
  │
③ 取消哨兵任务（SentinelTaskID > 0 时）
  │  m.dep.TaskClient().SetCompleteByID(ctx, session.SentinelTaskID)
  │
④ 触发后置处理
  │  ├─ 媒体元数据提取任务
  │  └─ 全文索引任务
  │
⑤ m.kv.Delete("callback_", sessionID)  // 清理 KV 会话
```

---

## 五、SessionID / CallbackSecret / 驱动签名层的分工与校验机制

本章是核心澄清章节，以"代码证据"形式逐要素解答：**回调密钥是否真的参与了校验？三者在整体安全体系中各自的角色是什么？**

### 5.1 CallbackSecret 完整生命周期追踪（代码证据链）

CallbackSecret 在系统中经历了 **生成 → 存储 → 嵌入URL → 客户端接收 → 存储商回调时带回** 共五个阶段：

| 阶段 | 代码位置 | 核心操作 | 是否"读取URL中的Secret并比较"？ |
|------|---------|---------|-------------------------------|
| **①生成** | [dbfs/upload.go:250](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/fs/dbfs/upload.go#L250) | `CallbackSecret: util.RandStringRunesCrypto(32)` | —（生成阶段） |
| **②存储到会话** | 同上行 | 存入 `UploadSession.CallbackSecret` 字段 | —（存储阶段） |
| **③嵌入回调URL** | [routes.go:46-49](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/cluster/routes/routes.go#L46-L49) | `MasterSlaveCallbackUrl(base, driver, id, secret)` → URL 路径为 `{driver}/{id}/{secret}` | —（构造URL阶段） |
| **④返回给客户端** | [manager/upload.go:113](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L113) | `credential.CallbackSecret = uploadSession.CallbackSecret` | —（传递给客户端阶段） |
| **⑤存储商回调带回到服务端** | [router.go:476-545](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/router.go#L476-L545) | URL 中的 `:key` 参数匹配 secret 进入路由 | ❌ **从未被 Gin Context 的 Param("key") 读取** |
| **⑥？？？** | 全局搜索 `c.Param("key")` / `Param(["']secret` | — | ❌ **整个代码库 0 处匹配** |
| **⑦？？？** | 全局搜索 `CallbackSecret` 用于校验的表达式 | 例如 `session.CallbackSecret == xxx` 等比较 | ❌ **仅出现于生成/传递处，从未出现于任何比较表达式** |

#### 5.1.1 结论：CallbackSecret 的真实作用

**CallbackSecret 从未被显式校验。** 它在整个安全体系中的角色是：

- **被动防御（路径不可枚举性）**：由于 CallbackSecret 是 32 位密码学随机串（约 190 bits 熵），攻击者即使猜到了 SessionID 也无法构造完整的回调 URL。
- **路由匹配屏障**：Gin 的路由是严格匹配的，`/callback/oss/{sessionID}/{key}` 中的 `{key}` 段必须存在才能进入这条路由。如果 URL 缺少第三段或第三段错误，直接返回 404，根本不会进入 UseUploadSession 中间件。

> ⚠️ 但反过来说：如果攻击者通过某种方式（如日志泄露、中间人攻击）获得了完整的回调 URL（包含正确的 sessionID + callbackSecret），那么这个 callbackSecret 并不会被进一步校验——**只要 KV 中 session 还没过期，请求就会通过 UseUploadSession**。之后是否能继续通过，取决于驱动签名层（见 5.3 节）。

### 5.2 SessionID 的完整生命周期追踪（代码证据链）

SessionID 在系统中的角色是**主动校验要素**：

| 阶段 | 代码位置 | 核心操作 | 是否"读取URL中的SessionID并校验"？ |
|------|---------|---------|----------------------------------|
| **①生成** | [manager/upload.go:80](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L80) | `sessionID := uuid.Must(uuid.NewV4()).String()` | — |
| **②存入KV** | [manager/upload.go:136-144](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L136-L144) | `m.kv.Set("callback_" + sessionID, ...)`，带 TTL | — |
| **③回调时读取URL** | [auth.go:195](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L195) | `sessionID := c.Param("sessionID")` | ✅ 是 |
| **④校验1：非空** | [auth.go:196-198](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L196-L198) | 空则返回 `CodeParamErr` | ✅ 是 |
| **⑤校验2：KV 存在** | [auth.go:200-204](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L200-L204) | `dep.KV().Get("callback_" + sessionID)`，不存在则 `CodeUploadSessionExpired` | ✅ 是（过期自动失效） |
| **⑥校验3：策略类型匹配** | [auth.go:208-210](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L208-L210) | `callbackSession.Policy.Type != string(policyType)` → `CodePolicyNotAllowed` | ✅ 是（防止跨驱动冒用） |
| **⑦恢复用户上下文** | [auth.go:212-214](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L212-L214) | `SetUserCtx(c, callbackSession.UID)` | 利用会话中的 UID |

#### 5.2.1 SessionID 的真实作用

**SessionID 是整个校验链的核心锚点**：
- 它是从 URL 参数 → 恢复内存中完整会话对象的唯一桥梁
- 它的校验包含"存在性（含过期TTL）+ 策略类型匹配"两道主动校验
- 通过它恢复出来的 Policy 中包含了 AK/SK/SlaveKey 等驱动签名层需要的密钥（见 5.3 节）

### 5.3 各存储驱动签名层的分工（代码证据链）

驱动签名层**完全独立于 SessionID 和 CallbackSecret**，它们使用"存储商的 AK/SK 体系"进行签名验证。

#### 5.3.1 签名层所用密钥从哪里来？

所有驱动签名层的密钥都**来自 SessionID 恢复出来的会话对象**（即 `callbackSession.Policy`），而非来自 URL 参数：

| 驱动 | 中间件 / 校验函数 | 使用的密钥来源 | 密钥形式 |
|-----|-----------------|--------------|---------|
| **OSS** | `OSSCallbackAuth()` → [VerifyCallbackSignature](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/oss/callback.go#L90-L123) | 不需要会话中的 AK/SK！使用请求头中的 x-oss-pub-key-url（阿里云公钥） | RSA 公钥（远程下载，白名单校验） |
| **七牛** | `QiniuCallbackValidate` → `mac.VerifyCallback` | `session.Policy.AccessKey` / `session.Policy.SecretKey` 重建 `qbox.Mac` | AK + SK |
| **又拍云** | `UpyunCallbackAuth` → [ValidateCallback](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/upyun/upyun.go#L356-L384) | `session.Policy.AccessKey` / `session.Policy.SecretKey` 传给 `sign()` 函数 | AK + SK（SK 先做 MD5） |
| **远程从机** | `RemoteCallbackAuth()` → `auth.CheckRequest` | `session.Policy.Edges.Node.SlaveKey` | SlaveKey（HMAC） |
| **COS/S3/KLS3/OBS** | **无此层** | 无驱动签名校验 | — |

#### 5.3.2 驱动签名层的算法细节（与 SessionID/CallbackSecret 无关）

各驱动签名校验算法**完全不包含** SessionID 或 CallbackSecret 作为签名原文的组成部分：

| 驱动 | 签名原文（即被签名的数据） | SessionID/CallbackSecret 是否在原文中？ |
|-----|------------------------|-------------------------------------|
| **OSS** | `URL.Path + "\n" + Body` 的 MD5 值 | ❌ 不在。URL.Path 包含它们，但原文是整个 Path+Body，不单独解析验证 |
| **七牛** | 请求的 Method / Path / Host / Content-Type / Body 等（SDK 内部封装） | ❌ 不在。SDK 按 HTTP 请求规范构造签名原文 |
| **又拍云** | `"POST&" + URL.Path + "&" + Date + "&" + Content-MD5` | ❌ 不在。URL.Path 整体参与，但不单独验证 |
| **远程从机** | 整个 HTTP 请求（Method/Path/Header/Body 等） | ❌ 不在。按通用 HMAC 请求签名规范 |

> 注意：OSS/七牛/又拍云的签名原文中虽然包含 `URL.Path`（其中自然含有 `{sessionID}` 和 `{secret}`），但这是"整体 Path 字符串"作为签名输入，并不意味着对它们进行了**单独的、显式的**解析校验。只要攻击者截获了某次合法回调的完整请求（含所有Header/Body）并原样重放，签名就必然匹配。签名层真正防御的是"篡改请求内容"而非"重放请求"。

### 5.4 三者的分工全景图

```
                 ┌─────────────────────────────────────────────────────────┐
                 │                    整体安全防线                         │
                 └─────────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
          ▼                           ▼                           ▼
┌─────────────────────┐   ┌─────────────────────┐   ┌──────────────────────────┐
│  第一层：路由屏障    │   │  第二层：会话锚点    │   │  第三层：驱动签名（可选） │
│  (CallbackSecret)   │   │   (SessionID)       │   │  (存储商 AK/SK 体系)     │
├─────────────────────┤   ├─────────────────────┤   ├──────────────────────────┤
│                     │   │                     │   │                          │
│ URL 必须包含正确的  │   │ 从 URL 中显式读取   │   │ 仅部分驱动有此层        │
│ 第三段(secret) 才能 │   │ 并进行 3 道主动校验:│   │                          │
│ 匹配路由，否则 404  │   │   ① 非空检查        │   │ • OSS: RSA 公钥验签      │
│                     │   │   ② KV 存在+TTL     │   │ • 七牛: HMAC-SHA1(AK/SK)│
│ ⚠️ 路由匹配成功后， │   │   ③ 策略类型匹配    │   │ • 又拍云: HMAC+Body MD5 │
│ 此 secret 再无任    │   │                     │   │ • 远程: HMAC(SlaveKey)  │
│ 何代码读取或校验    │   │ 恢复出会话后，将整  │   │ • COS/S3: 无此层       │
│                     │   │ 个 Policy(含AK/SK)  │   │                          │
│ 防御类型：被动防御  │   │ 塞入 Gin Context，供 │   │ 防御类型：主动校验      │
│ 防的是：瞎猜路径    │   │ 后续中间件使用      │   │ 防的是：篡改请求内容     │
│ 防不了：重放已知URL │   │                     │   │ 防不了：重放完整合法请求│
│                     │   │ 防御类型：主动校验  │   │                          │
│ 有效时间：永久      │   │ 防的是：过期/跨驱动 │   │                          │
│ (只要 URL 正确即可) │   │ 冒用会话            │   │                          │
│                     │   │                     │   │                          │
│                     │   │ 有效时间：TTL(24h)  │   │                          │
└─────────────────────┘   └─────────────────────┘   └──────────────────────────┘
          │                           │                           │
          └───────────────────────────┼───────────────────────────┘
                                      │
                                      ▼
                        ┌──────────────────────────────┐
                        │ 第四层：业务校验 + Complete  │
                        │ （文件大小二次校验等）         │
                        └──────────────────────────────┘
```

### 5.5 典型攻击场景下的防御能力评估

| 攻击场景 | CallbackSecret 能否防御？ | SessionID 校验能否防御？ | 驱动签名层能否防御？ | 整体结论 |
|---------|-------------------------|------------------------|-------------------|---------|
| 攻击者瞎猜 URL 路径 | ✅ 32 位随机熵足够大，不可能枚举通过 | ✅ UUID v4 同样无法枚举 | — | ✅ 安全 |
| 攻击者截获了某次合法回调的**完整请求**（含URL/Header/Body），在 TTL 内原样重放 | ❌ URL 正确即可匹配路由 | ❌ KV 仍在有效期 | ❌ 签名层校验的是"内容未被篡改"，重放即原样未篡改 | ⚠️ **存在重放风险**。TTL 是唯一防线（24h内仍可重放）。但重放成功只是再次调用 CompleteUpload，幂等或返回"会话已不存在"（若前次已成功清理 KV）。 |
| 攻击者篡改了截获请求中的 Body 内容（如把 size 改大） | ❌ URL 不变仍可匹配路由 | ❌ KV 会话仍有效 | ✅ OSS/七牛/又拍云/远程的签名会校验失败 | ✅ 有签名层的驱动安全。COS/S3 等无签名层的驱动需依赖 CompleteUpload 阶段的 Head Object 校验。 |
| 攻击者获取了有效 SessionID，但不知道 CallbackSecret | ✅ 无法构造正确的第三段 URL，路由不匹配，404 | —（根本进不来） | — | ✅ 安全。这是 CallbackSecret 的核心价值所在。 |
| 攻击者用 OSS 存储策略的 SessionID 去调用 COS 回调路由 | ❌ URL 正确即可匹配对应路由（如果同时知道 secret） | ✅ 策略类型校验不通过，CodePolicyNotAllowed | — | ✅ SessionID 的策略类型校验挡住了跨驱动冒用。 |

---

## 六、异常分支处理全景

### 6.1 凭证签发阶段失败

**处理函数：** [OnUploadFailed](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L369-L397)

在 CreateUploadSession 中，以下步骤任一失败都会触发：

```go
if err != nil {
    m.OnUploadFailed(ctx, uploadSession)  // 回滚
    return nil, err
}
```

OnUploadFailed 清理逻辑：

| 场景 | 清理动作 |
|------|---------|
| 有状态（主节点）+ 有 LockToken | 释放文件锁 `m.Unlock(LockToken)` |
| 有状态 + NewFileCreated = true | 硬删除占位文件 `m.Delete(Uri, SysSkipSoftDelete=true)` |
| 有状态 + 是更新操作 + 非导入 | 版本控制回滚 `VersionControl(Uri, EntityID, delete=true)` |
| 无状态（从节点） | 通过驱动删除已上传的物理文件 `d.Delete(SavePath)` |

> 所有清理错误仅记录 Warning 日志，不中断错误返回链。

### 6.2 用户主动取消上传

**处理函数：** [CancelUploadSession](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L223-L287)

```
① 从 KV 读取会话
  │
② 有状态时：
  │  ├─ DBFS.CancelUploadSession → 生成 staleEntities 和 indexDiff
  │  ├─ staleEntities 排队回收任务
  │  └─ indexDiff 处理索引变更
  │
③ 驱动层清理：
  │  ├─ 无状态：d.Delete(SavePath) 删除物理文件
  │  └─ 有状态：d.CancelToken(session) 取消存储端分片上传
  │
④ m.kv.Delete("callback_", sessionID)
```

### 6.3 回调过期（哨兵触发）

**处理函数：** [UploadSentinelCheckTask.Do](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L501-L548)

执行时机 = `ExpireAt + 5分钟` 后仍未完成回调：

```
① 二次检查任务状态（防止竞态）
  │  已完成 → 直接退出
  │
② 获取实体信息
  │  找不到 → 认为已清理，正常退出
  │
③ 驱动层清理
  │  ├─ d.Delete(Source) 删除物理文件
  │  └─ d.CancelToken(session) 取消存储端会话
  │
④ 任务正常结束
```

> 注意：哨兵任务**不**删除 DB 中的占位实体，只清理物理存储资源。这是一种保守设计——宁可留下脏数据等待手动清理，也不误删。

### 6.4 回调中 CompleteUpload 失败

ProcessCallback 返回 error，Controller 返回非 200 状态码：

| 驱动 | 失败响应码 | 响应格式 |
|-----|----------|---------|
| OSS | 400 | `serializer.Err` |
| COS/S3/KLS3 | 400 | `serializer.Err` |
| 七牛 | 400 | `{"error": "..."}` 通用格式 |
| 又拍云 | 400 | `serializer.Err` |
| Remote | 200 | `serializer.Err` |

> 注意：返回非 200 时，KV 中的上传会话**不会被删除**，理论上可以重试回调。但存储商通常只会重试有限次。

---

## 七、三段路径的校验关联图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           凭证签发（第一段）                                        │
│                                                                                      │
│  UploadSessionID(UUID) + CallbackSecret(32rand) ──┐                                  │
│                                                    │                                  │
│  Policy.{AK,SK,Type,Node.SlaveKey} ─────────────┐ │  存入 KV:                      │
│                                                  │ │  callback_{sessionID}           │
│  ExpireAt(now+TTL) ─────────────────────────────┼─┼──▶ TTL = 会话剩余秒数             │
│                                                  │ │                                  │
│  预签名URL/upToken/Policy ─────────────────────┐ │ │                                  │
│                                                 │ │ │                                  │
└─────────────────────────────────────────────────┼─┼─┼──────────────────────────────────┘
                                                  │ │ │
                                                  ▼ ▼ ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           客户端直传（第二段）                                       │
│                                                                                      │
│  PUT 分片到各 UploadURLs ── 签名已在URL/Token中                                      │
│  POST CompleteURL     ── 携带 callback 参数/Header                                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                                  │
                                                  ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           服务端回调（第三段）                                        │
│                                                                                      │
│  URL: /callback/{driver}/{sessionID}/{secret}                                        │
│         │                                                                             │
│         ├─ 路由匹配屏障（第一层，无代码校验）                                         │
│         │   {secret} 必须存在且非空才能进入此路由（否则 404）                         │
│         │   ⚠️ 此层之后，secret 再也不被任何代码读取或校验                               │
│         │                                                                             │
│         ├─① UseUploadSession（第二层，SessionID 主动校验）                            │
│         │   sessionID → KV 查找会话（过期则不存在）                                   │
│         │   会话中取出: Policy.Type / UID / Size / LockToken / Policy.AK/SK 等        │
│         │   校验: 非空 + KV存在(TTL) + 策略类型匹配                                   │
│         │                                                                             │
│         ├─② 驱动签名校验（第三层，可选）                                              │
│         │    ├─ OSS:   RSA(MD5(Path+Body)) 用阿里云公钥验证（白名单校验公钥来源）     │
│         │    ├─ 七牛:  HMAC-SHA1 用 session.Policy.AK/SK 重建 Mac 验证               │
│         │    ├─ 又拍云: HMAC-SHA1(MD5(SK)) + Body MD5 校验                           │
│         │    ├─ Remote:HMAC-SHA1 用 session.Policy.Node.SlaveKey 验证                 │
│         │    └─ COS/S3/KLS3/OBS: 无此层                                              │
│         │                                                                             │
│         ├─③ 业务校验（第四层，可选）                                                  │
│         │    └─ OSS: callbackBody.Size == session.Props.Size                         │
│         │                                                                             │
│         └─④ CompleteUpload                                                           │
│              ├─ 驱动层: COS 额外 Head 校验文件大小                                    │
│              ├─ DBFS层: 占位实体转正、释放锁、版本策略、事务提交                        │
│              ├─ 取消哨兵任务                                                           │
│              └─ KV 删会话                                                              │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 八、关键安全设计总结

| 设计点 | 说明 | 风险考虑 |
|-------|------|---------|
| **CallbackSecret 32位随机** | 密码学安全随机，作为 URL 路径的一部分，**仅提供路由匹配屏障+路径不可枚举性，不参与显式校验** | ⚠️ 一旦完整 URL 泄露，24h TTL 内可被重放（有签名层的驱动能防"篡改"但防不了"重放"）。**建议后续版本** 在 `uploadCallbackCheck` 中增加 `c.Param("key") == callbackSession.CallbackSecret` 的显式比对。 |
| **KV 中会话带 TTL** | 过期自动失效，防止会话永久有效 | TTL 默认 24h 偏长，重放窗口较大；TTL 内如果泄露仍可被冒用 |
| **SessionID 策略类型匹配** | OSS 会话不能用于 COS 回调（路由前缀+中间件双重校验） | 有效防止跨驱动冒用会话 |
| **存储商签名校验（可选层）** | OSS/七牛/又拍云/远程节点有独立签名机制，基于存储商 AK/SK 体系 | ⚠️ COS/S3/KLS3/OBS 无此层，仅靠 URL 随机性+哨兵兜底，安全等级较低 |
| **OSS 公钥白名单** | 只接受 gosspublic.alicdn.com 域下发的公钥，缓存 7 天 | 防止通过伪造公钥 URL 投毒 |
| **文件大小二次校验** | OSS 在回调层校验 Body size，COS 在 CompleteUpload 层 Head Object 校验 | 防止篡改上传内容大小（COS/S3 等无签名驱动的关键兜底校验） |
| **分布式锁** | PrepareUpload 时锁定目标路径加锁，Complete 时解锁 | 防止并发写同一文件造成数据错乱 |
| **哨兵宽限期** | ExpireAt+5分钟，防止正常回调因网络延迟被误杀 | 极端延迟下仍可能误杀，但5分钟足够缓冲；保守策略不删 DB 占位 |
| **签名层防篡改 / 防重放区分** | 驱动签名层（RSA/HMAC）只能防"请求内容被篡改" | ⚠️ **无法防重放攻击**。截获完整合法请求可在 TTL 内原样重放，幂等性是主要兜底（CompleteUpload 的二次处理或会因 KV 已删除而失败） |

---

## 九、核心文件索引

| 模块 | 文件路径 | 关键内容 |
|------|---------|---------|
| 上传管理器 | [manager/upload.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go) | CreateUploadSession / CompleteUpload / CancelUpload / OnUploadFailed / 哨兵任务 |
| 会话定义 | [fs/fs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/fs/fs.go) | UploadSession / UploadCredential / UploadProps 结构体 |
| DBFS 会话准备 | [fs/dbfs/upload.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/fs/dbfs/upload.go) | PrepareUpload（CallbackSecret 生成位置） / CompleteUpload |
| 回调控制器 | [controllers/callback.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/controllers/callback.go) | Qiniu/OSS/Upyun 回调校验中间件 |
| 回调服务 | [callback/upload.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/service/callback/upload.go) | ProcessCallback 入口 |
| 认证中间件 | [middleware/auth.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go) | UseUploadSession / OSSCallbackAuth / RemoteCallbackAuth |
| 路由定义 | [routers/router.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/router.go#L472-L545) | 各驱动回调路由及中间件链 |
| 驱动接口 | [driver/handler.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/handler.go) | Handler 接口（Token / CancelToken / CompleteUpload） |
| OSS 驱动 | [driver/oss/oss.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/oss/oss.go) | Token 签发 |
| OSS 回调验签 | [driver/oss/callback.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/oss/callback.go) | VerifyCallbackSignature / GetPublicKey |
| COS 驱动 | [driver/cos/cos.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/cos/cos.go) | Token / CompleteUpload（含大小校验）/ 哨兵需求声明 |
| S3 驱动 | [driver/s3/s3.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/s3/s3.go) | Token 签发 |
| 七牛驱动 | [driver/qiniu/qiniu.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/qiniu/qiniu.go) | Token / CancelToken 签发 |
| 又拍云驱动 | [driver/upyun/upyun.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/upyun/upyun.go) | Token / ValidateCallback / sign 函数 |
| URL 构造 | [cluster/routes/routes.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/cluster/routes/routes.go) | MasterSlaveCallbackUrl |
| TTL 配置 | [setting/provider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/setting/provider.go#L691-L693) | UploadSessionTTL 默认值 |
