# Cloudreve 离线下载任务处理路径分析

本文档详细剖析 Cloudreve v4 中，离线下载任务从用户提交到最终文件落盘的完整处理链路，重点覆盖**状态机**、**调度策略**、**失败重试**三段核心实现，并结合真实代码行为纠正"按 ResumeTime 堆排序"等常见误解。

---

## 一、核心代码文件索引

以下使用仓库相对路径定位代码：

| 模块 | 路径 |
|------|------|
| 离线下载工作流（任务阶段流转） | `pkg/filemanager/workflows/remote_download.go` |
| 队列框架（Worker、状态转移、重试） | `pkg/queue/queue.go` |
| 调度器（入队/出队） | `pkg/queue/scheduler.go` |
| Task 接口、DBTask、状态转移表 | `pkg/queue/task.go` |
| 队列参数默认值 | `pkg/queue/options.go` |
| 节点池（节点分配） | `pkg/cluster/pool.go` |
| Node 接口与 Master/Slave 实现 | `pkg/cluster/node.go` |
| Downloader 接口定义 | `pkg/downloader/downloader.go` |
| API 提交/列表/取消入口 | `service/explorer/workflows.go` |
| Slave 侧任务创建/查询 | `service/node/task.go` |
| 任务状态枚举（ent） | `ent/task/task.go` |
| 远程下载队列构造（热重载） | `application/dependency/dependency.go` |
| 工作流通用工具（allocateNode 等） | `pkg/filemanager/workflows/worfklows.go` |

---

## 二、三层状态机

Cloudreve 的离线下载状态机分为三层：**任务生命周期状态** → **业务阶段** → **下载器内部状态**，逐层嵌套。

### 2.1 第一层：任务生命周期状态（Task.Status）

定义于 `ent/task/task.go`，共 6 种：

| 枚举值 | 含义 |
|--------|------|
| `queued` | 已提交入队，等待 Worker 取走 |
| `processing` | 正在被 Worker 执行（调用 `t.Do()`） |
| `suspending` | 挂起中（需等待外部条件，如轮询间隔或重试退避） |
| `completed` | 全部流程完成 |
| `error` | 执行失败（重试耗尽或致命错误） |
| `canceled` | 用户主动取消 |

**状态转移表**定义于 `pkg/queue/task.go` 的 `stateTransitions` 变量：

```
  (初始空) ──► queued ──► processing ──┬──► suspending ──► processing (循环)
                 │          │            │
                 │          ├────────────┼──► completed
                 │          ├────────────┼──► error
                 │          └────────────┼──► canceled
                 └───────────────────────┘
                              queued ──► error
```

每个合法转移附带一个回调函数，关键动作如下：

- **`"" → queued`**：`persistTask` —— 新建任务首次入队时持久化到 DB。
- **`queued → processing`**：`persistTask` —— Worker 取到任务后更新状态。
- **`processing → suspending`**：`persistTask` 之后**立即调用 `q.QueueTask(ctx, task)` 重新入调度器**（这是"挂起"的本质——不是停在那儿等，而是重新塞回调度器排队，等 ResumeTime 到了再被取到）。
- **`processing → completed / error / canceled`**：先执行 `task.Cleanup()`（取消下载器任务、删临时目录、从 registry 移除），再持久化。
- **`suspending → processing`**：`persistTask`，同时 `metric.DecSuspendingTask()`。

### 2.2 第二层：业务阶段（RemoteDownloadTaskState.Phase）

状态保存在 `Task.PrivateState`（JSON 持久化），枚举值与行为定义于 `pkg/filemanager/workflows/remote_download.go`。由 `RemoteDownloadTask.Do()` 内的 switch 驱动：

| Phase | 行为 |
|-------|------|
| `""`（NotStarted） | 调用 `allocateNode` 分配节点、创建 Downloader 实例、校验 SSRF、调用下载器 `CreateTask`。之后 Phase 切到 `monitor`，返回 `StatusSuspending`。 |
| `monitor` / `seeding`（AwaitSeeding） | 按节点配置的 `Interval` 周期性调用下载器 `Info()`，根据第三层状态决定下一步。 |
| `transfer` | 若节点是 Master：本机开 Worker 池并发上传到目标存储；若是 Slave：通过 RPC 在从节点创建 `SlaveUploadTask` 并轮询。完成后切到 `seeding`（AwaitSeeding）。 |

`monitor` 阶段的关键分支（见 `remote_download.go` 的 `monitor()`）：

| 下载器状态 | 动作 |
|------------|------|
| `downloading` | `ResumeAfter(Interval)`，返回 `StatusSuspending` 等下次轮询 |
| `seeding` | 若尚未传输 → Phase 切 `transfer`，`ResumeAfter(0)` 立即再次执行；已传输且 `WaitForSeeding=false` → 直接 `StatusCompleted`；否则继续挂起等做种 |
| `completed` | 若尚未传输 → Phase 切 `transfer`；否则 → `StatusCompleted` |
| `error` / `unknown` | 返回 error 并包装 `CriticalErr`（不重试） |

