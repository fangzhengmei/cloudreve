# Cloudreve 离线下载任务处理路径分析

本文档详细剖析 Cloudreve v4 中，离线下载任务从用户提交到最终文件落盘的完整处理链路，重点覆盖**状态机**、**调度策略**、**失败重试**三段核心实现。

---

## 一、整体处理流程概览

```
用户提交 (API)
    │
    ▼
任务创建 (NewRemoteDownloadTask)
    │
    ▼
入队排队 (StatusQueued) ──────► 队列调度 (FIFO + ResumeTime 最小堆)
    │                                    ▲
    ▼                                    │
Worker 取任务 (StatusProcessing)         │
    │                                    │
    ├─► Phase: NotStarted               │
    │     └─ 分配节点 + 创建下载任务     │
    │          └─ 进入 Monitor 阶段 ────►│ 返回 StatusSuspending
    │                                    │
    ├─► Phase: Monitor                  │
    │     └─ 轮询下载器 (aria2/qBittorrent)
    │          ├─ 下载中 ───────────────►│
    │          ├─ 做种中 ───────────────►│
    │          └─ 完成/错误 ──► 进入 Transfer 阶段
    │
    ├─► Phase: Transfer
    │     ├─ Master: 并发上传 (worker pool)
    │     └─ Slave:  RPC 调从节点创建 SlaveUploadTask 轮询
    │          └─ 传输完成 ──► 进入 AwaitSeeding 阶段
    │
    └─► Phase: AwaitSeeding (等待做种结束)
          └─ 全部完成 ──► StatusCompleted / Cleanup
```

**核心代码文件索引**：

