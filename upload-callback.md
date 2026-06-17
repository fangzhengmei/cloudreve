# 分块上传三段路径校验分析笔记

## 一、整体架构总览

分块上传分为三条核心路径，它们通过 `UploadSessionID` 和 `CallbackSecret` 作为关联纽带：

```
┌────────────────────────┐     ┌──────────────────────┐     ┌──────────────────────┐
│  ① 临时凭证签发        │────▶│  ② 客户端直传        │────▶│  ③ 服务端回调        │
│  (服务端 → 客户端)     │     │  (客户端 → 存储商)    │     │  (存储商/客户端 → 服务端)
└────────────────────────┘     └──────────────────────┘     └──────────────────────┘
         │                              │                              │
         ▼                              ▼                              ▼
  UploadSession存入KV           各存储凭证/预签名URL         签名校验+会话恢复+完成上传
```

**关联标识：**
- `UploadSessionID`：UUID v4，用于 KV 键值查找会话
- `CallbackSecret`：32位加密随机字符串，作为回调 URL 的第三路径参数
- KV 缓存 Key：`callback_{sessionID}`

> ⚠️ **核心前置澄清（三层真相）**：
> 1. **第一层（Gin 路由匹配）**：路由第三段 `:key` 是 Gin 的**命名动态参数**，匹配**任意单个路径段**（任意非空、不含 `/` 的字符串）。不管填 "a" 还是 "正确的 CallbackSecret"，都能通过路由。CallbackSecret 的安全价值是"路径不可枚举的熵"，而非"路由校验逻辑"。
> 2. **第二层（SessionID 主动校验）**：仅校验 SessionID 的三道坎（非空、KV 存在+TTL、策略类型匹配）。**完全不读取、不校验 URL 中的第三段值**。
> 3. **第三层（驱动签名层，可选）**：对有签名层的驱动（OSS/七牛/又拍云/远程），签名原文包含完整 `URL.Path`，第三段值**通过签名层被隐式校验**——值不对则签名不通过。对无签名层驱动（COS/S3/KLS3/OBS），则无此保护，需依赖 CompleteUpload 的大小兜底（OBS 甚至无兜底！）。
>
> 详见下方"第五章"的完整证据链。

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

回调 URL 同时嵌入了 sessionID 和 callbackSecret，由存储商或客户端在回调时使用。

#### 2.3.3 各驱动签发方式对比