另外，`monitor` 中还包含：
- **Handle 跟随**：若下载器返回 `status.FollowedBy != nil`（例如 aria2 中种子任务创建完后产生新的下载任务），替换 `state.Handle` 并立即再执行一次。
- **容量预校验**：首次拿到或 Total 变化时，调 `validateFiles` → `fm.PreValidateUpload` 检查用户容量、命名合法性等，失败直接 `CriticalErr`。
- **状态查询失败容忍**：`GetTaskStatusTried` 连续失败 5 次才向上抛 error，中间每次 `ResumeAfter(Interval)` 挂起。

### 2.3 第三层：下载器内部状态（downloader.Status）

定义于 `pkg/downloader/downloader.go`，由 aria2/qBittorrent/slave 三种适配器各自实现：

| 枚举值 | 含义 |
|--------|------|
| `downloading` | 正在下载 |
| `seeding` | 下载完成、正在做种（BT） |
| `completed` | 所有动作结束 |
| `error` | 下载失败 |
| `unknown` | 下载器无响应或状态不明 |

适配器位置：
- Aria2：`pkg/downloader/aria2/aria2.go`
- qBittorrent：`pkg/downloader/qbittorrent/qbittorrent.go`
- Slave 代理（调从节点 RPC）：`pkg/downloader/slave/slave.go`

---

## 三、调度策略：入队、取任务、恢复执行

本节重点澄清：**`fifoScheduler` 并不是按 ResumeTime 排序的最小堆。**

### 3.1 实际数据结构：slice + 尾部弹出（近似 LIFO）

看 `pkg/queue/scheduler.go`：

```go
type fifoScheduler struct {
    taskQueue taskHeap   // 底层是 []Task
    count     int        // 元素数
    ...
}
type taskHeap []Task

// 虽然声明了 heap.Interface 的 5 个方法：
func (h taskHeap) Len() int           { ... }
func (h taskHeap) Less(i, j int) bool { return h[i].ResumeTime() < h[j].ResumeTime() }
func (h taskHeap) Swap(i, j int)      { ... }
func (h *taskHeap) Push(x any)        { *h = append(*h, x.(Task)) }
func (h *taskHeap) Pop() any          { x := (*h)[len(*h)-1]; *h = (*h)[:len(*h)-1]; return x }
```

**但 `container/heap` 从未被 import，`heap.Init/Push/Pop/Fix` 一个都没调用。** 所以：

- `Less()` 函数虽然写了"按 ResumeTime 升序"，但**从未被执行过**，只是死代码。
- `fifoScheduler.Queue(task)` → `taskQueue.Push(task)` → 只是 `append` 到 slice **末尾**。
- `fifoScheduler.Request()` → 检查 slice **最后一个**元素 `taskQueue[Len()-1]` 的 `ResumeTime <= now`：
  - 到期 → `Pop()` 取最后一个元素（栈顶）返回。
  - 未到期 → 直接返回 `ErrNoTaskInQueue`，**完全不检查 slice 前面的元素**。

所以 `fifoScheduler` 的真实语义是：**一个带"只看栈顶到期时间"的 LIFO 栈**，名字里的 FIFO 是误导的。

这带来的直接影响：
- 新提交的任务（append 到尾部）总是被优先检查，可能导致早提交但 ResumeTime 未到的任务长期被压在栈底（直到栈顶任务全部出空才有机会被检查）。
- 任务挂起后再次入队（见 2.1 节），会重新 append 到尾部，下次 `Request` 第一个就看到它——如果 ResumeTime 还没到，整个调度器就陷入"无任务可取"，直到 taskPullInterval 超时后再轮询一次。

### 3.2 提交入队流程

入口 1：用户 API 提交 → `service/explorer/workflows.go` 的 `CreateDownloadTask()`：
```
权限校验(GroupPermissionRemoteDownload)
  → 目标目录 / 种子文件合法性检查
  → 批量大小校验(Aria2BatchSize)
  → 逐个 src 调 workflows.NewRemoteDownloadTask()
  → dep.RemoteDownloadQueue(c).QueueTask(c, t)
```

入口 2：服务重启恢复 → `pkg/queue/queue.go` 的 `(q *queue).Start()`：
```
GetPendingTasks(ctx, "remote_download")  // 查 DB 中未完成的任务
  → 逐个 NewTaskFromModel() 反序列化
  → QueueTask(ctx, resumedTask)          // 与新提交共用同一条入队路径
```

`QueueTask` 的动作（`pkg/queue/queue.go`）：
1. 若 `t.Status() != suspending`，先做 `"" → queued` 的状态转移（持久化 DB，生成 Task ID）。
2. `q.scheduler.Queue(t)` → 把 Task append 到调度器 slice 尾部。
3. `registry.Set(t.ID(), t)` → 登记到内存注册表（供 API 查询进度、取消等使用）。

### 3.3 Worker 取任务与执行循环

`(q *queue).start()` 的结构：

```
for {
    q.schedule()             // 若忙 Worker < workerCount，向 ready 通道塞一个信号
    <-q.ready                // 等有空闲 Worker

    // 后台 goroutine 拉任务
    go func() {
        for {
            t, err := q.scheduler.Request()
            if t == nil {
                // 取不到：sleep taskPullInterval（远程下载队列是 10s），然后重试
                select {
                case <-time.After(q.taskPullInterval):
                case <-q.quit: return
                }
                continue
            }
            tasks <- t       // 拿到任务，送入主循环
            return
        }
    }()

    t := <-tasks
    q.metric.IncBusyWorker()
    go q.work(t)             // 执行任务
}
```

