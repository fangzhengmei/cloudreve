# Cloudreve 媒体预览与缩略图系统分析

## 一、整体架构

媒体预览（缩略图）系统分为三层：

```
API 层 (service/explorer/file.go)
    ↓ FileThumbService.Get()
Manager 层 (pkg/filemanager/manager/thumbnail.go)
    ↓ manager.Thumbnail() → manager.generateThumb()
    ↓                       ↓
驱动层 (driver/)            生成管线 (pkg/thumb/pipeline.go)
    ↓ handler.Thumb()           ↓ pipeline.Generate()
    ↓                           ↓ 逐个 Generator 尝试
实体源层 (entitysource/)       Builtin / Vips / FFmpeg / LibreOffice / MusicCover / LibRaw
    ↓ EntitySource.Url()
URL 输出 (带签名/过期时间)
```

入口 API：[FileThumbService.Get()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/service/explorer/file.go#L497-L524)

```go
type FileThumbService struct {
    Uri string `form:"uri" binding:"required"`
}
type FileThumbResponse struct {
    Url     string     `json:"url"`
    Expires *time.Time `json:"expires"`
}
```

前端请求缩略图时，传入文件 URI，后端返回签名 URL + 过期时间。

---

## 二、预览地址生成

### 2.1 核心流程：`manager.Thumbnail()`

入口：[manager.Thumbnail()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/thumbnail.go#L27-L101)

整个预览地址的生成遵循 4 步优先级决策：

| 步骤 | 条件 | 行为 | URL 生成方式 |
|------|------|------|-------------|
| Step 0 | 文件元数据含 `ThumbDisabledKey` 或非文件类型 | 直接返回 `ErrEntityNotExist` | — |
| Step 1 | 已存在 `EntityTypeThumbnail` 类型实体 | 复用已生成的缩略图实体 | 通过 `GetEntitySource` → `EntitySource.Url()` |
| Step 2 | 驱动原生支持缩略图（`ThumbSupportedExts` / `ThumbSupportAllExts`） | 原生实时缩略图 | `EntitySource.Url(WithUseThumb(true))` → `handler.Thumb()` |
| Step 3 | 驱动支持 `ThumbProxy`（本地生成代理） | 提交异步任务生成本地缩略图 | 生成后作为新实体存储 |
| Step 4 | 以上均不满足 | 标记 `ThumbDisabledKey`，永久禁用该文件缩略图 | — |

### 2.2 `EntitySource.Url()` — URL 生成核心

入口：[EntitySource.Url()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/entitysource/entitysource.go#L587-L668)

URL 生成有两条路径：

#### 路径 A：内部代理（Internal Proxy）

触发条件（满足任一）：
1. 驱动声明 `HandlerCapabilityProxyRequired`
2. 策略设置 `InternalProxy = true` 且未显式禁用
3. 实体为空（ID == 0）
4. 实体已加密且未显式禁用代理

```go
siteUrl := f.settings.SiteURL(ctx)
base := routes.MasterFileContentUrl(siteUrl, entityID, displayName, isDownload, isThumb, speedLimit)
srcUrl, err = auth.SignURI(ctx, f.generalAuth, base.String(), expire)
```

生成的 URL 格式：`{SiteURL}/api/v3/file/content/{hashid}?...`，由 Cloudreve 自身反代至后端存储。

#### 路径 B：存储服务直链

触发条件：无需内部代理。

```go
if f.o.IsThumb {
    srcUrlStr, err = f.handler.Thumb(ctx, expire, ext, f.e)
} else {
    srcUrlStr, err = f.handler.Source(ctx, f.e, &driver.GetSourceArgs{...})
}
```

- 如果是缩略图请求（`IsThumb=true`），调用 `handler.Thumb()` 获取带图片处理参数的签名 URL
- 否则调用 `handler.Source()` 获取普通下载链接

最后应用 CDN/代理覆盖：`driver.ApplyProxyIfNeeded(policy, srcUrl)`

---

## 三、缓存策略

### 3.1 已生成缩略图实体缓存（持久化）

[thumbnail.go Step 1](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/thumbnail.go#L39-L52)

缩略图生成后作为 `EntityTypeThumbnail` 类型实体持久化存储在文件系统中，与原始文件关联。下次请求时直接复用，无需重新生成。

存储方式因节点类型不同：
- **主节点**（非 stateless）：通过 `m.Update()` 上传至存储策略，保存路径由 `ThumbEntitySuffix` 配置决定
- **从节点**（stateless）：通过 `d.Put()` 上传为 sidecar 文件，保存路径为 `原始路径 + ThumbSlaveSidecarSuffix`

### 3.2 URL 签名缓存（内存级）

[entitysource.go cachedUrl](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/entitysource/entitysource.go#L178-L181)

```go
type entitySource struct {
    cachedUrl    string
    cachedExpiry time.Time
}
```

在 `getRsc()` 中，非本地文件请求会缓存已签名的 URL：

```go
if f.cachedUrl != "" && now.Before(f.cachedExpiry.Add(-time.Minute)) {
    urlStr = f.cachedUrl  // 命中缓存
} else {
    u, err := f.Url(...)  // 重新生成
    f.cachedUrl = u.Url
    f.cachedExpiry = expire
}
```

**缓存淘汰**：
- 过期时间前 1 分钟即视为失效（安全余量）
- 调用 `Apply()` 修改选项时主动清除缓存（`clearUrlCache()`）

### 3.3 HTTP 条件请求缓存（ETag）

[entitysource.go Serve()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/entitysource/entitysource.go#L270-L506)

本地文件通过 ETag 支持 HTTP 条件请求：

```go
etag := "\"" + hashid.EncodeEntityID(f.hasher, f.e.ID()) + "\""
```

- `If-None-Match` → 返回 `304 Not Modified`
- `If-Match` → 前置校验
- `If-Range` + `Range` → 条件范围请求

### 3.4 缩略图禁用标记缓存

[thumbnail.go disableThumb()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/thumbnail.go#L288-L296)

```go
func disableThumb(ctx context.Context, m *manager, uri *fs.URI) error {
    return m.fs.PatchMetadata(ctx, []*fs.URI{uri}, fs.MetadataPatch{
        Key:   dbfs.ThumbDisabledKey,
        Value: "",
    })
}
```

当缩略图生成失败或不被支持时，在文件元数据中写入 `ThumbDisabledKey`。下次请求在 Step 0 即被拦截，避免反复尝试。

**清除时机**：文件上传完成或覆盖时，自动移除该标记（见 [upload.go](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/fs/dbfs/upload.go#L316)）。

---

## 四、驱动能力差异

### 4.1 Capabilities 结构体

[driver/handler.go Capabilities](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/driver/handler.go#L89-L111)

```go
type Capabilities struct {
    StaticFeatures         *boolset.BooleanSet  // 静态能力位
    MaxSourceExpire        time.Duration        // 源 URL 最大有效期
    MinSourceExpire        time.Duration        // 源 URL 最小有效期
    MediaMetaSupportedExts []string             // 支持原生媒体元数据的扩展名
    MediaMetaProxy         bool                 // 是否使用本地代理生成媒体元数据
    ThumbSupportedExts     []string             // 支持原生缩略图的扩展名
    ThumbSupportAllExts    bool                 // 是否对所有扩展名生成缩略图
    ThumbMaxSize           int64                // 原生缩略图最大文件大小（0=无限制）
    ThumbProxy             bool                 // 是否使用本地代理生成缩略图
    BrowserRelayedDownload bool                 // 是否通过 stream-saver 中继下载
}
```

### 4.2 各驱动能力对比

| 驱动 | 原生 Thumb | ThumbSupportedExts | ThumbProxy | 原生 MediaMeta | MediaMetaProxy |
|------|-----------|-------------------|------------|---------------|----------------|
| **Local** | ✗（必须走代理） | — | ✓ | ✓ | ✓ |
| **S3** | ✗（not implemented） | — | 由策略决定 | ✗ | 由策略决定 |
| **OSS** | ✓ 阿里云图片处理 | 由策略配置 | 由策略决定 | ✓ 图片/视频/音频 | 由策略决定 |
| **COS** | ✓ 腾讯云数据万象 | 由策略配置 | 由策略决定 | ✓ | 由策略决定 |
| **OBS** | ✓ 华为云图片处理 | 由策略配置 | 由策略决定 | ✓ 图片EXIF | 由策略决定 |
| **Qiniu** | ✓ 七牛数据处理 | 由策略配置 | 由策略决定 | — | 由策略决定 |
| **Upyun** | ✓ 又拍云图片处理 | 由策略配置 | — | — | — |
| **KS3** | ✓ 金山云图片处理 | — | 由策略决定 | ✗ | 由策略决定 |
| **OneDrive** | ✗ | 由策略配置 | 由策略决定 | ✗ | 由策略决定 |
| **Remote** | 代理至远端 | 由策略配置 | 由策略决定 | 代理至远端 | 由策略决定 |

### 4.3 原生 Thumb 实现方式差异

各云厂商的图片处理参数格式不同：

| 驱动 | 处理参数格式 | 编码参数附加方式 |
|------|------------|----------------|
| **OSS** | `image/resize,m_lfit,h_{h},w_{w}/format,{fmt}/quality,q_{q}` | 附加到 `GetObjectRequest.Process` |
| **COS** | `imageMogr2/thumbnail/{w}x{h}/format/{fmt}/rquality/{q}` | 附加为 URL 查询参数 |
| **OBS** | `image/resize,m_lfit,w_{w},h_{h}/format,{fmt}/quality,q_{q}` | 通过 `QueryParams` 传入签名 |
| **Qiniu** | `imageView2/1/w/{w}/h/{h}/format/{fmt}/q/{q}` | 附加为 URL 查询参数（空值 key） |
| **Upyun** | `!/fwfh/{w}x{h}/format/{fmt}/quality/{q}` | 拼接到源路径后，再签名 |
| **KS3** | `@base@tag=imgScale&m=0&w={w}&h={h}&q={q}&F={fmt}` | 拼接到对象 Key 后，预签名 |
| **S3** | not implemented | — |
| **OneDrive** | not implemented | — |

所有驱动统一读取 `settings.ThumbSize()` 和 `settings.ThumbEncode()` 来获取目标尺寸和编码配置。

---

## 五、降级处理

### 5.1 缩略图请求降级链

[manager.Thumbnail()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/thumbnail.go#L27-L101)

```
Step 0: 检查 ThumbDisabledKey → 已禁用则直接返回失败
    ↓
Step 1: 已有缩略图实体 → 直接使用（最快路径）
    ↓
Step 2: 驱动原生支持 → 实时生成签名 URL（零存储开销）
    ↓ 条件: ThumbSupportAllExts || ext ∈ ThumbSupportedExts
    ↓       && (ThumbMaxSize == 0 || size <= ThumbMaxSize)
    ↓       && !encrypted
Step 3: ThumbProxy 支持 → 本地生成管线异步生成
    ↓ 条件: ThumbProxy == true && FS 支持 GenerateThumb
Step 4: 均不支持 → 调用 disableThumb() 永久禁用
```

### 5.2 缩略图生成管线降级链

[pipeline.Generate()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/thumb/pipeline.go#L86-L124)

生成管线内置 6 个 Generator，按 Priority 升序排列（数字越小优先级越高）：

| Generator | Priority | 支持格式 | 机制 | Continue |
|-----------|----------|---------|------|----------|
| **LibreOfficeGenerator** | 50 | 文档格式（doc/pdf/ppt 等） | 调用 `soffice --headless --convert-to png` | ✓（输出 PNG 后交给下一个 Generator 缩放） |
| **MusicCoverGenerator** | 50 | 音频格式（mp3/flac 等） | 使用 `tag.ReadFrom()` 提取封面图 | ✓（提取原始封面后交给下一个 Generator 缩放） |
| **LibRawGenerator** | 50 | RAW 图片格式 | 调用 `dcraw -e` 提取内嵌缩略图 | ✓（输出后交给下一个 Generator 缩放） |
| **VipsGenerator** | 100 | 由 `VipsThumbExts` 配置 | 调用 `vips thumbnail_source` | ✗ |
| **FfmpegGenerator** | 200 | 视频格式 | 调用 `ffmpeg -ss ... -i ... -vframes 1` | ✗ |
| **BuiltinGenerator** | 300 | jpg/jpeg/png/gif | Go 标准库 image 解码 + draw 缩放 | ✗ |

**降级机制**：

1. 每个 Generator 在不支持时返回 `ErrPassThrough`，管线自动跳到下一个
2. 当 Generator 返回 `Result.Continue = true` 时，表示其产出是中间结果（如提取了原始封面图），需要后续 Generator 进一步缩放处理
3. 所有 Generator 都跳过时返回 `ErrNotAvailable`

```
Generator 生成中间结果 (Continue=true)
    ↓ CloneToLocalSrc() 将中间文件转为本地 EntitySource
    ↓ 更新 ext 为中间文件扩展名
    ↓ 交给下一个 Generator 继续处理
    ↓
最终 Generator 输出最终缩略图 (Continue=false)
```

### 5.3 生成失败降级

[manager.generateThumb()](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/filemanager/manager/thumbnail.go#L129-L222)

```go
if err != nil {
    if res != nil && res.Path != "" {
        _ = os.Remove(res.Path)  // 清理临时文件
    }
    if !errors.Is(err, context.Canceled) && !m.stateless {
        if err := disableThumb(ctx, m, uri); err != nil {  // 永久禁用
            m.l.Warning("Failed to disable thumb: %v", err)
        }
    }
}
```

- 生成失败后清理临时文件
- 非取消原因导致的失败 + 非从节点 → 标记 `ThumbDisabledKey`
- 从节点（stateless）不标记禁用，因为文件可能下次上传到主节点

### 5.4 FFmpeg 生成器内的降级

[ffmpeg.go](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/thumb/ffmpeg.go#L32-L101)

```go
if es.IsLocal() && !es.Entity().Encrypted() {
    input = es.LocalPath(ctx)              // 本地文件：直接读路径
} else {
    src, err := es.Url(ctx, opts...)       // 远程文件：获取签名 URL
    input = src.Url                         // ffmpeg 通过 HTTP 拉取
}
```

对于非本地文件，FFmpeg 通过 HTTP URL 方式读取输入；加密文件则禁用内部代理绕过，保证 URL 可直接访问。

### 5.5 Vips 在 Windows 上的降级

[vips.go](file:///d:/fz/0601-1/solo-dogfeeding/code/47-Cloudreve/pkg/thumb/vips.go#L46-L78)

```go
if runtime.GOOS == "windows" {
    // Pipe IO is not working on Windows for VIPS
    if es.IsLocal() && !es.Entity().Encrypted() {
        input = fmt.Sprintf("[filename=\"%s\"]", es.LocalPath(ctx))
        usePipe = false
    } else {
        // 先下载到临时文件再处理
        tempInputFile, err := util.CreatNestedFile(tempPath)
        io.Copy(tempInputFile, es)
        input = fmt.Sprintf("[filename=\"%s\"]", tempPath)
    }
}
```

Windows 下 VIPS 不支持管道输入，降级为：
- 本地未加密文件 → 直接传文件路径
- 远程/加密文件 → 先下载到临时文件再传入路径

---

## 六、关键数据流总结

```
前端请求 → GET /api/v3/file/thumb?uri=xxx
    ↓
FileThumbService.Get()
    ↓ manager.Thumbnail(uri)
    ↓
    ├─ 已有缩略图实体? → GetEntitySource(thumbEntity) → Url()
    ├─ 驱动原生支持? → GetEntitySource(primaryEntity, WithUseThumb(true)) → Url()
    │                   → handler.Thumb() 生成带图片处理参数的签名直链
    ├─ ThumbProxy? → SubmitAndAwaitThumbnailTask() → pipeline.Generate()
    │                   → 生成临时文件 → 上传为缩略图实体 → Url()
    └─ 都不支持? → disableThumb() → 返回 ErrEntityNotExist
    ↓
EntitySource.Url()
    ├─ ShouldInternalProxy? → 生成 Cloudreve 内部代理 URL + 签名
    └─ 否则 → handler.Thumb()/Source() + ApplyProxyIfNeeded
    ↓
FileThumbResponse{Url, Expires}
```