| 模块 | 文件路径 |
|------|---------|
| 任务工作流 | [remote_download.go](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go) |
| 队列框架 | [queue.go](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/queue/queue.go) / [scheduler.go](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/queue/scheduler.go) / [task.go](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/queue/task.go) |
| 节点调度 | [pool.go](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/cluster/pool.go) / [node.go](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/cluster/node.go) |
| 下载器接口 | [downloader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/downloader/downloader.go) |
| API 入口 | [workflows.go](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/service/explorer/workflows.go) |
| 任务状态定义 | [task.go](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/ent/task/task.go) |
| 队列配置 | [dependency.go](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/application/dependency/dependency.go#L689-L717) |

---

## 二、状态机实现

Cloudreve 的离线下载采用**三层嵌套状态机**设计：

### 2.1 第一层：任务生命周期状态 (Task.Status)

定义在 [ent/task/task.go#L90-L103](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/ent/task/task.go#L90-L103)：

| 状态值 | 含义 | 触发时机 |
|--------|------|---------|
| `queued` | 已入队，等待被 Worker 取走 | `QueueTask()` 提交时 |
| `processing` | 正在执行中 | Worker 从调度器取到任务时 |
| `suspending` | 挂起中（等待外部条件，如下载完成/下次轮询） | 任务 `Do()` 返回 `StatusSuspending` |
| `completed` | 全部完成（下载+传输+做种结束） | 所有阶段完成 |
| `error` | 执行失败（超过重试上限/致命错误） | 重试耗尽/`CriticalErr` |
| `canceled` | 用户主动取消 | 下载器返回 `ErrTaskNotFount` 且有历史状态 |

**状态转移矩阵** 定义在 [pkg/queue/task.go#L375-L473](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/queue/task.go#L375-L473)：

```
          ┌───────────────────────────────────────────────────┐
          │                                                   ▼
  (初始空) ──► queued ──► processing ──► suspending ──► processing ...
                  │          │                                    │
                  │          ├──────────────► completed ◄─────────┘
                  │          ├──────────────► error
                  │          └──────────────► canceled
                  └──────────────► error
```

关键转移逻辑：
- **processing → suspending**：持久化状态后，调用 `q.QueueTask()` **重新入堆**等待 ResumeTime 到达
- **processing → completed/error/canceled**：调用 `task.Cleanup()` 清理下载任务、临时目录，从 registry 删除
- **suspending → processing**：调度器检测到 `ResumeTime <= now` 时出堆

### 2.2 第二层：任务内部阶段 (RemoteDownloadTaskPhase)

定义在 [pkg/filemanager/workflows/remote_download.go#L43-L78](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L43-L78)，存储在 `PrivateState.Phase` 字段（JSON 序列化持久化）：

| 阶段值 | 含义 | 关键动作 |
|--------|------|---------|
| `""` (NotStarted) | 刚创建，尚未分配节点 | SSRF 校验 → 分配节点 → 创建下载器实例 → 调用 `CreateTask()` 交给 aria2/qB |
| `monitor` | 监控下载进度 | 按 `NodeSetting.Interval` 轮询下载器 `Info()`，校验容量，检测状态变迁 |
| `transfer` | 从节点临时目录传输到目标存储策略 | Master 本地并发上传 / Slave 通过 RPC 调从节点上传 |
| `seeding` (AwaitSeeding) | 传输完成后等待做种结束 | 若 `WaitForSeeding=false` 直接完成 |

**阶段转移** 由 `Do()` 方法的 switch 驱动（[remote_download.go#L148-L159](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L148-L159)）：

```go
switch m.state.Phase {
case RemoteDownloadTaskPhaseNotStarted:
    next, err = m.createDownloadTask(ctx, dep)    // → Phase = monitor
case RemoteDownloadTaskPhaseMonitor, RemoteDownloadTaskPhaseAwaitSeeding:
    next, err = m.monitor(ctx, dep)               // 完成/做种 → Phase = transfer
case RemoteDownloadTaskPhaseTransfer:
    if m.node.IsMaster() {
        next, err = m.masterTransfer(ctx, dep)    // 本机上传
    } else {
        next, err = m.slaveTransfer(ctx, dep)     // 调从节点
    }                                              // → Phase = seeding
}
```

### 2.3 第三层：下载器内部状态 (downloader.Status)

定义在 [pkg/downloader/downloader.go#L64-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/downloader/downloader.go#L64-L72)：

| 状态值 | 含义 | → 第二层次映射 |
|--------|------|--------------|
| `downloading` | 下载进行中 | 保持 monitor，按 Interval 轮询 |
| `seeding` | 下载完成，正在做种 | → Phase = transfer（先传文件） |
| `completed` | 全部完成（含做种） | 若尚未传输 → Phase = transfer；若已传输 → StatusCompleted |
| `error` | 下载失败 | → StatusError（标记 CriticalErr） |
| `unknown` | 状态未知 | → StatusError（标记 CriticalErr） |

状态判定代码在 [monitor()](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L288-L324) 中：
- 下载中/做种中：返回 `StatusSuspending`，由 `ResumeAfter(Interval)` 控制下次轮询时间
- 完成但未进入 transfer：切换 Phase 后立即 `ResumeAfter(0)` 触发下次执行
- 做种配置关闭 (`WaitForSeeding=false`)：传输完成后直接 `StatusCompleted` 跳过等待

---

## 三、调度策略

调度分为**三层**：任务队列调度、节点分配调度、文件传输并发调度。

### 3.1 任务队列调度 (FIFO + ResumeTime 最小堆)

实现于 [pkg/queue/scheduler.go](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/queue/scheduler.go)：

**数据结构**：
```go
type fifoScheduler struct {
    taskQueue taskHeap   // heap.Interface，按 ResumeTime 升序的最小堆
    capacity  int        // 队列容量，0 表示无限
    count     int        // 当前元素数
}

func (h taskHeap) Less(i, j int) bool {
    return h[i].ResumeTime() < h[j].ResumeTime()  // 早到期的先出堆
}
```

**取任务逻辑**（[scheduler.go#L60-L79](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/queue/scheduler.go#L60-L79)）：
```go
func (s *fifoScheduler) Request() (Task, error) {
    // ...
    if s.taskQueue[s.taskQueue.Len()-1].ResumeTime() > time.Now().Unix() {
        return nil, ErrNoTaskInQueue   // 堆顶都未到期，暂无可执行任务
    }
    data := s.taskQueue.Pop()          // 弹出最早到期的任务
    return data.(Task), nil
}
```

**Worker 模型**（[queue.go#L380-L438](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/queue/queue.go#L380-L438)）：
- 固定 Worker 数（`workerCount`，默认 CPU 核数，可通过 `QueueSetting.WorkerNum` 配置）
- `schedule()` 检查忙 Worker 数未达上限时，向 `ready` 通道发信号
- 收到信号后异步从调度器 `Request()` 取任务（无任务则按 `taskPullInterval=10s` 间隔重试）
- 取到任务后用 goroutine 执行 `q.work(t)`

**RemoteDownloadQueue 专属配置**（[dependency.go#L689-L717](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/application/dependency/dependency.go#L689-L717)）：
- 启动时从 DB 恢复所有 `remote_download` 类型的挂起/排队任务
- `taskPullInterval = 10s`（远程下载任务对实时性不敏感）
- 最大执行时间 `maxTaskExecution`（默认 60h）
- 单例模式，可通过管理后台热重载

### 3.2 节点分配调度 (加权轮询 WRR)

实现于 [pkg/cluster/pool.go](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/cluster/pool.go)：

**节点池初始化**：
- 按 `NodeCapability`（如 `NodeCapabilityRemoteDownload`）分桶存储
- 每个节点有 `weight`（管理员配置）和 `current`（当前加权值，初始 0）

**选择算法**（[pool.go#L88-L146](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/cluster/pool.go#L88-L146)）——平滑加权轮询：
```
1. 若指定了 preferred NodeID（即 state.NodeID>0，任务恢复场景），优先用已分配节点
2. 否则遍历桶内所有节点：
     item.current += max(1, item.weight)
     total += max(1, item.weight)
     记录 current 最大的节点为 selected
3. selected.current -= total   // 扣减总权重，保证长期公平
4. 返回 selected.node
```

示例（权重 A=3, B=1）：
| 请求 | A.current | B.current | total | 选中 | 调整后 |
|------|-----------|-----------|-------|------|--------|
| 1 | 0+3=3 | 0+1=1 | 4 | A | A=-1, B=1 |
| 2 | -1+3=2 | 1+1=2 | 4 | A(同权先出现者) | A=-2, B=2 |
| 3 | -2+3=1 | 2+1=3 | 4 | B | A=1, B=-1 |
| 4 | 1+3=4 | -1+1=0 | 4 | A | A=0, B=0 |

结果分布 A:B = 3:1，符合权重。

### 3.3 文件传输并发调度

#### Master 节点（本机传输）

实现于 [masterTransfer()](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L428-L557)：
- 信号量模式：缓冲通道 `worker` 容量 = `MaxParallelTransfer`（系统配置）
- 每个待传文件从 `worker` 通道取一个 slot，goroutine 执行 `transferFunc`，完成后归还
- `Transferred` map 记录已成功上传的文件索引，**断点续传粒度为单个文件**
- 上传进度通过原子操作维护：单文件进度、总字节进度、总文件数进度

#### Slave 节点（从节点传输）

实现于 [slaveTransfer()](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L326-L426)：
- Master 不直接传文件，而是通过 `slaveNode.CreateTask()` 发起 RPC 在从节点创建 `SlaveUploadTask`
- 每 30 秒轮询一次从节点任务状态（`ResumeAfter(30s)`）
- 进度数据通过 `NodeState.progress` 从 SlaveTaskSummary 合并
- 从节点任务部分成功时，Master 将已成功的文件索引记入 `Transferred`，**下次重建 Slave 任务时跳过这些文件**

---

## 四、失败重试机制

重试机制同样是**分层设计**，区分队列级（框架通用）与业务级（离线下载特有）。

### 4.1 队列级重试 (指数退避 Backoff)

实现于 [queue.go#L296-L313](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/queue/queue.go#L296-L313)：

**触发条件**（三者同时满足）：
1. `t.Do()` 返回 error
2. `maxRetry - t.Retried() > 0`（未超重试上限）
3. 错误不包含 `CriticalErr`（非致命错误）
4. 队列未进入关闭流程

**退避算法**（`github.com/jpillora/backoff`）：
```
delay = retryDelay               // 若配置了固定延迟（RetryDelay）
否则 delay = backoff.ForAttempt(n)
       = min(backoffMaxDuration, 1000ms * backoffFactor^n)
```

默认参数（[options.go#L33-L43](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/queue/options.go#L33-L43)）：
- `backoffFactor = 2`
- `backoffMaxDuration = 60s`
- `retryDelay = 0`（启用指数退避）

**重试计数**：每次重试前调用 `t.OnRetry(err)`，将错误记入 `ErrorHistory`，`RetryCount++`（[task.go#L304-L316](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/queue/task.go#L304-L316)）。

### 4.2 业务级重试：获取下载状态

实现于 [monitor()](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L250-L266)：

```
GetTaskStatusMaxTries = 5
```

场景：下载器 RPC 调用失败（如 aria2 重启、网络波动），但任务本身仍可能有效。

- 失败时 `GetTaskStatusTried++`，记录 warning 日志
- 未达上限：按节点 `Interval` 正常挂起轮询
- 达上限：向上返回 error，进入队列级重试（此时可能走指数退避，时间更长）
- **特殊处理**：若 `ErrTaskNotFount` 且任务曾有状态记录 → 判定用户通过外部工具取消了任务 → `StatusCanceled`（不进入重试）

### 4.3 业务级重试：文件传输部分失败

Master 模式（[remote_download.go#L548-L552](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L548-L552)）：
- 传输中单个文件失败不立即中止，用 `AggregateError` 聚合
- 全部文件遍历结束后，若 `failed > 0`，返回 error 进入队列级重试
- `Transferred[index]` 已记录成功文件，**下次 Do() 迭代不会重复上传**（断点续传）

Slave 模式（[remote_download.go#L397-L416](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L397-L416)）：
- 从节点任务结束（成功或失败）时，比对 `Transferred` 长度与待传文件数
- 部分成功：将已传索引合并到 Master 的 `Transferred`，`SlaveUploadTaskID = 0`，返回 error
- 队列级重试时，会重新创建不包含已传文件的 SlaveUploadTask

### 4.4 致命错误标记 (CriticalErr)

定义于 [queue.go#L64-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/queue/queue.go#L64-L66)：
```go
var CriticalErr = errors.New("non-retryable error")
```

**会被包装为 `%w` CriticalErr 的场景**（直接跳过重试进入 StatusError）：

| 场景 | 代码位置 |
|------|---------|
| SSRF 校验失败（用户输入恶意内网 URL） | [remote_download.go#L189-L191](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L189-L191) |
| 种子文件 URI 解析失败 | [remote_download.go#L197-L199](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L197-L199) |
| 用户容量/配额预校验失败 | [remote_download.go#L279-L282](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L279-L282) |
| 下载器返回 `StatusError` / `StatusUnknown` | [remote_download.go#L318-L320](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L318-L320) |
| 目标 URI 解析失败（Transfer 阶段） | [remote_download.go#L334-L336](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L334-L336) |
| Slave 任务被 Cancel | [remote_download.go#L419-L421](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L419-L421) |
| Slave 状态反序列化失败 | [remote_download.go#L393-L395](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L393-L395) |

### 4.5 任务取消与清理

用户发起取消：[workflows.go#L398-L414](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/service/explorer/workflows.go#L398-L414) → 调 `CancelDownload()` → `downloader.Cancel(handle)`。

自动清理（StatusCompleted / StatusError 时）：[Cleanup()](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/pkg/filemanager/workflows/remote_download.go#L595-L609)
- 取消下载器中的任务
- Master 节点：删除 `SavePath` 临时目录（从节点需由对应流程单独清理）

---

## 五、提交入口汇总

| 动作 | API | 核心函数 |
|------|-----|---------|
| 提交下载任务 | `POST /api/v3/file/download` | [CreateDownloadTask()](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/service/explorer/workflows.go#L81-L169) |
| 查看任务列表 | `GET /api/v3/task?category=downloading/downloaded` | [ListTasks()](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/service/explorer/workflows.go#L324-L384) |
| 获取实时进度 | WebSocket / SSE | [TaskPhaseProgress()](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/service/explorer/workflows.go#L386-L396) |
| 设置要下载的文件（种子） | `PUT /api/v3/task/:id/download` | [SetDownloadFiles()](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/service/explorer/workflows.go#L423-L452) |
| 取消下载 | `DELETE /api/v3/task/:id/download` | [CancelDownloadTask()](file:///d:/fz/0601-2/solo-dogfeeding/code/32-Cloudreve/service/explorer/workflows.go#L398-L414) |

### 提交流程关键校验

```
CreateDownloadTask()
    ├─► 权限：GroupPermissionRemoteDownload
    ├─► 批量大小：Aria2BatchSize（用户组配置）
    ├─► 目标目录：必须存在且有 CreateFile 权限
    ├─► SrcFile（种子文件）：必须存在且有 DownloadFile 权限
    └─► 逐源创建 NewRemoteDownloadTask → RemoteDownloadQueue.QueueTask()
```

---

## 六、状态持久化说明

任务状态分两部分持久化到 DB（`ent.Task` 表）：

| 字段 | 内容 | 可见性 |
|------|------|--------|
| `status` | 第一层状态 (queued/processing/...) | 公开 |
| `public_state` | `TaskPublicState`：RetryCount、ErrorHistory、Error、ResumeTime、ExecutedDuration | 公开（前端可见） |
| `private_state` | `RemoteDownloadTaskState` JSON：Phase、Handle、Status、NodeID、Transferred、SlaveUploadTaskID 等 | 仅服务端内部使用 |

**服务重启恢复**：Queue `Start()` 时查询 DB 中 pending 状态的 `remote_download` 任务，用 `NewTaskFromModel()` 反序列化重建，重新入调度器。由于 PrivateState 完整保存了所有中间状态（包括下载器 Handle、已传文件列表），恢复后可从断点无缝继续。