`q.work(t)`（单任务生命周期）：
```
transitStatus → processing
for {
    next, err := q.run(ctx, t)     // 调 t.Do()，内含重试逻辑
    if err != nil {
        transitStatus → error
        break
    }
    t.OnIterationComplete(...)
    transitStatus → next           // 可能是 processing / suspending / completed / ...
    if next != processing { break }
}
```

注意 `q.run()` 内部会把需要重试的 error 吃掉、改写 next 为 `suspending`、设置 ResumeTime——所以 `work` 里看到的 `err` 已经是 nil，通过 `next == suspending` 走转移回调重新入队（见 2.1）。只有超过重试次数或属于 `CriticalErr` 的错误，才会让 `run` 返回 error，进而 `work` 中 transit 到 `error`。

### 3.4 恢复执行的两条路径

`RemoteDownloadTask` 有两个构造函数，对应两种恢复场景，实例复用策略完全不同：

```go
// 路径 1：用户新提交任务
func NewRemoteDownloadTask(ctx, src, srcFile, dst string) (queue.Task, error)

// 路径 2：服务重启从 DB 恢复
func NewRemoteDownloadTaskFromModel(task *ent.Task) queue.Task
```

**关键区别**：`RemoteDownloadTask` 结构体中有三个字段**不写入 DB**（只在内存存在）：

| 字段 | 类型 | 持久化？ | 作用 |
|------|------|---------|------|
| `m.node` | `cluster.Node` | ❌ | 当前选中的节点实例 |
| `m.d` | `downloader.Downloader` | ❌ | 下载器客户端实例（绑定到具体节点） |
| `m.state` | `*RemoteDownloadTaskState` | ❌ | `Do()` 执行期间的临时状态，每次重新反序列化 |

而 `m.Task.PrivateState`（JSON 字符串）**会写入 DB**，包含 `Handle`、`Phase`、`NodeID`、`Transferred`、`SlaveUploadTaskID` 等。

基于此，"挂起→恢复"有两条完全不同的路径：

#### 路径 A：同进程内挂起 → 恢复

同进程内 `processing → suspending → processing` 的场景（绝大多数情况）：

1. **`registry.Delete` 只在终态（completed/error/canceled）调用**，`suspending` 转移时**不**删除 Task 实例。
2. 调度器存储和取出的是**同一个 Task 指针**，实例内存地址不变。
3. `m.d` 字段保留上一次 `CreateDownloader` 创建的实例，`m.d != nil`。
4. 但每次 `Do()` 开头都会重新 `json.Unmarshal(m.State(), state)` 重建 `m.state`，并重新 `allocateNode` 选节点。

**同进程内恢复的 Do() 流程**：
```
Do()
  ├─► json.Unmarshal(m.State(), &state)     // 每次重建 m.state
  ├─► allocateNode → np.Get(cap, state.NodeID)  // 每次重新选节点
  │      ├─► preferred = state.NodeID（上一次写入 DB 的值）
  │      └─► 精确匹配成功 → 返回原节点
  ├─► m.node = 新选节点
  ├─► if m.d == nil { ... }                 // 同进程内 m.d != nil，跳过！
  └─► 继续 Phase switch
```

#### 路径 B：服务重启后从 DB 恢复

进程退出后内存清空，重启时 `queue.Start()` 从 DB 恢复：

1. `GetPendingTasks(ctx, "remote_download")` 从 ent.Task 表查出所有未完成的任务。
2. 逐个调用 `NewRemoteDownloadTaskFromModel(task)` 创建**全新**的 Task 实例。
3. 新实例的 `m.d == nil`、`m.node == nil`、`m.state == nil`。
4. `QueueTask` 重新入调度器，同时 `registry.Set(id, newTaskInstance)`。

**服务重启后首次 Do() 流程**：
```
Do()
  ├─► json.Unmarshal(m.State(), &state)     // 首次反序列化
  ├─► allocateNode → np.Get(cap, state.NodeID)  // 从 DB 恢复的 NodeID
  │      ├─► 若原节点仍在 → 精确匹配，返回原节点
  │      └─► 若原节点已删 → fallback WRR 选新节点，state.NodeID 被更新
  ├─► m.node = 新选节点
  ├─► if m.d == nil { ... }                 // 重启后 m.d == nil，执行！
  │      └─► node.CreateDownloader(...)     // 基于当前选中的节点创建全新 downloader
  └─► 继续 Phase switch
```

两条路径的核心差异可以用下表总结：

| 维度 | 同进程内挂起恢复 | 服务重启恢复 |
|------|-----------------|-------------|
| Task 实例 | 同一个指针（registry 未删除） | 全新实例（NewFromModel 创建） |
| `m.d` | 非 nil，不重新 `CreateDownloader` | nil，基于当前节点重新创建 |
| `m.node` | 每次重新选节点，可能与 m.d 不一致 | 每次重新选节点，与新创建的 m.d 一致 |
| 状态来源 | 反序列化上一次 Do() 写入的 PrivateState | 反序列化 DB 中最后一次持久化的 PrivateState |
| Handle 有效性 | 由原节点的下载器持有，若节点未变则有效 | 若原节点未变且下载器未重启则有效；否则失效 |