| 驱动类型 | 签发方式 | 客户端直传凭证 | 回调发起者 | 代码位置 |
|---------|---------|--------------|----------|---------|
| **阿里云OSS** | 每分片预签名URL + CompleteURL | `UploadURLs[]` + `CompleteURL` + Callback(base64) | **OSS 存储服务器**主动回调 | [oss.go:490-589](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/oss/oss.go#L490-L589) |
| **腾讯云COS** | 每分片预签名URL + CompleteURL | `UploadURLs[]` + `CompleteURL` | **客户端**主动 GET 回调 | [cos.go:452-542](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/cos/cos.go#L452-L542) |
| **AWS S3** | 每分片预签名URL + CompleteURL | `UploadURLs[]` + `CompleteURL` | **客户端**主动 GET 回调 | [s3.go:335-411](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/s3/s3.go#L335-L411) |
| **七牛云** | upToken + InitParts | `Credential` + `UploadURLs[]` + `UploadID` | **七牛服务器**主动回调 | [qiniu.go:363-408](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/qiniu/qiniu.go#L363-L408) |
| **又拍云** | Policy(base64) + HMAC签名 | `UploadPolicy` + `Credential` | **又拍云服务器**主动回调 | [upyun.go:280-321](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/upyun/upyun.go#L280-L321) |
| **华为OBS** | 每分片预签名URL + CompleteURL | 每分片预签名URL | **客户端/OBS服务器**POST回调 | [obs.go:405-490](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/obs/obs.go#L405-L490) |
| **金山KS3** | 每分片预签名URL + CompleteURL | 每分片预签名URL | **客户端**主动 GET 回调 | [ks3.go:391-478](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/ks3/ks3.go#L391-L478) |
| **OneDrive** | Graph API uploadSession | `UploadURL` | **客户端**完成后通知 | [onedrive.go:167](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/onedrive/onedrive.go#L167) |

> **关键差异**：OSS/七牛/又拍云的回调由**存储商服务器**发起（有能力附带签名），COS/S3/KLS3/OBS/OneDrive 的回调由**客户端**主动发起（无存储商签名能力，只能靠 URL 路径保护）。

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

| 驱动 | 分片上传方式 | 合并触发 | 回调触发者 |
|-----|------------|---------|----------|
| OSS | 逐个 PUT 已签名的分片URL | 客户端调用 CompleteURL | OSS 服务器主动 POST 回调 |
| COS/S3/KLS3 | 逐个 PUT 已签名的分片URL | 客户端调用 CompleteURL | **客户端**主动 GET 回调 URL |
| 七牛 | 分片直传到 upHost，带 upToken | 客户端调用七牛合并接口 | 七牛服务器主动 POST 回调 |
| 又拍云 | POST 表单上传，带 Policy + Signature | 表单上传即完成 | 又拍云服务器主动 POST 回调 |
| OBS | 逐个 PUT 已签名的分片URL | 客户端调用 CompleteURL | OBS 服务器 POST 回调 / 客户端触发 |

**注意：** COS/S3/KLS3 等驱动没有存储商的主动签名回调机制，回调由客户端上传完成后主动调用，安全等级天然更低，需要**哨兵任务**兜底。

---

## 四、第三段路径：服务端回调

### 4.1 回调路由总览

路由定义于 [router.go:472-545](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/router.go#L472-L545)，统一格式：

```
POST|GET /api/v4/callback/{driverType}/{sessionID}/{:key}
```

第三段参数名为 `:key`（即 CallbackSecret）。**Gin 的命名参数 `:key` 匹配任意单个路径段**——格式正确即可，不校验值内容。

### 4.2 中间件执行链（以 OSS 为例）

```
请求到达
  │
  ▼
① UseUploadSession(PolicyTypeOss)
  │  ├─ 从 URL Path 提取 sessionID（只提取 sessionID，不提取 :key）
  │  ├─ 从 KV 获取 callback_{sessionID} → 不存在则 401
  │  ├─ 验证 Policy.Type 匹配
  │  └─ 通过 callbackSession.UID 恢复用户上下文
  │
  ▼
② OSSCallbackAuth()  [仅部分驱动有此层]
  │  └─ RSA 公钥签名验证（签名原文含完整 URL.Path → 间接校验了第三段值）
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

> ⚠️ **三重否定证据（关键代码对照）**：
> **证据①**：URL 格式 `{driverType}/{sessionID}/{key}` 定义于 [router.go:476-545](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/router.go#L476-L545)，第三段命名为 `:key`。
> **证据②**：整个 `uploadCallbackCheck` 函数中**完全没有** `c.Param("key")`。全局搜索 `c.Param(["']key["'])` 在 `middleware/` 目录下 **0 条匹配**。
> **证据③**：所有后续中间件（OSSCallbackAuth / RemoteCallbackAuth / QiniuCallbackValidate / UpyunCallbackAuth）中，**也没有任何代码读取 `Param("key")` 并与 `callbackSession.CallbackSecret` 对比**。全局搜索 CallbackSecret 在比较表达式中的使用，**0 条匹配**。
>
> 结论：UseUploadSession 及所有后续中间件**从不显式校验第三段值是否等于 KV 中存储的 CallbackSecret**。

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
② 签名原文构造（关键代码：callback.go:76-84）
  │ strURLPathDecode = url.PathUnescape(r.URL.Path)       // 完整解码后的 Path
  │ 待签名内容 strAuth = strURLPathDecode + "\n" + Body   // ⚠️ 包含完整 Path，第三段值在其中
  │ MD5值 = md5(待签名内容)
  │
  ▼
③ RSA 验签
  └─ rsa.VerifyPKCS1v15(pubKey, crypto.MD5, MD5值, Authorization解码值)
```

**关键代码（[oss/callback.go:81](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/oss/callback.go#L81)）**：
```go
strAuth := fmt.Sprintf("%s\n%s", strURLPathDecode, string(body))  // URL.Path 在签名原文中
```

**异常分支：**
- 公钥 URL 不在白名单 → `public key url invalid`
- Authorization 头缺失 → `no authorization field in Request header`
- PEM 解码失败 → `pubBlock not exist`
- RSA 验签失败 → 对应 crypto 错误（包括因篡改第三段值导致的 MD5 不匹配）

#### 4.4.2 七牛云签名核验

**中间件：** [QiniuCallbackValidate](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/controllers/callback.go#L34-L54)

```go
// 使用七牛 SDK 自带的验证方法
mac := qbox.NewMac(session.Policy.AccessKey, session.Policy.SecretKey)
ok, err := mac.VerifyCallback(c.Request)
```

七牛 SDK 内部逻辑：基于请求的 `Authorization` 头，按 HTTP 签名规范（包含请求 Method / Path / Host / Content-Type / Body 等）使用 AK/SK 计算 HMAC-SHA1 并对比。**URL.Path（含第三段值）是签名原文的组成部分**。

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
② HMAC-SHA1 签名计算（关键代码：upyun.go:373-378）
  │ key = hex(md5(SecretKey))
  │ elements = ["POST", c.Request.URL.Path, Date, Content-Md5]  // ⚠️ 含完整 URL.Path
  │ 待签名字符串 = strings.Join(elements, "&")
  │ signature = "UPYUN " + AK + ":" + base64(hmac_sha1(key, 待签名字符串))
  │
  ▼
③ 对比 signature == Authorization 头
```

**关键代码（[upyun/upyun.go:373-378](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/upyun/upyun.go#L373-L378)）**：
```go
signature := sign(session.Policy.AccessKey, session.Policy.SecretKey, []string{
    "POST",
    c.Request.URL.Path,  // ⚠️ 包含完整 Path（含 sessionID 和第三段值）
    date,
    contentMD5,
})
```

#### 4.4.4 远程（从机节点）签名核验

**中间件：** [RemoteCallbackAuth](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L220-L240)

**校验链**：
1. `auth.CheckRequest(c, authInstance, c.Request)` → [auth/auth.go:78-89](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/auth/auth.go#L78-L89)
2. `getSignContent(ctx, r)` → [auth/auth.go:98-121](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/auth/auth.go#L98-L121)
3. `getUrlSignContent(ctx, r.URL)` → [auth/auth.go:212-227](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/auth/auth.go#L212-L227): `return url.Path`

**签名原文成分**：`NewRequestSignString(URL.Path, X-Cr-* Headers 拼接, Body)`，**包含完整 URL.Path**。

#### 4.4.5 无独立签名校验的驱动

| 驱动 | HTTP方法 | 中间件链 | 有签名层？ | CompleteUpload 大小兜底？ |
|-----|---------|---------|-----------|------------------------|
| COS | GET | UseUploadSession + ProcessCallback | ❌ 无 | ✅ 有（[cos.go:550-568](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/cos/cos.go#L550-L568) 做 Head Object） |
| S3 | GET | UseUploadSession + ProcessCallback | ❌ 无 | ✅ 有（[s3.go:504-523](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/s3/s3.go#L504-L523) 做 Meta → Head） |
| KS3 | GET | UseUploadSession + ProcessCallback | ❌ 无 | ✅ 有（[ks3.go:521-540](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/ks3/ks3.go#L521-L540) 做 Meta → Head） |
| **OBS** | POST | UseUploadSession + ProcessCallback | ❌ 无 | ❌ **空实现，无任何校验**（[obs.go:519-521](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/obs/obs.go#L519-L521)） |
| OneDrive | POST | UseUploadSession + ProcessCallback | ❌ 无 | 依赖 Graph API 状态 |

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

> 其他驱动（COS/S3/KS3）在 CompleteUpload 阶段也有类似的大小校验，见各驱动的 CompleteUpload 方法。**OBS 除外。**

### 4.6 第四层：ProcessCallback 完成上传

**位置：** [ProcessCallback](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/service/callback/upload.go#L45-L60) → [manager.CompleteUpload](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L289-L324)

执行步骤：

```
① d.CompleteUpload(ctx, session)        // 驱动层
  │  ├─ 七牛/又拍云/OSS：空实现（存储端已完成）
  │  ├─ COS/S3/KS3：Head Object 校验实际文件大小
  │  └─ OBS：空 return nil（⚠️ 无校验）
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

## 五、任意动态段 / SessionID / 驱动签名层的真实安全边界

本章是核心澄清章节，以代码证据形式逐要素解答：
- **路由第三段的任意值能否通过？**（Gin 动态参数匹配规则）
- **SessionID 缓存查找的真实边界在哪里？**
- **驱动签名层到底校验了什么？第三段值是否被隐式校验？**
- **各驱动的安全等级差异到底有多大？**

### 5.1 Gin 路由 `:key` 动态参数的真实匹配规则

#### 5.1.1 路由定义（[router.go:476-545](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/router.go#L476-L545)）

```go
// 所有回调路由格式统一为：
callback.POST("oss/:sessionID/:key", ...)      // 第三段命名为 :key
callback.GET("cos/:sessionID/:key", ...)
callback.POST("qiniu/:sessionID/:key", ...)
// ... 其他驱动同理
```

#### 5.1.2 Gin 路由匹配规则

Gin 框架中 `:key` 是**命名参数（Named Parameter）**，匹配规则是：
- 匹配**任意单个路径段**（即 URL 中从当前 `/` 之后到下一个 `/` 或结尾之前的所有字符）
- 匹配条件：**非空 + 不含 `/`**
- **不校验值内容**（不管是 "a"、"任意字符串" 还是正确的 CallbackSecret，只要格式对就匹配）

#### 5.1.3 匹配测试（逻辑推断）

| 请求 URL | 是否匹配路由 | 原因 |
|---------|------------|------|
| `/callback/cos/abc-123/x` | ✅ 匹配 | 第三段非空且不含 `/` |
| `/callback/cos/abc-123/任意字符` | ✅ 匹配 | 第三段非空且不含 `/` |
| `/callback/cos/abc-123/正确的32位secret` | ✅ 匹配 | 第三段非空且不含 `/` |
| `/callback/cos/abc-123/` | ❌ 404 | 第三段为空 |
| `/callback/cos/abc-123` | ❌ 404 | 缺少第三段 |
| `/callback/cos/abc-123/a/b` | ❌ 404 | 第三段含 `/`（Gin 认为是额外段） |

**结论：CallbackSecret 的安全价值不在于"路由会不会校验它的值"，而在于"它提供了 32 位密码学随机串的熵，使攻击者无法枚举到一个格式正确的 URL"。**

### 5.2 CallbackSecret 完整生命周期追踪（七阶段代码证据链）

| 阶段 | 代码位置 | 核心操作 | 是否读取 URL 中的 :key 并比较？ |
|------|---------|---------|-----------------------------|
| **①生成** | [dbfs/upload.go:250](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/fs/dbfs/upload.go#L250) | `CallbackSecret: util.RandStringRunesCrypto(32)` | — |
| **②存储到会话** | 同上行 | 存入 `UploadSession.CallbackSecret` 字段 | — |
| **③嵌入回调URL** | [routes.go:46-49](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/cluster/routes/routes.go#L46-L49) | `path.Join(..., driver, id, secret)` → 第三段 | — |
| **④返回给客户端** | [manager/upload.go:113](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go#L113) | `credential.CallbackSecret = uploadSession.CallbackSecret` | — |
| **⑤回调请求到达** | Gin 路由匹配层 | `:key` 匹配任意非空路径段进入路由 | ❌ Gin 不校验值，只校验格式 |
| **⑥UseUploadSession** | [auth.go:193-217](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L193-L217) | 仅读取 `c.Param("sessionID")` | ❌ **0 处读取 Param("key")** |
| **⑦驱动签名层/业务层** | 4.4.1 - 4.4.5 各中间件 | — | ❌ **0 处读取 Param("key") 并比较** |

#### 5.2.1 否定证据的三路交叉验证

| 搜索模式 | 搜索范围 | 匹配条数 | 结论 |
|---------|---------|---------|------|
| `c.Param\(["']key["']\)` | `middleware/` 目录 | 0 | 无中间件读取 URL 中的 :key |
| `CallbackSecret.*==\|==.*CallbackSecret` | 全项目 | 0 | 无代码将 CallbackSecret 用于比较表达式 |
| `Param\(["']key["'].*==\|==.*Param\(["']key["']` | 全项目 | 0 | 无代码用 URL 中的 key 参与比较 |

### 5.3 SessionID 的真实安全边界

SessionID 是整个校验链中**唯一被显式读取并做多道校验**的标识。

| 校验阶段 | 代码位置 | 校验内容 | 通过条件 |
|---------|---------|---------|---------|
| **①非空检查** | [auth.go:196-198](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L196-L198) | `sessionID == ""` | UUID v4 格式字符串，非空 |
| **②KV 存在检查** | [auth.go:200-204](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L200-L204) | `dep.KV().Get("callback_" + sessionID)` | KV 中存在 → 暗含 "未过期(TTL 有效)" |
| **③策略类型匹配** | [auth.go:208-210](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L208-L210) | `Policy.Type != string(policyType)` | URL 路由前缀（如 `cos`）与会话中 Policy.Type 一致 |

**SessionID 的安全边界**：只要攻击者持有**一个尚未过期的、策略类型与路由前缀匹配的 SessionID**，就可以通过 UseUploadSession 中间件。后续能否继续通过，取决于驱动签名层是否存在。

### 5.4 驱动签名层的真实安全边界（签名原文成分详解）

本小节回答关键问题：**第三段值（CallbackSecret）通过签名层被隐式校验了吗？**

答案：**对有签名层的驱动——是的。** 因为签名原文包含完整 `URL.Path`，第三段值变了 → Path 变了 → 签名原文变了 → 签名校验失败。

#### 5.4.1 各驱动签名原文成分对比

| 驱动 | 签名原文构造代码 | URL.Path 是否在原文中？ | 第三段值变化会导致签名失败？ |
|-----|---------------|----------------------|------------------------|
| **OSS** | `fmt.Sprintf("%s\n%s", URL.Path, Body)` → MD5 | ✅ 是（[oss/callback.go:81](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/oss/callback.go#L81)） | ✅ 会 |
| **七牛** | SDK 内部按 HTTP 规范：Method/Path/Host/ContentType/Body 等 | ✅ 是（行业标准 HMAC 规范） | ✅ 会 |
| **又拍云** | `strings.Join(["POST", URL.Path, Date, MD5], "&")` | ✅ 是（[upyun/upyun.go:373-378](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/upyun/upyun.go#L373-L378)） | ✅ 会 |
| **远程从机** | `NewRequestSignString(URL.Path, X-Cr-Headers, Body)` | ✅ 是（[auth/auth.go:119](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/auth/auth.go#L119) + [auth/auth.go:226](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/auth/auth.go#L226)） | ✅ 会 |
| **COS** | 无签名层 | ❌ 无 | ❌ 不会（直接通过） |
| **S3** | 无签名层 | ❌ 无 | ❌ 不会（直接通过） |
| **KS3** | 无签名层 | ❌ 无 | ❌ 不会（直接通过） |
| **OBS** | 无签名层 | ❌ 无 | ❌ **不会（且 CompleteUpload 无大小兜底）** |

#### 5.4.2 驱动签名层的能力边界

| 能力 | 是否能防？ | 说明 |
|-----|----------|------|
| **防篡改请求 Body** | ✅ 能 | Body 在签名原文中，篡改则签名不匹配 |
| **防篡改 URL.Path（含第三段）** | ✅ 能 | Path 在签名原文中，篡改则签名不匹配 |
| **防重放完整合法请求** | ❌ 不能 | 原样请求 → 原文和签名都不变 → 签名必然匹配 |
| **防过期会话冒用** | ❌ 不能（需 SessionID 层） | 签名层不校验 KV TTL |
| **防跨策略类型冒用** | ❌ 不能（需 SessionID 层） | 签名层不比较 Policy.Type |

### 5.5 三层防线的分工全景图（修正版）

```
                 ┌──────────────────────────────────────────────────────────────┐
                 │                     整体安全防线                              │
                 └──────────────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼─────────────────────────────┐
          │                           │                             │
          ▼                           ▼                             ▼
┌─────────────────────────┐ ┌───────────────────────┐ ┌───────────────────────────────┐
│  第一层：路由匹配屏障     │ │  第二层：SessionID      │ │  第三层：驱动签名（可选）       │
│  (Gin :key 动态参数)     │ │   主动校验锚点         │ │  (含URL.Path → 隐式校验第三段) │
├─────────────────────────┤ ├───────────────────────┤ ├───────────────────────────────┤
│                         │ │                       │ │                               │
│ 校验规则：               │ │ 校验规则：             │ │ 校验规则（有签名层驱动）：      │
│ •第三段必须非空          │ │ • sessionID 非空       │ │ • 用 AK/SK 或 RSA 公钥验签     │
│ •第三段不能包含 "/"      │ │ • KV 键存在（含 TTL）  │ │ • 签名原文 = Path + Body + ... │
│ •不校验值内容            │ │ • Policy.Type 匹配     │ │ • Path 变了 → 签名必然失败     │
│                         │ │                       │ │                               │
│ ⚠️ 填 "a"、"任意值" 都   │ │                       │ │ ⚠️ 防篡改，但不防重放           │
│   能匹配路由             │ │ 恢复出会话对象后，     │ │                               │
│                         │ │ 塞入 Gin Context       │ │ 无签名层驱动（COS/S3/KS3/OBS）：│
│ 防御类型：被动防御       │ │                       │ │   • 直接跳过此层               │
│ 防的是：路径枚举         │ │ 防御类型：主动校验     │ │   • COS/S3/KS3 有大小兜底      │
│ 防不了：知道 SessionID   │ │ 防的是：过期 / 跨策略  │ │   • ⚠️ OBS 无大小兜底！        │
│         后随便填值       │ │                       │ │                               │
│                         │ │ 防御窗口：TTL(24h)    │ │                               │
└─────────────────────────┘ └───────────────────────┘ └───────────────────────────────┘
          │                           │                             │
          └───────────────────────────┼─────────────────────────────┘
                                      │
                                      ▼
                        ┌──────────────────────────────────────┐
                        │ 第四层：业务校验 + CompleteUpload    │
                        │ • OSSCallbackValidate: size 校验     │
                        │ • 驱动层 CompleteUpload: Head 大小   │
                        │   （⚠️ OBS 空实现）                   │
                        └──────────────────────────────────────┘
```

### 5.6 各驱动安全等级对比矩阵（完整视图）

| 驱动 | 回调发起者 | 路由屏障 | SessionID 校验 | 驱动签名层 | 签名含 Path | 大小兜底 | 综合安全等级 | 典型风险 |
|-----|----------|---------|--------------|-----------|-----------|---------|-----------|---------|
| **OSS** | 存储商 | ✅ | ✅ 三道 | ✅ RSA | ✅ 隐式校验第三段 | ✅ Body size | **A+** | 重放合法请求 |
| **七牛** | 存储商 | ✅ | ✅ 三道 | ✅ HMAC-SHA1 | ✅ 隐式校验第三段 | ❌ 依赖 CompleteUpload | **A** | 重放合法请求 |
| **又拍云** | 存储商 | ✅ | ✅ 三道 | ✅ HMAC + MD5 | ✅ 隐式校验第三段 | ✅ Body MD5 | **A** | 重放合法请求 |
| **远程** | 从机 | ✅ | ✅ 三道 | ✅ HMAC-SHA256 | ✅ 隐式校验第三段 | ❌ | **A** | 重放合法请求 |
| **COS** | 客户端 | ✅ | ✅ 三道 | ❌ 无 | N/A | ✅ Head 大小 | **B** | 知 SessionID 可冒用，需大小恰好匹配 |
| **S3** | 客户端 | ✅ | ✅ 三道 | ❌ 无 | N/A | ✅ Head 大小 | **B** | 同 COS |
| **KS3** | 客户端 | ✅ | ✅ 三道 | ❌ 无 | N/A | ✅ Head 大小 | **B** | 同 COS |
| **OneDrive** | 客户端 | ✅ | ✅ 三道 | ❌ 无 | N/A | Graph API | **B** | 同 COS |
| **OBS** | 客户端/OBS | ✅ | ✅ 三道 | ❌ 无 | N/A | ❌ **空实现** | **C** | ⚠️ **知 SessionID 可直接冒用，无任何兜底** |

### 5.7 七种典型攻击场景的防御能力评估

| # | 攻击场景 | Gin 路由 | SessionID 层 | 驱动签名层 | CompleteUpload 兜底 | 最终结论 |
|---|---------|---------|-------------|-----------|------------------|---------|
| 1 | 攻击者瞎猜 URL 路径（枚举） | ✅ 32位熵不可枚举 | ✅ UUID不可枚举 | — | — | ✅ 安全 |
| 2 | 攻击者知 SessionID，不知 Secret，填"任意值"当第三段 | ✅ 通过（格式合法） | ✅ 通过 | **有签名层：❌ 签名失败**<br>**无签名层：✅ 通过** | **无签名层：**<br>COS/S3/KS3: ✅ 大小兜底<br>**OBS: ❌ 无兜底** | **有签名层：安全**<br>**COS/S3/KS3: 依赖大小**<br>**⚠️ OBS: 直接通过** |
| 3 | 攻击者截获完整合法请求，TTL 内原样重放 | ✅ 通过 | ✅ KV 仍有效 | ✅ 签名匹配（内容未篡改） | — | ⚠️ **重放风险**。兜底：首次成功后 KV 被删，二次重放会失效 |
| 4 | 攻击者篡改 Body 中的 size | ✅ 通过 | ✅ 通过 | ✅ 有签名层：签名失败<br>❌ 无签名层：通过 | **无签名层：**<br>✅ Head 大小兜底<br>❌ OBS: 无 | **有签名层：安全**<br>**COS/S3/KS3: 安全**<br>**⚠️ OBS: 直接通过** |
| 5 | 攻击者用 OSS SessionID 调 COS 路由（同时知两者 secret） | ✅ 通过 | ❌ Policy.Type 不匹配 | — | — | ✅ SessionID 层挡住 |
| 6 | 攻击者在无签名层驱动下，知 SessionID + 猜中正确大小 | ✅ 通过 | ✅ 通过 | — | ❌ 大小匹配通过 | ⚠️ **可冒用**。攻击者需预先知道要上传文件的确切大小（来自上传凭证） |
| 7 | 攻击者 TTL 内截获合法 URL，过期后重放 | ✅ 通过 | ❌ KV 不存在（TTL 失效） | — | — | ✅ 安全 |

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

## 七、三段路径的校验关联图（修正版）

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                            凭证签发（第一段）                                          │
│                                                                                       │
│  UploadSessionID(UUID) + CallbackSecret(32rand) ──┐                                   │
│                                                    │                                   │
│  Policy.{AK,SK,Type,Node.SlaveKey} ─────────────┐ │  存入 KV:                        │
│                                                  │ │  callback_{sessionID}             │
│  ExpireAt(now+TTL) ─────────────────────────────┼─┼──▶ TTL = 会话剩余秒数               │
│                                                  │ │                                   │
│  预签名URL/upToken/Policy ─────────────────────┐ │ │                                   │
│                                                 │ │ │                                   │
└─────────────────────────────────────────────────┼─┼─┼───────────────────────────────────┘
                                                  │ │ │
                                                  ▼ ▼ ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                            客户端直传（第二段）                                         │
│                                                                                       │
│  OSS/七牛/又拍云：存储商收到 complete 请求后，主动 POST 回调 URL（带签名）                │
│  COS/S3/KLS3/OBS：客户端合并完成后，主动 GET/POST 回调 URL（无签名，仅靠路径）            │
│                                                                                       │
└──────────────────────────────────────────────────────────────────────────────────────┘
                                                  │
                                                  ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                            服务端回调（第三段）                                          │
│                                                                                       │
│  URL: /callback/{driver}/{sessionID}/{:key}  ← Gin 动态参数，任意非空值均可匹配          │
│         │                                                                              │
│         ├─① 路由匹配（第一层，被动屏障）                                                │
│         │   :key 非空且不含 / → 匹配成功，进 UseUploadSession                           │
│         │   ⚠️ 不管填什么，只要格式合法就通过                                             │
│         │                                                                              │
│         ├─② UseUploadSession（第二层，SessionID 主动校验）                              │
│         │   sessionID → KV 查找（3道坎：非空 + KV存在+TTL + Policy.Type）               │
│         │   恢复出：Policy.Type / UID / Size / LockToken / Policy.AK/SK / Secret 等     │
│         │   ⚠️ 此层仍然完全不读取、不校验 URL 中的 :key 值                                │
│         │                                                                              │
│         ├─③ 驱动签名校验（第三层，可选）                                                │
│         │    ├─ 有签名层（OSS/七牛/又拍云/远程）：                                      │
│         │    │   签名原文含完整 URL.Path → 第三段值不对 → 签名校验失败                  │
│         │    │   ✅ 通过签名层"隐式"校验了第三段值的正确性                               │
│         │    │                                                                          │
│         │    └─ 无签名层（COS/S3/KLS3/OBS）：                                          │
│         │        直接跳过此层 → 第三段填任何值都不影响                                    │
│         │                                                                              │
│         ├─④ 业务校验 + CompleteUpload（第四层）                                        │
│         │    ├─ OSS: callbackBody.Size == session.Props.Size                           │
│         │    ├─ COS/S3/KS3: Head Object 校验实际大小                                    │
│         │    └─ ⚠️ OBS: CompleteUpload 空 return nil（无校验！）                         │
│         │                                                                              │
│         └─⑤ 收尾：取消哨兵 + 删 KV + 后置任务                                           │
│                                                                                       │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 八、关键安全设计总结

| 设计点 | 说明 | 风险考虑 |
|-------|------|---------|
| **Gin `:key` 动态参数** | 匹配任意非空路径段，不校验值 | CallbackSecret 仅提供"路径不可枚举熵"，不提供逻辑校验。建议在 `uploadCallbackCheck` 中增加 `c.Param("key") == callbackSession.CallbackSecret` 的显式比对，将第三段值纳入主动校验。 |
| **KV 中会话带 TTL** | 过期自动失效，防止会话永久有效 | TTL 默认 24h 偏长，重放窗口较大。建议根据上传文件大小动态调整 TTL（大文件长 TTL，小文件短 TTL）。 |
| **SessionID 策略类型匹配** | OSS 会话不能用于 COS 回调（路由前缀+中间件双重校验） | 有效防止跨驱动冒用会话，设计良好。 |
| **驱动签名层含 Path** | OSS/七牛/又拍云/远程的签名原文包含完整 URL.Path | **无意间**通过签名层"隐式校验"了第三段值的正确性。这是无显式校验代码时的重要安全保障。但无签名层驱动（COS/S3/KLS3/OBS）无此保障。 |
| **存储商签名校验（可选层）** | OSS/七牛/又拍云/远程节点有独立签名机制 | ⚠️ COS/S3/KLS3/OBS 无此层，仅靠 URL 随机性+大小兜底，安全等级较低。 |
| **OSS 公钥白名单** | 只接受 gosspublic.alicdn.com 域下发的公钥，缓存 7 天 | 防止通过伪造公钥 URL 投毒，设计良好。 |
| **文件大小二次校验** | OSS 在回调层校验 Body size，COS/S3/KS3 在 CompleteUpload 层 Head 校验 | 防止篡改上传内容大小（无签名驱动的关键兜底校验）。**⚠️ OBS CompleteUpload 是空实现，无此兜底！** |
| **分布式锁** | PrepareUpload 时锁定目标路径，Complete 时解锁 | 防止并发写同一文件造成数据错乱，设计良好。 |
| **哨兵宽限期** | ExpireAt+5 分钟，防止正常回调因网络延迟被误杀 | 保守策略不删 DB 占位。 |
| **签名层防篡改 / 防重放区分** | 驱动签名层（RSA/HMAC）只能防"请求内容被篡改" | ⚠️ **无法防重放攻击**。截获完整合法请求可在 TTL 内原样重放，幂等性+KV 删除是主要兜底。 |
| **OBS 特殊风险** | CompleteUpload 空实现 `return nil` | ⚠️ **高风险**：知 SessionID + 填任意第三段即可冒用，无任何大小校验兜底。建议立即实现 OBS 的 CompleteUpload Head 校验。 |

### 8.1 改进建议清单

| 优先级 | 建议 | 预期收益 |
|-------|------|---------|
| **P0** | 在 [uploadCallbackCheck](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go#L193-L217) 中增加 `c.Param("key") == callbackSession.CallbackSecret` 显式比对 | 补上所有驱动（尤其 OBS/COS/S3/KLS3）的第三段值主动校验 |
| **P0** | 实现 [OBS.CompleteUpload](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/obs/obs.go#L519-L521) 的 Head Object 大小校验 | 消除 OBS "知 SessionID 即可任意冒用" 的高危漏洞 |
| **P1** | 在 TTL 过期后，结合 nonce（如 KV 记录已消费的签名/请求 ID）防止重放攻击 | 缩短合法重放窗口 |
| **P1** | 根据文件大小动态调整 UploadSessionTTL | 小文件上传的 TTL 可大幅缩短至数小时 |
| **P2** | 对回调请求增加调用方 IP 白名单校验（仅接受存储商公布的 IP 段） | 进一步降低存储商回调驱动的重放/冒用风险 |

---

## 九、核心文件索引

| 模块 | 文件路径 | 关键内容 |
|------|---------|---------|
| 上传管理器 | [manager/upload.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/manager/upload.go) | CreateUploadSession / CompleteUpload / CancelUpload / OnUploadFailed / 哨兵任务 |
| 会话定义 | [fs/fs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/fs/fs.go) | UploadSession / UploadCredential / UploadProps 结构体 |
| DBFS 会话准备 | [fs/dbfs/upload.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/fs/dbfs/upload.go) | PrepareUpload（CallbackSecret 生成位置 L250） / CompleteUpload |
| 回调控制器 | [controllers/callback.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/controllers/callback.go) | Qiniu / OSS / Upyun 回调校验中间件 |
| 回调服务 | [callback/upload.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/service/callback/upload.go) | ProcessCallback 入口 |
| 认证中间件 | [middleware/auth.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/middleware/auth.go) | UseUploadSession(L178-217) / OSSCallbackAuth / RemoteCallbackAuth |
| 路由定义 | [routers/router.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/routers/router.go#L472-L545) | 各驱动回调路由（`:key` 动态参数）及中间件链 |
| 驱动接口 | [driver/handler.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/handler.go) | Handler 接口（Token / CancelToken / CompleteUpload） |
| OSS 驱动 | [driver/oss/oss.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/oss/oss.go) | Token 签发 |
| OSS 回调验签 | [driver/oss/callback.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/oss/callback.go) | VerifyCallbackSignature（签名原文含 URL.Path, L81） / GetPublicKey |
| COS 驱动 | [driver/cos/cos.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/cos/cos.go) | Token(L452-542) / CompleteUpload（Head 大小校验 L550-568）/ 哨兵声明 |
| S3 驱动 | [driver/s3/s3.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/s3/s3.go) | Token 签发(L335-411) / CompleteUpload（大小校验 L504-523） |
| KS3 驱动 | [driver/ks3/ks3.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/ks3/ks3.go) | Token 签发(L391-478) / CompleteUpload（大小校验 L521-540） |
| OBS 驱动 | [driver/obs/obs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/obs/obs.go) | Token 签发(L405-490) / **CompleteUpload 空实现(L519-521)** |
| 七牛驱动 | [driver/qiniu/qiniu.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/qiniu/qiniu.go) | Token / CancelToken 签发 |
| 又拍云驱动 | [driver/upyun/upyun.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/filemanager/driver/upyun/upyun.go) | Token / ValidateCallback（签名原文含 URL.Path, L373-378） / sign 函数 |
| URL 构造 | [cluster/routes/routes.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/cluster/routes/routes.go) | MasterSlaveCallbackUrl(L46-49) |
| HMAC 签名实现 | [auth/auth.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/auth/auth.go) | CheckRequest / getSignContent（含 URL.Path L119+L226） |
| HMAC 密钥处理 | [auth/hmac.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/auth/hmac.go) | HMACAuth.Sign / Check |
| TTL 配置 | [setting/provider.go](file:///d:/fz/0601-2/solo-dogfeeding/code/14-Cloudreve/pkg/setting/provider.go#L691-L693) | UploadSessionTTL 默认值 |
