# 远程下载任务代码分析

## 1. 整体架构与代码路径

### 1.1 核心文件清单

| 文件 | 作用 |
|------|------|
| [pkg/downloader/downloader.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/downloader/downloader.go) | 下载器抽象接口与通用状态定义 |
| [pkg/downloader/aria2/aria2.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/downloader/aria2/aria2.go) | Aria2 下载器具体实现 |
| [pkg/downloader/qbittorrent/qbittorrent.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/downloader/qbittorrent/qbittorrent.go) | qBittorrent 下载器具体实现 |
| [pkg/downloader/slave/slave.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/downloader/slave/slave.go) | 从机节点下载器代理（主机 → 从机 HTTP 调用） |
| [pkg/filemanager/workflows/remote_download.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/workflows/remote_download.go) | 远程下载任务工作流核心逻辑（状态机） |
| [pkg/filemanager/workflows/upload.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/workflows/upload.go) | 从机上传任务工作流（下载完成后文件传输） |
| [pkg/filemanager/workflows/worfklows.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/workflows/worfklows.go) | 工作流公共工具（节点分配、临时目录） |
| [pkg/queue/task.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/queue/task.go) | 通用任务接口、DBTask 实现、状态转换机 |
| [pkg/queue/queue.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/queue/queue.go) | 任务队列调度器（FIFO、重试、Worker） |
| [service/explorer/workflows.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/service/explorer/workflows.go) | HTTP API 服务层（创建下载任务、列表、取消） |
| [routers/controllers/file.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/routers/controllers/file.go) | HTTP Controller 层（CreateRemoteDownload 等） |
| [routers/controllers/slave.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/routers/controllers/slave.go) | 从机节点 API（SlaveDownloadTaskCreate/Status/Cancel 等） |
| [ent/task/task.go](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/ent/task/task.go) | 数据库 Task 表 Schema 与 Status 枚举 |

### 1.2 请求入口路由