**代码没有区分这两条路径**，`Do()` 的开头对两种场景走同样的流程。这导致同进程内如果发生节点切换（原节点被删），`m.node` 变成新节点但 `m.d` 还是旧节点的 downloader，两者不一致。

### 3.5 节点分配：加权轮询（WRR）

离线下载任务首次执行（Phase `""`）时，在 `allocateNode`（`pkg/filemanager/workflows/worfklows.go`）中调 `NodePool.Get()` 分配一个具备 `NodeCapabilityRemoteDownload` 能力的节点。实现位于 `pkg/cluster/pool.go`：

```go
func (p *weightedNodePool) Get(ctx, capability, preferred int) (Node, error) {
    // 1) 若 state.NodeID 已指定（恢复执行场景），优先复用原节点
    // 2) 否则对桶内每个节点：item.current += max(1, item.weight)
    //    记录 current 最大的节点为 selected
    // 3) selected.current -= total（所有权重之和）
    // 4) 返回 selected.node
}
```

这是标准**平滑加权轮询**（Nginx WRR 同类算法）：权重 A=3, B=1 时的分配序列为 `A, A, B, A`，长周期比例精确匹配权重。

节点分配后缓存到 `state.NodeID`，后续迭代（挂起→恢复）复用，不会在每次 `Do()` 时重新挑节点。

---

## 四、失败重试：队列级 + 业务级

### 4.1 队列级重试（指数退避）

位置：`pkg/queue/queue.go` 的 `(q *queue).run()`，在 `t.Do(ctx)` 返回后：

```go
if err != nil
    && q.maxRetry - t.Retried() > 0           // 还有重试配额
    && !errors.Is(err, CriticalErr)            // 非致命错误
    && atomic.LoadInt32(&q.stopFlag) != 1 {    // 队列未关闭

    t.OnRetry(err)                             // RetryCount++, 记入 ErrorHistory
    b := &backoff.Backoff{Max: q.backoffMaxDuration, Factor: q.backoffFactor}
    delay := q.retryDelay                      // 若配置了固定延迟就用它
    if q.retryDelay == 0 {
        delay = b.ForAttempt(float64(t.Retried()))   // 否则指数退避
    }
    t.OnSuspend(time.Now().Add(delay).Unix())  // 写 ResumeTime
    err = nil
    next = task.StatusSuspending               // 改走 suspending 路径
}
```

默认参数（`pkg/queue/options.go`，可通过管理后台的队列配置覆盖）：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `backoffFactor` | 2 | 指数底数 |
| `backoffMaxDuration` | 60s | 单次延迟上限 |
| `retryDelay` | 0 | 非 0 时忽略指数退避，用固定延迟 |
| `maxRetry` | 0（具体值由管理后台配置） | 最多重试次数 |
| `maxTaskExecution` | 60h | 单任务累计执行时间上限（含所有重试） |

队列级重试后的路径：
`run` 返回 `(StatusSuspending, nil)` → `work` 中 `transitStatus(processing → suspending)` → 转移回调 `q.QueueTask(ctx, task)` 重新 append 到调度器尾部 → 下次 Request 时检查 ResumeTime。

### 4.2 业务级重试：下载状态查询

位置：`remote_download.go` 的 `monitor()`。调用 `m.d.Info(ctx, handle)` 获取下载状态失败时：

| 情况 | 处理 |
|------|------|
| `errors.Is(err, ErrTaskNotFount)` 且 `state.Status != nil` | 判定任务被外部（如 aria2 Web UI）手动删除 → 返回 `StatusCanceled` |
| 其他错误，且 `GetTaskStatusTried < 5` | `GetTaskStatusTried++`，`ResumeAfter(Interval)` 挂起，**不上抛 error**，所以不走队列级退避（立即用节点 Interval 重试） |
| 达到 `GetTaskStatusMaxTries = 5` | 向上返回 error → 进入队列级重试（指数退避） |

这是两级重试的配合：网络抖动等临时故障先用短周期重试，持续失败再走长退避。

### 4.3 业务级重试：文件传输部分失败

传输阶段不使用"整个任务重来"，而是以**单个文件**为粒度断点续传。

**Master 本机上传**（`masterTransfer`）：
- 信号量通道控制并发 `MaxParallelTransfer`。
- 每个文件上传成功后，写入 `state.Transferred[file.Index] = nil`。
- 全部文件处理完后，若 `failed > 0`，整体返回 error，触发队列级重试。下次 Do() 进入 transfer 阶段时，已在 `Transferred` 中的文件直接跳过，进度字节数也原子累加到总进度中。

**Slave 上传**（`slaveTransfer`）：
- Master 通过 `node.CreateTask` 在从节点创建 `SlaveUploadTask`，记下 `state.SlaveUploadTaskID`。
- 每 30s 轮询一次从节点任务状态。
- 若从节点任务结束（完成或失败）但 `len(Transferred) < len(Files)`：
  - 把从节点已成功的索引合并进 `state.Transferred`
  - `SlaveUploadTaskID = 0`（下次重新创建一个只包含剩余文件的 SlaveUploadTask）
  - 返回 error，走队列级重试

