# Cloudreve 媒体预览与缩略图系统分析

## 一、系统总览

媒体预览系统是一个分层解耦的架构，从上到下依次为：API 层负责对外暴露缩略图接口，Manager 层负责驱动决策与任务调度，Driver 层封装各存储后端的原生能力，Pipeline 层负责本地缩略图生成，EntitySource 层统一输出带签名与过期时间的访问地址。

```
API 层 (service/explorer/file.go)
    ↓ FileThumbService.Get()
Manager 层 (pkg/filemanager/manager/thumbnail.go)
    ↓ manager.Thumbnail()
    ├─ 已有缩略图实体复用
    ├─ 驱动原生缩略图能力
    ├─ 本地生成代理（ThumbProxy）
    └─ 兜底禁用标记
    ↓
Driver 层        ←→      Pipeline 层 (pkg/thumb/)
handler.Thumb()           多 Generator 责任链
    ↓                          ↓
EntitySource 层 (entitysource/)
    ↓ EntitySource.Url()
签名 URL + 过期时间
```

外部入口：[FileThumbService.Get()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/service/explorer/file.go#L497-L524)

```go
type FileThumbService struct {
    Uri string `form:"uri" binding:"required"`
}
type FileThumbResponse struct {
    Url     string     `json:"url"`
    Expires *time.Time `json:"expires"`
}
```

前端通过文件 URI 请求缩略图，后端返回最终可访问 URL 及其过期时间。

---

## 二、从缩略图接口到图片数据的完整链路

### 2.1 两阶段分离：URL 获取 vs 图片加载

缩略图的完整消费过程分为**两个阶段**，由前端在不同的 HTTP 请求中完成：

**阶段一 — 获取缩略图 URL**

```
前端 → GET /api/v3/file/thumb?uri=xxx
    ↓
路由注册: routers/router.go#L662
    file.GET("thumb",
        middleware.ContextHint(),
        controllers.FromQuery[explorer.FileThumbService](...),
        controllers.Thumb)
    ↓
控制器: routers/controllers/file.go#L147
    service.Get(c) → FileThumbResponse{Url, Expires}
    ↓
前端收到: {"code":0,"data":{"url":"https://...","expires":"..."}}
```

此阶段只返回 URL 字符串和过期时间，**不返回图片二进制数据**。

**阶段二 — 加载图片数据**

前端拿到 URL 后，将其设为 `<img>` 标签的 `src`，浏览器发第二次请求加载图片。URL 的目标取决于阶段一生成的路径：

| URL 形态 | 浏览器请求目标 | 数据来源 |
|----------|-------------|---------|
| 存储直链（模式 B） | 云存储签名 URL | OSS/COS/OBS/OneDrive 等直接返回 |
| 内部代理 URL（模式 A） | `{SiteURL}/api/v3/file/content/{hashid}?thumb` | Cloudreve 反代到存储 |

### 2.2 内部代理路径的详细流转

当缩略图走内部代理模式时，图片数据的流转经过两层 HTTP 请求：

```
浏览器
  │ GET /api/v3/file/content/{entityHashId}/0/{filename}?thumb
  │ （签名参数在 query string 中）
  ▼
路由层: routers/router.go#L647
  content.GET(":id/:speed/:name",
      middleware.SignRequired(dep.GeneralAuth()),   ← 签名校验
      middleware.HashID(hashid.EntityID),            ← hashid 解码
      middleware.Sandbox(),                          ← 沙箱隔离
      controllers.ServeEntity)
  ▼
控制器: routers/controllers/file.go#L177
  EntityDownloadService.Serve(c)
  ▼
服务层: service/explorer/entity.go#L26
  m.GetEntitySource(c, entityID)    ← 获取 EntitySource
  entitySource.Serve(w, r,
      WithThumb(isThumb),           ← 告知 EntitySource 这是缩略请求
      WithContext(c))
  ▼
EntitySource.Serve(): entitysource.go#L270
  ├─ IsLocal? → 直接从本地磁盘读取，写入 HTTP Response
  └─ 非 Local? → Url() 获取存储直链 → ReverseProxy 反代
```

关键点：当浏览器通过内部代理 URL 加载缩略图时，`EntitySource.Serve()` 会再次调用 `Url()` 获取存储端的直链（此时不带 `IsThumb` 标志，因为实体已经是缩略图本身），然后通过 `httputil.ReverseProxy` 把请求反向代理到存储端。

### 2.3 从节点的缩略图流转

从节点（Slave）的缩略图走独立的 API 路径：

```
主节点请求从节点: GET /api/v3/slave/file/thumb/{base64src}/{ext}
    ↓
路由: routers/router.go#L92
  file.GET("thumb/:src/:ext",
      controllers.SlaveThumb)
    ↓
服务: service/explorer/slave.go#L157
  SlaveThumbService.Thumb(c)
    ├─ 尝试读取已有的本地 sidecar 缩略文件
    │   local.NewLocalFileEntity(EntityTypeThumbnail, src+ThumbSlaveSidecarSuffix)
    ├─ 不存在? → 从原文件生成
    │   local.NewLocalFileEntity(EntityTypeVersion, src)
    │   m.SubmitAndAwaitThumbnailTask(c, nil, ext, srcEntity)
    └─ 获取 EntitySource → Serve() 直接写回 HTTP Response
```

与主节点的区别：从节点**直接把图片数据写入 HTTP 响应**，而非返回 JSON URL。

---

## 三、预览地址生成

### 3.1 Manager 层的决策路径

决策入口：[manager.Thumbnail()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/thumbnail.go#L27-L101)

缩略图请求在 Manager 层经历一系列概念上分层的判定：

**第一层 — 预检：文件级禁用与类型过滤**
- 若文件元数据中已存在 `ThumbDisabledKey` 标记，或者目标不是文件实体，直接宣告不可预览并返回。这是为了避免对已知不支持的文件反复尝试。

**第二层 — 复用：已生成实体优先**
- 若该文件已关联 `EntityTypeThumbnail` 类型的实体（即缩略图已被生成并持久化），直接把该实体包装为 EntitySource 返回。这是成本最低的路径——不涉及任何计算或远程调用。

**第三层 — 原生：驱动侧实时缩略**
- 若驱动声明对当前扩展名/大小支持原生缩略图（且文件未加密），则由存储服务端在请求时实时完成缩略，Cloudreve 只负责签名下发 URL，不消耗自身计算资源。判断条件是：扩展名被支持（或被设为全部支持），文件大小在阈值内，且内容未加密。

**第四层 — 代理：本地异步生成**
- 若驱动声明支持 `ThumbProxy`（即允许 Cloudreve 自己在本地生成缩略图），且文件系统具备生成新缩略图的权限，则进入本地生成管线：把任务加入缩略图队列，等待从原图中生成缩略图，上传为新实体后再返回。

**第五层 — 兜底：永久标记不可用**
- 若以上条件均不满足，则在文件元数据中写入 `ThumbDisabledKey`，后续请求在第一层就被拦截，避免重复尝试。

### 3.2 EntitySource 层的 URL 路由

URL 生成入口：[EntitySource.Url()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/entitysource/entitysource.go#L587-L668)

当 Manager 层返回 EntitySource 后，其 `Url()` 方法会根据条件决定具体 URL 形态：

**模式 A — 内部代理模式（Cloudreve 反代）**
触发条件（满足任一即可）：
- 驱动静态能力声明 `HandlerCapabilityProxyRequired`（例如 Local 驱动）
- 存储策略配置了 `InternalProxy` 且调用方未显式关闭
- 实体尚未落库（ID 为 0）
- 实体已加密且调用方未显式关闭代理

此模式下，URL 由 [MasterFileContentUrl()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/cluster/routes/routes.go#L147-L166) 构造，格式为：

```
{SiteURL}/api/v3/file/content/{entityHashId}/{speedLimit}/{filename}?thumb
```

其中 `?thumb` 查询参数由 [IsThumbQuery](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/cluster/routes/routes.go#L16)（值 `"thumb"`）设置，告知后续的 `EntityDownloadService.Serve()` 这是缩略图请求。整个 URL 会被 `auth.SignURI()` 添加 HMAC 签名和过期时间。

**模式 B — 存储直链模式**
无需内部代理时，URL 直接由驱动生成：
- 若请求的是缩略图（`IsThumb=true`），调用 `handler.Thumb()` 返回带图片处理参数的签名 URL
- 否则调用 `handler.Source()` 返回普通下载签名 URL

两种模式最终都会经过 `driver.ApplyProxyIfNeeded()`，用于叠加 CDN 域名或外部反代配置。

### 3.3 原生缩略 URL 的生成细节

当走模式 B + `IsThumb=true` 时，各驱动通过不同方式把"尺寸+编码"参数注入签名 URL：

- **OSS**：`image/resize,m_lfit,h_{h},w_{w}/format,{fmt}/quality,q_{q}` 通过 `GetObjectRequest.Process` 字段传入签名
- **COS**：`imageMogr2/thumbnail/{w}x{h}/format/{fmt}/rquality/{q}` 作为 URL 查询参数附加
- **OBS**：`image/resize,m_lfit,w_{w},h_{h}/format,{fmt}/quality,q_{q}` 通过 `QueryParams` 传入签名
- **Qiniu**：`imageView2/1/w/{w}/h/{h}/format/{fmt}/q/{q}` 作为空值 key 的查询参数附加
- **Upyun**：`!/fwfh/{w}x{h}/format/{fmt}/quality/{q}` 直接拼接到源路径后再整体签名
- **KS3**：`@base@tag=imgScale&m=0&w={w}&h={h}&q={q}&F={fmt}` 直接拼接进对象 Key 后再预签名
- **OneDrive**：通过 Microsoft Graph 的缩略图端点 `GetThumbURL` 生成（非 URL 参数拼接）
- **Remote**：构造从节点缩略图 URL 并签名，转发给实际持有文件的从节点处理
- **Local / S3**：`handler.Thumb()` 直接返回 "not implemented"，因此在 Manager 层的第三层决策中永远不会命中；但 S3 可通过策略配置走第四层代理

所有驱动共享同一配置源：`settings.ThumbSize()`（目标尺寸）和 `settings.ThumbEncode()`（输出格式与质量）。

---

## 四、出错时的返回与默认图片

### 4.1 FileThumbService.Get() 内部的三个错误位点

[FileThumbService.Get()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/service/explorer/file.go#L497-L524) 内部有三个独立的错误触发点，每个错误被不同方式包装和传递：

```go
func (s *FileThumbService) Get(c *gin.Context) (*FileThumbResponse, error) {
    // ...
    uri, err := fs.NewUriFromString(s.Uri)       // ① 位点 A: URI 解析
    if err != nil {
        return nil, serializer.NewError(serializer.CodeParamErr, "unknown uri", err)
    }
    thumb, err := m.Thumbnail(c, uri)             // ② 位点 B: Manager 决策层
    if err != nil {
        return nil, fmt.Errorf("failed to get thumbnail: %w", err)
    }
    thumbUrl, err := thumb.Url(c, ...)            // ③ 位点 C: EntitySource 生成 URL
    if err != nil {
        return nil, fmt.Errorf("failed to get thumbnail url: %w", err)
    }
    // ...
}
```

### 4.2 错误响应的序列化机制

控制器层统一调用 `serializer.Err(c, err)`，其内部实现为：

[serializer.ErrWithDetails()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/serializer/error.go#L341-L375)

```go
func ErrWithDetails(c context.Context, errCode int, msg string, err error) Response {
    res := Response{Code: errCode, Msg: msg, CorrelationID: ...}
    var appError AppError
    if errors.As(err, &appError) {       // ← 关键：沿 %w 包装链递归查找 AppError
        res.Code = appError.ErrCode()    // ErrCode() 继续递归解包嵌套 AppError
        err = appError.RawError
        res.Msg = appError.Msg
    }
    if err != nil && gin.Mode() != gin.ReleaseMode {
        res.Error = err.Error()          // 非生产环境暴露完整错误文本
    }
    return res
}
```

`errors.As(err, &appError)` 会沿 `fmt.Errorf("%w", ...)` 的包装链一直深入，直到找到最内层的 `AppError`。如果整条链中不存在 `AppError`，则保留初始传入的 `CodeNotSet = -1` 和空 `Msg`。

基于此机制，三个错误位点分别产生以下几类响应：

### 4.3 错误类型分类表

| 错误来源 | 触发场景 | 错误包装链 | 最内层是否 AppError | 最终 code | 最终 msg | 生产环境 error 字段 |
|---------|---------|-----------|-------------------|-----------|----------|-------------------|
| **位点 A** URI 解析 | URI 格式非法 | `NewError(CodeParamErr, "unknown uri", err)` | 是 | **40001** | `"unknown uri"` | 空 |
| **位点 B** Manager 预检 + 决策兜底 | ThumbDisabledKey 命中 / 所有降级路径均不可用 / 目录无主实体 | `"failed to get thumbnail: %w"` → `fs.ErrEntityNotExist` → `NewError(40077, "Entity not exist")` | 是 | **40077** | `"Entity not exist"` | 空 |
| **位点 B** Manager 中间步骤失败 | 缩略图实体已关联但 EntitySource 创建失败 / 队列任务执行失败 | `"failed to get thumbnail: %w"` → `"failed to get entity source: %w"` / `"failed to execute thumb task: %w"` → 可能是 DB 错误 / 上下文取消等 | 可能是 | 取决于内层 | 取决于内层 | 空 |
| **位点 C** 存储直链签名失败（S3/OSS/COS 等） | 签名密钥异常 / 临时凭证过期 | `"failed to get thumbnail url: %w"` → 各 SDK 原生错误 | 否 | **-1** (CodeNotSet) | **空字符串** | 空 |
| **位点 C** OneDrive Graph API 失败（※重要） | 不支持的文件格式 / Token 过期 / 网络异常 | `"failed to get thumbnail url: %w"` → `"thumb not supported in OneDrive: %w"` / 其他 Graph 错误 | 否 | **-1** (CodeNotSet) | **空字符串** | 空 |
| **位点 C** 内部代理 URL 构建失败 | 签名密钥缺失 / HashID 编码失败 | `"failed to get thumbnail url: %w"` → 底层错误 | 可能是 | 取决于内层 | 取决于内层 | 空 |

### 4.4 OneDrive 错误响应的隐蔽性

这是最容易误解的一类错误：OneDrive 的缩略图错误全部发生在 **位点 C**，即在 Manager 决策之后的 URL 生成阶段。

代码路径：
- Manager 层判定 OneDrive 具备原生能力，返回 EntitySource ✓
- 调用 `thumb.Url()` → `handler.Thumb()` → `client.GetThumbURL()` 发起 Graph API 请求 ✗
- [onedrive.go#L139-L150](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/driver/onedrive/onedrive.go#L139-L150) 中对错误的包装使用 `fmt.Errorf("thumb not supported in OneDrive: %w", err)`

由于整个包装链中**没有任何 AppError**，最终序列化结果在生产环境为：

```json
{
  "code": -1,
  "msg": "",
  "correlation_id": "xxxxx"
}
```

前端看到 `code = -1`，只能判定为"出错了"，但无法区分具体是 OneDrive 不支持该文件、还是网络问题、亦或是 Token 过期。`msg` 字段为空字符串，`error` 字段在生产环境被隐藏。

**调试提示**：在 debug 模式（`GIN_MODE=debug`）下，`error` 字段会暴露完整错误链，例如：`"failed to get thumbnail url: thumb not supported in OneDrive: large thumbnail size not found"`。

### 4.5 成功响应格式

```json
{
  "code": 0,
  "data": {
    "url": "https://xxx.oss-cn-hangzhou.aliyuncs.com/path/to/thumb.jpg?x-oss-process=image%2Fresize...&OSSAccessKeyId=...&Expires=...&Signature=...",
    "expires": "2026-06-13T10:05:00Z"
  }
}
```

### 4.6 前端无服务端默认图片

**Cloudreve 后端不提供任何默认的缩略图占位图 URL**。当缩略图 API 返回 `code != 0` 时：

- 前端仅能通过 `code` 字段判定失败，无 fallback URL 可用
- 前端自行决定展示方式（通常使用本地打包的文件类型 SVG 图标作为占位）
- 不存在让 `<img>` 从后端加载一张"占位图"的机制

这与社交分享场景不同——社交分享在服务端有 PWA 图标作为兜底（见第八章）。

### 4.7 内部代理路径中的 HTTP 状态码错误

当浏览器通过内部代理 URL（`/api/v3/file/content/{id}?thumb`）实际加载图片二进制数据时，错误以原始 HTTP 状态码直接返回：

[EntitySource.Serve()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/entitysource/entitysource.go#L270-L384)

| 场景 | HTTP 状态码 | 响应体 |
|------|-----------|--------|
| 本地实体底层数据不存在（文件被删但 DB 记录还在） | 404 | `"Entity data does not exist."` |
| 远程文件签名 URL 生成失败 | 500 | 错误信息字符串（取决于运行环境） |
| 远程文件签名 URL 格式解析失败 | 500 | 错误信息字符串 |
| 反向代理到存储端时连接失败 / 超时 | 502 | `"[Cloudreve] Bad Gateway"` |
| 本地文件 Seek 失败（Range 请求异常） | 500 | `"seeker can't seek"` |
| 签名校验不通过（URL 被篡改 / 过期） | 401 | `{"code":401, "msg":"signature invalid"}` |

浏览器收到这些非 2xx 状态码后，`<img>` 标签显示为破碎图标，前端可监听 `onerror` 事件切换到本地占位图。

---

## 五、缓存策略

缩略图系统的缓存设计横跨四个概念层次：

### 5.1 持久化实体缓存

缩略图一旦成功生成，就会作为 `EntityTypeThumbnail` 类型的实体被持久化存储，与原文件建立关联。后续请求直接复用该实体（Manager 层的第二层判定），不再重新计算。

两种持久化模式对应不同部署形态：
- **主节点模式**：缩略图通过 `m.Update()` 写入存储策略，保存路径由 `ThumbEntitySuffix` 模板决定
- **从节点模式**（stateless）：缩略图作为 sidecar 文件写入源路径同目录，保存路径为 `原始路径 + ThumbSlaveSidecarSuffix`

### 5.2 签名 URL 内存缓存

非本地驱动需要反复生成带签名的访问 URL。EntitySource 内部维护了对最近一次生成 URL 的内存缓存：

```go
type entitySource struct {
    cachedUrl    string
    cachedExpiry time.Time
}
```

在读取远程资源时，若缓存 URL 距离过期还有至少 1 分钟余量，则直接复用；否则重新生成并刷新缓存。调用 `Apply()` 变更选项（会影响 URL 内容）时主动清理缓存。

### 5.3 HTTP 条件请求缓存

对本地驱动提供的文件，Serve 逻辑使用 ETag（由实体 ID 哈希生成）支持标准 HTTP 条件请求：
- `If-None-Match` 命中时返回 `304 Not Modified`
- `If-Match` 用于前置一致性校验
- `If-Range` 与 `Range` 结合用于条件范围请求

### 5.4 不可预览标记缓存

当缩略图生成失败或驱动明确不支持时，系统在文件元数据中写入 `ThumbDisabledKey`：

```go
func disableThumb(ctx context.Context, m *manager, uri *fs.URI) error {
    return m.fs.PatchMetadata(ctx, []*fs.URI{uri}, fs.MetadataPatch{
        Key:   dbfs.ThumbDisabledKey,
        Value: "",
    })
}
```

此标记使得 Manager 层在预检阶段就宣告不可预览，避免了无效重试。当文件被重新上传或覆盖时，该标记会被自动清除。

---

## 六、驱动能力差异

### 6.1 能力模型

所有缩略图相关能力由驱动的 `Capabilities()` 返回，核心字段定义：

[driver/handler.go Capabilities](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/driver/handler.go#L89-L111)

```go
type Capabilities struct {
    ThumbSupportedExts  []string  // 原生支持的缩略图扩展名白名单
    ThumbSupportAllExts bool      // 是否对所有扩展名启用原生缩略图
    ThumbMaxSize        int64     // 原生缩略图最大文件大小（0=无限）
    ThumbProxy          bool      // 是否允许 Cloudreve 本地代理生成
    StaticFeatures      *boolset.BooleanSet  // 静态能力位（含 ProxyRequired 等）
}
```

这些能力字段共同决定 Manager 层在第三层（原生）和第四层（代理）之间的路径选择。

### 6.2 各驱动能力对照

| 驱动 | 原生 Thumb 是否实现 | ThumbSupportedExts | ThumbProxy | 是否强制内部代理 |
|------|--------------------|-------------------|------------|-----------------|
| **Local** | ✗（返回 "not implemented"） | nil（零值） | ✓（硬编码 true） | ✓（ProxyRequired 强制） |
| **S3** | ✗（返回 "not implemented"） | nil | 由策略决定 | ✗ |
| **OSS** | ✓（阿里云图片处理 API） | 由策略配置 | 由策略决定 | ✗ |
| **COS** | ✓（腾讯云数据万象） | 由策略配置 | 由策略决定 | ✗ |
| **OBS** | ✓（华为云图片处理） | 由策略配置 | 由策略决定 | ✗ |
| **Qiniu** | ✓（七牛数据处理） | 由策略配置 | 由策略决定 | ✗ |
| **Upyun** | ✓（又拍云图片处理） | 由策略配置 | 由策略决定 | ✗ |
| **KS3** | ✓（金山云图片处理） | nil | 由策略决定 | ✗ |
| **OneDrive** | ✓（Graph API GetThumbURL） | 由策略配置 | 由策略决定 | ✗ |
| **Remote** | ✓（转发到从节点） | 由策略配置 | 由策略决定 | ✗ |

**对 Local 驱动的特别说明**：
由于 `ThumbSupportedExts=nil`、`ThumbSupportAllExts=false`，第三层（原生）的判定条件永远为假；而 `ThumbProxy=true` 导致它必然进入第四层（本地代理生成）。又因为 `HandlerCapabilityProxyRequired`，其最终输出的 URL 永远走内部代理模式，不会出现外部直链。

---

## 七、降级处理

### 7.1 请求链路的分级降级

缩略图请求的处理过程，本质上是一条从"最省资源"到"最耗资源"的分级降级链，由 Manager 层的 Thumbnail 方法串联：

```
低成本  ←——————————————————————————————————————→ 高成本
  预检     复用实体     驱动原生     本地代理     标记禁用
(拦截)    (零开销)    (零存储)    (生成+上传)    (持久失败)
```

- **预检**：成本最低，只检查一个元数据字段，命中即终止
- **复用实体**：只读取已存在的缩略图实体，不做任何计算
- **驱动原生**：只签名一个 URL，存储侧承担全部计算
- **本地代理**：需下载原图、调外部进程（如 vips/ffmpeg）、上传结果，成本最高
- **标记禁用**：当所有路径均不可行时，将失败结论持久化，避免后续重复消耗

### 7.2 OneDrive 原生缩略图的降级缺口

OneDrive 的缩略图处理存在一个**判定与执行分离**的架构问题，导致降级链在"驱动原生"这一环断裂。

**问题根源**：Manager 层的决策（走第三层原生路径）仅依据静态 `Capabilities` 字段判断，而真正调用 Graph API 获取缩略图是在后续 `EntitySource.Url()` 中执行。这两个阶段在调用栈上是分离的。

**判定阶段**（Manager 层 `Thumbnail()` 中）：
- 检查 `ThumbSupportAllExts` / `ThumbSupportedExts` / `ThumbMaxSize` 等字段
- 只要这些字段匹配，就判定为"原生支持"，返回 EntitySource
- 此阶段**不**实际调用 Graph API

**执行阶段**（`EntitySource.Url()` 中）：
- 调用 `handler.Thumb()` → `client.GetThumbURL()`
- 发起实际的 Graph API 请求 `/drive/root:/{path}:/thumbnails/0/large`
- 此时才可能发生错误

**OneDrive 缩略图错误分类**：
[onedrive.go#L139-L150](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/driver/onedrive/onedrive.go#L139-L150)

```go
func (handler *Driver) Thumb(ctx context.Context, expire *time.Time, ext string, e fs.Entity) (string, error) {
    res, err := handler.client.GetThumbURL(ctx, e.Source())
    if err != nil {
        var apiErr *RespError
        if errors.As(err, &apiErr); err == ErrThumbSizeNotFound ||
           (apiErr != nil && apiErr.APIError.Code == notFoundError) {
            return "", fmt.Errorf("thumb not supported in OneDrive: %w", err)
        }
    }
    return res, nil
}
```

- **业务不可用错误**（可识别的"不支持"类）：
  - `ErrThumbSizeNotFound`：Graph API 响应中不存在 `large` 尺寸的缩略图（[api.go#L460](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/driver/onedrive/api.go#L460)）
  - `itemNotFound`：文件/项在 OneDrive 中不存在
  - 以上两种会被包装为 `fmt.Errorf("thumb not supported in OneDrive: %w", err)` 返回
- **其他运行时错误**：网络错误、Token 过期、权限不足等，直接原样向上抛出

**降级缺口**：
```
Manager.Thumbnail()               EntitySource.Url()
      │                               │
      ├─ 判定 Capabilities ✓           ├─ handler.Thumb() ✗
      └─ 返回 EntitySource              └─ 错误向上冒泡
         （已无法回头）                     （无法回退到 ThumbProxy）
```

错误发生时，调用栈已离开 Manager 层的决策逻辑，无法触发第三层 → 第四层的降级。这意味着：
- 对 OneDrive 中 Graph API 不支持的文件，`FileThumbService.Get()` 会返回 `code=40077` 的错误
- 前端收到错误后，不会获得任何缩略图 URL
- 只有当策略配置 `ThumbProxy=true` 且 `ThumbSupportedExts` 不包含该扩展名时，才能避开此问题，走正常的代理生成路径

### 7.3 本地生成管线内部的责任链降级

当请求进入本地代理路径后，缩略图由 Pipeline 内的一组 Generator 按优先级责任链尝试：

[pipeline.Generate()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/thumb/pipeline.go#L86-L124)

六个内置 Generator 的优先级与职责：

| Generator | Priority | 典型职责 | 结果语义 |
|-----------|----------|---------|---------|
| **LibreOfficeGenerator** | 50 | 文档转 PNG（doc/pdf/ppt 等） | 中间结果 → 交付下游缩放 |
| **MusicCoverGenerator** | 50 | 音频封面提取（mp3/flac 等） | 中间结果 → 交付下游缩放 |
| **LibRawGenerator** | 50 | RAW 图片内嵌缩略图提取 | 中间结果 → 交付下游缩放 |
| **VipsGenerator** | 100 | 高性能通用图片缩放/转码 | 最终结果 |
| **FfmpegGenerator** | 200 | 视频帧抽取生成缩略图 | 最终结果 |
| **BuiltinGenerator** | 300 | Go 原生 image 库解码缩放（jpg/png/gif） | 最终结果 |

责任链的降级机制：
1. 当前 Generator 不支持该格式时，返回 `ErrPassThrough`，管线自动切换到下一个 Generator
2. 若 Generator 仅产出中间产物（如从文档渲染出第一页的 PNG、从音频里抽出原始封面），会把 `Result.Continue` 置为 true。此时管线会把中间产物转为本地 EntitySource，并把扩展名同步更新，再交给后续 Generator 继续处理
3. 所有 Generator 都 `ErrPassThrough` 时，管线返回 `ErrNotAvailable`

中间结果的清理函数（`Result.Cleanup`）通过 `defer` 注册，在管线结束后执行临时目录/文件清理。

### 7.4 生成失败后的处理

当整条管线最终返回错误时：

[manager.generateThumb()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/thumbnail.go#L129-L222)

- 先删除已生成但不可用的临时缩略图文件
- 若错误不是上下文取消、且当前节点不是从节点（stateless），则把该文件标记为永久不可预览（写入 `ThumbDisabledKey`）
- 从节点不做永久标记，因为文件可能在主节点重新生成

### 7.5 Generator 内部的平台与场景降级

在单个 Generator 内部，也存在针对运行环境和输入条件的降级分支：

**FFmpeg 的输入降级**：
- 本地未加密文件 → 直接传本地文件路径给 ffmpeg
- 远程/加密文件 → 先获取签名下载 URL，ffmpeg 通过 HTTP 拉取

**Vips 在 Windows 上的降级**：
- Linux/macOS 下，Vips 支持通过 stdin 管道接收输入（零拷贝）
- Windows 下 Vips 的管道实现存在问题，因此退化：
  - 本地未加密文件 → 改为传文件路径字符串 `[filename="xxx"]`
  - 远程/加密文件 → 先把输入完整落盘到临时文件，再传路径给 Vips

这些降级逻辑的目的是在不改变最终产物的前提下，尽可能适配不同平台的执行环境。

---

## 八、社交分享预览入口

社交媒体分享时（如微信、Twitter、Facebook、Discord 等），爬虫会抓取页面的 Open Graph 元标签来生成卡片预览。Cloudreve 对此提供了专门的中间件链路，复用了现有的缩略图系统。

### 8.1 爬虫识别与 OG 页面渲染

入口中间件：[SharePreview](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/middleware/share_preview.go#L84-L103)

```go
func SharePreview(dep dependency.Dep) gin.HandlerFunc {
    return func(c *gin.Context) {
        if !isSocialMediaBot(c.GetHeader("User-Agent")) {
            c.Next()  // 非爬虫 → 正常走后续路由
            return
        }
        // 爬虫 → 渲染 OG 页面并返回
        id, password := extractShareParams(c)
        html := renderShareOGPage(c, dep, id, password)
        c.String(200, html)
        c.Abort()  // 中止后续路由
    }
}
```

**爬虫识别**：通过 User-Agent 关键字匹配：
- `facebookexternalhit`、`facebot`（Facebook / Messenger）
- `twitterbot`（Twitter / X）
- `linkedinbot`（LinkedIn）
- `discordbot`（Discord）
- `telegrambot`（Telegram）
- `slackbot`（Slack）
- `whatsapp`（WhatsApp）

识别到爬虫后，不走正常的前端重定向逻辑，而是返回一个包含 Open Graph / Twitter Card 元标签的 HTML 页面。

### 8.2 OG 页面结构与缩略图注入

OG 页面模板：[ogHTMLTemplate](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/middleware/share_preview.go#L37-L57)

```html
<!DOCTYPE html>
<html>
<head>
    <meta property="og:title" content="{{.Title}}">
    <meta property="og:description" content="{{.Description}}">
    <meta property="og:image" content="{{.ImageURL}}">
    <meta property="og:url" content="{{.ShareURL}}">
    <meta property="og:type" content="website">
    <meta name="twitter:card" content="summary">
    <meta name="twitter:image" content="{{.ImageURL}}">
    ...
</head>
<body>
    <script>window.location.href = "{{.RedirectURL}}";</script>
</body>
</html>
```

- `og:image` / `twitter:image`：卡片预览图，即缩略图 URL
- `og:title`：文件名
- `og:description`：文件大小 + 上传者昵称
- `<script>` 标签：真实用户点击链接时重定向到前端页面

### 8.3 缩略图逻辑的复用

社交分享的缩略图完全复用现有的 `FileThumbService` 逻辑，入口：

[loadShareThumbnail()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/middleware/share_preview.go#L181-L201)

```go
func loadShareThumbnail(c *gin.Context, shareID, password string, shareInfo *explorer.Share) (string, error) {
    shareUri, err := fs.NewUriFromString(fs.NewShareUri(shareID, password))
    subService := &explorer.FileThumbService{
        Uri: shareUri.Join(shareInfo.Name).String(),
    }
    if err := SetUserCtx(c, 0); err != nil {
        return "", err
    }
    res, err := subService.Get(c)
    return res.Url, nil
}
```

**上下文构造**：
- 通过 `fs.NewShareUri()` 构造分享专属 URI（格式：`share://{id}/{password}`）
- 通过 `SetUserCtx(c, 0)` 注入匿名（访客）用户身份，因为爬虫不会携带登录态
- 调用与登录用户完全相同的 `FileThumbService.Get()` 方法

### 8.4 社交分享的降级路径

社交分享场景下的缩略图有四层降级：

```
已生成缩略图实体? → 使用文件缩略图 URL
          ↓
驱动原生支持? → 生成原生缩略 URL
          ↓
本地代理生成? → 生成并上传 → 使用
          ↓
获取缩略图失败? → 静默降级到 PWA 图标作为默认 og:image
```

前三层与普通缩略图逻辑完全一致，**社交分享特有的兜底**在：

[renderShareOGPage()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/middleware/share_preview.go#L129-L179)

```go
if pwa.LargeIcon != "" {
    data.ImageURL = resolveURL(base, pwa.LargeIcon)
} else if pwa.MediumIcon != "" {
    data.ImageURL = resolveURL(base, pwa.MediumIcon)
}

thumbnail, err := loadShareThumbnail(c, id, password, shareInfo)
if err == nil {
    data.ImageURL = thumbnail
}
```

**与普通缩略图的不同**：普通缩略图失败时返回 `code=40077` 的 API 错误；而社交分享失败时**静默降级**到 PWA 应用图标，保证 OG 页面始终有一张图可供爬虫抓取。**这是整个系统中唯一存在服务端默认图片的场景**——普通文件预览没有后端提供的默认占位图。

### 8.5 社交分享与普通预览的调用对比

| 维度 | 普通文件预览 | 社交分享预览 |
|------|------------|------------|
| 入口 | `/api/v3/file/thumb` | 中间件识别爬虫 UA |
| 用户身份 | 当前登录用户 | 匿名访客（UID=0） |
| URI 类型 | 普通文件 URI | Share URI（含ID和密码） |
| 缩略图逻辑 | `FileThumbService.Get()` | 完全复用 `FileThumbService.Get()` |
| 失败处理 | 返回 API 错误（code=40077） | 降级到 PWA 图标，静默处理 |
| 输出格式 | JSON（URL + Expires） | HTML（含 og:image 元标签） |
| URL 过期策略 | 可配置 | 与普通预览相同 |
| 是否有默认图片 | 否（前端自行处理） | 是（PWA 图标） |

---

## 九、全景调用关系总结

```
┌────────────────────────────────────────────────────────────────────┐
│  外部触发                                                            │
│  ├─ 前端: GET /api/v3/file/thumb?uri=xxx (阶段一：获取 URL)         │
│  ├─ 浏览器: GET {thumbUrl} (阶段二：加载图片数据)                    │
│  └─ 社交爬虫: GET /s/{id}/{pwd} (UA 含 facebookexternalhit 等)     │
└──────────────────────────────────────┬─────────────────────────────┘
                                       │
                    ┌──────────────────┴──────────────────┐
                    │                                      │
                    ▼                                      ▼
    ┌─────────────────────────────┐      ┌─────────────────────────────┐
    │  普通预览 / 内部代理加载      │      │  社交分享预览               │
    │  FileThumbService.Get()     │      │  SharePreview 中间件         │
    │  → JSON{Url, Expires}       │      │  → 识别爬虫 UA              │
    │                             │      │  → 复用 FileThumbService    │
    │  或 EntityDownloadService   │      │  → 失败降级到 PWA 图标      │
    │  .Serve()                   │      │  → HTML OG 页面             │
    │  → 图片二进制数据            │      └─────────────────────────────┘
    └──────────────┬──────────────┘
                   │
                   ▼
    ┌────────────────────────────────────┐
    │  manager.Thumbnail() 决策链         │
    │  [预检] → [复用] → [原生] → [代理] │
    └───────────────────┬────────────────┘
                        │
    ┌───────────────────┴────────────────┐
    │                                      │
    ▼                                      ▼
┌─────────────────────┐      ┌─────────────────────┐
│ EntitySource.Url()  │      │ pipeline.Generate() │
│ ├─ 模式 A: 内部代理 │      │ Generator 责任链     │
│ │  /api/v3/file/    │      │  (6 个 Generator)   │
│ │   content/{id}    │      └───────────┬─────────┘
│ └─ 模式 B: 存储直链 │                  │
│   handler.Thumb()   │                  ▼
│   → 签名 URL        │      上传为缩略图实体
└─────────────────────┘      → 再次走 Url() 生成访问地址
```