在 [router.go:576-596](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/routers/router.go#L576-L596) 注册：

```
POST   /api/v4/workflow/download          创建远程下载任务
PATCH  /api/v4/workflow/download/:id      设置下载的目标文件（BT 多文件选择）
DELETE /api/v4/workflow/download/:id      取消下载任务
```

## 2. 任务状态体系

### 2.1 三层状态模型

远程下载任务的状态分为三个层次，每层职责不同：

#### 第一层：数据库任务状态（ent/task.Status）

定义于 [ent/task/task.go:97-104](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/ent/task/task.go#L97-L104)：

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

状态转换逻辑定义在 [pkg/queue/task.go:375-473](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/queue/task.go#L375-L473) 的 `stateTransitions` 映射表中：

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

定义于 [pkg/downloader/downloader.go:64-70](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/downloader/downloader.go#L64-L70)：

```go
const (
    StatusDownloading Status = "downloading"  // 下载中
    StatusSeeding     Status = "seeding"      // 做种中（BT）
    StatusCompleted   Status = "completed"    // 下载完成
    StatusError       Status = "error"        // 下载出错
    StatusUnknown     Status = "unknown"      // 未知状态
)
```

Aria2 的状态映射见 [aria2.go:105-122](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/downloader/aria2/aria2.go#L105-L122)：
- `active` + 完成度 100% + BT 模式 → `StatusSeeding`
- `active` → `StatusDownloading`
- `waiting/paused` → `StatusDownloading`
- `complete` → `StatusCompleted`
- `error` → `StatusError`
- `cancelled/removed` → 返回 `ErrTaskNotFount`

#### 第三层：工作流阶段（RemoteDownloadTaskPhase）

定义于 [remote_download.go:60-65](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/workflows/remote_download.go#L60-L65)：

```go
const (
    RemoteDownloadTaskPhaseNotStarted   RemoteDownloadTaskPhase = ""          // 初始，尚未创建下载器任务
    RemoteDownloadTaskPhaseMonitor                              = "monitor"   // 监控下载进度
    RemoteDownloadTaskPhaseTransfer                              = "transfer"  // 下载完成，正在传输到存储策略
    RemoteDownloadTaskPhaseAwaitSeeding                          = "seeding"   // 传输完成，等待做种结束
)
```

状态机入口是 [RemoteDownloadTask.Do()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/workflows/remote_download.go#L119-L170)，根据当前 Phase 分发到不同处理函数。

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

核心函数 [monitor()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/workflows/remote_download.go#L246-L324)：

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
| 创建任务 | [slave.go:CreateTask()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/downloader/slave/slave.go#L38-L71) | `POST /api/v4/slave/download/task` | [SlaveDownloadTaskCreate()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/routers/controllers/slave.go#L133-L144) |
| 查询状态 | [slave.go:Info()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/downloader/slave/slave.go#L73-L109) | `POST /api/v4/slave/download/status` | [SlaveDownloadTaskStatus()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/routers/controllers/slave.go#L147-L164) |
| 取消任务 | [slave.go:Cancel()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/downloader/slave/slave.go#L111-L138) | `POST /api/v4/slave/download/cancel` | [SlaveCancelDownloadTask()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/routers/controllers/slave.go#L167-L178) |
| 选择文件 | [slave.go:SetFilesToDownload()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/downloader/slave/slave.go#L140-L168) | `POST /api/v4/slave/download/select` | [SlaveSelectFilesToDownload()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/routers/controllers/slave.go#L181-L192) |

从机收到请求后，使用本地真实下载器（Aria2/qBittorrent）执行操作，并通过 Gob 编码返回结果。

## 5. 文件落库流程

下载完成后进入 `RemoteDownloadTaskPhaseTransfer` 阶段，根据节点角色分两种路径：

### 5.1 主机节点传输（masterTransfer）

代码见 [remote_download.go:428-557](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/workflows/remote_download.go#L428-L557)

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

核心落库路径 [manager.Update()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/manager/upload.go#L326-L367)：
1. `PrepareUpload()`：在 DBFS 中准备上传会话（创建文件 Entity、锁定目标路径）
2. `Upload()`：将文件流写入实际存储策略（本地/OSS/S3 等）
3. `CompleteUpload()`：更新文件元数据、清除上传会话、触发媒体元数据提取和全文索引任务

### 5.2 从机节点传输（slaveTransfer）

代码见 [remote_download.go:326-426](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/workflows/remote_download.go#L326-L426)

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

从机端执行 SlaveUploadTask 的逻辑见 [upload.go:69-223](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/workflows/upload.go#L69-L223) 的 `SlaveUploadTask.Do()`，它使用 **stateless 模式**调用主机 API 完成落库：

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
- 队列启动时 [queue.go:93-131](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/queue/queue.go#L93-L131) 的 `Start()` 会：
  1. 调用 `taskClient.GetPendingTasks()` 拉取未完成任务
  2. 通过 `NewTaskFromModel()` 用注册的工厂恢复为具体 Task 实例
  3. 重新入队 `QueueTask()` 继续执行

### 6.2 节点分配的固定性

[allocateNode()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/workflows/worfklows.go#L29-L42) 会优先使用 `state.NodeID` 中已保存的节点：
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

[createDownloadTask()](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/workflows/remote_download.go#L172-L226) 入口检查：
```go
if m.state.Handle != nil {
    m.state.Phase = RemoteDownloadTaskPhaseMonitor
    return task.StatusSuspending, nil
}
```
如果 `Handle` 已存在（即下载器任务已创建），直接跳过创建进入监控阶段，避免重复创建。

### 6.5 失败重试机制

- **队列级重试**：[queue.go:296-313](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/queue/queue.go#L296-L313)，非 CriticalErr 且未超 maxRetry 时，使用指数退避延迟重试
- **getTaskStatus 级重试**：`monitor()` 中连续获取状态失败最多 5 次
- **CriticalErr**：被标记为 `CriticalErr` 的错误（如 URL 非法、容量不足、参数错误）不进行重试，直接返回 `StatusError`

## 7. 任务清理（Cleanup）

任务到达 `StatusCompleted/Error/Canceled` 时，状态转换机自动调用 `task.Cleanup()`：

[remote_download.go:595-609](file:///d:/fz/0601-1/solo-dogfeeding/code/46-Cloudreve/pkg/filemanager/workflows/remote_download.go#L595-L609)：
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