### 4.4 致命错误（CriticalErr）

`CriticalErr`（`pkg/queue/queue.go`）是一个 sentinel，`errors.Is(err, CriticalErr)` 为 true 的错误**直接跳过所有队列级重试**，立即转 `StatusError`。在 `remote_download.go` 中会被包装的场景：

| 场景 | 位置 |
|------|------|
| 用户提交的 URL 未通过 SSRF 校验 | `createDownloadTask` |
| 种子文件内部 URI 非法 | `createDownloadTask` |
| 用户容量 / 文件命名预校验失败 | `monitor`（首次拿到 Total 时） |
| 下载器返回 `StatusError` / `StatusUnknown` | `monitor` |
| 目标 URI 非法（transfer 阶段） | `slaveTransfer` / `masterTransfer` |
| Slave 任务被 Cancel | `slaveTransfer` |
| Slave 返回的状态反序列化失败 | `slaveTransfer` |

### 4.5 取消与清理

- **用户取消**：API 调 `service/explorer/workflows.go` 的 `CancelDownloadTask` → 类型断言到 `*RemoteDownloadTask` → `m.d.Cancel(ctx, handle)` 通知下载器停掉任务；但生命周期状态仍由后续 `Do()` 迭代中的 `monitor` 感知（遇到 `ErrTaskNotFount` 后走 `StatusCanceled`）。
- **自动清理**：`transitStatus(processing → completed/error/canceled)` 回调中统一调 `task.Cleanup()`，对于远程下载即 `RemoteDownloadTask.Cleanup()`：
  - 取消下载器任务（若 handle 存在）
  - Master 节点：删除 `SavePath` 临时目录
  - 从内存 `registry` 中删除任务（由转移回调处理）

---

## 五、完整流转时序示例

以"用户提交一个 BT 种子 → 分配到从节点 Slave → 下载 → 传输 → 做种完成"为例，串起所有环节：

```
用户 POST /api/v3/file/download
  │
  ├─► CreateDownloadTask 校验权限/容量/批量大小
  ├─► NewRemoteDownloadTask（Phase="", 状态=" "）
  └─► RemoteDownloadQueue.QueueTask
        ├─► transit "" → queued（持久化生成 Task ID）
        └─► scheduler.Queue = append(slice, task)
        └─► registry.Set(id, task)

Worker 调度循环
  │
  ├─► scheduler.Request: 取 slice 尾部 → ResumeTime=0（默认值）< now → 出栈
  ├─► transit queued → processing
  ├─► Do(): Phase=""
  │      ├─► allocateNode → NodePool.Get(NodeCapabilityRemoteDownload) → 选 Slave A
  │      ├─► node.CreateDownloader → slave.NewSlaveDownloader
  │      ├─► SSRF 校验 SrcUri
  │      ├─► 调 slave RPC CreateTask(aria2, seedUrl) → 返回 TaskHandle
  │      ├─► state.Phase = "monitor", state.Handle = {...}
  │      └─► return StatusSuspending
  ├─► transit processing → suspending
  │      ├─► persistTask（写 DB，PrivateState 含 Phase/Handle/NodeID）
  │      └─► q.QueueTask(task) → append 到 slice 尾部
  │
  ├─► 再过一段时间，调度器又取到它（若栈顶没有更晚提交的任务）
  ├─► transit suspending → processing
  ├─► Do(): Phase="monitor"
  │      ├─► 复用 state.NodeID 对应的 Slave A
  │      ├─► d.Info(handle) → 返回 downloading + 进度
  │      ├─► ResumeAfter(Interval)
  │      └─► return StatusSuspending
  │
  ├─► (重复 monitor → suspending → 入队 若干轮)
  │
  ├─► 某轮 d.Info() 返回 seeding
  │      ├─► state.Phase = "transfer"
  │      ├─► ResumeAfter(0)
  │      └─► return StatusSuspending
  │
  ├─► 立即又被取到 → Do(): Phase="transfer", Slave
  │      ├─► 构造 SlaveUploadTaskState（含每文件 src/dst/size/index）
  │      ├─► node.CreateTask("slave_upload", state) → 得到 SlaveUploadTaskID
  │      └─► return StatusSuspending
  │
  ├─► 每 30s 轮询 slaveNode.GetTask(SlaveUploadTaskID)
  │      ├─► 合并进度到 NodeState.progress
  │      ├─► Slave 任务 StatusCompleted + 所有文件 Transferred
  │      ├─► state.Phase = "seeding"（AwaitSeeding）
  │      └─► return StatusSuspending
  │
  └─► 后续 monitor 轮询：下载器状态仍为 seeding
         ├─► 若节点配置 WaitForSeeding=true → 继续挂起轮询
         └─► 某次 d.Info() 返回 completed
               └─► return StatusCompleted → Cleanup → 结束
```

---

## 六、边界场景与主流程衔接

本节补全四个容易和主流程脱节的边界：节点切换与旧句柄失效、种子文件选择、取消操作、落盘前校验。

### 6.1 节点切换与旧下载句柄失效

