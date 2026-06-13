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

## 二、预览地址生成

### 2.1 Manager 层的决策路径

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

### 2.2 EntitySource 层的 URL 路由

URL 生成入口：[EntitySource.Url()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/entitysource/entitysource.go#L587-L668)

当 Manager 层返回 EntitySource 后，其 `Url()` 方法会根据两个概念决定具体 URL 形态：

**模式 A — 内部代理模式（Cloudreve 反代）**
触发条件（满足任一即可）：
- 驱动静态能力声明 `HandlerCapabilityProxyRequired`（例如 Local 驱动）
- 存储策略配置了 `InternalProxy` 且调用方未显式关闭
- 实体尚未落库（ID 为 0）
- 实体已加密且调用方未显式关闭代理

此模式下，URL 指向 Cloudreve 自身接口：`{SiteURL}/api/v3/file/content/{hashid}?...`，由 Cloudreve 再向底层存储拉取数据并回传给客户端。所有响应会经过签名与过期校验。

**模式 B — 存储直链模式**
无需内部代理时，URL 直接由驱动生成：
- 若请求的是缩略图（`IsThumb=true`），调用 `handler.Thumb()` 返回带图片处理参数的签名 URL
- 否则调用 `handler.Source()` 返回普通下载签名 URL

两种模式最终都会经过 `driver.ApplyProxyIfNeeded()`，用于叠加 CDN 域名或外部反代配置。

### 2.3 原生缩略 URL 的生成细节

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

## 三、缓存策略

缩略图系统的缓存设计横跨四个概念层次：

### 3.1 持久化实体缓存

缩略图一旦成功生成，就会作为 `EntityTypeThumbnail` 类型的实体被持久化存储，与原文件建立关联。后续请求直接复用该实体（Manager 层的第二层判定），不再重新计算。

两种持久化模式对应不同部署形态：
- **主节点模式**：缩略图通过 `m.Update()` 写入存储策略，保存路径由 `ThumbEntitySuffix` 模板决定
- **从节点模式**（stateless）：缩略图作为 sidecar 文件写入源路径同目录，保存路径为 `原始路径 + ThumbSlaveSidecarSuffix`

### 3.2 签名 URL 内存缓存

非本地驱动需要反复生成带签名的访问 URL。EntitySource 内部维护了对最近一次生成 URL 的内存缓存：

```go
type entitySource struct {
    cachedUrl    string
    cachedExpiry time.Time
}
```

在读取远程资源时，若缓存 URL 距离过期还有至少 1 分钟余量，则直接复用；否则重新生成并刷新缓存。调用 `Apply()` 变更选项（会影响 URL 内容）时主动清理缓存。

### 3.3 HTTP 条件请求缓存

对本地驱动提供的文件，Serve 逻辑使用 ETag（由实体 ID 哈希生成）支持标准 HTTP 条件请求：
- `If-None-Match` 命中时返回 `304 Not Modified`
- `If-Match` 用于前置一致性校验
- `If-Range` 与 `Range` 结合用于条件范围请求

### 3.4 不可预览标记缓存

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

## 四、驱动能力差异

### 4.1 能力模型

所有缩略图相关能力由驱动的 `Capabilities()` 返回，核心字段定义：

[driver/handler.go Capabilities](file:///d:/fz/0601-1\solo-dogfeeding\code\47-Cloudreve\pkg\filemanager\driver\handler.go#L89-L111)

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

### 4.2 各驱动能力对照

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

**对 OneDrive 驱动的特别说明**：
其 `Thumb()` 已完整实现，调用 Microsoft Graph API 获取缩略图 URL。但 Graph API 本身对不支持的文件会返回错误，此时仍需要 `ThumbProxy` 配置作为兜底，回退到本地生成模式。

---

## 五、降级处理

### 5.1 请求链路的分级降级

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

### 5.2 本地生成管线内部的责任链降级

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

### 5.3 生成失败后的处理

当整条管线最终返回错误时：

[manager.generateThumb()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/thumbnail.go#L129-L222)

- 先删除已生成但不可用的临时缩略图文件
- 若错误不是上下文取消、且当前节点不是从节点（stateless），则把该文件标记为永久不可预览（写入 `ThumbDisabledKey`）
- 从节点不做永久标记，因为文件可能在主节点重新生成

### 5.4 Generator 内部的平台与场景降级

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

## 六、端到端数据流

```
前端 → GET /api/v3/file/thumb?uri=xxx
  ↓
FileThumbService.Get()
  ↓ manager.Thumbnail(uri)
  │
  ├─ [预检] ThumbDisabledKey? → 失败
  ├─ [复用] 已有缩略图实体?    → GetEntitySource → Url()
  ├─ [原生] 驱动支持且大小/扩展名匹配?
  │     → GetEntitySource(WithUseThumb=true)
  │     → EntitySource.Url()
  │        ├─ ProxyRequired / 加密 / 策略代理?
  │        │   → Cloudreve 内部代理 URL（模式 A）
  │        └─ 否则
  │            → handler.Thumb() 生成存储侧缩略 URL（模式 B）
  │            → ApplyProxyIfNeeded 叠加 CDN/反代
  ├─ [代理] ThumbProxy + 有 GenerateThumb 权限?
  │     → SubmitAndAwaitThumbnailTask
  │        → pipeline.Generate()
  │           ├─ LibreOffice/MusicCover/LibRaw 提取中间结果
  │           ├─ Vips/FFmpeg/Builtin 生成最终缩略
  │        → 上传为缩略图实体
  │        → GetEntitySource → Url()
  └─ [兜底] 以上均不满足 → disableThumb → 失败
  ↓
FileThumbResponse{Url, Expires}
```
