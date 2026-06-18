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

### 3.4 节点分配：加权轮询（WRR）

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

## 六、设计要点总结

1. **"Suspending"不是阻塞，而是立即重新入调度器。** 转移回调里调 `q.QueueTask` 把任务 append 回去，ResumeTime 作为"到时才能取"的门槛——但该门槛只在栈顶生效。
2. **`fifoScheduler` 实际是只看栈顶到期时间的 LIFO 栈，不是堆也不是 FIFO。** `Less` 方法是死代码；晚提交、短 ResumeTime 的任务会被优先取出，早提交的任务可能被压栈。
3. **重试分两层：** 业务层轻量重试（如状态查询，5 次以内，短周期）失败后才交给队列层指数退避；致命错误用 `CriticalErr` 哨兵直接跳过重试。
4. **断点续传以单文件为粒度：** `Transferred map[int]` 记录成功索引，Master 本地传与 Slave RPC 传均遵循此约定，失败重跑不会重复传已落盘的文件。
5. **恢复能力依赖 PrivateState JSON 持久化：** Phase、Handle、NodeID、Transferred、SlaveUploadTaskID 等全量保存在 DB 中，进程重启后 `GetPendingTasks` 恢复即可无缝继续。