节点切换的真实行为取决于**是同进程内挂起恢复还是服务重启恢复**（见 3.4 节），两种场景下 `m.d` 是否重建、Handle 是否有效完全不同。

**节点选择流程**（每次 `Do()` 开头，`remote_download.go`）：

```
allocateNode(ctx, dep, &m.state.NodeState, NodeCapabilityRemoteDownload)
    └─► NodePool.Get(ctx, capability, preferred = state.NodeID)
```

`weightedNodePool.Get`（`pkg/cluster/pool.go`）的行为：

1. 若 `state.NodeID > 0`（已分配过节点），先在节点池中按 ID 精确匹配。
2. **如果原节点已被管理员停用/删除**：`Upsert` 时会把它从 `p.nodes[capability]` 列表中移除，此时精确匹配失败，`selected == nil`。
3. Fallback 到 WRR 加权轮询，选一个新节点，回写 `state.NodeID = newNode.ID()`。

下面分两种场景分析：

#### 场景 A：同进程内节点切换（节点 A 被删 → 重新选节点 B）

同进程内 Task 实例是同一个，`m.d != nil`，所以 `CreateDownloader` **不会**被重新调用。

```
节点 A 被管理员停用 → np.Get 精确匹配失败 → fallback 选节点 B
    ├─► state.NodeID 被 allocateNode 更新为 B
    ├─► m.node = B          // 新节点
    ├─► m.d == nil? 否      // 同进程内 m.d 还是 A 的 downloader！
    ├─► switch Phase:
    │      case monitor:
    │          m.d.Info(ctx, handle)
    │              │  // 用 A 的 downloader 去查 A 的 Handle
    │              │  // 碰巧能查到（Handle 确实在 A 上）
    │              └─► 返回正常状态
    └─► 下次 Do():
           allocateNode → preferred = B（新 NodeID）
             └─► 精确匹配 B 成功 → 返回 B
           m.d 还是 A 的 downloader！！  // ← m.node 和 m.d 不一致
           m.d.Info(handle)
               │  // A 已被停用，RPC 失败
               ▼
           返回错误
               ├─► GetTaskStatusTried++（最多 5 次）
               └─► 最终 → error → 队列级重试
```

**关键不一致**：`m.node` 已经是节点 B，但 `m.d` 还是节点 A 的 downloader。第一次节点切换后的 `Info` 调用碰巧能成功（因为 Handle 确实还在 A 上），但第二次迭代就会因为 A 已停用而 RPC 失败。

#### 场景 B：服务重启后节点切换（节点 A 被删 → 重新选节点 B）

服务重启后 Task 是全新实例，`m.d == nil`，会基于新选的节点重新创建 downloader。

```
服务重启 → NewRemoteDownloadTaskFromModel → 全新实例
    ├─► json.Unmarshal → state.NodeID = A, state.Handle = {A 的 GID}
    ├─► allocateNode → preferred = A
    │       └─► A 已被删 → fallback 选 B → state.NodeID = B
    ├─► m.node = B
    ├─► m.d == nil → 是 → node.CreateDownloader → 创建 B 的 downloader
    └─► switch Phase:
            case monitor:
                m.d.Info(ctx, handle)
                    │  // 用 B 的 downloader 去查 A 的 GID
                    ▼
                返回 ErrTaskNotFount
                    │
                    ├─► state.Status != nil（之前在 A 上已拿到过状态）
                    │       → 判定"用户手动取消" → 返回 StatusCanceled
                    │
                    └─► state.Status == nil（还没来得及在 A 上拿到任何状态）
                            → GetTaskStatusTried++，ResumeAfter(Interval) 挂起重试
                            → 连续 5 次后 → 返回 error → 队列级重试（指数退避）
```

#### 共同问题：代码没有"检测到节点切换时自动在新节点重建下载任务"的逻辑

`createDownloadTask` 中有短路：
```go
if m.state.Handle != nil {
    m.state.Phase = RemoteDownloadTaskPhaseMonitor
    return task.StatusSuspending, nil
}
```
即只要 `Handle` 存在就直接进 monitor，**不会检查当前节点和 Handle 所属节点是否一致**，也不会尝试在新节点上重新创建下载任务。

因此无论哪种场景，只要发生节点切换且 Handle 还在，结果都是：
- 有历史状态（`state.Status != nil`）→ 误判为 `StatusCanceled`
- 无历史状态 → 重试 5 次后失败，再走队列级重试（最终还是失败）

slave 模式下 `Info` 返回 `ErrTaskNotFount` 的具体路径（`pkg/downloader/slave/slave.go`）：从节点 RPC 返回 `CodeNotFound` → `fmt.Errorf("%s (%w)", err, downloader.ErrTaskNotFount)`。

### 6.2 种子文件选择（SetFilesToDownload）与主流程衔接

**入口**：`service/explorer/workflows.go` 的 `SetDownloadFilesService.SetDownloadFiles()`。

**关键特征**：这个操作**绕过队列调度**，直接从内存 `TaskRegistry` 取 Task 实例同步调用下载器 API。

