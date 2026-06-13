# 远程下载任务代码分析

## 1. 整体架构与代码路径

### 1.1 核心文件清单

| 文件 | 作用 |
|------|------|
| [pkg/downloader/downloader.go](pkg/downloader/downloader.go) | 下载器抽象接口与通用状态定义 |
| [pkg/downloader/aria2/aria2.go](pkg/downloader/aria2/aria2.go) | Aria2 下载器具体实现 |
| [pkg/downloader/qbittorrent/qbittorrent.go](pkg/downloader/qbittorrent/qbittorrent.go) | qBittorrent 下载器具体实现 |
| [pkg/downloader/slave/slave.go](pkg/downloader/slave/slave.go) | 从机节点下载器代理（主机 → 从机 HTTP 调用） |
| [pkg/filemanager/workflows/remote_download.go](pkg/filemanager/workflows/remote_download.go) | 远程下载任务工作流核心逻辑（状态机） |
| [pkg/filemanager/workflows/upload.go](pkg/filemanager/workflows/upload.go) | 从机上传任务工作流（下载完成后文件传输） |
| [pkg/filemanager/workflows/worfklows.go](pkg/filemanager/workflows/worfklows.go) | 工作流公共工具（节点分配、临时目录） |
| [pkg/queue/task.go](pkg/queue/task.go) | 通用任务接口、DBTask 实现、状态转换机 |
| [pkg/queue/queue.go](pkg/queue/queue.go) | 任务队列调度器（FIFO、重试、Worker） |
| [service/explorer/workflows.go](service/explorer/workflows.go) | HTTP API 服务层（创建下载任务、列表、取消） |
| [routers/controllers/file.go](routers/controllers/file.go) | HTTP Controller 层（CreateRemoteDownload 等） |
| [routers/controllers/slave.go](routers/controllers/slave.go) | 从机节点 API（SlaveDownloadTaskCreate/Status/Cancel 等） |
| [ent/task/task.go](ent/task/task.go) | 数据库 Task 表 Schema 与 Status 枚举 |

### 1.2 请求入口路由