```
SetDownloadFiles(c, taskID)
    ├─► registry.Get(taskID)                    // 只能操作内存中仍存在的 Task
    ├─► 校验：owner == 当前用户
    ├─► 校验：Status == Suspending || Processing // 必须在活跃状态
    ├─► 校验：Summary.Phase == "monitor"        // 必须在监控阶段
    └─► downloadTask.SetDownloadTarget(c, files...)
            ├─► 校验：state.Handle != nil
            └─► m.d.SetFilesToDownload(ctx, handle, args...)  // 直接调下载器
```

**与主流程的衔接点**：

- 用户的选择结果**不保存在 Task 的 PrivateState 中**，而是保存在下载器侧（aria2 通过 `changeUri` / `changePosition` RPC 实现）。
- 下次 `monitor()` 调 `m.d.Info()` 时，下载器返回的 `status.Files` 中每个 `TaskFile.Selected` 字段会反映用户的选择。
- `validateFiles`（6.4 节）和 `transfer` 阶段读取的 `status.Files` 中 `Selected=true` 的文件就是用户选中的。
- BT 种子场景：aria2 创建任务后先解析种子文件，此阶段 `status.Files` 可能为空或未完整；解析完成后 `status.Total` 会变化，触发 `monitor` 中的 `Total != status.Total` 分支重新校验。用户需在此时才能看到文件列表并选择。

**限制**：
- 只能在 `monitor` 阶段调用，`transfer`/`seeding` 阶段会返回 `Task not in monitoring loop` 错误。
- Task 从 registry 中删除后（completed/error/canceled 后由状态转移回调删除）无法再操作。

### 6.3 取消操作与主流程衔接

**入口**：`service/explorer/workflows.go` 的 `CancelDownloadTask()`，同样绕过队列调度，直接从 registry 取 Task。

```
CancelDownloadTask(c, taskID)
    ├─► registry.Get(taskID)
    ├─► 校验：owner == 当前用户
    └─► downloadTask.CancelDownload(c)
            ├─► if state.Handle == nil → return nil    // 还没创建下载任务，无需取消
            └─► m.d.Cancel(ctx, handle)                 // 通知下载器取消
```

**与主流程的关系——取消是异步生效的**：

取消 API 只通知下载器停掉任务，**不会立即改变 Task 的生命周期状态**。Task 的状态变迁要等下一次 `Do()` 迭代中 `monitor` 感知到：

```
Cancel API 调用 ──► 下载器取消任务
                        │
                        │  (Task 仍处于 suspending，等 ResumeTime 到期)
                        ▼
下次 Do() → monitor → m.d.Info(handle)
                ├─► 返回 ErrTaskNotFount
                ├─► state.Status != nil（已有历史状态）
                └─► 返回 StatusCanceled
                        │
                        ▼
            transit processing → canceled → Cleanup
```

**延迟问题**：如果 Task 处于 `suspending` 且 `ResumeTime` 在 1 分钟后，取消操作要等 Worker 下次取到该任务（最长 1 分钟 + `taskPullInterval`）才会生效。代码中没有"取消时主动唤醒调度器立即执行"的机制。

**边界 1：Handle 为空时取消（真实缺陷）**

`CancelDownload` 中 `if m.state.Handle == nil { return nil }` 直接返回 nil（API 层显示取消成功），但 Task 的生命周期丝毫不受影响。

```
时序：
  0s:  用户提交下载 → QueueTask → queued
  0.5s: 调度器取到任务 → processing → Do()
  0.6s: Do() Phase="" → allocateNode 选了节点 A
  0.7s: 用户点击 Cancel → CancelDownload → state.Handle == nil → return nil（显示成功）
  0.8s: Do() 继续 → m.d.CreateTask(...) → 返回 Handle，写入 state.Handle
  0.9s: state.Phase = "monitor"，return StatusSuspending
  ...:  下载器正常开始下载
```

**后果**：用户看到取消成功，但实际上 `CreateTask` 已经执行，下载器已开始干活。下次 `Do()` 进入 monitor 后 `d.Info(handle)` 返回正常的 downloading 状态，任务继续执行。代码中没有"置一个取消标记让 createDownloadTask 跳过"的逻辑。

这个路径是真实存在的，因为 `createDownloadTask` 中只检查 `Handle != nil` 才短路，而 `CancelDownload` 在 Handle 为 nil 时什么都不做，两者没有联动。

**边界 2：Task 不在 registry 中**

`registry.Get(taskID)` 找不到 → API 返回 `Task not found`。registry 删除发生在 `processing → completed/error/canceled` 转移回调中，所以已结束的任务无法取消。

**边界 3：Cleanup 兜底**

`processing → completed/error/canceled` 转移回调中会统一调 `task.Cleanup()`，`RemoteDownloadTask.Cleanup()` 会再次调 `m.d.Cancel(ctx, handle)` 兜底，所以即使取消 API 没调用，任务结束时也会通知下载器清理。

### 6.4 落盘前校验与触发时机

**入口**：`remote_download.go` 的 `validateFiles()`，由 `monitor()` 在特定条件下调用。

**触发时机**（`monitor()` 中）：

```go
if m.state.Status == nil || m.state.Status.Total != status.Total {
    // 首次拿到下载器状态，或总大小变化（如种子解析完成、文件列表更新）
    if err := m.validateFiles(ctx, dep, status); err != nil {
        m.state.Status = status   // 即使校验失败也存状态
        return task.StatusError, fmt.Errorf("... (%w)", queue.CriticalErr)
    }
}
```

两种触发条件：
1. **首次拿到状态**（`state.Status == nil`）：下载器刚返回第一个 `TaskStatus`，此时才知道有多少文件、多大。
2. **Total 变化**（`state.Status.Total != status.Total`）：例如 BT 种子刚创建时 aria2 还在解析，文件列表不完整；解析完成后 Total 变大，触发重新校验。

**校验内容**（`validateFiles()` + `dbfs.PreValidateUpload()`）：

```
validateFiles(ctx, dep, status)
    ├─► 解析 Dst URI
    ├─► 过滤 status.Files 中 Selected=true 的文件
    ├─► 校验至少有一个 Selected 文件
    ├─► 构造 PreValidateFile 列表（文件名经 sanitizeFileName 替换非法字符）
    └─► fm.PreValidateUpload(ctx, dstUri, validateArgs...)
            ├─► 获取目标目录 navigator
            ├─► 校验目标是文件夹
            ├─► 校验当前用户是目标目录 owner
            ├─► 获取存储策略
            ├─► 逐文件校验：
            │     ├─► validateFileSize（单文件大小限制）
            │     └─► validateNewFile（扩展名黑白名单、文件名正则）
            └─► validateUserCapacity(total)（用户剩余容量是否够）
```

**与主流程的关系——这是 transfer 之前的"预检"**：

```
monitor 轮询
    ├─► d.Info() 返回 status
    ├─► 首次/Total 变化 → validateFiles → 预检容量、扩展名、命名
    │     ├─► 通过 → 继续
    │     └─► 失败 → CriticalErr → StatusError（不重试）
    │
    ├─► status.State == seeding/completed
    │       → Phase = transfer → ResumeAfter(0)
    │
    └─► transfer 阶段
            ├─► 再次解析 Dst URI（CriticalErr）
            ├─► 读取 status.Files 中 Selected=true 的文件
            ├─► 跳过 Transferred 中已传的文件
            └─► 逐文件上传到目标存储策略
```

校验的目的是**在下载完成后、传输开始前提前拦截**容量不足、扩展名被禁、文件名非法等问题，避免下载完了才发现传不上去浪费带宽。但注意：

- **只校验 Selected 文件**：如果用户没通过 `SetFilesToDownload` 选过文件，下载器默认全部 Selected，则校验所有文件。
- **失败不重试**：因为包装了 `CriticalErr`，队列级重试被跳过。
- **Total 变化时重复校验**：如果下载器返回的 Total 在多次轮询中变化（例如种子分阶段解析），每次变化都会重新校验，避免漏检新增文件。

---

## 七、设计要点总结

1. **"Suspending"不是阻塞，而是立即重新入调度器。** 转移回调里调 `q.QueueTask` 把任务 append 回去，ResumeTime 作为"到时才能取"的门槛——但该门槛只在栈顶生效。
2. **`fifoScheduler` 实际是只看栈顶到期时间的 LIFO 栈，不是堆也不是 FIFO。** `Less` 方法是死代码；晚提交、短 ResumeTime 的任务会被优先取出，早提交的任务可能被压栈。
3. **重试分两层：** 业务层轻量重试（如状态查询，5 次以内，短周期）失败后才交给队列层指数退避；致命错误用 `CriticalErr` 哨兵直接跳过重试。
4. **断点续传以单文件为粒度：** `Transferred map[int]` 记录成功索引，Master 本地传与 Slave RPC 传均遵循此约定，失败重跑不会重复传已落盘的文件。
5. **恢复能力依赖 PrivateState JSON 持久化：** Phase、Handle、NodeID、Transferred、SlaveUploadTaskID 等全量保存在 DB 中，进程重启后 `GetPendingTasks` 恢复即可继续。
6. **恢复执行分同进程挂起和服务重启两条路径：** 同进程内 Task 实例复用、`m.d` 保留；服务重启后新建实例、`m.d` 重建。代码未区分两条路径，导致同进程内节点切换时 `m.node` 和 `m.d` 不一致。
7. **节点切换是未处理的边界缺陷：** 旧 Handle 在新节点上无效，服务重启后会被判 `Canceled` 或重试失败；同进程内切换还可能出现 `m.node` 是新节点但 `m.d` 还是旧节点 downloader 的不一致。`createDownloadTask` 只看 Handle 是否存在，不检查节点匹配性，不会自动重建下载任务。
8. **文件选择、取消操作绕过队列直接操作下载器：** 通过内存 `TaskRegistry` 同步调用，不改变 PrivateState；效果在下一次 `monitor` 轮询中通过 `d.Info()` 返回值感知。取消是异步生效的，延迟取决于 ResumeTime。
9. **取消时 Handle 为空是真实缺陷：** `CancelDownload` 在 `Handle == nil` 时直接返回 nil（显示成功），但后续 `createDownloadTask` 仍会正常创建 Handle 并开始下载，等于取消静默失效。两者没有联动标记。
10. **落盘前校验在 monitor 阶段触发：** 首次拿到状态或 Total 变化时校验容量、扩展名、命名，失败直接 `CriticalErr` 不重试，避免下载完成后才发现无法传输。