在 [router.go:576-596](routers/router.go#L576-L596) 注册：

```
POST   /api/v4/workflow/download          创建远程下载任务
PATCH  /api/v4/workflow/download/:id      设置下载的目标文件（BT 多文件选择）
DELETE /api/v4/workflow/download/:id      取消下载任务
```

## 2. 任务状态体系

### 2.1 三层状态模型

远程下载任务的状态分为三个层次，每层职责不同：

#### 第一层：数据库任务状态（ent/task.Status）

定义于 [task.go:97-104](ent/task/task.go#L97-L104)：

```go
const (
    StatusQueued     Status = "queued"      // 已入队，等待执行
    StatusProcessing Status = "processing"  // 正在执行
    StatusSuspending Status = "suspending"  // 挂起（等待轮询/重试）
    StatusError      Status = "error"       // 失败
    StatusCanceled   Status = "canceled"    // 已取消
    StatusCompleted  Status = "completed"   // 已完成
)
```

状态转换逻辑定义在 [task.go:375-473](pkg/queue/task.go#L375-L473) 的 `stateTransitions` 映射表中：

```
"" → StatusQueued                  （新任务持久化）
StatusQueued → StatusProcessing    （开始执行）
StatusQueued → StatusError         （入队即失败）
StatusProcessing → StatusQueued    （重新入队）
StatusProcessing → StatusCompleted （成功完成 → 触发 Cleanup）
StatusProcessing → StatusError     （执行失败 → 触发 Cleanup）
StatusProcessing → StatusCanceled  （被取消 → 触发 Cleanup）
StatusProcessing → StatusSuspending（挂起，ResumeTime 后自动恢复）
StatusSuspending → StatusProcessing（恢复执行）
StatusSuspending → StatusError      （挂起期间失败）
```

关键：**StatusSuspending 是远程下载的核心状态**，用于轮询下载进度。每次 `Do()` 返回 `StatusSuspending`，队列会根据 `ResumeTime` 将任务重新入队等待下一次调度。

#### 第二层：下载器内部状态（downloader.Status）

定义于 [downloader.go:64-70](pkg/downloader/downloader.go#L64-L70)：

```go
const (
    StatusDownloading Status = "downloading"  // 下载中
    StatusSeeding     Status = "seeding"      // 做种中（BT）
    StatusCompleted   Status = "completed"    // 下载完成
    StatusError       Status = "error"        // 下载出错
    StatusUnknown     Status = "unknown"      // 未知状态
)
```

Aria2 的状态映射见 [aria2.go:105-122](pkg/downloader/aria2/aria2.go#L105-L122)：
- `active` + 完成度 100% + BT 模式 → `StatusSeeding`
- `active` → `StatusDownloading`
- `waiting/paused` → `StatusDownloading`
- `complete` → `StatusCompleted`
- `error` → `StatusError`
- `cancelled/removed` → 返回 `ErrTaskNotFount`

#### 第三层：工作流阶段（RemoteDownloadTaskPhase）

定义于 [remote_download.go:60-65](pkg/filemanager/workflows/remote_download.go#L60-L65)：

```go
const (
    RemoteDownloadTaskPhaseNotStarted   RemoteDownloadTaskPhase = ""          // 初始，尚未创建下载器任务
    RemoteDownloadTaskPhaseMonitor                              = "monitor"   // 监控下载进度
    RemoteDownloadTaskPhaseTransfer                              = "transfer"  // 下载完成，正在传输到存储策略
    RemoteDownloadTaskPhaseAwaitSeeding                          = "seeding"   // 传输完成，等待做种结束
)
```

状态机入口是 [RemoteDownloadTask.Do()](pkg/filemanager/workflows/remote_download.go#L119-L170)，根据当前 Phase 分发到不同处理函数。

## 3. 任务执行完整流程

```
用户请求
  ↓
controllers.CreateRemoteDownload
  ↓
explorer.DownloadWorkflowService.CreateDownloadTask()
  ├─ 权限校验（GroupPermissionRemoteDownload）
  ├─ 校验目标路径 Dst（必须是可创建文件的目录）
  ├─ 校验来源（Src URL 列表 或 SrcFile 种子文件 URI）
  ├─ 批量创建 workflows.NewRemoteDownloadTask()
  └─ 入队 dep.RemoteDownloadQueue().QueueTask()
       ↓
queue.QueueTask() → StatusQueued → persistTask() → 写入 DB
       ↓
队列 Worker 调度 → StatusProcessing → 调用 task.Do()
       ↓
workflows.RemoteDownloadTask.Do()
  ├─ 反序列化 PrivateState → RemoteDownloadTaskState
  ├─ allocateNode() 分配/复用节点（NodeCapabilityRemoteDownload）
  ├─ node.CreateDownloader() 创建下载器实例
  └─ switch Phase:
     ├─ NotStarted   → createDownloadTask()
     ├─ Monitor/Seeding → monitor()
     └─ Transfer     → masterTransfer() / slaveTransfer()
```

## 4. 回调处理（轮询机制）

远程下载不使用 Webhook 回调，而是采用**轮询（Polling）**模式。

### 4.1 轮询调度流程

核心函数 [monitor()](pkg/filemanager/workflows/remote_download.go#L246-L324)：

```go
func (m *RemoteDownloadTask) monitor(ctx, dep) (task.Status, error) {
    resumeAfter := node.Settings.Interval * time.Second  // 轮询间隔

    // 1. 获取下载器状态
    status, err := m.d.Info(ctx, m.state.Handle)
    if err != nil {
        if errors.Is(err, downloader.ErrTaskNotFount) && m.state.Status != nil {
            return task.StatusCanceled, nil  // 任务被外部删除视为取消
        }
        m.state.GetTaskStatusTried++
        if m.state.GetTaskStatusTried >= GetTaskStatusMaxTries(5) {
            return task.StatusError, ...     // 连续失败 5 次报错
        }
        m.ResumeAfter(resumeAfter)          // 否则延迟重试
        return task.StatusSuspending, nil
    }

    // 2. 如果 Handle 变更（BT 磁力链接解析完成后会 Follow 到新任务）
    if status.FollowedBy != nil {
        m.state.Handle = status.FollowedBy
        m.ResumeAfter(0)                     // 立即再次执行
        return task.StatusSuspending, nil
    }

    // 3. 首次获取 / 大小变更时，校验用户容量
    if m.state.Status == nil || m.state.Status.Total != status.Total {
        if err := m.validateFiles(ctx, dep, status); err != nil {
            return task.StatusError, ...      //  CriticalErr，不可重试
        }
    }

    // 4. 根据下载器状态决策
    switch status.State {
    case StatusSeeding:
        if Phase == Monitor {
            Phase = Transfer                   // 进入传输阶段
            return StatusSuspending, nil       // 立即执行传输
        }
        if !node.Settings.WaitForSeeding {
            return StatusCompleted, nil        // 不等待做种，直接完成
        }
        // 继续等待做种
        ResumeAfter(resumeAfter)
        return StatusSuspending, nil

    case StatusCompleted:
        if Phase == Monitor {
            Phase = Transfer                   // 非 BT，直接进入传输
            return StatusSuspending, nil
        }
        return StatusCompleted, nil            // 全部完成

    case StatusDownloading:
        ResumeAfter(resumeAfter)               // 继续轮询
        return StatusSuspending, nil

    case StatusUnknown / StatusError:
        return StatusError, ...                // CriticalErr，不可重试
    }
}
```

### 4.2 主机 → 从机通信

当下载任务分配在从机节点时，主机通过 `slaveDownloader` 代理 HTTP 调用从机的 Slave API：

| 操作 | 主机代理方法 | 从机 API 路径 | 从机 Controller |
|------|-------------|---------------|----------------|
| 创建任务 | [slave.go:CreateTask()](pkg/downloader/slave/slave.go#L38-L71) | `POST /api/v4/slave/download/task` | [SlaveDownloadTaskCreate()](routers/controllers/slave.go#L133-L144) |
| 查询状态 | [slave.go:Info()](pkg/downloader/slave/slave.go#L73-L109) | `POST /api/v4/slave/download/status` | [SlaveDownloadTaskStatus()](routers/controllers/slave.go#L147-L164) |
| 取消任务 | [slave.go:Cancel()](pkg/downloader/slave/slave.go#L111-L138) | `POST /api/v4/slave/download/cancel` | [SlaveCancelDownloadTask()](routers/controllers/slave.go#L167-L178) |
| 选择文件 | [slave.go:SetFilesToDownload()](pkg/downloader/slave/slave.go#L140-L168) | `POST /api/v4/slave/download/select` | [SlaveSelectFilesToDownload()](routers/controllers/slave.go#L181-L192) |

从机收到请求后，使用本地真实下载器（Aria2/qBittorrent）执行操作，并通过 Gob 编码返回结果。

## 5. 文件落库流程

下载完成后进入 `RemoteDownloadTaskPhaseTransfer` 阶段，根据节点角色分两种路径：

### 5.1 主机节点传输（masterTransfer）

代码见 [remote_download.go:428-557](pkg/filemanager/workflows/remote_download.go#L428-L557)

```
masterTransfer()
  ├─ 构建并发 worker 池（MaxParallelTransfer，默认由系统配置）
  ├─ 统计待传输文件总数与总大小
  ├─ 遍历所有 Selected=true 的 TaskFile:
  │   ├─ 已在 Transferred map 中的跳过（断点续传）
  │   └─ goroutine transferFunc():
  │       ├─ os.Open(src) 打开本地临时文件
  │       ├─ 构造 fs.UploadRequest（带 ProgressFunc）
  │       └─ fm.Update(ctx, fileData, fs.WithNoEntityType())
  │           └─ manager.Update()
  │               ├─ m.fs.PrepareUpload()  → DB 创建占位文件/版本记录
  │               ├─ m.Upload()            → 上传到存储策略驱动
  │               └─ m.CompleteUpload()    → 完成落库、触发媒体元数据/全文索引
  └─ 全部成功 → Phase = AwaitSeeding，返回 StatusSuspending
```

核心落库路径 [manager.Update()](pkg/filemanager/manager/upload.go#L326-L367)：
1. `PrepareUpload()`：在 DBFS 中准备上传会话（创建文件 Entity、锁定目标路径）
2. `Upload()`：将文件流写入实际存储策略（本地/OSS/S3 等）
3. `CompleteUpload()`：更新文件元数据、清除上传会话、触发媒体元数据提取和全文索引任务

### 5.2 从机节点传输（slaveTransfer）

代码见 [remote_download.go:326-426](pkg/filemanager/workflows/remote_download.go#L326-L426)

从机不直接访问主机数据库，而是通过创建 **SlaveUploadTask** 由从机自身执行：

```
slaveTransfer()
  ├─ 若 SlaveUploadTaskID == 0（首次进入）:
  │   ├─ 构建 SlaveUploadTaskState（包含所有待上传文件）
  │   ├─ m.node.CreateTask(SlaveUploadTaskType, payload)  → HTTP 调用从机创建任务
  │   └─ 记录 SlaveUploadTaskID，返回 StatusSuspending
  │
  └─ 否则（轮询 SlaveUploadTask 状态）:
      ├─ m.node.GetTask(SlaveUploadTaskID)  → HTTP 查询从机任务
      ├─ 同步 progress 和 Transferred 状态
      └─ 判断结果:
          ├─ Completed 且全部传输完成 → Phase = AwaitSeeding
          ├─ Completed 但有未传输 → 记录已传输文件索引，重置 SlaveUploadTaskID 重新创建任务
          ├─ Error → 同上，记录已传输，下次重试剩余文件
          ├─ Canceled → 返回 StatusError（CriticalErr）
          └─ 其他 → 30 秒后再查
```

从机端执行 SlaveUploadTask 的逻辑见 [upload.go:69-223](pkg/filemanager/workflows/upload.go#L69-L223) 的 `SlaveUploadTask.Do()`，它使用 **stateless 模式**调用主机 API 完成落库：

```
SlaveUploadTask.Do()（在从机上执行）
  └─ 遍历待传文件:
      └─ fm.Update(ctx, req, fs.WithNode(node), fs.WithStatelessUserID(userID), fs.WithNoEntityType())
          └─ updateStateless()
              ├─ node.PrepareUpload()    → 调主机 /slave/upload/prepare 创建占位
              ├─ m.Upload()              → 从机直传到存储策略
              └─ node.CompleteUpload()   → 调主机 /slave/upload/complete 完成落库
```

## 6. 重复任务与断点续传处理

系统通过多层机制确保任务可恢复、不重复执行：

### 6.1 数据库持久化与启动恢复

- 所有任务状态（PublicState + PrivateState）均持久化在 DB 的 `tasks` 表中
- PrivateState 是 JSON 字符串，包含 `RemoteDownloadTaskState` 完整信息（Handle、Phase、Transferred、NodeID 等）
- 队列启动时 [queue.go:93-131](pkg/queue/queue.go#L93-L131) 的 `Start()` 会：
  1. 调用 `taskClient.GetPendingTasks()` 拉取未完成任务
  2. 通过 `NewTaskFromModel()` 用注册的工厂恢复为具体 Task 实例
  3. 重新入队 `QueueTask()` 继续执行

### 6.2 节点分配的固定性

[allocateNode()](pkg/filemanager/workflows/worfklows.go#L29-L42) 会优先使用 `state.NodeID` 中已保存的节点：
```go
func allocateNode(ctx, dep, state *NodeState, capability) (cluster.Node, error) {
    node, err := np.Get(ctx, capability, state.NodeID)  // 传入 NodeID 优先复用
    state.NodeID = node.ID()
    return node, nil
}
```
这样任务恢复后会连接同一台节点的下载器，继续监控同一个下载任务。

### 6.3 Transferred 索引去重

- `RemoteDownloadTaskState.Transferred` 是 `map[int]interface{}`，key 为下载器文件 Index
- 传输前检查：`if _, ok := m.state.Transferred[f.Index]; ok { skip }`
- masterTransfer：传输成功后写入 `m.state.Transferred[file.Index] = nil`
- slaveTransfer：SlaveUploadTask 完成后同步其 Transferred 到主任务
- 每次状态保存时 Transferred 被序列化入库，重启后不丢失

### 6.4 已创建下载任务的去重

[createDownloadTask()](pkg/filemanager/workflows/remote_download.go#L172-L226) 入口检查：
```go
if m.state.Handle != nil {
    m.state.Phase = RemoteDownloadTaskPhaseMonitor
    return task.StatusSuspending, nil
}
```
如果 `Handle` 已存在（即下载器任务已创建），直接跳过创建进入监控阶段，避免重复创建。

### 6.5 失败重试机制

- **队列级重试**：[queue.go:296-313](pkg/queue/queue.go#L296-L313)，非 CriticalErr 且未超 maxRetry 时，使用指数退避延迟重试
- **getTaskStatus 级重试**：`monitor()` 中连续获取状态失败最多 5 次
- **CriticalErr**：被标记为 `CriticalErr` 的错误（如 URL 非法、容量不足、参数错误）不进行重试，直接返回 `StatusError`

## 7. 任务清理（Cleanup）

任务到达 `StatusCompleted/Error/Canceled` 时，状态转换机自动调用 `task.Cleanup()`：

[remote_download.go:595-609](pkg/filemanager/workflows/remote_download.go#L595-L609)：
```go
func (m *RemoteDownloadTask) Cleanup(ctx) error {
    // 1. 取消下载器中的任务
    if m.state.Handle != nil {
        m.d.Cancel(ctx, m.state.Handle)
    }
    // 2. 主机节点删除本地临时下载目录
    if m.state.Status != nil && m.node.IsMaster() && m.state.Status.SavePath != "" {
        os.RemoveAll(m.state.Status.SavePath)
    }
    return nil
}
```

Aria2 的 `Cancel()` 还会额外延迟 120 秒删除临时目录（避免 aria2 还在占用文件句柄）。

## 8. 关键数据结构

### 8.1 RemoteDownloadTaskState

```go
type RemoteDownloadTaskState struct {
    SrcFileUri         string                 // 种子文件 URI（二选一）
    SrcUri             string                 // 下载 URL（二选一）
    Dst                string                 // 目标存储路径 URI
    Handle             *downloader.TaskHandle // 下载器任务句柄（ID+Hash）
    Status             *downloader.TaskStatus // 最新下载状态
    NodeState          // 内嵌：NodeID, progress
    Phase              RemoteDownloadTaskPhase // 当前工作流阶段
    SlaveUploadTaskID  int                     // 从机上传任务 ID（从机模式）
    SlaveUploadState   *SlaveUploadTaskState   // 从机上传任务状态快照
    GetTaskStatusTried int                     // 连续获取状态失败次数
    Transferred        map[int]interface{}     // 已传输的文件 Index 集合
    Failed             int                     // 传输失败文件数
}
```

### 8.2 downloader.TaskStatus

```go
type TaskStatus struct {
    FollowedBy    *TaskHandle // BT 磁力链接解析后指向新任务
    SavePath      string      // 本地临时保存路径
    Name          string      // 任务名称
    State         Status      // downloading/seeding/completed/error/unknown
    Total         int64       // 总大小
    Downloaded    int64       // 已下载
    DownloadSpeed int64       // 下载速度 B/s
    Uploaded      int64       // 已上传（做种）
    UploadSpeed   int64       // 上传速度
    Hash          string      // BT InfoHash
    Files         []TaskFile  // 文件列表
    Pieces        []byte      // Piece 完成位图
    NumPieces     int         // 总 Piece 数
    ErrorMessage  string      // 错误信息
}
```

### 8.3 ent.Task 数据库字段

```go
type Task struct {
    ID            int
    CreatedAt     time.Time
    UpdatedAt     time.Time
    DeletedAt     *time.Time
    Type          string           // "remote_download"
    Status        Status           // queued/processing/suspending/...
    PublicState   TaskPublicState  // {RetryCount, ExecutedDuration, Error, ErrorHistory, ResumeTime, ...}
    PrivateState  string           // JSON 序列化的 RemoteDownloadTaskState
    CorrelationID uuid.UUID        // 日志关联 ID
    UserTasks     int              // 所属用户 ID
}
```

## 9. 进度查询机制

前端获取远程下载任务状态与进度有两条 API 路径：

### 9.1 API 入口

| API | 路径 | 用途 |
|-----|------|------|
| 任务列表 | `GET /api/v4/workflow?category=downloading|downloaded` | 查询任务列表与 Summary |
| 任务进度 | `GET /api/v4/workflow/progress/:id` | 查询单个任务的实时 Progress |

路由注册见 [router.go:554-561](routers/router.go#L554-L561)。

### 9.2 任务列表与 Summary

`ListTasks()` 定义于 [workflows.go:324-384](service/explorer/workflows.go#L324-L384)，按 `category` 分类查询：

- **downloading**：查询状态为 `StatusSuspending / StatusProcessing / StatusQueued` 的远程下载任务，`PageSize` 强制设为 `intsets.MaxInt`（即一次性返回所有进行中任务）
- **downloaded**：查询状态为 `StatusCanceled / StatusError / StatusCompleted` 的远程下载任务

每个任务通过 `task.Summarize(hasher)` 生成摘要，返回到前端的 `TaskResponse` 结构体（[response.go:88-101](service/explorer/response.go#L88-L101)）：

```go
type TaskResponse struct {
    CreatedAt    time.Time      `json:"created_at,"`
    UpdatedAt    time.Time      `json:"updated_at"`
    ID           string         `json:"id"`          // HashID 编码
    Status       string         `json:"status"`      // DB 任务状态
    Type         string         `json:"type"`        // "remote_download"
    Node         *user.Node     `json:"node,omitempty"`
    Summary      *queue.Summary `json:"summary,omitempty"` // 核心：包含 Phase 和下载器状态
    Error        string         `json:"error,omitempty"`
    ErrorHistory []string       `json:"error_history,omitempty"`
    Duration     int64          `json:"duration,omitempty"`
    ResumeTime   int64          `json:"resume_time,omitempty"`
    RetryCount   int            `json:"retry_count,omitempty"`
}
```

### 9.3 Summarize 返回内容

[RemoteDownloadTask.Summarize()](pkg/filemanager/workflows/remote_download.go#L629-L661) 返回的 Summary 结构：

```go
&queue.Summary{
    Phase:  string(m.state.Phase),    // "" | "monitor" | "transfer" | "seeding"
    NodeID: m.state.NodeID,
    Props: map[string]any{
        "src_str":    m.state.SrcUri,           // 下载 URL
        "src":        m.state.SrcFileUri,        // 种子文件 URI
        "dst":        m.state.Dst,               // 目标路径
        "failed":     failed,                    // 失败文件数
        "download":   status,                    // downloader.TaskStatus 快照（SavePath 已脱敏为空）
    },
}
```

**状态显示的关键问题**：前端拿到的 DB 状态 `Status` 字段只能是 `queued / processing / suspending / error / canceled / completed` 之一，而真正的下载进度信息在 `Summary.Props["download"]` 中。对用户来说：

| 用户感知 | DB Status | Phase | Summary.download.state |
|---------|-----------|-------|----------------------|
| 等待中 | queued | "" | 无 |
| 下载中 | suspending | monitor | downloading |
| 做种中 | suspending | monitor / seeding | seeding |
| 传输中 | processing | transfer | completed/seeding |
| 等待做种结束 | suspending | seeding | seeding |
| 已完成 | completed | seeding | completed |
| 失败 | error | 任意 | error/unknown |
| 已取消 | canceled | 任意 | 无 |

**问题：DB 状态与用户感知的映射不直观**。`suspending` 在不同 Phase 下含义完全不同，需要前端结合 `Phase` + `download.state` 才能正确展示。若前端仅依赖 `status` 字段，会出现"下载中"和"传输中"都显示为 `suspending/processing` 的困惑。

### 9.4 实时进度查询

[TaskPhaseProgress()](service/explorer/workflows.go#L386-L396) 从内存中的 `TaskRegistry` 获取运行中任务的 `Progress()`：

```go
func TaskPhaseProgress(c *gin.Context, taskID int) (queue.Progresses, error) {
    r := dep.TaskRegistry()
    t, found := r.Get(taskID)       // 仅内存中的活跃任务
    if !found || (权限校验) {
        return queue.Progresses{}, nil
    }
    return t.Progress(c), nil
}
```

[RemoteDownloadTask.Progress()](pkg/filemanager/workflows/remote_download.go#L663-L679) 合并两个来源的进度：

```go
func (m *RemoteDownloadTask) Progress(ctx) queue.Progresses {
    merged := make(queue.Progresses)
    // 1. masterTransfer 的上传进度（仅主机节点有值）
    for k, v := range m.progress {
        merged[k] = v
    }
    // 2. slaveTransfer 的从机上传进度（仅从机节点有值）
    if m.state.NodeState.progress != nil {
        for k, v := range m.state.NodeState.progress {
            merged[k] = v
        }
    }
    return merged
}
```

进度 key 的含义（定义于 [archive.go:69-72](pkg/filemanager/workflows/archive.go#L69-L72) 和 [remote_download.go:71-72](pkg/filemanager/workflows/remote_download.go#L71-L72)）：

| key | 含义 | Total | Current |
|-----|------|-------|---------|
| `upload` | 传输总字节数 | 选中文件总大小 | 已传输字节数 |
| `upload_count` | 传输文件计数 | 选中文件数 | 已传输文件数 |
| `upload_single_{n}` | 单文件传输 | 单文件大小 | 已传输字节，Identifier 为目标 URI |
| `relocate` | 重定位计数 | 重定位文件数 | 已完成数 |

**重要限制**：`Progress()` 仅在任务存在于内存 `TaskRegistry` 时可用。任务一旦到达终态（completed/error/canceled），会从 Registry 中 `Delete()`，此时 `Progress()` 返回空 map。

### 9.5 下载阶段的进度

在 `Phase=monitor`（下载中）阶段，`Progress()` 返回空 map，因为下载进度的信息不在 `m.progress` 中，而是存在于 `Summary.Props["download"]` 中：

```json
{
  "download": {
    "state": "downloading",
    "total": 1073741824,
    "downloaded": 536870912,
    "download_speed": 1048576,
    "name": "ubuntu-22.04.iso",
    "files": [...]
  }
}
```

前端需要从 `Summary.download.downloaded / total` 计算下载百分比，而非从 `Progress()` API 获取。

## 10. 重复提交规则分析

### 10.1 创建入口无 URL 级去重

[CreateDownloadTask()](service/explorer/workflows.go#L81-L169) 的逻辑：

```go
// 批量创建——遍历 Src 列表，每个 URL 创建一个独立任务
for _, src := range service.Src {
    t, _ := workflows.NewRemoteDownloadTask(c, src, service.SrcFile, service.Dst)
    dep.RemoteDownloadQueue(c).QueueTask(c, t)
    tasks = append(tasks, t)
}
```

**没有任何去重校验**。每次调用都会创建一个全新的 `RemoteDownloadTask`，即使 URL 完全相同。系统不检查：
- 同一用户是否已有相同 URL 的进行中任务
- 同一 URL 是否已被其他用户下载
- 相同 Dst 下是否已有同名文件

### 10.2 批量数量限制

唯一的限流手段是 `Aria2BatchSize`（[workflows.go:114-117](service/explorer/workflows.go#L114-L117)）：

```go
limit := user.Edges.Group.Settings.Aria2BatchSize
if limit > 0 && len(service.Src) > limit {
    return nil, serializer.NewError(serializer.CodeBatchAria2Size, "", nil)
}
```

这只是限制单次请求的批量大小，不是去重。

### 10.3 队列级也无去重

[QueueTask()](pkg/queue/queue.go#L184-L209) 只做以下操作：
1. 状态转 `StatusQueued`
2. 持久化到 DB
3. 推入 FIFO 调度器
4. 注册到 TaskRegistry（`registry.Set(t.ID(), t)`）

FIFO 调度器（[scheduler.go](pkg/queue/scheduler.go)）按 `ResumeTime` 最小堆排序，无去重逻辑。TaskRegistry 是 `map[int]Task`，按 DB 自增 ID 索引，也不会检测重复。

### 10.4 下载器级可能的隐式去重

Aria2 本身可能对相同 URL 的重复添加产生不同 GID（不同任务），因此**不会在下载器层面自动去重**。但如果 URL 指向同一资源，两个 Cloudreve 任务会各自监控独立的 Aria2 GID，最终各自传输文件到 Dst，**导致 Dst 下出现文件覆盖或版本冲突**。

### 10.5 文件级冲突处理

当两个任务同时完成下载并传输到同一 Dst 时，`fm.Update()` → `PrepareUpload()` 会在 DBFS 中处理：
- 如果目标文件已存在，会创建新版本（Entity Type = Version）
- 不会因为文件已存在而拒绝上传

### 10.6 重启恢复时的重复风险

[GetPendingTasks()](inventory/task.go#L165-L187) 查询所有 `StatusIn(processing, queued, suspending)` 的任务并重新入队。如果重启前有重复 URL 的任务，重启后都会恢复执行，不存在去重。

**总结**：当前系统**不提供任何 URL 级去重机制**，同一 URL 可以被重复提交，产生独立的下载任务并最终在目标路径创建文件版本。

## 11. 本机路径问题分析（文档路径 vs 运行时路径）

> **核心区分**：本文档中出现的路径有两类，需要明确区分：
> - **文档引用路径**（如 `pkg/downloader/aria2/aria2.go`）：是相对于仓库根目录的文件路径，仅用于让读者定位源码，与程序运行时完全无关
> - **运行时文件系统路径**（如 `SavePath`、`file.Name`、`src`）：是程序运行时在服务器磁盘上实际读写的路径，由下载器、操作系统和配置共同决定

### 11.1 运行时路径的完整生命周期

运行时路径经历 **生成 → 序列化 → 反序列化 → 打开文件** 四个阶段，下面沿代码逐一追踪。

#### 阶段一：生成临时目录（CreateTask）

Aria2 在 [aria2.go:277-291](pkg/downloader/aria2/aria2.go#L277-L291) 生成临时下载目录：

```go
func (a *aria2Client) tempPath(ctx context.Context) string {
    guid, _ := uuid.NewV4()
    base := util.RelativePath(a.options.TempPath)
    if a.options.TempPath == "" {
        base = util.DataPath(a.settings.TempPath(ctx))
    }
    path := filepath.Join(base, Aria2TempFolder, guid.String())
    return path
}
```

- `util.RelativePath()`（[path.go:59-69](pkg/util/path.go#L59-L69)）的行为：
  - 如果已经是绝对路径，原样返回
  - 否则基于可执行文件所在目录拼接：`filepath.Join(filepath.Dir(os.Executable()), name)`
- `filepath.Join` 使用操作系统原生分隔符拼接：
  - Linux: `/var/cloudreve/data/aria2/550e8400-e29b-41d4-a716-446655440000`
  - Windows: `C:\cloudreve\data\aria2\550e8400-e29b-41d4-a716-446655440000`
- **此路径以 OS 原生格式传入 Aria2 RPC 的 `dir` 选项**，Aria2 直接使用此路径保存文件

qBittorrent 在 [qbittorrent.go:251-263](pkg/downloader/qbittorrent/qbittorrent.go#L251-L263) 同样调用 `filepath.Join` 生成路径，通过 `savepath` 字段传给 qBittorrent API。

#### 阶段二：序列化到 SavePath（Info）

当工作流调用 `Info()` 获取下载状态时，下载器将 RPC 返回的 `Dir` 转为正斜杠格式：

Aria2（[aria2.go:130](pkg/downloader/aria2/aria2.go#L130)）：
```go
savePath := filepath.ToSlash(status.Dir)
```

qBittorrent（[qbittorrent.go:210](pkg/downloader/qbittorrent/qbittorrent.go#L210)）：
```go
SavePath: filepath.ToSlash(torrents[0].SavePath),
```

**为什么用 `filepath.ToSlash`？** 因为 `SavePath` 和 `file.Name` 会被 JSON 序列化存入数据库（`PrivateState` 字段），跨平台传输时正斜杠是通用格式。特别是从机模式下，主机和从机的操作系统可能不同，正斜杠是安全的中间表示。

Aria2 的 `file.Name`（[aria2.go:148-152](pkg/downloader/aria2/aria2.go#L148-L152)）也用正斜杠相对路径：
```go
relPath := strings.TrimPrefix(filepath.ToSlash(item.Path), savePath)
if len(relPath) > 0 {
    relPath = relPath[1:]  // 去掉前导 "/"
}
```

qBittorrent 的 `file.Name`（[qbittorrent.go:216](pkg/downloader/qbittorrent/qbittorrent.go#L216)）直接用 `filepath.ToSlash`：
```go
Name: filepath.ToSlash(item.Name),
```

> **注意 qBittorrent 与 Aria2 的 file.Name 语义差异**：
> - Aria2：`file.Name` 是相对于 `SavePath` 的相对路径（去掉了前缀），如 `subdir/file.txt`
> - qBittorrent：`file.Name` 是 torrent 内部的文件名（可能包含子目录），如 `torrent-name/subdir/file.txt`

#### 阶段三：反序列化并在工作流中使用

从 DB 反序列化后，`SavePath` 是正斜杠字符串，`file.Name` 也是正斜杠字符串。

#### 阶段四：打开文件（masterTransfer / slaveTransfer）

**主机节点**在 [remote_download.go:471](pkg/filemanager/workflows/remote_download.go#L471) 拼接并打开：

```go
src := filepath.FromSlash(path.Join(m.state.Status.SavePath, file.Name))
// ↓ 在 Windows 上
// path.Join("C:/cloudreve/data/aria2/uuid", "subdir/file.txt")
//   → "C:/cloudreve/data/aria2/uuid/subdir/file.txt"  (POSIX 风格中间结果)
// filepath.FromSlash(...)
//   → "C:\cloudreve\data\aria2\uuid\subdir\file.txt"  (OS 原生路径)
// os.Open(src)  → 用 OS 原生路径打开文件
```

流程：`path.Join`（POSIX 拼接）→ `filepath.FromSlash`（转 OS 分隔符）→ `os.Open`。

**从机节点**在 [remote_download.go:357](pkg/filemanager/workflows/remote_download.go#L357) 构建路径：

```go
src := path.Join(m.state.Status.SavePath, f.Name)
// ↓ 结果是纯 POSIX 正斜杠字符串，如 "C:/cloudreve/data/aria2/uuid/subdir/file.txt"
```

这个 POSIX 字符串通过 JSON 传给从机的 `SlaveUploadTask`，从机在 [upload.go:134](pkg/filemanager/workflows/upload.go#L134) 打开时才转换：

```go
handle, err := os.Open(filepath.FromSlash(file.Src))
// ↓ 在 Windows 上转为 "C:\cloudreve\data\aria2\uuid\subdir\file.txt"
```

**结论：两条路径最终效果一致，只是 `filepath.FromSlash` 的调用时机不同——主机在拼接时立即转换，从机在打开时才转换。**

### 11.2 Cleanup 中的路径

[Cleanup()](pkg/filemanager/workflows/remote_download.go#L595-L609) 中删除临时目录：

```go
if m.state.Status != nil && m.node.IsMaster() && m.state.Status.SavePath != "" {
    os.RemoveAll(m.state.Status.SavePath)
}
```

此处 `SavePath` 仍是正斜杠格式（从 DB 反序列化而来）。`os.RemoveAll` 在 Windows 上也能正确处理正斜杠路径（Go 的 `os` 包内部会转换），所以不会有问题。

Aria2 的 `Cancel()` 在 [aria2.go:206-212](pkg/downloader/aria2/aria2.go#L206-L212) 使用 `status.SavePath`（也是正斜杠）传给 `os.RemoveAll`，同样安全。

### 11.3 运行时路径的潜在风险

1. **qBittorrent file.Name 不是相对路径**：qBittorrent 返回的 `file.Name` 是 `filepath.ToSlash(item.Name)`，通常包含种子名称前缀（如 `ubuntu-22.04/file.txt`）。当 `masterTransfer` 拼接 `path.Join(SavePath, file.Name)` 时，路径变为 `savepath/ubuntu-22.04/file.txt`。而 qBittorrent 实际保存的文件路径是 `SavePath/ubuntu-22.04/file.txt`，与拼接结果一致，**不会出问题**。但如果 qBittorrent 返回的 `Name` 字段与实际目录结构不完全匹配，就会出现 `os.Open` 找不到文件的错误。

2. **跨 OS 主从部署**：如果主机是 Linux、从机是 Windows（或反过来），`SavePath` 的正斜杠中间表示是正确的——因为 `filepath.FromSlash` 会在实际使用端按本地 OS 转换。但如果 Aria2/qBittorrent 运行在从机上，而 `SavePath` 中的根路径（如 `/tmp/`）在 Windows 上无意义，则 `os.Open` 会失败。**这不是代码 bug，而是部署约束**——从机下载器的临时路径必须是本机有效路径。

3. **Windows 长路径**：临时路径经过 `filepath.Join(base, "aria2", uuid)` 三层嵌套，再加上种子内部目录结构，可能超过 Windows 260 字符限制。Go 默认使用长路径前缀（`\\?\`）可缓解，但 Aria2/qBittorrent 自身不一定支持。

### 11.4 目标路径的 sanitize

目标文件名（非源文件路径）通过 [sanitizeFileName()](pkg/filemanager/workflows/remote_download.go#L681-L684) 处理：

```go
func sanitizeFileName(name string) string {
    r := strings.NewReplacer("\\", "_", ":", "_", "*", "_", "?", "_", "\"", "_", "<", "_", ">", "_", "|", "_")
    return r.Replace(name)
}
```

将 Windows 不允许的字符替换为下划线。源文件路径 `src` 不做 sanitize——因为它是下载器生成的本地路径，被假定为文件系统上合法且存在的路径。

### 11.5 Summarize 中的路径脱敏

[Summarize()](pkg/filemanager/workflows/remote_download.go#L629-L661) 中将 `SavePath` 置空后返回给前端：

```go
status := &*m.state.Status
status.SavePath = ""   // 脱敏：不暴露服务器本地路径
```

因此前端永远看不到 `SavePath`，只能通过 `Summary.download.downloaded/total` 等字段获取进度信息。
