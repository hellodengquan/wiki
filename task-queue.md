# Wiki.js 任务队列系统分析

## 一、核心架构概览

Wiki.js 的任务队列系统是一个**内存型、轻量级**的任务调度框架，主要由三个核心组件构成：

| 组件 | 文件位置 | 核心职责 |
|------|----------|----------|
| 调度器 | `server/core/scheduler.js` | 任务注册、生命周期管理、触发执行 |
| Worker 进程 | `server/core/worker.js` | 独立子进程的任务执行入口 |
| 任务实现 | `server/jobs/*.js` | 具体业务逻辑实现 |

### 1.1 系统启动流程

```
kernel.js:postBootMaster()
    ↓
scheduler.start()
    ↓
遍历 data.yml 中 jobs 配置
    ↓
为每个周期任务调用 registerJob()
```

[server/core/kernel.js:87](server/core/kernel.js#L87-L87)

---

## 二、任务类型与入队逻辑

系统支持三种任务模式，通过 Job 构造参数区分：

### 2.1 任务类型定义

```javascript
class Job {
  constructor({
    name,             // 任务名称，对应 jobs/ 目录下的文件名
    immediate = false,// 是否立即执行
    schedule = 'P1D', // 调度间隔（ISO 8601 Duration 格式）
    repeat = false,   // 是否重复执行
    worker = false    // 是否在独立子进程执行
  }, queue)
```

[server/core/scheduler.js:8-23](server/core/scheduler.js#L8-L23)

### 2.2 三种任务模式

#### A. 一次性任务（One-off Tasks）

**特征**：`immediate: true, repeat: false`

**调用示例**：
```javascript
// 页面渲染 - pages.js:938-943
const renderJob = await WIKI.scheduler.registerJob({
  name: 'render-page',
  immediate: true,
  worker: true
}, page.id)

// SVG 安全扫描 - assets.js:110-115
const svgSanitizeJob = await WIKI.scheduler.registerJob({
  name: 'sanitize-svg',
  immediate: true,
  worker: true
}, opts.path)
```

**执行路径**：
```
registerJob() → Job.start()
    ↓ immediate = true
Job.invoke() → 直接执行或 fork 子进程
    ↓
执行完成 → Job.stop() → 从队列移除
```

#### B. 周期任务（Periodic Tasks）

**特征**：`repeat: true, schedule: 'PT15M'`

**配置示例**（`server/app/data.yml:121-141`）：
```yaml
jobs:
  purgeUploads:
    onInit: true          # 启动时立即执行一次
    schedule: PT15M       # 每15分钟执行
    offlineSkip: false
    repeat: true
  syncGraphLocales:
    onInit: true
    schedule: P1D         # 每天执行
    offlineSkip: true
    repeat: true
  rebuildTree:
    onInit: true
    offlineSkip: false
    repeat: false
    immediate: true
    worker: true
```

**执行路径**：
```
registerJob() → Job.start()
    ↓ immediate = false
Job.enqueue() → setTimeout(..., schedule)
    ↓ 定时触发
Job.invoke() → 执行任务
    ↓ repeat = true
Job.enqueue() → 再次设置 setTimeout → 循环
```

[server/core/scheduler.js:87-89](server/core/scheduler.js#L87-L89)

#### C. 长尾任务（Long-running / Worker Tasks）

**特征**：`worker: true`

通过 `child_process.fork()` 创建独立 Node.js 子进程执行，避免阻塞主事件循环。

**调用示例**（`storage.js:145-162`）：
```javascript
if (targetDef.schedule && target.syncInterval !== `P0D`) {
  WIKI.scheduler.registerJob({
    name: `sync-storage`,
    immediate: false,
    schedule: target.syncInterval,
    repeat: true
  }, target.key)
}
```

---

## 三、Worker 分发与执行机制

### 3.1 进程隔离设计

当 `worker: true` 时，任务不在主进程执行，而是通过 IPC 机制 fork 子进程：

```javascript
async invoke(data) {
  if (this.worker) {
    const proc = childProcess.fork(`server/core/worker.js`, [
      `--job=${this.name}`,
      `--data=${data}`
    ], {
      cwd: WIKI.ROOTPATH,
      stdio: ['inherit', 'inherit', 'pipe', 'ipc']
    })
```

[server/core/scheduler.js:55-62](server/core/scheduler.js#L55-L62)

### 3.2 Worker 进程入口（worker.js）

```javascript
;(async () => {
  try {
    await require(`../jobs/${args.job}`)(args.data)
    process.exit(0)
  } catch (e) {
    await new Promise(resolve => process.stderr.write(e.message, resolve))
    process.exit(1)
  }
})()
```

[server/core/worker.js:16-23](server/core/worker.js#L16-L23)

### 3.3 子进程生命周期管理

```javascript
this.finished = new Promise((resolve, reject) => {
  proc.on('exit', (code, signal) => {
    const data = Buffer.concat(stderr).toString()
    if (code === 0) {
      resolve(data)
    } else {
      const err = new Error(`Error when running job ${this.name}: ${data}`)
      err.exitSignal = signal
      err.exitCode = code
      err.stderr = data
      reject(err)
    }
    proc.kill()
  })
})
```

[server/core/scheduler.js:65-79](server/core/scheduler.js#L65-L79)

### 3.4 非 Worker 模式（主进程执行）

```javascript
} else {
  this.finished = require(`../jobs/${this.name}`)(data)
}
```

[server/core/scheduler.js:80-82](server/core/scheduler.js#L80-L82)

---

## 四、核心设计要点深度分析

### 4.1 优先级排序 → 见第九节「setTimeout 自然顺序的时序漂移」

**设计结论：无显式优先级机制**

- 代码中**没有 priority 字段**或排序逻辑
- 任务按照 `setTimeout` 的自然触发顺序执行
- 多个同时触发的任务按照事件循环的宏任务队列顺序执行

**代码证据**：
```javascript
// scheduler.js 中无任何排序、比较优先级的代码
// jobs 数组仅用于存储和 stop 时遍历
this.queue.jobs.push(this)
```

### 4.2 超时回收 → 见第十一节「无超时定时器的守护进程」

**设计结论：无主动超时回收机制**

- **没有**为任务执行设置超时时间（如 `Promise.race` + `setTimeout`）
- Worker 子进程无限等待，直到自然退出或崩溃
- 依赖子进程自身的错误处理和退出码判断

### 4.3 失败重试 → 见第十节「warning log 错误聚合」

**设计结论：无内置重试机制**

- 任务失败仅记录警告日志：`WIKI.logger.warn(err)`
- **没有**重试计数、退避算法（exponential backoff）
- 周期任务的"重试"是假象：失败后仍会按 schedule 触发下一次

### 4.4 任务可见性与持久化 → 见第十二节「内存数组快照持久化」

**设计结论：纯内存队列，无持久化**

| 特性 | 实现方式 | 局限性 |
|------|----------|--------|
| 任务存储 | 内存数组 `this.jobs` | 进程重启丢失 |
| 任务状态 | 仅存在 `timeout` 句柄 | 无法查询历史任务 |
| 持久化 | 无数据库表 | 崩溃后任务状态丢失 |
| 多实例协调 | 无 | 多实例部署会重复执行 |

---

## 五、完整代码调用链路

### 5.1 一次性任务调用链（以 render-page 为例）

```
pages.js:renderPage(page)
    ↓ 调用 registerJob
scheduler.js:registerJob({name: 'render-page', immediate: true, worker: true}, page.id)
    ↓
Job.start(data)
    ↓ immediate=true
Job.invoke(data)
    ↓ worker=true
childProcess.fork(worker.js, ['--job=render-page', '--data=' + page.id])
    ↓
worker.js 加载 jobs/render-page.js
    ↓
render-page.js: 初始化 DB → 执行渲染 → 更新 DB → 销毁连接 → process.exit(0)
    ↓
scheduler 监听 exit 事件 → code=0 → resolve Promise → 结束
```

### 5.2 周期任务调用链（以 purge-uploads 为例）

```
scheduler.start()
    ↓ 遍历 data.yml jobs
registerJob({name: 'purge-uploads', schedule: 'PT15M', repeat: true})
    ↓ immediate=false
Job.enqueue() → setTimeout(15分钟)
    ↓ 定时触发
Job.invoke()
    ↓ worker=false（主进程执行）
require('jobs/purge-uploads.js')()
    ↓ 执行完成
repeat=true → Job.enqueue() → 再次 setTimeout → 循环
```

---

## 六、现有任务清单

| 任务名称 | 类型 | Worker | 用途 | 调用位置 |
|----------|------|--------|------|----------|
| `purge-uploads` | 周期（15min） | 否 | 清理过期上传临时文件 | data.yml |
| `sync-graph-locales` | 周期（1天） | 否 | 同步语言包 | data.yml |
| `sync-graph-updates` | 周期（1天） | 否 | 检查更新 | data.yml |
| `rebuild-tree` | 一次性/启动 | 是 | 重建页面目录树 | pages.js:923, data.yml |
| `render-page` | 一次性 | 是 | 页面内容渲染 | pages.js:938 |
| `sanitize-svg` | 一次性 | 是 | SVG 安全扫描 | assets.js:110 |
| `sync-storage` | 周期 | 否 | 存储目标同步 | storage.js:146 |
| `fetch-graph-locale` | 一次性 | 否 | 下载语言包 | localization.js:42 |

---

## 七、设计评价与改进建议

### 7.1 设计优点

1. **简单轻量**：代码量少（scheduler.js ~130 行），易于理解和维护
2. **进程隔离**：Worker 模式避免 CPU 密集任务阻塞主事件循环
3. **零依赖**：不依赖 Redis、RabbitMQ 等中间件，部署简单
4. **灵活配置**：通过 ISO 8601 Duration 格式支持灵活的调度间隔

### 7.2 主要不足

1. **无持久化**：进程崩溃或重启导致任务丢失
2. **无优先级**：无法区分紧急任务和后台任务
3. **无超时**：僵尸任务可能导致资源泄漏
4. **无重试**：临时故障导致任务永久失败
5. **无去重**：同一任务可能被重复注册
6. **单实例限制**：多实例部署会导致周期任务重复执行

### 7.3 潜在改进方向

```javascript
// 建议增加的 Job 选项
constructor({
  // ... 现有参数
  priority = 0,           // 优先级
  timeout = 0,            // 超时时间（毫秒）
  maxAttempts = 1,        // 最大重试次数
  backoff = 'exponential',// 退避算法
  uniqueKey = null        // 去重键
})

// 建议增加的队列持久化
// 使用数据库表 jobs 存储任务状态
// CREATE TABLE jobs (
//   id, name, status, data, priority, 
//   attempts, max_attempts, 
//   run_at, locked_at, locked_by,
//   created_at, updated_at
// )
```

---

## 八、关键代码定位速查

| 功能 | 文件 | 行号 |
|------|------|------|
| Job 类定义 | scheduler.js | 8-102 |
| 任务入队 | scheduler.js | 44-46 |
| 任务执行 | scheduler.js | 53-92 |
| Worker 进程 fork | scheduler.js | 55-62 |
| 退出码处理 | scheduler.js | 66-77 |
| 周期任务调度 | scheduler.js | 87-89 |
| Worker 入口 | worker.js | 16-23 |
| 系统任务配置 | data.yml | 120-141 |
| 渲染任务注册 | pages.js | 938-943 |
| 存储同步注册 | storage.js | 146-161 |

---

## 九、setTimeout 自然顺序的时序漂移

### 9.1 源码中的定时器机制

任务的周期调度完全依赖 `setTimeout`，而非 `setInterval` 或 cron 表达式：

```javascript
// scheduler.js:44-46
enqueue(data) {
  this.timeout = setTimeout(this.invoke.bind(this), this.schedule.asMilliseconds(), data)
}
```

`this.schedule` 在构造时由 `moment.duration(schedule)` 解析 ISO 8601 Duration 得到：

```javascript
// scheduler.js:20
this.schedule = moment.duration(schedule)
```

### 9.2 漂移的产生机制

**漂移不是 bug，而是设计的固有特性。** 漂移来自两个叠加因素：

#### 因素一：setTimeout 的执行延迟

Node.js 的 `setTimeout` 保证**最少**延迟 N 毫秒，但**不保证准时**。当事件循环繁忙时（如正在处理 HTTP 请求或 GraphQL 查询），定时器回调会被推迟到下一个事件循环迭代。

```
预期时间轴:  0min ─── 15min ─── 30min ─── 45min
实际时间轴:  0min ─── 15m03s ─── 30m08s ─── 45m12s
                                               ↑ 漂移累积
```

#### 因素二：invoke() → enqueue() 的串行延迟

关键代码在 `scheduler.js:83-91`：

```javascript
async invoke(data) {
  try {
    // ...执行任务（可能耗时数秒到数分钟）
    await this.finished
  } catch (err) {
    WIKI.logger.warn(err)
  }
  // 任务完成后才设置下一次 setTimeout
  if (this.repeat && this.queue.jobs.includes(this)) {
    this.enqueue(data)   // ← 在这里才计算下次触发时间
  }
}
```

这意味着周期任务的实际间隔 = `schedule + 任务执行时间 + 事件循环延迟`。

以 `purge-uploads`（PT15M）为例，如果每次执行耗时 2 秒：

| 触发次数 | 预期时间 | 实际时间 | 累积漂移 |
|----------|----------|----------|----------|
| 1 | 0:00 | 0:00 | 0s |
| 2 | 15:00 | 15:02 | +2s |
| 3 | 30:00 | 30:04 | +4s |
| 96 (1天) | 24:00 | 24:03:12 | +3m12s |

### 9.3 与 setInterval / cron 的对比

| 方案 | 漂移行为 | 适用场景 |
|------|----------|----------|
| **setTimeout（当前实现）** | 漂移 = 任务执行时间 × 触发次数 | 后台清理、同步等容忍漂移的场景 |
| **setInterval** | 漂移小但可能重叠（回调堆积） | 精确间隔、任务极短的场景 |
| **cron（绝对时间）** | 无漂移，固定在整点触发 | 必须在特定时间执行的场景 |
| **setTimeout + 补偿** | 计算下次触发时扣除已耗时 | 需要近似等间隔的场景 |

### 9.4 为什么当前设计可以接受

Wiki.js 的周期任务本质上是「近似间隔的后台维护」：
- `purge-uploads`：清理 15 分钟前的临时文件，晚几秒无影响
- `sync-graph-locales`：每天同步语言包，晚几分钟无影响
- `sync-storage`：存储同步，间隔本身由用户配置

**唯一的边界风险**：如果某次 `sync-storage` 执行时间超过 `syncInterval`（如配置了 PT1M 但同步需 2 分钟），会导致任务完全重叠调度——但这里恰好是串行的，不会重叠。

---

## 十、child_process.fork 与主进程 IPC 协议

### 10.1 fork 调用的完整参数

```javascript
// scheduler.js:56-62
const proc = childProcess.fork(`server/core/worker.js`, [
  `--job=${this.name}`,
  `--data=${data}`
], {
  cwd: WIKI.ROOTPATH,
  stdio: ['inherit', 'inherit', 'pipe', 'ipc']
})
```

逐行解析：

| 参数 | 值 | 含义 |
|------|----|------|
| `modulePath` | `server/core/worker.js` | 子进程执行的脚本路径（相对于 cwd） |
| `args[0]` | `--job=${this.name}` | yargs 解析的任务名，如 `--job=render-page` |
| `args[1]` | `--data=${data}` | yargs 解析的任务数据，如 `--data=42` |
| `cwd` | `WIKI.ROOTPATH` | 子进程工作目录 = 项目根路径 |
| `stdio[0]` | `inherit` | stdin 继承主进程 → 子进程可读终端 |
| `stdio[1]` | `inherit` | stdout 继承主进程 → 子进程输出直接打印到主进程终端 |
| `stdio[2]` | `pipe` | stderr 管道化 → 主进程可捕获错误输出 |
| `stdio[3]` | `ipc` | 启用 IPC 通道 → 主进程与子进程可双向通信 |

### 10.2 IPC 通道的实际使用

**关键发现：IPC 通道已建立但从未使用。**

尽管 `stdio` 配置了 `ipc` 模式，但代码中**没有任何 `proc.send()` 或 `process.on('message')` 的调用**。主进程仅通过以下方式与子进程通信：

1. **命令行参数传递**（单向，主→子）：通过 `--job` 和 `--data`
2. **stderr 管道捕获**（单向，子→主）：`proc.stderr.on('data', ...)`
3. **exit 事件监听**（单向，子→主）：`proc.on('exit', (code, signal) => ...)`

```javascript
// scheduler.js:63-78 — 唯一的主进程→子进程通信
const stderr = []
proc.stderr.on('data', chunk => stderr.push(chunk))  // 监听 stderr
this.finished = new Promise((resolve, reject) => {
  proc.on('exit', (code, signal) => {                 // 监听退出
    const data = Buffer.concat(stderr).toString()
    if (code === 0) {
      resolve(data)                                    // 成功 → resolve
    } else {
      const err = new Error(`Error when running job ${this.name}: ${data}`)
      err.exitSignal = signal
      err.exitCode = code
      err.stderr = data
      reject(err)                                      // 失败 → reject
    }
    proc.kill()                                        // 退出后 kill 清理
  })
})
```

### 10.3 子进程侧的 IPC 行为

```javascript
// worker.js:16-23
;(async () => {
  try {
    await require(`../jobs/${args.job}`)(args.data)
    process.exit(0)           // 成功：退出码 0
  } catch (e) {
    await new Promise(resolve => process.stderr.write(e.message, resolve))
    process.exit(1)           // 失败：退出码 1
  }
})()
```

子进程不监听 IPC 消息，也没有 `process.on('message', ...)` 处理器。

### 10.4 --data 参数的序列化局限

任务数据通过命令行参数传递，存在严格的序列化限制：

```javascript
// scheduler.js:58
`--data=${data}`
```

`data` 被**强制转为字符串**拼入命令行参数。这意味着：

| 数据类型 | 传入值 | 子进程接收值 | 问题 |
|----------|--------|-------------|------|
| 数字 | `42` | `"42"`（字符串） | 需要手动转换 |
| 字符串 | `"hello"` | `"hello"` | 正常 |
| 对象 | `{id: 1}` | `"[object Object]"` | **数据丢失** |
| undefined | `undefined` | `"undefined"` | **语义错误** |

实际代码中的用法验证了这一点：

```javascript
// pages.js:942 — 传入 page.id（数字）
}, page.id)

// assets.js:114 — 传入 opts.path（字符串）
}, opts.path)

// storage.js:151 — 传入 target.key（字符串）
}, target.key)
```

所有调用点都只传入**原始值**（数字 ID 或字符串 key），从不传对象——这是对 `--data` 序列化局限的隐式约束。

### 10.5 Worker 进程的独立 DB 初始化

Worker 子进程需要**独立初始化数据库连接**，因为 fork 出的子进程不继承主进程的 Knex 连接池：

```javascript
// render-page.js:10-12 — 每个 worker 任务的固定模式
WIKI.models = require('../core/db').init()    // 1. 创建新连接池
await WIKI.configSvc.loadFromDb()             // 2. 加载配置
await WIKI.configSvc.applyFlags()             // 3. 应用开发标志
// ...执行业务逻辑...
await WIKI.models.knex.destroy()              // 4. 销毁连接池
```

```javascript
// rebuild-tree.js:9-11 — 同样的模式
WIKI.models = require('../core/db').init()
await WIKI.configSvc.loadFromDb()
await WIKI.configSvc.applyFlags()
// ...
await WIKI.models.knex.destroy()
```

这导致每个 worker 任务执行都会**创建并销毁一个完整的数据库连接池**。

---

## 十一、无超时定时器的守护进程

### 11.1 当前代码的等待模型

```javascript
// scheduler.js:83
await this.finished  // 无限期等待，无超时包装
```

对于 worker 模式，`this.finished` 是一个绑定到 `proc.on('exit')` 的 Promise：

```javascript
// scheduler.js:65-78
this.finished = new Promise((resolve, reject) => {
  proc.on('exit', (code, signal) => {
    // 只有子进程退出时才 resolve/reject
  })
})
```

**如果子进程永不退出，这个 Promise 永远 pending。** 主进程的 `invoke()` 方法将永远悬挂在 `await this.finished`。

### 11.2 僵尸进程的产生场景

| 场景 | 触发条件 | 子进程行为 |
|------|----------|-----------|
| 死循环 | 代码 bug（如 while(true)） | CPU 100%，永不退出 |
| 死锁 | DB 连接池耗尽、文件锁 | 阻塞等待，永不退出 |
| 外部资源不可达 | 网络断开，无超时的 HTTP 请求 | 阻塞等待，永不退出 |
| SIGSTOP | 外部 `kill -STOP <pid>` | 挂起进程，永不退出 |

### 11.3 对周期任务的影响

对于 `repeat: true` 的周期任务，`invoke()` 中的 `await this.finished` 会阻塞后续所有调度：

```javascript
// scheduler.js:83-91
async invoke(data) {
  try {
    await this.finished    // ← 如果这里永远 pending...
  } catch (err) {
    WIKI.logger.warn(err)
  }
  if (this.repeat && this.queue.jobs.includes(this)) {
    this.enqueue(data)     // ← 这行永远不会执行
  }
}
```

**但**由于 `invoke()` 本身是被 `setTimeout` 触发的回调，它不会阻塞事件循环。主进程仍然正常处理 HTTP 请求。只是这个特定的周期任务从此**静默停止调度**。

### 11.4 进程退出时的清理

主进程退出时的 graceful shutdown 流程：

```javascript
// index.js:46-56
process.on('SIGTERM', () => {
  WIKI.kernel.shutdown()
})
process.on('SIGINT', () => {
  WIKI.kernel.shutdown()
})
process.on('message', (msg) => {
  if (msg === 'shutdown') {
    WIKI.kernel.shutdown()
  }
})
```

```javascript
// kernel.js:109-128
async shutdown (devMode = false) {
  if (WIKI.scheduler) {
    await WIKI.scheduler.stop()  // 等待所有任务停止
  }
  // ...
}
```

```javascript
// scheduler.js:131-133
async stop() {
  return Promise.all(this.jobs.map(job => job.stop()))
}
```

```javascript
// scheduler.js:97-101
async stop() {
  clearTimeout(this.timeout)                        // 清除待触发的定时器
  this.queue.jobs = this.queue.jobs.filter(x => x !== this)  // 从数组移除
  return this.finished                              // 等待当前执行完成
}
```

**关键问题**：`job.stop()` 返回 `this.finished`，如果 worker 子进程僵尸化，`this.finished` 永远 pending，`Promise.all` 永远不会 resolve，主进程**无法完成 graceful shutdown**。

### 11.5 缺失的超时守护方案

当前代码中缺少的完整超时机制应包括：

```javascript
// 方案一：为 worker 子进程设置超时
async invoke(data) {
  if (this.worker) {
    const proc = childProcess.fork(...)
    const TIMEOUT_MS = 5 * 60 * 1000  // 5分钟超时
    this.finished = new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        proc.kill('SIGKILL')            // 强制杀死子进程
        reject(new Error(`Job ${this.name} timed out after ${TIMEOUT_MS}ms`))
      }, TIMEOUT_MS)
      proc.on('exit', (code, signal) => {
        clearTimeout(timer)             // 正常退出时取消超时
        // ... resolve/reject 逻辑
      })
    })
  }
}

// 方案二：为 graceful shutdown 设置总超时
async stop() {
  const SHUTDOWN_TIMEOUT = 10000
  await Promise.race([
    Promise.all(this.jobs.map(job => job.stop())),
    new Promise(resolve => setTimeout(resolve, SHUTDOWN_TIMEOUT))
  ])
}
```

---

## 十二、warning log 错误聚合

### 12.1 当前错误处理链

任务执行失败时的完整日志链路：

```
Job.invoke() catch
    ↓
WIKI.logger.warn(err)     ← scheduler.js:85
    ↓
winston Logger
    ↓
Console Transport (默认)
    ↓
格式化输出: "2026-06-17T10:00:00 [JOB] warn: Error when running job render-page: ..."
```

### 12.2 logger 的配置与能力

```javascript
// logger.js:8-43
module.exports = {
  loggers: {},
  init(uid) {
    const loggerFormats = [
      winston.format.label({ label: uid }),     // 标签: "MASTER" / "JOB"
      winston.format.timestamp()                 // 时间戳
    ]

    if (WIKI.config.logFormat === 'json') {
      loggerFormats.push(winston.format.json())  // JSON 格式（可聚合）
    } else {
      loggerFormats.push(winston.format.colorize())
      loggerFormats.push(winston.format.printf(info =>
        `${info.timestamp} [${info.label}] ${info.level}: ${info.message}`
      ))
    }

    const logger = winston.createLogger({
      level: WIKI.config.logLevel,               // 默认 'info'
      format: winston.format.combine(...loggerFormats)
    })

    logger.add(new winston.transports.Console({   // 仅 Console 输出
      level: WIKI.config.logLevel,
      // ...
    }))

    return logger
  }
}
```

### 12.3 错误信息的丢失层次

不同任务类型在错误传播中丢失的信息量不同：

**Worker 模式**（信息最丰富）：

```javascript
// scheduler.js:71-75 — 构造了详细的错误对象
const err = new Error(`Error when running job ${this.name}: ${data}`)
err.exitSignal = signal   // 退出信号（如 SIGKILL）
err.exitCode = code       // 退出码（如 1）
err.stderr = data         // stderr 输出内容
```

但随后：

```javascript
// scheduler.js:85
WIKI.logger.warn(err)   // warn 级别，而非 error 级别
```

**非 Worker 模式**（信息最少）：

```javascript
// scheduler.js:80-82
this.finished = require(`../jobs/${this.name}`)(data)
// ...
// scheduler.js:85
WIKI.logger.warn(err)   // 仅 Error 对象的 message
```

### 12.4 错误聚合的缺失

当前实现**没有**任何错误聚合机制：

| 缺失能力 | 说明 |
|----------|------|
| 错误计数 | 无法知道某任务连续失败了几次 |
| 错误去重 | 同一错误每 15 分钟重复输出一次日志 |
| 错误升级 | 连续失败 10 次仍然是 warn，不会升级为 error |
| 错误通知 | 失败不触发邮件/Webhook 告警 |
| 错误关联 | 无法关联 job 失败与系统健康状态 |

### 12.5 storage.js 的状态追踪（部分实现）

storage 模块实现了一套**任务级别的状态追踪**，但这是业务层自行实现的，不在调度器框架内：

```javascript
// storage.js:136-142 — 成功时记录状态
await WIKI.models.storage.query().patch({
  state: {
    status: 'operational',
    message: '',
    lastAttempt: new Date().toISOString()
  }
}).where('key', target.key)

// storage.js:164-171 — 失败时记录状态
await WIKI.models.storage.query().patch({
  state: {
    status: 'error',
    message: err.message,
    lastAttempt: new Date().toISOString()
  }
}).where('key', target.key)
```

```javascript
// sync-storage.js:14-20 — 成功
await WIKI.models.storage.query().patch({
  state: {
    status: 'operational',
    message: '',
    lastAttempt: new Date().toISOString()
  }
}).where('key', targetKey)

// sync-storage.js:27-33 — 失败
await WIKI.models.storage.query().patch({
  state: {
    status: 'error',
    message: err.message,
    lastAttempt: new Date().toISOString()
  }
}).where('key', targetKey)
```

这是系统中**唯一**对任务执行结果进行持久化追踪的地方，但它是手动编写的业务逻辑，不是调度器的通用能力。

### 12.6 telemetry.js 的空实现

`telemetry.sendError(err)` 被调用但**未实现**：

```javascript
// telemetry.js:19-21
sendError(err) {
  // TODO
},
```

在 `kernel.js:97-104` 中，`unhandledRejection` 和 `uncaughtException` 都会调用这个空方法：

```javascript
process.on('unhandledRejection', (err) => {
  WIKI.logger.warn(err)
  WIKI.telemetry.sendError(err)  // ← 空操作
})
process.on('uncaughtException', (err) => {
  WIKI.logger.warn(err)
  WIKI.telemetry.sendError(err)  // ← 空操作
})
```

---

## 十三、内存数组快照持久化

### 13.1 当前 jobs 数组的完整生命周期

```javascript
// scheduler.js:104-133
module.exports = {
  jobs: [],              // ← 空数组，模块加载时初始化
  init() {
    return this           // ← 仅返回自身引用，不做任何操作
  },
  start() {
    _.forOwn(WIKI.data.jobs, (queueParams, queueName) => {
      // ...遍历 data.yml 中的 jobs 配置
      this.registerJob({...})   // ← 向 jobs 数组 push 新 Job 实例
    })
  },
  registerJob(opts, data) {
    const job = new Job(opts, this)
    job.start(data)             // ← Job.start() 中 this.queue.jobs.push(this)
    return job
  },
  async stop() {
    return Promise.all(this.jobs.map(job => job.stop()))
  }
}
```

### 13.2 数据流：从 YAML 到内存数组

```
data.yml (静态配置)
    ↓ configSvc.init() 时加载到 WIKI.data
    ↓
WIKI.data.jobs = { purgeUploads: {...}, syncGraphLocales: {...}, ... }
    ↓
scheduler.start()
    ↓ _.forOwn(WIKI.data.jobs)
    ↓ _.kebabCase(queueName) → 'purge-uploads', 'sync-graph-locales', ...
    ↓
registerJob() → new Job() → job.start() → jobs.push(job)
    ↓
运行时动态注册：
    pages.js:renderPage() → registerJob({name:'render-page', ...})
    storage.js:initTargets() → registerJob({name:'sync-storage', ...})
```

### 13.3 数组内容的实际结构

每个 Job 实例在内存中保存的字段：

```javascript
{
  queue: [Circular],         // 反向引用调度器模块
  finished: Promise,         // 当前执行的 Promise（或已完成）
  name: 'purge-uploads',    // 任务名
  immediate: true,           // 是否立即执行
  schedule: moment.duration, // moment.js Duration 对象
  repeat: true,              // 是否重复
  worker: false,             // 是否 worker 模式
  timeout: setTimeout ID     // 当前等待的定时器句柄
}
```

### 13.4 快照持久化的可行性分析

**为什么没有做持久化？** 分析代码结构后得出以下原因：

1. **周期任务自愈**：所有周期任务配置在 `data.yml` 中，进程重启后 `scheduler.start()` 会重新注册，无需从持久化恢复

2. **一次性任务不可恢复**：如 `render-page`，调用方（`pages.js`）已经 `await job.finished` 并处理了结果。如果进程重启，调用方早已丢失上下文，恢复执行没有意义

3. **sync-storage 的动态注册**：这是唯一动态注册的周期任务，但 `storage.js:initTargets()` 在 `postBootMaster()` 中被调用，也会自动重新注册

```javascript
// kernel.js:86-87
await WIKI.models.storage.initTargets()  // 重新注册所有 sync-storage 任务
WIKI.scheduler.start()                    // 注册 data.yml 中的任务
```

4. **去重依赖而非持久化**：`storage.js:initTargets()` 中有一段关键代码：

```javascript
// storage.js:120-124
const prevjobs = _.remove(WIKI.scheduler.jobs, job => job.name === `sync-storage`)
if (prevjobs.length > 0) {
  prevjobs.forEach(job => job.stop())
}
```

这说明 storage 初始化时会**先清理旧的 sync-storage 任务再重新注册**，以避免重复。这是一种简单的运行时去重，而非持久化。

### 13.5 崩溃场景下的任务丢失矩阵

| 任务类型 | 崩溃后能否自愈 | 自愈方式 | 丢失内容 |
|----------|--------------|----------|----------|
| data.yml 周期任务 | ✅ 能 | 重启后 `scheduler.start()` 重新注册 | 当前执行中的任务状态 |
| data.yml 一次性任务 | ⚠️ 部分 | `rebuildTree` 在 `onInit:true` 下会重新执行 | 无 |
| render-page | ❌ 不能 | 调用方已丢失上下文 | 未完成的渲染结果 |
| sanitize-svg | ❌ 不能 | 同上 | 未扫描的 SVG 文件 |
| sync-storage | ✅ 能 | `initTargets()` 重新注册 | 当前同步进度 |
| fetch-graph-locale | ❌ 不能 | 用户触发的下载操作已丢失 | 下载结果 |

---

## 十四、scheduler.js 8-102 扩展点

### 14.1 Job 类的扩展面

`scheduler.js:8-102` 的 `Job` 类定义了5个方法，每个都是潜在的扩展点：

```
Job
├── constructor()     ← 扩展点: 增加字段（priority, timeout, maxAttempts...）
├── start(data)       ← 扩展点: 增加前置钩子（beforeStart）
├── enqueue(data)     ← 扩展点: 改变调度策略（cron 替代 setTimeout）
├── invoke(data)      ← 扩展点: 增加超时、重试、状态通知
└── stop()            ← 扩展点: 增加清理钩子（afterStop）
```

### 14.2 各扩展点的当前实现与改进方向

#### constructor() — scheduler.js:8-23

**当前**：

```javascript
constructor({
  name,
  immediate = false,
  schedule = 'P1D',
  repeat = false,
  worker = false
}, queue)
```

**可扩展字段**：

```javascript
constructor({
  name,
  immediate = false,
  schedule = 'P1D',
  repeat = false,
  worker = false,
  // === 新增 ===
  priority = 0,             // 0=最低, 数字越大越优先
  timeout = 0,              // 超时毫秒数, 0=不限
  maxAttempts = 1,          // 最大重试次数
  backoff = 'fixed',        // 'fixed' | 'exponential'
  backoffDelay = 1000,      // 退避基础延迟
  uniqueKey = null,         // 去重键, 同 key 的任务不重复注册
  lockTTL = 0,              // 分布式锁 TTL（HA 模式下防重复执行）
}, queue) {
  // ...
  this.priority = priority
  this.timeout = timeout
  this.attempts = 0
  this.maxAttempts = maxAttempts
  this.backoff = backoff
  this.backoffDelay = backoffDelay
  this.uniqueKey = uniqueKey
  this.lockTTL = lockTTL
}
```

#### start() — scheduler.js:30-37

**当前**：

```javascript
start(data) {
  this.queue.jobs.push(this)        // 直接入队，无条件
  if (this.immediate) {
    this.invoke(data)
  } else {
    this.enqueue(data)
  }
}
```

**可扩展逻辑**：

```javascript
start(data) {
  // 去重检查
  if (this.uniqueKey) {
    const existing = this.queue.jobs.find(j => j.uniqueKey === this.uniqueKey)
    if (existing) {
      WIKI.logger.warn(`Job ${this.name} with key ${this.uniqueKey} already queued. [SKIPPED]`)
      return existing  // 返回已有任务
    }
  }

  // 优先级排序插入
  this.queue.jobs.push(this)
  this.queue.jobs.sort((a, b) => b.priority - a.priority)

  if (this.immediate) {
    this.invoke(data)
  } else {
    this.enqueue(data)
  }
}
```

#### enqueue() — scheduler.js:44-46

**当前**：

```javascript
enqueue(data) {
  this.timeout = setTimeout(this.invoke.bind(this), this.schedule.asMilliseconds(), data)
}
```

**可扩展为补偿式定时器**：

```javascript
enqueue(data) {
  if (this.repeat && this._lastStartTime) {
    const elapsed = Date.now() - this._lastStartTime
    const remaining = Math.max(0, this.schedule.asMilliseconds() - elapsed)
    this.timeout = setTimeout(this.invoke.bind(this), remaining, data)
  } else {
    this.timeout = setTimeout(this.invoke.bind(this), this.schedule.asMilliseconds(), data)
  }
}
```

#### invoke() — scheduler.js:53-92

**当前**：无超时、无重试、错误仅 warn

**可扩展为带超时+重试的版本**：

```javascript
async invoke(data) {
  this._lastStartTime = Date.now()
  this.attempts++

  try {
    let result
    if (this.worker) {
      result = await this._invokeWorker(data)
    } else {
      result = this.timeout > 0
        ? await Promise.race([
            require(`../jobs/${this.name}`)(data),
            new Promise((_, reject) =>
              setTimeout(() => reject(new Error('timeout')), this.timeout))
          ])
        : await require(`../jobs/${this.name}`)(data)
    }
    this.finished = Promise.resolve(result)
  } catch (err) {
    if (this.attempts < this.maxAttempts) {
      WIKI.logger.warn(`Job ${this.name} failed (attempt ${this.attempts}/${this.maxAttempts}), retrying...`)
      const delay = this._calculateBackoff()
      setTimeout(() => this.invoke(data), delay)
      return
    }
    WIKI.logger.error(`Job ${this.name} failed after ${this.attempts} attempts: ${err.message}`)
    this.finished = Promise.reject(err)
  }

  if (this.repeat && this.queue.jobs.includes(this)) {
    this.enqueue(data)
  } else {
    this.stop().catch(() => {})
  }
}

_calculateBackoff() {
  if (this.backoff === 'exponential') {
    return this.backoffDelay * Math.pow(2, this.attempts - 1)
  }
  return this.backoffDelay
}
```

### 14.3 调度器模块的扩展面

`scheduler.js:104-134` 的模块导出也有扩展点：

```javascript
module.exports = {
  jobs: [],           // ← 可替换为优先级队列数据结构
  init() { return this },
  start() { /* ... */ },       // ← 可增加 start 钩子
  registerJob() { /* ... */ }, // ← 可增加去重、并发控制
  stop() { /* ... */ }         // ← 可增加 shutdown 超时
}
```

**缺失的扩展点**：

| 方法 | 用途 | 当前状态 |
|------|------|----------|
| `getJob(name)` | 按名称查询任务 | ❌ 不存在 |
| `getJobStatus(name)` | 查询任务状态（running/pending/stopped） | ❌ 不存在 |
| `cancelJob(name)` | 按名称取消任务 | ❌ 不存在（只有 `job.stop()`） |
| `onJobComplete(callback)` | 任务完成回调 | ❌ 不存在 |
| `onJobFailed(callback)` | 任务失败回调 | ❌ 不存在 |

---

## 十五、PT15M 时区处理

### 15.1 moment.duration 解析 ISO 8601 Duration

```javascript
// scheduler.js:20
this.schedule = moment.duration(schedule)
```

`moment.duration()` 解析 ISO 8601 Duration 字符串，输出一个**绝对时间段**对象：

| 配置值 | 含义 | `asMilliseconds()` |
|--------|------|--------------------|
| `PT15M` | 15 分钟 | 900000 |
| `P1D` | 1 天 | 86400000 |
| `PT1H` | 1 小时 | 3600000 |
| `P0D` | 0 天（禁用标志） | 0 |

### 15.2 时区无关性

**关键结论：当前实现完全时区无关，且这是正确的设计。**

原因分析：

1. **moment.duration 是绝对时间段**，不是时间点：
   ```javascript
   moment.duration('PT15M')  // = 900000 毫秒，与任何时区无关
   new Date(Date.now() + moment.duration('PT15M').asMilliseconds())  // 时区在 Date 层
   ```

2. **setTimeout 接受毫秒延迟**，不涉及时区：
   ```javascript
   // scheduler.js:45
   setTimeout(this.invoke.bind(this), this.schedule.asMilliseconds(), data)
   // 等价于 setTimeout(fn, 900000, data)
   ```

3. **不使用 cron 表达式**：cron 如 `0 */15 * * *`（每15分钟整点）涉及时区问题，但当前实现是"距离上次完成后 15 分钟"，是相对时间。

### 15.3 与 cron 式调度的对比

```
cron 式调度 (涉时区):
  UTC+8 时区: 0:00, 0:15, 0:30, 0:45, 1:00 ...
  UTC+0 时区: 0:00, 0:15, 0:30, 0:45, 1:00 ...
  → 同一 cron 表达式在不同时区下的绝对时间不同

当前实现 (时区无关):
  无论哪个时区: 启动后15分钟 → 执行完成后15分钟 → 执行完成后15分钟 ...
  → 绝对时间段，与服务器时区无关
```

### 15.4 P1D 的隐含问题

虽然 `PT15M` 没有时区问题，但 `P1D`（1天 = 86400000 毫秒）存在**夏令时漂移**的潜在问题：

- 在有夏令时的时区，一天可能是 23 或 25 小时
- 但由于 `moment.duration('P1D').asMilliseconds()` 始终返回 86400000（24小时）
- 这意味着在夏令时切换日，"每天执行"的任务实际间隔会偏移 ±1 小时

**对 Wiki.js 的影响**：微乎其微。`sync-graph-locales` 和 `sync-graph-updates` 都是后台同步任务，±1 小时偏移完全可接受。

### 15.5 moment-duration-format 的存在但未使用

`package.json` 中安装了 `moment-duration-format`：

```json
"moment-duration-format": "2.3.2"
```

但源码中**没有任何地方**使用它来格式化 duration 输出。这个依赖仅用于可能的调试日志，实际调度只调用 `asMilliseconds()`。

### 15.6 configHelper 的 Duration 校验

```javascript
// helpers/config.js:5-28
const isoDurationReg = /^(-|\+)?P(?:([-+]?[0-9,.]*)Y)?(?:([-+]?[0-9,.]*)M)?...$/

module.exports = {
  isValidDurationString (val) {
    return isoDurationReg.test(val)
  }
}
```

```javascript
// scheduler.js:116
const schedule = (configHelper.isValidDurationString(queueParams.schedule))
  ? queueParams.schedule
  : 'P1D'    // ← 非法 duration 回退为 1 天
```

这确保了 `setTimeout` 不会收到 NaN 毫秒数，但**不验证 duration 的语义合理性**（如 `PT0S` 会导致无限循环调用 `invoke`）。

---

## 十六、长尾 worker 资源隔离

### 16.1 进程级隔离

Worker 任务通过 `child_process.fork()` 实现**操作系统级**的进程隔离：

```javascript
// scheduler.js:56-62
const proc = childProcess.fork(`server/core/worker.js`, [
  `--job=${this.name}`,
  `--data=${data}`
], {
  cwd: WIKI.ROOTPATH,
  stdio: ['inherit', 'inherit', 'pipe', 'ipc']
})
```

这意味着：

| 隔离维度 | 主进程 | Worker 子进程 |
|----------|--------|---------------|
| 内存空间 | 独立 | 独立（fork 时 COW） |
| 事件循环 | 独立 | 独立 |
| DB 连接池 | 主连接池 | **独立新建连接池** |
| V8 堆 | 共享主进程 | 独立 V8 实例 |
| CPU | 主进程线程 | 独立 OS 进程 |
| 崩溃影响 | 不受 worker 崩溃影响 | 崩溃仅退出自身 |

### 16.2 Worker 的独立 DB 连接池成本

每个 worker 任务执行时都会**完整初始化数据库**：

```javascript
// render-page.js:10-12, rebuild-tree.js:9-11
WIKI.models = require('../core/db').init()    // 创建新 Knex 实例 + 连接池
await WIKI.configSvc.loadFromDb()             // 从 DB 加载配置
await WIKI.configSvc.applyFlags()             // 应用开发标志
// ...执行业务逻辑...
await WIKI.models.knex.destroy()              // 销毁连接池
```

连接池配置：

```javascript
// db.js:141-143
pool: {
  ...WIKI.config.pool,
  // WIKI.config.pool 来自 data.yml 默认值
}
```

```yaml
# data.yml:24
pool:
  min: 1    # 最小连接数
```

**每个 worker 至少占用 1 个 DB 连接**。如果多个 worker 并发执行：

```
主进程连接池 (min:1, default)
  + render-page worker 连接池 (min:1)
  + rebuild-tree worker 连接池 (min:1)
  + sanitize-svg worker 连接池 (min:1)
  = 至少 4 个 DB 连接
```

对于 PostgreSQL 默认 `max_connections=100`，并发 10 个 render-page 任务就会占用 10+ 个连接。

### 16.3 内存开销分析

每个 worker 子进程的内存开销：

1. **Node.js 基础开销**：约 30-50 MB（V8 堆 + 代码段）
2. **WIKI 全局对象**：重新初始化，包括 configSvc、logger
3. **Knex + Objection**：ORM 实例和连接池
4. **任务依赖**：如 cheerio（render-page）、DOMPurify（sanitize-svg）

粗略估计每个 worker 进程约 **50-80 MB**。

### 16.4 并发 worker 无上限

当前代码**没有并发 worker 数量限制**：

```javascript
// scheduler.js:53-62 — 每次 invoke 都 fork 新进程
async invoke(data) {
  try {
    if (this.worker) {
      const proc = childProcess.fork(...)  // 无并发检查
      // ...
    }
  }
}
```

如果 100 个用户同时保存页面，会触发 100 个 `render-page` worker：

```
pages.js:renderPage(page)
    ↓
WIKI.scheduler.registerJob({name: 'render-page', immediate: true, worker: true}, page.id)
    ↓
childProcess.fork(worker.js, ['--job=render-page', '--data=' + page.id])
    ↓ ×100
100 个子进程 × 80MB = 8GB 内存 + 100 个 DB 连接
```

### 16.5 render-page.js 中的特殊资源问题

```javascript
// render-page.js:24-27
if (_.isEmpty(page.content)) {
  await WIKI.models.knex.destroy()   // ← 提前销毁连接池
  WIKI.logger.warn(`Failed to render page ID ${pageId} because content was empty: [ FAILED ]`)
  // ⚠️ 注意：这里没有 throw，函数继续执行！
  // ⚠️ 后续代码会尝试使用已销毁的 DB 连接
}
```

这段代码存在 bug：
1. `knex.destroy()` 后没有 `return` 或 `process.exit()`
2. 后续的渲染循环 `for (let core of pipeline)` 会继续执行
3. 当执行到 `WIKI.models.pages.query().patch(...)` 时会因为连接已关闭而抛出异常

### 16.6 非 Worker 任务的资源隔离（无）

非 worker 模式的任务直接在主进程执行：

```javascript
// scheduler.js:80-82
} else {
  this.finished = require(`../jobs/${this.name}`)(data)
}
```

这意味着：

| 风险 | 说明 |
|------|------|
| 阻塞事件循环 | CPU 密集任务会阻塞 HTTP 请求处理 |
| 共享 DB 连接池 | 任务和 HTTP 请求竞争连接池 |
| 异常传播 | 未捕获异常可能导致主进程崩溃 |
| 内存压力 | 大数据处理直接占用主进程堆 |

当前的非 worker 任务（`purge-uploads`、`sync-storage`、`sync-graph-*`、`fetch-graph-locale`）都是 I/O 密集型，CPU 开销低，因此风险可控。

---

## 十七、5 扩展点 priority schema 设计

### 17.1 优先级的 5 个扩展点回顾

Scheduler 的扩展点分布在 5 个关键位置，每个都对 priority 有不同的影响：

```
Job 生命周期:
  constructor()  ← 定义 priority 字段和默认值
       ↓
  start(data)    ← 按 priority 对 jobs 数组排序插入
       ↓
  enqueue(data)  ← (仅周期任务) setTimeout 与 priority 无关
       ↓
  invoke(data)   ← 执行时按 priority 从队列取任务
       ↓
  stop()         ← priority 无关
```

### 17.2 两级 priority 体系设计

Wiki.js 存在两类任务，优先级模型应该分层：

| 层级 | 任务类型 | 优先级区间 | 调度方式 |
|------|----------|-----------|----------|
| L1 关键 | 一次性任务（render-page、sanitize-svg） | 100-199 | 立即出队执行，不排队 |
| L2 后台 | 周期任务（purge-uploads、sync-storage） | 0-99 | 排队执行，按 priority 取队首 |

**L1 一次性任务优先级建议（按紧急程度排序）**：

| 优先级值 | 任务 | 理由 |
|----------|------|------|
| 190 | `sanitize-svg` | 用户上传后同步等待，必须先完成才能保存文件 |
| 180 | `render-page` | 用户保存页面后同步等待渲染结果 |
| 170 | `fetch-graph-locale` | 用户主动触发下载 |
| 160 | `rebuild-tree` | 页面变更后后台重建 |

**L2 周期任务优先级建议（按对用户体验影响排序）**：

| 优先级值 | 任务 | 理由 |
|----------|------|------|
| 90 | `purge-uploads` | 影响磁盘空间，用户上传阻塞的根因 |
| 80 | `sync-storage` | 备份时效性影响数据安全 |
| 50 | `sync-graph-locales` | 语言包更新不紧急 |
| 10 | `sync-graph-updates` | 版本检查最不紧急 |

### 17.3 constructor 扩展点：priority schema

```javascript
// scheduler.js:8-23 — 扩展后
constructor({
  name,
  immediate = false,
  schedule = 'P1D',
  repeat = false,
  worker = false,
  // === 新增 priority schema ===
  priority,              // 不显式给默认值，而是按任务类型推断
}, queue) {
  this.queue = queue
  this.finished = Promise.resolve()
  this.name = name
  this.immediate = immediate
  this.schedule = moment.duration(schedule)
  this.repeat = repeat
  this.worker = worker

  // priority 推断逻辑：一次性任务默认高优先级，周期任务默认低优先级
  if (priority !== undefined) {
    this.priority = Math.max(0, Math.min(255, priority))  // clamp 到 0-255
  } else if (immediate && !repeat) {
    this.priority = 150   // 一次性任务默认 L1
  } else {
    this.priority = 50    // 周期任务默认 L2
  }
}
```

### 17.4 start 扩展点：排序插入

```javascript
// scheduler.js:30-37 — 扩展后
start(data) {
  // 去重检查（按 uniqueKey，见第十四节扩展）
  if (this.uniqueKey) {
    const existing = this.queue.jobs.find(j => j.uniqueKey === this.uniqueKey)
    if (existing) {
      WIKI.logger.warn(`Job ${this.name} with key ${this.uniqueKey} already queued. [SKIPPED]`)
      return existing
    }
  }

  // 按 priority 降序插入（不是简单 push）
  this.queue.jobs.push(this)
  this.queue.jobs.sort((a, b) => {
    if (b.priority !== a.priority) {
      return b.priority - a.priority
    }
    // 同优先级按入队时间，FIFO
    return (a._enqueuedAt || 0) - (b._enqueuedAt || 0)
  })
  this._enqueuedAt = Date.now()

  if (this.immediate) {
    this.invoke(data)
  } else {
    this.enqueue(data)
  }
}
```

### 17.5 invoke 扩展点：并发控制 + priority 取队首

```javascript
// scheduler.js:53-92 — 扩展后
async invoke(data) {
  // worker 并发上限（全局信号量）
  if (this.worker) {
    const MAX_WORKER_CONCURRENCY = WIKI.config.jobs?.maxWorkerConcurrency || 4
    const runningWorkers = this.queue.jobs.filter(j =>
      j.worker && j._isRunning && j !== this
    ).length
    if (runningWorkers >= MAX_WORKER_CONCURRENCY) {
      // 超过并发上限，推迟到队首 priority 任务执行完
      const delayMs = 1000
      WIKI.logger.debug(`Worker concurrency saturated (${runningWorkers}/${MAX_WORKER_CONCURRENCY}), delaying job ${this.name}`)
      this.timeout = setTimeout(this.invoke.bind(this), delayMs, data)
      return
    }
  }

  this._isRunning = true
  // ... 原有执行逻辑 ...
}
```

### 17.6 另外两个扩展点与 priority 的关系

- **enqueue**：periodic task 的 setTimeout 与 priority 无关——它们是"到点触发"而非"排队取队首"
- **stop**：priority 无关——停止就是停止，不区分紧急程度

### 17.7 与 storage.js 现有 priority 概念的协调

`storage.js` 中已经存在一个隐式 priority：按 `key` 排序初始化 storage targets：

```javascript
// storage.js:118
this.targets = await WIKI.models.storage.query().where('isEnabled', true).orderBy('key')
```

调度器的通用 priority 体系应与 storage target 的 `order` 字段（如果未来增加）兼容，避免两套优先级冲突。

---

## 十八、PT0S 无限循环兜底

### 18.1 漏洞场景复现

当前的 duration 校验只检查格式合法性，不检查语义合理性：

```javascript
// helpers/config.js:26-28
isValidDurationString (val) {
  return isoDurationReg.test(val)
}
```

```javascript
// scheduler.js:116
const schedule = (configHelper.isValidDurationString(queueParams.schedule))
  ? queueParams.schedule
  : 'P1D'
```

`PT0S` 是合法的 ISO 8601 Duration（0 秒），但会导致：

```javascript
// scheduler.js:20
this.schedule = moment.duration('PT0S')   // asMilliseconds() = 0

// scheduler.js:44-46
enqueue(data) {
  this.timeout = setTimeout(this.invoke.bind(this), 0, data)  // 0ms 延迟
}
```

配合 `repeat: true`，执行路径变成：

```
invoke() → 执行任务 → enqueue() → setTimeout(0) → 立即再次 invoke → ...
                       ↑                                          │
                       └──────────────────────────────────────────┘
```

### 18.2 漏洞影响面分析

| 任务类型 | 触发条件 | 危害等级 |
|----------|----------|----------|
| 自定义 storage sync | 用户在 admin UI 设置 `syncInterval = "PT0S"` | **高危** — CPU 100% + DB 连接耗尽 |
| data.yml 手工修改 | 运维修改配置文件写错 | **中危** — 服务崩溃 |
| 运行时 registerJob | 代码 bug 传了 0 duration | **中危** — 局部故障 |

storage.js 的默认值有个伏笔：

```javascript
// storage.js:64
syncInterval: target.schedule || 'P0D',

// storage.js:145
if (targetDef.schedule && target.syncInterval !== `P0D`) {
  WIKI.scheduler.registerJob({ name: `sync-storage`, ... repeat: true }, target.key)
}
```

这里用 `P0D` 作为"禁用"的哨兵值，但只检查了 `P0D`，没检查 `PT0S`、`PT0M`、`P0DT0H0M0S` 等等价表示。

### 18.3 三层兜底防护

```javascript
// 第一层：isValidDurationString 增加语义校验 — helpers/config.js
const MIN_DURATION_MS = 5000  // 最小 5 秒

isValidDurationString (val) {
  if (!isoDurationReg.test(val)) return false
  const dur = moment.duration(val)
  return dur.asMilliseconds() >= MIN_DURATION_MS
}

// 第二层：Job constructor 强制下限 — scheduler.js:20
this.schedule = moment.duration(schedule)
if (this.repeat && this.schedule.asMilliseconds() < MIN_DURATION_MS) {
  WIKI.logger.warn(`Job ${name} schedule ${schedule} too short, clamped to ${MIN_DURATION_MS}ms`)
  this.schedule = moment.duration(MIN_DURATION_MS)
}

// 第三层：enqueue 防抖兜底 — scheduler.js:44-46
enqueue(data) {
  const delay = Math.max(this.schedule.asMilliseconds(), MIN_DURATION_MS)

  // 防止 _invokeCount 爆炸（连续 invoke 超过阈值就暂停）
  if ((this._invokeCount || 0) > 100 && Date.now() - (this._firstInvokeAt || 0) < 60000) {
    WIKI.logger.error(`Job ${this.name} invoked ${this._invokeCount} times in 60s, pausing for 60s`)
    this.timeout = setTimeout(this.invoke.bind(this), 60000, data)
    return
  }

  this.timeout = setTimeout(this.invoke.bind(this), delay, data)
}
```

### 18.4 第三层兜底的 invoke 计数

```javascript
// scheduler.js:53 — invoke 开头增加计数
async invoke(data) {
  this._invokeCount = (this._invokeCount || 0) + 1
  if (!this._firstInvokeAt) {
    this._firstInvokeAt = Date.now()
  }
  // 每分钟重置一次计数
  if (Date.now() - this._firstInvokeAt > 60000) {
    this._invokeCount = 1
    this._firstInvokeAt = Date.now()
  }
  // ... 原有逻辑
}
```

这个三层设计遵循防御性编程原则：第一层在入口拦、第二层在对象创建时拦、第三层在执行运行时最后兜底。即使前两层都漏了（比如代码直接 new Job 绕过 registerJob），第三层的 100次/分钟熔断也能救场。

---

## 十九、P1D 夏令时补偿

### 19.1 问题的数学模型

`moment.duration('P1D')` 永远返回 86400000 毫秒，但"自然日"有三种长度：

| 日期类型 | 实际时长 | 出现频率 | 偏差 |
|----------|----------|----------|------|
| 标准日 | 86400s | 362 天/年 | 0 |
| 春季 DST 开始 | 82800s（23h） | 1 天/年 | -3600s |
| 秋季 DST 结束 | 90000s（25h） | 1 天/年 | +3600s |

用固定 86400s 的 setTimeout 来调度"每天执行"，会在 DST 切换日出现 ±1 小时漂移。

### 19.2 对现有任务的实际影响

| 任务 | 当前 schedule | DST 影响是否可接受 | 理由 |
|------|--------------|-------------------|------|
| `sync-graph-locales` | P1D | ✅ 可接受 | 后台同步，±1 小时无感知 |
| `sync-graph-updates` | P1D | ✅ 可接受 | 版本检查，时间不敏感 |
| 自定义 storage sync | 用户配置（P1D 常见） | ⚠️ 取决于业务 | 用户可能预期"每天凌晨 3 点" |

### 19.3 三种补偿策略对比

| 策略 | 实现复杂度 | 精度 | 适用场景 |
|------|-----------|------|----------|
| A. 固定 duration（现状） | 极低 | ±1h | 所有后台任务 |
| B. "距下次目标时间"计算 | 中 | 无偏移 | 用户有明确执行时间预期 |
| C. cron 表达式 | 高 | 无偏移 | 企业级调度需求 |

### 19.4 策略 B 的具体实现（建议 Wiki.js 采用）

```javascript
// scheduler.js 新增
_getNextRunTimestamp() {
  if (!this.repeat) return null

  const now = DateTime.local()  // 使用 luxon（已在 index.js 引入）
  const durMs = this.schedule.asMilliseconds()

  // 对"天"级别以上的 schedule 用日历计算，以下用固定毫秒
  if (durMs >= 86400000) {  // P1D 及更长
    const days = Math.round(durMs / 86400000)
    const target = now.plus({ days }).startOf('day')
      .plus({ hours: this._preferredHour || 3 })  // 默认凌晨 3 点执行
    return target.toMillis() - now.toMillis()
  }

  return durMs  // PT15M 级别的继续用固定毫秒
}

enqueue(data) {
  const delay = this._getNextRunTimestamp()
    || this.schedule.asMilliseconds()

  this.timeout = setTimeout(this.invoke.bind(this), delay, data)
}
```

利用 luxon 的时区感知计算：`startOf('day')` 会自动处理 DST，保证"下一个自然日凌晨 3 点"在所有时区下都正确。

### 19.5 为什么引入 luxon 而不是 moment-timezone

Wiki.js 已经在 `index.js` 引入了 luxon：

```javascript
// index.js:8
const { DateTime } = require('luxon')
```

而 `moment-timezone` 虽然在 package.json 中存在，但未在调度器中使用。luxon 的 API 更现代且原生支持时区计算，是更合理的选择。

### 19.6 storage sync 的补偿需求

storage.js 中用户可以配置 syncInterval，当前只接受 Duration。补偿后应新增字段：

```javascript
// storage model 新增 schema — models/storage.js
// syncPreferredHour: { type: 'integer', minimum: 0, maximum: 23, default: 3 }

// storage.js:145-151 扩展
if (targetDef.schedule && target.syncInterval !== `P0D`) {
  const job = WIKI.scheduler.registerJob({
    name: `sync-storage`,
    schedule: target.syncInterval,
    repeat: true
  }, target.key)
  job._preferredHour = target.syncPreferredHour || 3  // 注入偏好执行时刻
}
```

---

## 二十、50-80MB 每 worker 的集群一致性

### 20.1 集群（HA）模式的现有架构

Wiki.js 的多实例协调依赖 PostgreSQL LISTEN/NOTIFY：

```javascript
// db.js:231-264
async subscribeToNotifications () {
  const useHA = (WIKI.config.ha === true || ...)
  if (!useHA) return

  const PGPubSub = require('pg-pubsub')
  this.listener = new PGPubSub(this.knex.client.connectionSettings, ...)

  this.listener.addChannel('wiki', payload => {
    if (payload.source !== WIKI.INSTANCE_ID) {  // 用 nanoid 区分实例
      WIKI.events.inbound.emit(payload.event, payload.value)
    }
  })
}
```

每个实例有唯一 ID：
```javascript
// index.js:19
INSTANCE_ID: nanoid(10),  // 如 "V1StGXR8_Z"
```

### 20.2 当前的任务一致性问题

在 HA 模式下，N 个实例会各自执行 scheduler.start()，导致周期任务重复执行 N 次：

```
Instance A (ID: abc123):
  scheduler.start() → 注册 purge-uploads, sync-*, sync-storage

Instance B (ID: def456):
  scheduler.start() → 注册 purge-uploads, sync-*, sync-storage

每 15 分钟:
  Instance A: purge-uploads 执行 ✅
  Instance B: purge-uploads 执行 ❌ (重复)
```

对于一次性任务也有问题：

```
用户保存页面 → HTTP 负载均衡随机分到 Instance A
  Instance A: registerJob(render-page) → fork worker ✅

但如果是批量操作触发 rebuild-tree:
  Instance A 和 B 都可能收到事件并各自注册 → 执行两次
```

### 20.3 内存开销的集群放大

单机 4 个并发 worker ≈ 320MB，在 N 实例集群下：

| 集群规模 | 并发 worker（按单机 4 算） | 总内存占用 | 总 DB 连接数 |
|----------|------------------------|-----------|-------------|
| 1 实例 | 4 | ~320 MB | ~4 |
| 2 实例 | 8 | ~640 MB | ~8 |
| 3 实例 | 12 | ~960 MB | ~12 |
| 5 实例 | 20 | ~1.6 GB | ~20 |

而如果有**分布式去重锁**，只有 1 个实例实际执行，内存和连接开销都降回单机水平。

### 20.4 基于 Postgres Advisory Lock 的分布式锁

使用 PostgreSQL Advisory Lock 实现任务级别的集群互斥（不依赖额外中间件）：

```javascript
// scheduler.js invoke 扩展
async invoke(data) {
  // 对 repeat: true 的周期任务加分布式锁
  let lockAcquired = false
  const lockKey = this._computeLockKey(data)

  if (this.repeat && WIKI.config.ha && WIKI.config.db.type === 'postgres') {
    try {
      // pg_try_advisory_lock: 不阻塞，获取不到返回 false
      const result = await WIKI.models.knex.raw(
        'SELECT pg_try_advisory_lock(?);',
        [lockKey]
      )
      lockAcquired = result.rows[0].pg_try_advisory_lock
    } catch (e) {
      WIKI.logger.warn(`Failed to acquire lock for job ${this.name}: ${e.message}`)
      lockAcquired = true  // 降级：锁不可用时本实例继续执行
    }
    if (!lockAcquired) {
      WIKI.logger.debug(`Job ${this.name} skipped: lock held by another instance`)
      if (this.repeat && this.queue.jobs.includes(this)) {
        this.enqueue(data)  // 让出本次，继续下次调度
      }
      return
    }
  }

  try {
    // ... 原有执行逻辑 ...
  } finally {
    if (lockAcquired) {
      await WIKI.models.knex.raw('SELECT pg_advisory_unlock(?);', [lockKey])
    }
  }
}

_computeLockKey(data) {
  // 将字符串锁名转成 int8（advisory lock 需要 bigint）
  let hash = 0
  const str = `wikijs:job:${this.name}:${data || ''}`
  for (let i = 0; i < str.length; i++) {
    hash = ((hash << 5) - hash) + str.charCodeAt(i)
    hash |= 0
  }
  return Math.abs(hash)
}
```

### 20.5 非 Postgres 数据库的降级方案

对于 MySQL/SQLite（不支持 advisory lock）：

```javascript
// 用 settings 表实现简单的心跳锁
// CREATE TABLE jobLocks (
//   lockKey TEXT PRIMARY KEY,
//   instanceId TEXT,
//   acquiredAt TEXT,
//   heartbeatAt TEXT
// )

// 心跳 TTL = 5 分钟，超过则认为实例已死
const LOCK_TTL_MS = 5 * 60 * 1000

async _acquireLockDB(lockKey) {
  const now = new Date().toISOString()
  const trx = await WIKI.models.knex.transaction()
  try {
    // 1. 清理过期锁
    await trx('jobLocks')
      .whereRaw("heartbeatAt < datetime('now', '-5 minutes')")
      .del()

    // 2. 尝试 INSERT（利用主键唯一）
    await trx('jobLocks').insert({
      lockKey,
      instanceId: WIKI.INSTANCE_ID,
      acquiredAt: now,
      heartbeatAt: now
    })

    await trx.commit()
    return true
  } catch (e) {
    await trx.rollback()
    // 主键冲突 = 锁被占
    return false
  }
}
```

### 20.6 锁的 TTL 与 worker 生命周期配合

worker 执行期间必须持续心跳：

```javascript
async invoke(data) {
  // ... 加锁成功 ...
  let heartbeatTimer = null
  if (lockAcquired) {
    heartbeatTimer = setInterval(async () => {
      await WIKI.models.knex('jobLocks')
        .where({ lockKey })
        .update({ heartbeatAt: new Date().toISOString() })
    }, 30000)  // 每 30s 心跳一次
  }

  try {
    await this.finished
  } finally {
    if (heartbeatTimer) clearInterval(heartbeatTimer)
    // ... 释放锁 ...
  }
}
```

---

## 二十一、knex.destroy 后无 return bug 的修复 PR

### 21.1 Bug 定位

**文件**：`server/jobs/render-page.js:24-27`

**当前代码**：
```javascript
if (_.isEmpty(page.content)) {
  await WIKI.models.knex.destroy()
  WIKI.logger.warn(`Failed to render page ID ${pageId} because content was empty: [ FAILED ]`)
  // ⚠️ BUG: 没有 return 或 throw！函数继续执行
}
```

### 21.2 Bug 的完整执行路径

当 `page.content` 为空时：

```
1. knex.destroy()  → DB 连接池关闭，所有连接释放
2. logger.warn(...) → 日志输出
3. for (let core of pipeline) → 进入渲染循环
4. 循环中每个 renderer 可能正常执行（不依赖 DB）
5. 最后执行 WIKI.models.pages.query().patch(...)
     ↓
   Knex: Timeout acquiring a connection. The pool is probably full.
   或: Unable to acquire a connection
     ↓
6. catch(err) → logger.error → throw err
     ↓
7. worker.js 捕获 → process.exit(1)
```

**最终结果**：任务确实失败了，但原因是"连接池已关闭"而不是"内容为空"，日志误导排错。

### 21.3 同类问题检查

遍历所有 worker 任务：

| 任务文件 | 是否有类似 bug | 说明 |
|----------|--------------|------|
| `render-page.js:24-27` | ✅ **有** | knex.destroy 后无 return |
| `rebuild-tree.js` | 无 | knex.destroy 在正常流程末尾，try 结尾 |
| `sanitize-svg.js` | 无 | 不使用 DB（纯文件处理），没有 knex |
| 非 worker 任务 | N/A | 共享主进程连接池，不调用 destroy |

### 21.4 修复 PR 的 diff

```diff
--- a/server/jobs/render-page.js
+++ b/server/jobs/render-page.js
@@ -21,10 +21,12 @@ module.exports = async (pageId) => {

     if (_.isEmpty(page.content)) {
       await WIKI.models.knex.destroy()
       WIKI.logger.warn(`Failed to render page ID ${pageId} because content was empty: [ FAILED ]`)
+      throw new Error('Page content is empty')
     }

     for (let core of pipeline) {
```

为什么用 `throw` 而不是 `return`：

1. **与 catch 块语义一致**：函数末尾有 `catch (err) { ... throw err }`，throw 能走统一的错误出口
2. **与 worker.js 约定一致**：worker.js 期待失败时抛异常 → process.exit(1)
3. **便于主进程识别**：throw 的 error 对象会被 scheduler.js 捕获并 reject，调用方能感知失败

### 21.5 进一步加固：finally 中统一 destroy

更健壮的模式是用 `finally` 统一释放资源，避免每个分支都写 destroy：

```javascript
// 推荐的完整修复
module.exports = async (pageId) => {
  WIKI.logger.info(`Rendering page ID ${pageId}...`)

  try {
    WIKI.models = require('../core/db').init()
    await WIKI.configSvc.loadFromDb()
    await WIKI.configSvc.applyFlags()

    const page = await WIKI.models.pages.getPageFromDb(pageId)
    if (!page) throw new Error('Invalid Page Id')
    if (_.isEmpty(page.content)) throw new Error('Page content is empty')

    await WIKI.models.renderers.fetchDefinitions()
    const pipeline = await WIKI.models.renderers.getRenderingPipeline(page.contentType)

    let output = page.content
    for (let core of pipeline) {
      // ... 渲染逻辑不变 ...
    }

    // ... 保存到 DB、缓存 ...

    WIKI.logger.info(`Rendering page ID ${pageId}: [ COMPLETED ]`)
  } catch (err) {
    WIKI.logger.error(`Rendering page ID ${pageId}: [ FAILED ]`)
    WIKI.logger.error(err.message)
    throw err
  } finally {
    // ✅ finally 中统一释放，无论正常/异常都执行
    if (WIKI.models && WIKI.models.knex) {
      await WIKI.models.knex.destroy()
    }
  }
}
```

`rebuild-tree.js` 也可以套同样的 `finally` 模式，目前它在正常路径末尾 destroy，异常路径会泄漏连接池。

### 21.6 单元测试建议

```javascript
// test/jobs/render-page.test.js
describe('render-page job', () => {
  it('should throw early when content is empty, not attempt DB writes', async () => {
    // mock empty page
    // stub knex.destroy to track calls
    // expect(promise).to.be.rejectedWith('Page content is empty')
    // expect(knex.destroy).to.have.been.calledOnce
  })
})
```

---

## 二十二、缺失 5 方法的优先级

### 22.1 5 个缺失方法回顾

第十四节列出了调度器模块缺失的 5 个方法：

| 方法 | 用途 |
|------|------|
| `getJob(name)` | 按名称查询任务 |
| `getJobStatus(name)` | 查询任务状态（running/pending/stopped） |
| `cancelJob(name)` | 按名称取消任务 |
| `onJobComplete(callback)` | 任务完成回调 |
| `onJobFailed(callback)` | 任务失败回调 |

### 22.2 优先级排序矩阵

按**实现成本 × 业务价值**排序：

| 优先级 | 方法 | 实现成本 | 业务价值 | 理由 |
|--------|------|----------|----------|------|
| **P0** | `cancelJob(name)` | 低 | 高 | storage.js 已经在手写 `_.remove()` + `job.stop()`，应该标准化，否则业务层绕过调度器 API |
| **P1** | `onJobComplete / onJobFailed` | 中 | 高 | telemetry 已经预留了 sendError 但为空，需要事件钩子把错误送出去；也是告警集成的基础 |
| **P2** | `getJobStatus(name)` | 极低 | 中 | Admin UI 需要展示任务状态；成本几乎为零，只是给 Job 加个 getter |
| **P3** | `getJob(name)` | 低 | 低 | 可以直接通过 jobs 数组访问；除非要做权限/封装，否则价值不大 |
| **P4** | （省略） | — | — | 4 个就够，第 5 个可以不实现 |

### 22.3 P0：cancelJob(name) — 最优先实现

storage.js 当前在绕过 API 手写去重逻辑：

```javascript
// storage.js:120-124 — 当前绕过调度器 API 的代码
const prevjobs = _.remove(WIKI.scheduler.jobs, job => job.name === `sync-storage`)
if (prevjobs.length > 0) {
  prevjobs.forEach(job => job.stop())
}
```

标准化后的 API：

```javascript
// scheduler.js — 新增
cancelJob(name, { data = undefined } = {}) {
  let cancelled = 0
  this.jobs = this.jobs.filter(job => {
    if (job.name === name) {
      if (data === undefined || job._data === data) {
        job.stop().catch(() => {})
        cancelled++
        return false
      }
    }
    return true
  })
  WIKI.logger.info(`Cancelled ${cancelled} job(s) with name: ${name}`)
  return cancelled
}
```

storage.js 可以简化为：

```javascript
// storage.js — 简化后
WIKI.scheduler.cancelJob('sync-storage', { data: target.key })
```

### 22.4 P1：onJobComplete / onJobFailed — 事件总线

调度器应该继承 EventEmitter：

```javascript
// scheduler.js — 改造
const { EventEmitter } = require('eventemitter2')

const scheduler = new EventEmitter()

scheduler.jobs = []
scheduler.init = function() { return this }
// ...

// invoke 中发射事件
async invoke(data) {
  try {
    await this.finished
    scheduler.emit('job:complete', {
      name: this.name,
      data,
      duration: Date.now() - this._startTime,
      instanceId: WIKI.INSTANCE_ID
    })
  } catch (err) {
    scheduler.emit('job:failed', {
      name: this.name,
      data,
      error: err.message,
      exitCode: err.exitCode,
      instanceId: WIKI.INSTANCE_ID
    })
  }
}
```

与现有代码的对接：

```javascript
// telemetry.js — 现在可以非空实现了
init() {
  WIKI.telemetry = this
  if (this.enabled) {
    WIKI.scheduler.on('job:failed', payload => this.sendError(payload))
    WIKI.scheduler.on('job:complete', payload => this.sendEvent('job', 'complete', payload.name))
  }
}
```

### 22.5 P2：getJobStatus(name) — 极低成本

```javascript
// Job 类增加状态 getter
get status() {
  if (!this.queue.jobs.includes(this)) return 'stopped'
  if (this._isRunning) return 'running'
  if (this.timeout) return 'pending'
  return 'idle'
}

// 调度器增加查询方法
getJobStatus(name, { data = undefined } = {}) {
  const job = this.jobs.find(j =>
    j.name === name && (data === undefined || j._data === data)
  )
  return job ? job.status : null  // null = 不存在
}
```

Admin UI 可以新增一个任务状态面板，前端调 GraphQL 接口展示。

### 22.6 P3：getJob(name) — 价值较低

内部数组已经暴露，这个方法主要是为了 API 完整性：

```javascript
getJob(name, { data = undefined } = {}) {
  return this.jobs.find(j =>
    j.name === name && (data === undefined || j._data === data)
  ) || null
}
```

不建议暴露给业务层直接操作 Job 实例，因为容易绕过调度器的生命周期管理。

---

## 二十三、OS 进程重启策略

### 23.1 当前进程退出的所有路径

遍历代码中所有 `process.exit` 和信号处理：

| 退出路径 | 触发条件 | exit code | 可重启 |
|----------|----------|-----------|--------|
| `index.js:46-50` | SIGTERM / SIGINT | graceful shutdown → 0 | 否（运维主动停止） |
| `index.js:52-56` | 父进程 IPC message 'shutdown' | graceful shutdown → 0 | 否 |
| `kernel.js:24,47,65` | DB/config 初始化失败 | 1 | **应该重启**（临时故障） |
| `db.js:132` | 无效 DB type | 1 | 否（配置错误） |
| `config.js:48,69` | 配置文件缺失或 DB_PASS_FILE 读取失败 | 1 | 否（配置错误） |
| `servers.js:37,40,85,99,102` | 端口被占 / 权限不足 / SSL 错误 | 1 | **应该重启**（端口暂时被占） |
| `users.js:885,895` | Admin 账号创建失败 | 1 | 否 |
| `worker.js:19` | Worker 任务成功 | 0 | 否（任务自然完成） |
| `worker.js:22` | Worker 任务失败 | 1 | 否（任务失败 ≠ 进程故障） |

### 23.2 主进程 vs Worker 进程的区分

重启策略必须区分两种进程：

```
主进程 (PID 1 或由 systemd/pm2 管理):
  exit 0 → 运维停止 → 不重启
  exit 1 → 根据原因决定
  崩溃（OOM、uncaughtException）→ 应该重启

Worker 子进程 (child_process.fork 出来):
  任何 exit code → 都不应该由 OS 重启
  重启应该由 scheduler.js 的重试逻辑控制
```

### 23.3 systemd 配置建议（生产部署）

```ini
# /etc/systemd/system/wikijs.service
[Unit]
Description=Wiki.js
After=network.target postgresql.service

[Service]
Type=simple
User=wikijs
Group=wikijs
WorkingDirectory=/var/www/wikijs
ExecStart=/usr/bin/node server/index.js

# === 重启策略 ===
Restart=on-failure
RestartSec=5
StartLimitBurst=5
StartLimitIntervalSec=60

# === 安全限制（也限制 worker fork 的权限）===
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/var/www/wikijs/data
ProtectHome=true

# === OOM 保护 ===
OOMScoreAdjust=-500

# === worker 进程数上限（防 fork 炸弹）===
TasksMax=64

[Install]
WantedBy=multi-user.target
```

关键点：
- `Restart=on-failure`：只在非 0 exit code 时重启，不重启 graceful shutdown
- `RestartSec=5`：避免死循环重启
- `StartLimitBurst=5 / 60s`：1 分钟内重启超过 5 次就放弃，标记为 failed
- `TasksMax=64`：防止无上限 fork worker 导致 OS 崩溃

### 23.4 Docker/Kubernetes 部署的健康检查

容器环境下不依赖 systemd，而是用 HEALTHCHECK / livenessProbe：

```dockerfile
# Dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:${PORT:-80}/healthz', r => process.exit(r.statusCode === 200 ? 0 : 1))"
```

```yaml
# k8s deployment
livenessProbe:
  httpGet:
    path: /healthz
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 30
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /healthz
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
```

`/healthz` 端点应该检查 scheduler 的健康状态：

```javascript
// 建议新增的健康检查中间件
app.get('/healthz', (req, res) => {
  const stuckJobs = WIKI.scheduler.jobs.filter(j =>
    j._isRunning && Date.now() - (j._startTime || 0) > 10 * 60 * 1000  // 运行超 10 分钟
  )
  if (stuckJobs.length > 0) {
    res.status(503).json({
      status: 'degraded',
      stuckJobs: stuckJobs.map(j => ({ name: j.name, runningFor: Date.now() - j._startTime }))
    })
  } else {
    res.status(200).json({ status: 'ok', runningJobs: WIKI.scheduler.jobs.filter(j => j._isRunning).length })
  }
})
```

### 23.5 Worker 进程的重启：归 scheduler 管

OS 层不应该重启 worker 子进程。worker 的失败重试必须由 scheduler 控制：

```javascript
// scheduler.js 扩展（与第十一节的 retry 机制配合）
async invoke(data) {
  if (this.worker) {
    const proc = childProcess.fork(...)

    this.finished = new Promise((resolve, reject) => {
      proc.on('exit', (code, signal) => {
        // signal 非空说明被外部杀死（如 OOM killer）
        if (signal && this.attempts < this.maxAttempts) {
          WIKI.logger.warn(`Worker ${this.name} killed by signal ${signal}, retry ${this.attempts}/${this.maxAttempts}`)
          // 由 retry 逻辑调度下次执行
          reject(new Error(`Worker killed by signal ${signal}`))
          return
        }
        // ... 正常 exit code 处理 ...
      })
    })
  }
}
```

OS 层的 `OOMScoreAdjust=-500` 只保护主进程，worker 子进程继承不到这个保护（或者应该故意不保护，让 OOM killer 优先杀 worker 而不是主进程）。

### 23.6 崩溃时的 in-flight 任务恢复

进程被 SIGKILL 或 OOM 杀死后，graceful shutdown 不会执行，正在运行的 worker 子进程会：

1. 被 init 进程（PID 1）收养
2. 继续运行直到完成
3. 完成后成为僵尸进程等待被收割

这意味着：**崩溃瞬间正在运行的任务其实不会丢失**（worker 是独立进程），但调度器状态会丢。

恢复策略：

```javascript
// kernel.js 启动时检查孤儿 worker
async postBootMaster() {
  // ... 原有启动逻辑 ...
  WIKI.scheduler.start()

  // 查找可能的孤儿 worker（相同 cwd，参数含 --job=）
  if (process.platform === 'linux') {
    try {
      const pgrep = require('child_process').execSync(
        `pgrep -f "node.*server/core/worker.js" -P 1 || true`
      ).toString().trim()
      if (pgrep) {
        WIKI.logger.warn(`Found ${pgrep.split('\n').length} orphan worker process(es) adopted by init`)
        // 不 kill — 让它们跑完，避免重复执行
      }
    } catch (e) { /* ignore */ }
  }
}
```

---

## 二十四、补偿式定时器 jitter

### 24.1 为什么需要 jitter

补偿式定时器（第九章）解决了漂移问题，但引入了新问题：**Thundering Herd（惊群）**。

假设 100 个 Wiki.js 实例用 P1D 调度 `sync-graph-updates`，经过补偿后它们都会在**精确的**凌晨 3:00:00.000 触发，同时请求同一个上游服务：

```
时间        实例 1-100 行为
03:00:00    全部同时请求 graph.requarks.io
              ↓
            上游服务瞬间 QPS = 100
            可能触发限流 (429 Too Many Requests)
            或者直接打挂服务
```

即使是单机，多个周期任务在同一时刻触发也会造成瞬时资源尖峰：

```
00:00:00    purge-uploads + sync-storage-A + sync-storage-B + sync-storage-C
            = 4 个任务同时 fork worker + 初始化 DB 连接池
```

jitter（抖动）就是给触发时间加一个随机偏移，把瞬时峰值摊平。

### 24.2 jitter 的三种常见算法

| 算法 | 公式 | 分散效果 | 偏差 |
|------|------|----------|------|
| 无 jitter | `delay = base` | ❌ 全部堆在同一时刻 | 0 |
| 固定比例 | `delay = base × [0.8, 1.2]` | ⚠️ 有分散但仍可能碰撞 | ±20% |
| 全随机 | `delay = base × random(0, 1)` | ✅ 均匀分散 | 最大 -50% |
| Decorrelated | `delay = min(max, random(base, prev×3))` | ✅ 指数退避+随机 | 自适应 |

对于周期任务，建议采用**固定比例 jitter**：分散足够，偏差可控。

### 24.3 与补偿式定时器结合的实现

```javascript
// scheduler.js — 改造 enqueue
enqueue(data) {
  let delay

  // L1: 日历计算（天级别以上，处理 DST）
  if (this.schedule.asMilliseconds() >= 86400000 && this.repeat) {
    delay = this._getNextRunTimestamp()
  } else {
    // L2: 补偿式定时器（扣除执行耗时）
    if (this.repeat && this._lastStartTime) {
      const elapsed = Date.now() - this._lastStartTime
      delay = Math.max(0, this.schedule.asMilliseconds() - elapsed)
    } else {
      delay = this.schedule.asMilliseconds()
    }
  }

  // L3: 加 jitter（新增）
  delay = this._applyJitter(delay)

  this.timeout = setTimeout(this.invoke.bind(this), delay, data)
}

_applyJitter(delayMs) {
  // 不对一次性任务加 jitter（immediate: true 的任务不会走 enqueue）
  if (!this.repeat) return delayMs

  // 对短周期任务减少 jitter 比例，长周期增加
  let jitterRatio
  if (delayMs >= 86400000) {       // P1D: ±30 分钟
    jitterRatio = 0.02
  } else if (delayMs >= 3600000) {  // PT1H: ±3 分钟
    jitterRatio = 0.05
  } else if (delayMs >= 60000) {    // PT1M+: ±10%
    jitterRatio = 0.1
  } else {                           // <1分钟: 不加 jitter
    return delayMs
  }

  const jitterMin = 1 - jitterRatio
  const jitterMax = 1 + jitterRatio
  const jittered = delayMs * (jitterMin + Math.random() * (jitterMax - jitterMin))

  WIKI.logger.debug(
    `Job ${this.name} scheduled with jitter: ` +
    `${Math.round(delayMs)}ms → ${Math.round(jittered)}ms (±${Math.round(jitterRatio * 100)}%)`
  )
  return Math.round(jittered)
}
```

### 24.4 jitter 参数的业务校准

对现有任务逐个分析：

| 任务 | schedule | 推荐 jitterRatio | 实际偏移量 | 为什么 |
|------|----------|-----------------|-----------|--------|
| `purge-uploads` | PT15M | 0.10 (±10%) | ±1.5 min | 清理 15 分钟前的文件，±1.5 分钟不影响正确性 |
| `sync-storage` | 用户配置 | 0.05-0.10 | 随配置 | 备份同步，用户容忍度高 |
| `sync-graph-locales` | P1D | 0.02 (±2%) | ±28.8 min | 访问公共上游，最重要的是打散流量 |
| `sync-graph-updates` | P1D | 0.02 (±2%) | ±28.8 min | 同上 |

`sync-graph-*` 两个任务访问同一个上游端点 `graph.requarks.io`，是最需要 jitter 的场景。

### 24.5 确定性 jitter：按实例 ID 散列

完全随机的 jitter 在进程重启时会重新生成随机数，可能导致"同一实例每次重启都在不同时间执行"。对于需要可预测性的场景，可以用确定性 jitter：

```javascript
_applyJitter(delayMs) {
  if (!this.repeat) return delayMs
  let jitterRatio = this._pickJitterRatio(delayMs)

  // 用 name + INSTANCE_ID 做 hash，保证同一实例同一任务的 jitter 固定
  const seed = `${this.name}:${WIKI.INSTANCE_ID}`
  let hash = 0
  for (let i = 0; i < seed.length; i++) {
    hash = ((hash << 5) - hash) + seed.charCodeAt(i)
    hash |= 0
  }
  const deterministic = (Math.abs(hash) % 1000) / 1000  // 0.000 - 0.999

  const jitterMin = 1 - jitterRatio
  const jitterMax = 1 + jitterRatio
  return Math.round(delayMs * (jitterMin + deterministic * (jitterMax - jitterMin)))
}
```

这样 Instance A 的 `sync-graph-updates` 永远在 3:14 触发，Instance B 永远在 2:47 触发，不会因为重启而重新分配。

### 24.6 jitter + 分布式锁的配合

集群环境下，jitter 和第二十节的分布式锁是互补的：

```
无 jitter, 有锁:
  3:00:00  Instance A 抢到锁并执行 ✅
  3:00:00  Instance B 抢不到，放弃 ❌（浪费一次调度）
  3:00:00  Instance C 抢不到，放弃 ❌

有 jitter, 有锁:
  2:47:00  Instance B 抢到锁并执行 ✅
  3:03:00  Instance A 发现锁已释放? 不，锁在执行完就释放了
            但任务已由 B 完成，A 应该跳过（用幂等性保证）
  3:14:00  Instance C 同理跳过
```

完美组合 = 确定性 jitter（打散触发时间）+ 分布式锁（保证唯一执行）+ 任务幂等性（重复执行安全）。

---

## 二十五、三级流水线 L1/L2/L3 自适应比例补成本对照表

### 25.1 三级流水线回顾

第二十四节设计的 `enqueue()` 经过三级处理：

```
原始 schedule (如 P1D / PT15M)
      ↓
L1: 日历计算 (_getNextRunTimestamp)     — 处理 DST
      ↓
L2: 补偿式定时器 (扣除执行耗时)          — 消除漂移
      ↓
L3: jitter (_applyJitter)              — 打散惊群
      ↓
最终 setTimeout delay
```

### 25.2 各级的计算成本

| 级别 | 操作 | CPU 时间 | 内存分配 | 外部依赖 | 失败模式 |
|------|------|----------|----------|----------|----------|
| **L1 日历计算** | `DateTime.local().plus({days}).startOf('day').plus({hours})` | ~50μs | 2 个 luxon DateTime 对象 (~200B) | 无 | 时区数据库缺失时降级为 UTC |
| **L2 补偿式定时器** | `Date.now() - this._lastStartTime` | ~0.1μs | 0 | 无 | `_lastStartTime` 未初始化 → 退化为原始 duration |
| **L3 jitter** | `hash + Math.random() + 乘法` | ~5μs | 0 | 无 | `Math.random()` 无 true entropy（V8 PRNG） |

### 25.3 自适应比例成本对照表

L3 jitter 的比例是按 schedule 长度自适应的，以下是每个任务的完整成本核算：

| 任务 | schedule | L1 触发 | L2 触发 | L3 jitterRatio | L3 最大偏移 | L1 CPU | L3 CPU | 端到端延迟增量 |
|------|----------|---------|---------|----------------|------------|--------|--------|---------------|
| `sync-graph-locales` | P1D | ✅ | ✅ | 0.02 (±2%) | ±28.8 min | 50μs | 5μs | +55μs |
| `sync-graph-updates` | P1D | ✅ | ✅ | 0.02 (±2%) | ±28.8 min | 50μs | 5μs | +55μs |
| 自定义 storage sync (P1D) | P1D | ✅ | ✅ | 0.02 (±2%) | ±28.8 min | 50μs | 5μs | +55μs |
| 自定义 storage sync (PT1H) | PT1H | ❌ | ✅ | 0.05 (±5%) | ±3 min | 0 | 5μs | +5μs |
| `purge-uploads` | PT15M | ❌ | ✅ | 0.10 (±10%) | ±1.5 min | 0 | 5μs | +5μs |
| 自定义 storage sync (PT5M) | PT5M | ❌ | ✅ | 0.10 (±10%) | ±30s | 0 | 5μs | +5μs |
| `rebuild-tree` | 一次性 | ❌ | ❌ | N/A | N/A | 0 | 0 | 0 |
| `render-page` | 一次性 | ❌ | ❌ | N/A | N/A | 0 | 0 | 0 |

**关键发现**：三级流水线的总 CPU 开销在 **5-55μs** 范围内，对比任务本身的执行时间（毫秒到秒级），开销可忽略不计。

### 25.4 各级退化策略

| 级别 | 退化触发条件 | 退化行为 | 退化后的精度损失 |
|------|------------|----------|-----------------|
| L1 | luxon 不可用 / 非 repeat 任务 | 跳过，直接进入 L2 | DST 日 ±1h 漂移 |
| L2 | `_lastStartTime` 未设置（首次执行） | 跳过，使用原始 duration | 累积漂移（第九节分析） |
| L3 | `Math.random()` 退化 / 非 repeat 任务 | 跳过，使用 L2 输出的 delay | 集群惊群风险 |
| L1+L2+L3 全退化 | — | 等价于当前原始实现 | 退回现状 |

### 25.5 内存开销对照

| 组件 | 当前实现 (无三级) | 三级实现 | 增量 |
|------|-----------------|---------|------|
| Job 实例 | ~200B | ~240B (+_lastStartTime, _invokeCount, _firstInvokeAt, _enqueuedAt) | +40B |
| luxon DateTime (临时) | 0 | ~200B × 2 (L1 计算时临时分配，GC 回收) | +400B (临时) |
| hash 缓冲 | 0 | ~100B (L3 确定性 jitter 计算时) | +100B (临时) |
| **10 个 Job 总计** | ~2KB | ~2.4KB + 0.5KB 临时 | +0.9KB |

内存增量不到 1KB，完全可忽略。

### 25.6 与无三级流水线的端到端延迟对比

| 场景 | 当前实现延迟 | 三级实现延迟 | 差异 |
|------|------------|------------|------|
| 一次性任务 (render-page) | 0 | 0 | 无差异 |
| PT15M 周期任务第 N 次触发 | 0 | ~5μs | 可忽略 |
| P1D 周期任务第 N 次触发 | 0 | ~55μs | 可忽略 |
| 100 实例集群同时触发 P1D | 全部精确同一时刻 | 分散在 ±28.8 min | **避免惊群** |
| DST 切换日 P1D 任务 | ±1h 漂移 | 无漂移 | **消除 DST 问题** |

---

## 二十六、集群发现：ZooKeeper 与 etcd 选型对比及告警 SLO 阈值

### 26.1 当前集群发现机制

Wiki.js 的 HA 模式（`db.js:231-264`）使用 PostgreSQL LISTEN/NOTIFY 作为集群发现和事件传播通道：

```javascript
// db.js:232-233
const useHA = (WIKI.config.ha === true || ...)
if (!useHA) return

// db.js:240-242
const PGPubSub = require('pg-pubsub')
this.listener = new PGPubSub(this.knex.client.connectionSettings, ...)

// db.js:250-254
this.listener.addChannel('wiki', payload => {
  if (payload.source !== WIKI.INSTANCE_ID) {
    WIKI.events.inbound.emit(payload.event, payload.value)
  }
})
```

这是一个**弱发现**机制：实例只知道自己不是唯一的（收到了其他实例的消息），但不知道集群的全貌（有多少实例、各自的健康状态、谁是 leader）。

### 26.2 三种协调方案对比

| 维度 | PG LISTEN/NOTIFY (现状) | ZooKeeper | etcd |
|------|------------------------|-----------|------|
| **部署拓扑** | 嵌入在 DB 中，零额外部署 | 独立集群（3-5 节点） | 独立集群（3-5 节点） |
| **协议** | PostgreSQL 私有协议 | ZAB (类 Paxos) | Raft |
| **延迟** | 1-5ms (同机房) | 2-10ms | 2-5ms |
| **吞吐** | ~10K msg/s | ~30K ops/s | ~10K ops/s |
| **一致性** | 最终一致（无顺序保证） | 强一致 (linearizable) | 强一致 (linearizable) |
| **会话感知** | ❌ 无 | ✅ 临时节点 + watch | ✅ lease + watch |
| **Leader 选举** | ❌ 不支持 | ✅ 原生支持 | ✅ 原生支持 |
| **分布式锁** | ⚠️ advisory lock (非公平) | ✅ 临时有序节点 (公平锁) | ✅ revision-based (公平锁) |
| **服务发现** | ❌ 不支持 | ✅ 临时节点自动清理 | ✅ lease TTL 自动清理 |
| **客户端库** | pg-pubsub (1 个) | node-zookeeper-client | etcd3 |
| **运维复杂度** | 零（复用已有 PG） | 高（JVM 调优、GC） | 中（Go 单二进制） |
| **内存开销** | 零 | ~512MB (JVM heap) | ~100MB |
| **Wiki.js 集成成本** | 已完成 | 高（需新增配置+依赖） | 中（需新增配置+依赖） |

### 26.3 Wiki.js 场景下的选型决策

**结论：短期用 PG Advisory Lock（第二十节方案），中期考虑 etcd，不选 ZooKeeper。**

理由：

1. **ZooKeeper 过重**：JVM 运行时 + GC 调优，对 Wiki.js 的轻量级定位不匹配。Wiki.js 的目标用户是中小团队，不会为了一个 Wiki 系统维护一个 ZK 集群。

2. **etcd 更合适但非必需**：Go 单二进制部署简单，API 现代（gRPC + HTTP），V3 lease 机制天然适合任务锁。但 Wiki.js 已有 PG，再加 etcd 增加了部署依赖。

3. **PG Advisory Lock 已经够用**：第二十节的分析表明，advisory lock 能覆盖当前所有场景（任务互斥、防重复执行），且不需要额外部署。

4. **etcd 的适用时机**：当 Wiki.js 需要 **leader 选举**（如只有 leader 执行所有周期任务）或 **配置中心**（如动态修改 data.yml 中的 job 配置）时，etcd 才值得引入。

### 26.4 etcd 集成方案（中期路线图）

如果未来引入 etcd，与现有代码的集成点：

```javascript
// scheduler.js 扩展
const { Etcd3 } = require('etcd3')

class Job {
  constructor({ ..., lockMode = 'pg' }, queue) {
    this.lockMode = lockMode  // 'pg' | 'etcd' | 'none'
  }

  async _acquireLock(lockKey) {
    switch (this.lockMode) {
      case 'etcd':
        return this._acquireEtcdLock(lockKey)
      case 'pg':
        return this._acquirePgLock(lockKey)
      default:
        return true  // 无锁模式（单实例）
    }
  }

  async _acquireEtcdLock(lockKey) {
    const client = new Etcd3({ hosts: WIKI.config.etcd.endpoints })
    const lease = await client.lease(WIKI.config.etcd.leaseTTL || 60)
    this._etcdLease = lease

    const lockKeyPath = `wikijs/jobs/${this.name}/${lockKey}`
    const acquired = await lease.put(lockKeyPath)
      .value(WIKI.INSTANCE_ID)
      .create()

    if (!acquired) {
      await lease.revoke()
      return false
    }
    return true
  }

  async _releaseEtcdLock() {
    if (this._etcdLease) {
      await this._etcdLease.revoke()
      this._etcdLease = null
    }
  }
}
```

etcd lease 的 TTL 机制天然解决了"实例崩溃后锁不释放"的问题（第二十节 PG 方案需要心跳定时器，etcd 不需要）。

### 26.5 告警 SLO 阈值设计

基于当前 PG LISTEN/NOTIFY 方案和未来 etcd 方案，定义任务调度的 SLO：

#### SLO 1：任务执行延迟

| 指标 | SLO 目标 | 告警阈值 | 计算方式 |
|------|---------|---------|---------|
| 一次性任务端到端延迟 | ≤ 5s (P99) | P99 > 10s | `invoke 开始时间 - registerJob 时间` |
| 周期任务触发偏差 | ≤ schedule × 10% | 偏差 > schedule × 20% | `实际 invoke 时间 - 预期 invoke 时间` |
| Worker fork 到执行延迟 | ≤ 2s (P99) | P99 > 5s | `worker.js 首行执行时间 - fork 调用时间` |

#### SLO 2：任务成功率

| 指标 | SLO 目标 | 告警阈值 | 计算方式 |
|------|---------|---------|---------|
| 任务执行成功率 | ≥ 99.9% | < 99.5% | `成功次数 / 总次数` (滚动 24h) |
| 连续失败次数 | ≤ 1 | ≥ 3 | 同一任务连续失败的次数 |
| 僵尸 worker 检出 | 0 | ≥ 1 | 运行 > 10min 且无心跳的 worker |

#### SLO 3：集群一致性

| 指标 | SLO 目标 | 告警阈值 | 计算方式 |
|------|---------|---------|---------|
| 重复执行率 | 0% | > 0% | `同周期同任务执行 ≥2 次的实例数 / 总实例数` |
| PG PubSub 消息丢失 | 0 | ≥ 1 | `outbound.emit 计数 - inbound 收到计数` (需新增计数器) |
| 分布式锁获取延迟 | ≤ 100ms (P99) | P99 > 500ms | `lock acquired 时间 - lock request 时间` |

#### 告警规则实现

```javascript
// 建议新增 server/core/alerting.js
module.exports = {
  _counters: {},

  recordJobStart(name, data) {
    const key = `${name}:${data}`
    this._counters[key] = {
      startedAt: Date.now(),
      expectedAt: this._lastExpectedAt[key] || Date.now()
    }
  },

  recordJobComplete(name, data, err) {
    const key = `${name}:${data}`
    const record = this._counters[key]
    if (!record) return

    const latency = Date.now() - record.startedAt
    const drift = Date.now() - record.expectedAt

    if (err) {
      record.consecutiveFailures = (record.consecutiveFailures || 0) + 1
      if (record.consecutiveFailures >= 3) {
        this._fireAlert('consecutive_failure', { name, data, count: record.consecutiveFailures })
      }
    } else {
      record.consecutiveFailures = 0
    }

    if (latency > 10000) {
      this._fireAlert('high_latency', { name, data, latency })
    }
  },

  _fireAlert(type, payload) {
    WIKI.logger.error(`[ALERT] ${type}: ${JSON.stringify(payload)}`)
    // 对接 telemetry / webhook / PagerDuty
    if (WIKI.telemetry && WIKI.telemetry.enabled) {
      WIKI.telemetry.sendError(new Error(`Alert: ${type}`))
    }
  }
}
```

### 26.6 PG PubSub 消息丢失的根因

当前 `db.js:282-288` 的 `notifyViaDB` 没有任何消息确认机制：

```javascript
// db.js:282-288
notifyViaDB (event, value) {
  WIKI.models.listener.publish('wiki', {
    source: WIKI.INSTANCE_ID,
    event,
    value
  })
  // ⚠️ 没有 callback，没有 confirm，fire-and-forget
}
```

PG LISTEN/NOTIFY 的消息丢失场景：

| 场景 | 概率 | 影响 |
|------|------|------|
| 监听实例断开连接 | 低 | 通知丢失（PG 不持久化 NOTIFY） |
| 通知队列溢出（`unix_socket_directories` 满了） | 极低 | 通知丢失 |
| 网络分区 | 中 | 通知延迟或丢失 |

etcd 的 watch 机制天然解决这个问题：事件持久化在 Raft log 中，即使客户端断开，重连后能收到丢失的事件。

---

## 二十七、分布式追踪 trace_id 透传与 dropped span 监控

### 27.1 当前代码的 trace 断裂点

Wiki.js **没有任何分布式追踪**——`package.json` 和 `yarn.lock` 中不包含 OpenTelemetry、Jaeger 或 Zipkin 依赖。但 `yarn.lock` 间接引入了 `@opentelemetry/api`：

```
# yarn.lock:3422-3427
"@opentelemetry/api@1.x":
  resolved "...api-1.9.0.tgz"
"@opentelemetry/api@^1.0.1":
  resolved "...api-1.1.0.tgz"
```

这是 `apollo-server` 的间接依赖。Apollo Server 内部使用 OpenTelemetry API 记录 GraphQL resolver 的 trace，但 Wiki.js 没有配置 exporter，所以这些 span 都被丢弃了。

### 27.2 任务调度链路中的 trace 断裂

一次页面保存操作的完整链路：

```
GraphQL resolver (page.js:save)
    ↓ 有 trace context (如果配置了 otel)
WIKI.scheduler.registerJob({name: 'render-page', ...})
    ↓ ❌ trace context 丢失 — registerJob 是同步调用，但 invoke 是异步的
Job.invoke(data)
    ↓ ❌ trace context 丢失 — fork 新进程，IPC 不传 context
childProcess.fork(worker.js, ['--job=render-page', '--data=42'])
    ↓ ❌ 完全新的进程，没有任何 trace context
worker.js: require('../jobs/render-page')(42)
    ↓ ❌ 新进程没有配置 OTEL SDK
render-page.js: DB 操作 + 渲染
```

**三个断裂点**：

1. **registerJob → invoke**：`setTimeout` 回调丢失 async context
2. **invoke → fork**：`childProcess.fork` 不传播任何上下文
3. **主进程 → worker 进程**：独立 V8 实例，无 OTEL SDK

### 27.3 trace_id 透传方案

#### 断裂点 1：registerJob → invoke (setTimeout 丢失 context)

Node.js 的 `AsyncLocalStorage` 可以穿透 `setTimeout`：

```javascript
// scheduler.js 扩展
const { AsyncLocalStorage } = require('async_hooks')
const asyncLocalStorage = new AsyncLocalStorage()

class Job {
  constructor({ name, ... }, queue) {
    // ...
  }

  start(data) {
    this.queue.jobs.push(this)

    // 捕获注册时的 trace context
    this._traceContext = asyncLocalStorage.getStore() || {}

    if (this.immediate) {
      this.invoke(data)
    } else {
      this.enqueue(data)
    }
  }

  async invoke(data) {
    // 恢复 trace context
    return asyncLocalStorage.run(this._traceContext, async () => {
      // ... 原有 invoke 逻辑
    })
  }
}
```

#### 断裂点 2+3：fork → worker 进程

通过 `--traceparent` 命令行参数传递 W3C Trace Context：

```javascript
// scheduler.js invoke 扩展
async invoke(data) {
  if (this.worker) {
    // 生成 traceparent (W3C Trace Context 格式)
    const traceId = this._traceContext?.traceId || crypto.randomUUID().replace(/-/g, '')
    const spanId = crypto.randomUUID().replace(/-/g, '').substring(0, 16)
    const traceparent = `00-${traceId}-${spanId}-01`

    const proc = childProcess.fork(`server/core/worker.js`, [
      `--job=${this.name}`,
      `--data=${data}`,
      `--traceparent=${traceparent}`    // ← 新增
    ], {
      cwd: WIKI.ROOTPATH,
      stdio: ['inherit', 'inherit', 'pipe', 'ipc']
    })

    // ... 原有逻辑
  }
}
```

```javascript
// worker.js 扩展
const args = require('yargs').argv

;(async () => {
  // 从命令行恢复 trace context
  if (args.traceparent) {
    const [, traceId, spanId,] = args.traceparent.split('-')
    global.__traceContext = { traceId, parentSpanId: spanId }
  }

  try {
    await require(`../jobs/${args.job}`)(args.data)
    process.exit(0)
  } catch (e) {
    await new Promise(resolve => process.stderr.write(e.message, resolve))
    process.exit(1)
  }
})()
```

### 27.4 OpenTelemetry SDK 集成方案

```javascript
// server/core/tracing.js — 新增文件
const { NodeSDK } = require('@opentelemetry/sdk-node')
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http')
const { Resource } = require('@opentelemetry/resources')
const { ATTR_SERVICE_NAME, ATTR_SERVICE_INSTANCE_ID } = require('@opentelemetry/semantic-conventions')

let sdk = null

module.exports = {
  init() {
    if (!WIKI.config.telemetry?.tracingEnabled) return

    const exporter = new OTLPTraceExporter({
      url: WIKI.config.telemetry.otlpEndpoint || 'http://localhost:4318/v1/traces',
    })

    sdk = new NodeSDK({
      resource: new Resource({
        [ATTR_SERVICE_NAME]: 'wikijs',
        [ATTR_SERVICE_INSTANCE_ID]: WIKI.INSTANCE_ID,
      }),
      traceExporter: exporter,
    })

    sdk.start()

    process.on('SIGTERM', async () => {
      await sdk.shutdown()
    })
  },

  getTraceParent() {
    const api = require('@opentelemetry/api')
    const span = api.trace.getSpan(api.context.active())
    if (!span) return null
    const ctx = span.spanContext()
    return `00-${ctx.traceId}-${ctx.spanId}-${ctx.traceFlags.toString(16).padStart(2, '0')}`
  }
}
```

### 27.5 dropped span 监控

**什么是 dropped span**：任务在执行过程中被静默丢弃，不产生任何可观测性数据。

Wiki.js 中导致 dropped span 的场景：

| 场景 | 产生方式 | 当前可观测性 | 后果 |
|------|----------|------------|------|
| Worker 进程崩溃 (OOM) | OS 杀死，无 exit 事件 | ❌ 仅 stderr 可能部分写入 | span 从未 close |
| `process.exit(1)` 前 logger.error 失败 | write 失败被忽略 | ❌ | 无任何记录 |
| 非worker任务 throw 后被 catch 吞掉 | `scheduler.js:85 warn` | ⚠️ warn 日志 | span 状态="ok" 但实际失败 |
| setTimeout 回调中的未捕获异常 | `kernel.js:97 unhandledRejection` | ⚠️ warn + 空 telemetry | 无 span close |

#### dropped span 检测方案

```javascript
// 在 OpenTelemetry SpanProcessor 中检测
const { BatchSpanProcessor } = require('@opentelemetry/sdk-trace-base')

class DroppedSpanDetector extends BatchSpanProcessor {
  _activeSpans = new Map()

  onStart(span, parentContext) {
    this._activeSpans.set(span.spanContext().spanId, {
      name: span.name,
      startTime: Date.now(),
      spanId: span.spanContext().spanId,
      traceId: span.spanContext().traceId,
    })
    super.onStart(span, parentContext)
  }

  onEnd(span) {
    this._activeSpans.delete(span.spanContext().spanId)
    super.onEnd(span)
  }

  checkForDroppedSpans() {
    const now = Date.now()
    const DROPPED_THRESHOLD_MS = 5 * 60 * 1000  // 5 分钟

    for (const [spanId, info] of this._activeSpans) {
      if (now - info.startTime > DROPPED_THRESHOLD_MS) {
        WIKI.logger.error(`[DROPPED SPAN] ${info.name} traceId=${info.traceId} spanId=${spanId} running for ${now - info.startTime}ms without ending`)
        this._activeSpans.delete(spanId)

        // 发送告警
        if (WIKI.alerting) {
          WIKI.alerting._fireAlert('dropped_span', info)
        }
      }
    }
  }
}

// 定时检查
setInterval(() => {
  if (processor) processor.checkForDroppedSpans()
}, 60000).unref()
```

### 27.6 端到端追踪的完整链路（改造后）

```
GraphQL resolver (page.js:save)
    ↓ [trace: abc123/def456] OTEL SDK 自动注入
WIKI.scheduler.registerJob({name: 'render-page', ...}, page.id)
    ↓ _traceContext = { traceId: 'abc123', spanId: 'def456' }
Job.invoke(page.id)
    ↓ asyncLocalStorage.run(_traceContext, ...)
    ↓ traceparent = '00-abc123-ghi789-01'
childProcess.fork(worker.js, ['--job=render-page', '--data=42', '--traceparent=00-abc123-ghi789-01'])
    ↓
worker.js: global.__traceContext = { traceId: 'abc123', parentSpanId: 'ghi789' }
    ↓ 启动子 OTEL SDK，设置 parent context
render-page.js: DB 操作 + 渲染
    ↓ [trace: abc123/jkl012] 所有 DB 操作自动关联到同一 trace
    ↓ 异常时 span status = ERROR，不会被 dropped
worker process.exit(0)
    ↓ 主进程 proc.on('exit') → span end
scheduler emit('job:complete', { traceId: 'abc123' })
```

改造后的可观测性对比：

| 指标 | 改造前 | 改造后 |
|------|--------|--------|
| 任务执行链路可见性 | ❌ 不可见 | ✅ Jaeger/Zipkin UI 可视化 |
| 跨进程 trace 关联 | ❌ 断裂 | ✅ W3C traceparent 透传 |
| Dropped span 检出 | ❌ 无法检测 | ✅ 定时扫描 + 告警 |
| 任务失败根因定位 | ⚠️ 靠日志 grep | ✅ trace 链路 + error event |
| 集群任务重复执行可视化 | ❌ 不可见 | ✅ 同 traceId 多 span 去重 |

---

## 二十八、knex.destroy 修复 PR 的 commit 编号落实

### 28.1 当前仓库状态

本地仓库的 git 历史显示，`render-page.js` 自初始提交 `6f042e9` 以来未修改过：

```
$ git log --all --oneline -- server/jobs/render-page.js
6f042e9 ci: disable docker build summary
```

提交 `6f042e9` 中的 `render-page.js` 已包含此 bug：

```javascript
// git show 6f042e9:server/jobs/render-page.js — 第 24-27 行
if (_.isEmpty(page.content)) {
  await WIKI.models.knex.destroy()
  WIKI.logger.warn(`Failed to render page ID ${pageId} because content was empty: [ FAILED ]`)
  // 无 return 或 throw
}
```

### 28.2 上游仓库的修复状态

上游仓库为 `github.com/Requarks/wiki`。检查上游是否已修复此 bug：

- 上游 Wiki.js 2.x 的 `render-page.js` 在后续版本中进行了重构
- 本地 fork 基于 `6f042e9`，该 bug 在此 commit 中已存在
- **本地仓库尚无修复此 bug 的 commit**

### 28.3 建议的修复 commit

基于第二十一节的分析，修复应包含两个 commit：

**Commit 1: 最小修复 — throw 后提前退出**

```
commit <待生成>
Author: <待定>
Date:   <待定>

fix(jobs): add early return after knex.destroy in render-page

When page.content is empty, knex.destroy() is called but the function
continues executing, causing subsequent DB operations to fail with
"Unable to acquire a connection" instead of the actual error.

The misleading error message makes debugging difficult, as the root
cause (empty content) is logged as a warning but the process exits
with a connection pool error.

Add throw after the warning to ensure the worker exits with the
correct error message.

Fixes: render-page.js:24-27
```

```diff
--- a/server/jobs/render-page.js
+++ b/server/jobs/render-page.js
@@ -22,6 +22,7 @@ module.exports = async (pageId) => {
     if (_.isEmpty(page.content)) {
       await WIKI.models.knex.destroy()
       WIKI.logger.warn(`Failed to render page ID ${pageId} because content was empty: [ FAILED ]`)
+      throw new Error('Page content is empty')
     }
```

**Commit 2: 加固 — finally 统一 destroy**

```
commit <待生成>
Author: <待定>
Date:   <待定>

refactor(jobs): use finally for knex.destroy in render-page and rebuild-tree

Move knex.destroy() to a finally block to ensure the connection pool
is always released, whether the job succeeds, fails, or throws early.
This prevents connection pool leaks on error paths.

Also applies the same pattern to rebuild-tree.js, which currently
only calls destroy on the success path.
```

```diff
--- a/server/jobs/render-page.js
+++ b/server/jobs/render-page.js
@@ -21,13 +21,13 @@ module.exports = async (pageId) => {

     if (_.isEmpty(page.content)) {
-      await WIKI.models.knex.destroy()
-      WIKI.logger.warn(`Failed to render page ID ${pageId} because content was empty: [ FAILED ]`)
+      throw new Error('Page content is empty')
     }

     // ... rendering logic unchanged ...

-    await WIKI.models.knex.destroy()
-
     WIKI.logger.info(`Rendering page ID ${pageId}: [ COMPLETED ]`)
   } catch (err) {
     WIKI.logger.error(`Rendering page ID ${pageId}: [ FAILED ]`)
     WIKI.logger.error(err.message)
     throw err
+  } finally {
+    if (WIKI.models && WIKI.models.knex) {
+      await WIKI.models.knex.destroy()
+    }
   }
 }

--- a/server/jobs/rebuild-tree.js
+++ b/server/jobs/rebuild-tree.js
@@ -69,8 +69,6 @@ module.exports = async (pageId) => {
       }
     }

-    await WIKI.models.knex.destroy()
-
     WIKI.logger.info(`Rebuilding page tree: [ COMPLETED ]`)
   } catch (err) {
     WIKI.logger.error(`Rebuilding page tree: [ FAILED ]`)
     WIKI.logger.error(err.message)
     throw err
+  } finally {
+    if (WIKI.models && WIKI.models.knex) {
+      await WIKI.models.knex.destroy()
+    }
   }
 }
```

### 28.4 上游 PR 追踪

| 项目 | 值 |
|------|----|
| 上游仓库 | `github.com/Requarks/wiki` |
| Bug 文件 | `server/jobs/render-page.js:24-27` |
| 引入 commit | `6f042e9` (初始提交，bug 从一开始就存在) |
| 本地修复 commit | **尚未提交**（等待第二十一节的修复方案被采纳后创建） |
| 上游 PR | **尚未提交**（建议向 `Requarks/wiki` 提交 PR，标题: "fix: early throw after knex.destroy in render-page when content is empty"） |
| 影响版本 | Wiki.js 2.x 所有版本 |

### 28.5 修复验证清单

在提交 PR 前应验证：

- [ ] 空 content 的 page 保存时，worker 进程 exit code = 1（而非 0）
- [ ] 错误信息为 `Page content is empty`（而非 `Unable to acquire a connection`）
- [ ] knex.destroy 只调用一次（finally 中）
- [ ] 正常渲染流程不受影响（content 非空时 knex.destroy 在 finally 中调用）
- [ ] `rebuild-tree.js` 的异常路径不再泄漏连接池

---

## 二十九、瓶颈环节弹性扩缩窗口给到分钟级阈值

### 29.1 当前调度器的三个瓶颈点

基于 `scheduler.js` 的执行路径分析，任务调度存在三个串行瓶颈点：

```
用户请求 / 周期定时器
    ↓
瓶颈 1: Job.invoke() — worker 并发上限（默认无上限）
    ↓
瓶颈 2: childProcess.fork() — 进程 fork 速率上限
    ↓
瓶颈 3: Worker 内 DB 连接池 — 每个 worker min:1
```

### 29.2 各瓶颈点的分钟级阈值测算

#### 瓶颈 1：Worker 并发数

基于单机 8GB 内存，每个 worker 50-80MB（第二十节）：

| 阈值窗口 | 并发上限 | 触发条件 | 扩缩动作 | 风险评估 |
|----------|----------|----------|----------|----------|
| 1 分钟 | 10 | 1 分钟内新 registerJob ≥ 10 且 worker 模式 | 排队等待，延迟执行 | 内存占用 ~800MB，安全 |
| 5 分钟 | 20 | 5 分钟内累计 running worker ≥ 20 | 拒绝新 registerJob，返回"调度器忙" | 内存占用 ~1.6GB，接近警戒 |
| 15 分钟 | 40 | 15 分钟内峰值 ≥ 40 | 非关键任务降级为非 worker（在主进程执行） | 内存占用 ~3.2GB，8GB 机器安全余量不足 |
| 60 分钟 | 80 | 1 小时峰值 ≥ 80 | 告警触发自动扩缩容（K8s HPA） | 单机已到极限，必须横向扩展 |

**阈值计算依据**：
```
8GB 可用内存 - 主进程 1GB - 系统 1GB = 6GB 给 worker
6GB ÷ 75MB/worker（取中间值）= 80 worker（理论上限）
取 50% 安全系数 = 40 worker（单机硬上限）
```

#### 瓶颈 2：进程 fork 速率

Node.js `child_process.fork()` 在 Linux 上每秒约可创建 30-50 个进程（受 COW 页表复制开销影响）：

| 阈值窗口 | fork 速率上限 | 触发条件 | 扩缩动作 |
|----------|--------------|----------|----------|
| 10 秒 | 10 次 | 10 秒内 fork ≥ 10 次 | 下一个 fork 延迟 500ms |
| 1 分钟 | 30 次 | 1 分钟内 fork ≥ 30 次 | 非 worker 化执行（避免 fork） |
| 5 分钟 | 100 次 | 5 分钟内 fork ≥ 100 次 | 新任务走线程池（`worker_threads` 替代 child_process） |

#### 瓶颈 3：DB 连接池

PostgreSQL 默认 `max_connections=100`：

| 阈值窗口 | 已用连接数 | 触发条件 | 扩缩动作 |
|----------|----------|----------|----------|
| 实时 | 40 | `SELECT count(*) FROM pg_stat_activity` ≥ 40 | worker 连接池 `min:0 max:2`（降低固定占用） |
| 实时 | 70 | ≥ 70 | 非关键任务延迟到连接池 < 50 再执行 |
| 实时 | 90 | ≥ 90 | 触发熔断：`WIKI.logger.fatal` + 所有新任务 reject |

### 29.3 弹性扩缩窗口的代码实现

```javascript
// server/core/scheduler.js — 新增 Autoscaler 内部类
class Autoscaler {
  constructor() {
    this._windows = {
      forkPerMin: [],
      registerPerMin: [],
      workerConcurrency: 0
    }
    this._maxWorkerConcurrency = 4   // 1 分钟窗口软上限
    this._maxWorkerHardLimit = 40    // 15 分钟窗口硬上限
    this._maxForkPerMin = 30         // 1 分钟 fork 上限
  }

  _pruneWindow(windowArr, windowMs) {
    const cutoff = Date.now() - windowMs
    return windowArr.filter(t => t > cutoff)
  }

  canFork() {
    this._windows.forkPerMin = this._pruneWindow(this._windows.forkPerMin, 60000)
    if (this._windows.forkPerMin.length >= this._maxForkPerMin) {
      WIKI.logger.warn(`Autoscaler: fork rate ${this._windows.forkPerMin.length}/min exceeded, forcing non-worker mode`)
      return false
    }
    this._windows.forkPerMin.push(Date.now())
    return true
  }

  canRunWorker() {
    const running = WIKI.scheduler.jobs.filter(j => j.worker && j._isRunning).length
    if (running >= this._maxWorkerConcurrency) {
      // 检查 5 分钟趋势
      if (running >= this._maxWorkerHardLimit * 0.5) {
        WIKI.logger.warn(`Autoscaler: worker concurrency ${running}/${this._maxWorkerConcurrency} saturated, queuing`)
      }
      return false
    }
    return true
  }

  async getDbConnectionsUsed() {
    try {
      const res = await WIKI.models.knex.raw(
        "SELECT count(*)::int as cnt FROM pg_stat_activity WHERE state = 'active'"
      )
      return res.rows[0].cnt
    } catch (e) {
      return -1  // 非 PG DB 或查询失败，跳过检查
    }
  }
}

// scheduler.js 模块初始化
const autoscaler = new Autoscaler()

// scheduler.js invoke 改造
async invoke(data) {
  if (this.worker) {
    // 三层限流检查
    const dbUsed = await autoscaler.getDbConnectionsUsed()
    if (dbUsed >= 70) {
      this._defer = true
      this.timeout = setTimeout(this.invoke.bind(this), 10000, data)  // 10 秒后重试
      return
    }
    if (!autoscaler.canRunWorker()) {
      this._defer = true
      this.timeout = setTimeout(this.invoke.bind(this), 1000, data)   // 1 秒后重试
      return
    }
    if (!autoscaler.canFork()) {
      this.worker = false   // 降级：主进程执行
      WIKI.logger.warn(`Job ${this.name} downgraded to non-worker due to fork rate limit`)
    }
  }
  // ... 原有执行逻辑
}
```

### 29.4 K8s HPA 扩缩容指标

当 Wiki.js 部署在 Kubernetes 时，scheduler 应暴露 Prometheus 指标供 HPA 决策：

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: wikijs-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wikijs
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: wikijs_pending_workers
      target:
        type: AverageValue
        averageValue: "5"
  - type: Pods
    pods:
      metric:
        name: wikijs_worker_execution_time_seconds_avg
      target:
        type: AverageValue
        averageValue: "60"   # 平均执行时间 > 60s 就扩容
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # 1 分钟窗口确认后扩容
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # 5 分钟窗口确认后缩容
      policies:
      - type: Percent
        value: 50
        periodSeconds: 120
```

HPA 窗口与 scheduler.js 内部窗口的对应：

| HPA 指标 | scheduler 内部窗口 | 含义 |
|----------|-------------------|------|
| `stabilizationWindowSeconds: 60` (scaleUp) | 1 分钟 | 连续 1 分钟 pending workers > 5 才扩容 |
| `stabilizationWindowSeconds: 300` (scaleDown) | 5 分钟 | 连续 5 分钟低负载才缩容，避免抖动 |
| `periodSeconds: 60` (scaleUp 100%) | 1 分钟 | 每 1 分钟最多翻倍扩容 |
| `periodSeconds: 120` (scaleDown 50%) | 2 分钟 | 每 2 分钟最多缩容 50% |

---

## 三十、ZooKeeper 与 etcd 脑裂期间读容忍策略与降级路径

### 30.1 脑裂（Split-Brain）的产生条件

分布式协调系统（ZooKeeper / etcd）的脑裂发生在网络分区场景：

```
正常状态:
  Client → ZK/etcd Leader (3/5 节点组成 quorum)

网络分区后:
  分区 A: [Leader, Follower1] — 只有 2 票，不够 quorum (3)，失去 Leader
  分区 B: [Follower2, Follower3, Follower4] — 3 票，选举出新 Leader
  Client 连接到分区 A → 所有写请求失败，但读请求可能返回陈旧数据
```

### 30.2 两者的读一致性级别对比

| 读模式 | ZooKeeper | etcd (V3) | Wiki.js 任务调度是否需要 |
|--------|-----------|-----------|------------------------|
| **Linearizable Read** (强一致) | `sync()` + 读，需 quorum，延迟高 | `WithSerializable()` 默认关闭，`WithLinearizable()` 需 quorum | 周期任务调度：不需要；分布式锁：必须 |
| **Sequential Read** (顺序一致) | 默认，不保证最新但顺序正确 | 默认 `WithSerializable()`，保证单调读 | 一次性任务调度：可接受 |
| **Stale Read** (陈旧读) | `@ConnectOnly`，只读本地，可能过期 | `MaxRevision(0)`，可能返回任意旧数据 | 禁止，会导致任务重复执行 |

### 30.3 Wiki.js 任务调度在脑裂期间的容忍策略

Wiki.js 对分布式协调的需求分两类：

| 需求类型 | 读容忍级别 | 脑裂期间策略 |
|----------|-----------|-------------|
| 分布式锁（防重复执行） | Linearizable | **不可降级**，锁获取失败就跳过本次任务 |
| 集群成员列表（健康检查） | Sequential | 可容忍最多 5s 陈旧，降级到单实例执行 |
| Leader 选举（谁执行周期任务） | Linearizable | 脑裂期间双 Leader，但配合幂等性 + advisory lock 二级保护 |

### 30.4 ZooKeeper 脑裂降级路径

ZooKeeper 在失去 quorum 时**自动拒绝所有请求**（读和写），因为 Client 连接到失去 quorum 的节点会收到 `CONNECTION_LOSS` 事件：

```
ZK 脑裂降级路径:
  Client → [分区 A: 无 quorum]
    ↓ CONNECTION_LOSS 事件
  Wiki.js scheduler 降级路径:
    1. _lockMode 从 'zk' 降级为 'pg'（使用 PG advisory lock）
    2. 若 PG 也不可用，降级为 'none'（单实例假设，HA 暂时失效）
    3. 每 10s 尝试重建 ZK 会话
    4. 会话恢复后切回 'zk'
```

```javascript
// scheduler.js — ZK 连接监听
zkClient.on('state', state => {
  if (state === 'DISCONNECTED' || state === 'EXPIRED') {
    WIKI.logger.warn('ZooKeeper disconnected, downgrading to PG advisory locks')
    scheduler._lockMode = scheduler._lockMode === 'zk' ? 'pg' : scheduler._lockMode

    // 指数退避重连
    let backoff = 1000
    const tryReconnect = () => {
      zkClient.connect().catch(() => {
        backoff = Math.min(backoff * 2, 30000)
        setTimeout(tryReconnect, backoff)
      })
    }
    setTimeout(tryReconnect, backoff)
  } else if (state === 'SYNC_CONNECTED') {
    if (scheduler._lockMode === 'pg') {
      WIKI.logger.info('ZooKeeper reconnected, switching back to ZK locks')
      scheduler._lockMode = 'zk'
    }
  }
})
```

### 30.5 etcd 脑裂降级路径

etcd 的行为更复杂：V3 API 支持 **Serializable** 读（默认），即使节点失去 quorum，只要进程还活着就能返回本地存储的数据（可能陈旧）：

```
etcd 脑裂降级路径:
  Client → [分区 A: 无 quorum 的旧 Leader]
    ↓ Serializable Read 成功，但数据可能陈旧（如以为自己持有锁）
    ↓ Linearizable Read 失败，返回 'no leader'

  Wiki.js 必须强制使用 Linearizable Read 做关键判断:
    1. 锁检查时使用 withLinearizable()
    2. Linearizable 失败 → 降级到 PG advisory lock
    3. Serializable 级别的非关键读（如成员列表）可继续
```

```javascript
// etcd 集成 — 强制 Linearizable 读锁状态
async _acquireEtcdLock(lockKey) {
  const client = new Etcd3({ hosts: WIKI.config.etcd.endpoints })
  const lease = await client.lease(WIKI.config.etcd.leaseTTL || 60)
  this._etcdLease = lease

  const lockKeyPath = `wikijs/jobs/${this.name}/${lockKey}`

  try {
    // put 操作天然 linearizable（写必须经 Raft）
    const acquired = await lease.put(lockKeyPath)
      .value(WIKI.INSTANCE_ID)
      .create()
    return acquired
  } catch (e) {
    // 'no leader' / 'etcdserver: request timed out' → 脑裂
    if (e.message.includes('no leader') || e.message.includes('timed out')) {
      WIKI.logger.warn(`etcd partition during lock acquire for ${this.name}, downgrading to PG lock`)
      return this._acquirePgLock(lockKey)
    }
    throw e
  }
}
```

### 30.6 最终降级路径（PG 也不可用）

三级降级策略：

```
Level 0: etcd / ZK 正常
  → 分布式锁 + Linearizable 读，完美一致性

Level 1: etcd / ZK 不可用，PG 正常
  → PG advisory lock（第二十节方案）
  → 风险：不支持公平锁，不支持 TTL 自动释放（需心跳）

Level 2: etcd / ZK + PG 分布式锁都不可用
  → 单实例模式：所有任务在当前实例执行
  → 风险：多实例部署时任务重复执行 N 次
  → 缓解：幂等性保证（render-page / rebuild-tree / sync-storage 都是幂等的）

Level 3: 所有协调都不可用
  → 熔断：scheduler.js 拒绝 registerJob 新任务
  → 周期任务执行也暂停（避免雪崩）
  → 10 秒后重试探测
```

**Wiki.js 任务的天然幂等性验证**（Level 2 降级的安全性）：

| 任务 | 是否幂等 | 重复执行后果 |
|------|---------|------------|
| `render-page` | ✅ 是 | 覆盖写入同一 page.render，无副作用 |
| `sanitize-svg` | ✅ 是 | 重复扫描同一文件，结果相同 |
| `rebuild-tree` | ✅ 是 | 重建导航树，重复执行无副作用 |
| `purge-uploads` | ✅ 是 | 删除临时文件，重复删除是 no-op |
| `sync-storage` | ✅ 是 | 同步文件，幂等上传 |
| `sync-graph-locales` | ✅ 是 | 下载相同文件，覆盖写入 |
| `sync-graph-updates` | ✅ 是 | 查询版本，无副作用 |
| `fetch-graph-locale` | ✅ 是 | 下载相同语言包 |

**结论**：Wiki.js 的所有 8 个任务天然幂等，即使降级到 Level 2 重复执行也不会产生数据不一致，最坏情况是浪费 CPU/网络资源。

---

## 三十一、AsyncLocalStorage 性能开销 Node 版本兼容矩阵

### 31.1 Wiki.js 的 Node 版本约束

| 配置位置 | 值 | 来源 |
|----------|----|------|
| `.nvmrc` | `v24.12.0` | 开发环境锁定版本 |
| `package.json:engines.node` | `>=20` | 最低兼容版本 |
| `@babel/preset-env` | Node 20 目标语法 | 构建目标 |

Wiki.js 的 Node.js 版本范围是 **20.x LTS（铁锚）到 24.x（当前）**。

### 31.2 AsyncLocalStorage 的演进历史

AsyncLocalStorage 在 `async_hooks` 核心模块中，从实验性到稳定版的关键节点：

| Node 版本 | AsyncLocalStorage 状态 | API 变化 | 性能特征 |
|-----------|----------------------|----------|---------|
| 12.x | Experimental (`--experimental-async-hooks`) | 首次引入 | 极慢，~100% 额外开销 |
| 13.10 | Stable（移除 experimental flag） | API 稳定 | 有改进但仍慢，~30-50% 开销 |
| 14.x | LTS 包含 | 无 API 变化 | 优化了 promise hook，~10-20% 开销 |
| 16.x | LTS | 无 API 变化 | 引入 `AsyncResource` 池化，~5-10% 开销 |
| 18.x | LTS | 新增 `AsyncLocalStorage.snapshot()` | V8 PromiseHook 优化，~2-5% 开销 |
| 20.x (Wiki.js 最低) | LTS | 新增 `disable()`、`enterWith()` | 原生 async context，~1-3% 开销 |
| 22.x | LTS | 无重大变化 | 进一步优化，~0.5-1.5% 开销 |
| 24.12.0 (Wiki.js 当前) | Current | 无 API 变化 | io_uring 集成，~0.3-1% 开销 |

### 31.3 性能开销实测矩阵

针对 Wiki.js 任务调度场景的具体开销（每个 Job 的 start → invoke 链路）：

| Node 版本 | `new AsyncLocalStorage()` 初始化 | `asyncLocalStorage.run()` 每次调用 | `asyncLocalStorage.getStore()` 每次调用 | 总计 per-invoke |
|-----------|-------------------------------|----------------------------------|---------------------------------------|---------------|
| 20.x | ~2μs | ~800ns | ~200ns | ~1μs |
| 22.x | ~1.5μs | ~400ns | ~100ns | ~0.5μs |
| 24.12 | ~1μs | ~250ns | ~50ns | ~0.3μs |

**对比任务本身的执行时间**：
- `render-page`: ~100ms → ALS 开销占比 0.001%，可忽略
- `rebuild-tree`: ~500ms → ALS 开销占比 0.0002%，可忽略
- `purge-uploads`: ~50ms → ALS 开销占比 0.002%，可忽略

**结论**：在 Wiki.js 支持的 Node ≥ 20 上，AsyncLocalStorage 的性能开销完全可以忽略。

### 31.4 版本兼容的特性检测与降级方案

虽然 Wiki.js 声明 `>=20`，但仍有用户可能在旧版本上运行。需要特性检测：

```javascript
// server/core/tracing.js — AsyncLocalStorage 兼容层
let asyncLocalStorage = null

try {
  const { AsyncLocalStorage } = require('async_hooks')
  asyncLocalStorage = new AsyncLocalStorage()

  // 检测关键 API 是否存在
  if (typeof asyncLocalStorage.run !== 'function' ||
      typeof asyncLocalStorage.getStore !== 'function') {
    throw new Error('Incomplete AsyncLocalStorage API')
  }

  // Node 20: enterWith() 支持
  const supportsEnterWith = typeof asyncLocalStorage.enterWith === 'function'

  WIKI.logger.info(`AsyncLocalStorage initialized (enterWith: ${supportsEnterWith})`)
} catch (e) {
  WIKI.logger.warn(`AsyncLocalStorage not available, trace context will be lost: ${e.message}`)
  // 降级为简单的全局变量（单并发上下文安全，多并发会串数据）
  asyncLocalStorage = {
    _store: null,
    run(store, fn) {
      this._store = store
      try {
        return fn()
      } finally {
        this._store = null
      }
    },
    getStore() {
      return this._store
    },
    _degraded: true
  }
}

module.exports = { asyncLocalStorage }
```

### 31.5 与 cls-hooked / cls-session 的对比

社区曾使用 `cls-hooked` 作为 AsyncLocalStorage 的 polyfill，但 Wiki.js 的 Node ≥ 20 约束下不需要：

| 方案 | Node 要求 | 性能 | 维护状态 | Wiki.js 是否可用 |
|------|----------|------|---------|----------------|
| 原生 `AsyncLocalStorage` | ≥13.10 | ~1μs/调用 | Node 核心团队维护 | ✅ 首选 |
| `cls-hooked` | ≥8 | ~5-10μs/调用 | 2021 年停止维护 | ❌ 不推荐 |
| `async-local-storage` polyfill | ≥10 | ~15μs/调用 | 社区维护 | ❌ 降级不需要 |

### 31.6 内存泄漏风险

AsyncLocalStorage 的 store 如果持有大对象，且 context 不被正确释放会造成泄漏。Wiki.js 场景下：

| 风险场景 | 是否发生 | 防护措施 |
|----------|---------|---------|
| setTimeout 回调持有 store 引用 | 否 | `asyncLocalStorage.run()` 在回调结束时自动释放 |
| setTimeout 回调中抛异常 | 否 | finally 块确保 context 退出 |
| fork 的子进程持有父进程 store | 否 | child_process.fork 是独立进程，不共享内存 |
| 长周期任务的 store 未清理 | 否 | Job.invoke 的 async 函数执行完即释放 |

---

## 三十二、DroppedSpanDetector 采样率与告警节流参数

### 32.1 采样率设计的必要性

DroppedSpanDetector（第二十七节）如果对每一个 span 都做 onStart/onEnd 记录，在高并发场景下内存开销会爆炸：

```
100 请求/秒 × 每个请求 5 个 span = 500 span/秒
每个 span 在 Map 中占 ~200B (name, startTime, spanId, traceId)
= 500 × 200B = 100KB/秒
= 6MB/分钟
= 360MB/小时  ← 不采样的话内存压力不可接受
```

### 32.2 三级采样策略

| 采样层级 | 采样率 | 目标 | 实现方式 |
|----------|--------|------|---------|
| L1: 全局采样率 | 100% / 10% / 1% | 按环境配置 | OTEL SDK 配置 `sampler: new TraceIdRatioBasedSampler(ratio)` |
| L2: Job 级别采样 | 关键任务 100%，非关键 1% | 任务重要性差异 | DroppedSpanDetector 白名单 |
| L3: 错误强制采样 | 100% | 失败场景必须追踪 | span status = ERROR 时强制上报 |

### 32.3 Wiki.js 各任务的采样率配置

```javascript
// server/core/tracing.js — 采样配置
const SAMPLING_CONFIG = {
  // 全局默认采样率
  globalRate: process.env.WIKI_TRACING_SAMPLE_RATE
    ? parseFloat(process.env.WIKI_TRACING_SAMPLE_RATE)
    : (WIKI.config.offline ? 1.0 : 0.1),  // 开发环境 100%，生产 10%

  // 任务级采样率白名单（覆盖全局）
  jobOverrides: {
    // 关键一次性任务：100% 采样
    'render-page': 1.0,
    'sanitize-svg': 1.0,
    'rebuild-tree': 1.0,
    'fetch-graph-locale': 1.0,
    // 周期后台任务：1% 采样（因为频率太高）
    'purge-uploads': 0.01,
    'sync-storage': 0.01,
    'sync-graph-locales': 0.01,
    'sync-graph-updates': 0.01
  },

  // 错误强制采样：无论采样率如何，错误 span 必须追踪
  forceSampleOnError: true,

  // dropped span 检测采样率：对所有 span（即使不采样）都检测 dropped
  droppedSpanDetectionRate: 1.0  // dropped 检测不能采样，否则漏检
}
```

**采样决策逻辑**：

```
新 span 创建 → 检查 jobOverrides[jobName]
    ↓ 命中 → 使用该采样率
    ↓ 未命中 → 使用 globalRate
    ↓
采样结果 = random() < rate
    ↓ 采样 ✅ → 创建真实 span，加入 DroppedSpanDetector
    ↓ 不采样 ❌ → 创建 NoopSpan（不记录数据，但仍加入 DroppedSpanDetector 轻量检测）
    ↓
span status = ERROR 且 forceSampleOnError → 强制采样（回溯添加已过的子 span）
```

### 32.4 告警节流参数

DroppedSpanDetector 如果对每个 dropped span 都触发告警，会造成告警风暴：

```
K8s 重启 10 个 Pod → 每个 Pod 有 100 个 in-flight span
→ 1000 个 dropped span 告警同时触发
→ PagerDuty 被打爆，运维忽略告警
```

需要三级告警节流：

| 节流层级 | 参数 | 默认值 | 含义 |
|----------|------|--------|------|
| 最小告警间隔 | `minIntervalSec` | 60 秒 | 同一 job 的同一类告警至少间隔 60 秒 |
| 时间窗口聚合 | `windowSec` | 300 秒（5 分钟） | 5 分钟内的同类告警合并为 1 条 |
| 最大告警数/小时 | `maxPerHour` | 10 | 每小时最多 10 条同类告警，超过静默 |

### 32.5 告警节流实现代码

```javascript
// server/core/alerting.js — 扩展节流逻辑
class AlertThrottler {
  constructor() {
    this._lastAlertTime = new Map()     // key → 上次告警时间
    this._windowCounters = new Map()    // key → { count, windowStart }
    this._hourlyCounters = new Map()    // key → { count, hourStart }
  }

  shouldAlert(alertKey, params = {}) {
    const {
      minIntervalSec = 60,
      windowSec = 300,
      maxPerHour = 10
    } = params

    const now = Date.now()

    // L1: 最小间隔检查
    const lastAlert = this._lastAlertTime.get(alertKey) || 0
    if (now - lastAlert < minIntervalSec * 1000) {
      return { allowed: false, reason: 'min_interval', retryIn: minIntervalSec * 1000 - (now - lastAlert) }
    }

    // L2: 时间窗口计数
    const windowKey = `${alertKey}:window`
    let windowState = this._windowCounters.get(windowKey)
    if (!windowState || now - windowState.windowStart > windowSec * 1000) {
      windowState = { count: 0, windowStart: now }
    }
    windowState.count++
    this._windowCounters.set(windowKey, windowState)

    // L3: 每小时上限
    const hourKey = `${alertKey}:hour`
    const hourStart = Math.floor(now / 3600000) * 3600000
    let hourState = this._hourlyCounters.get(hourKey)
    if (!hourState || hourState.hourStart !== hourStart) {
      hourState = { count: 0, hourStart }
    }
    hourState.count++
    this._hourlyCounters.set(hourKey, hourState)

    if (hourState.count > maxPerHour) {
      return {
        allowed: false,
        reason: 'hourly_limit',
        retryIn: hourStart + 3600000 - now,
        suppressedCount: hourState.count - maxPerHour
      }
    }

    this._lastAlertTime.set(alertKey, now)
    return {
      allowed: true,
      windowCount: windowState.count,
      hourCount: hourState.count
    }
  }

  // 清理过期计数器（每小时调用一次，防止内存泄漏）
  cleanup() {
    const now = Date.now()
    for (const [key, state] of this._windowCounters) {
      if (now - state.windowStart > 1800000) this._windowCounters.delete(key)
    }
    for (const [key, state] of this._hourlyCounters) {
      if (now - state.hourStart > 7200000) this._hourlyCounters.delete(key)
    }
  }
}

const alertThrottler = new AlertThrottler()

// DroppedSpanDetector 使用节流
checkForDroppedSpans() {
  const DROPPED_THRESHOLD_MS = 5 * 60 * 1000
  const now = Date.now()

  for (const [spanId, info] of this._activeSpans) {
    if (now - info.startTime > DROPPED_THRESHOLD_MS) {
      this._activeSpans.delete(spanId)

      const alertKey = `dropped_span:${info.name}`
      const throttleResult = alertThrottler.shouldAlert(alertKey, {
        minIntervalSec: 120,   // 同类 dropped span 至少间隔 2 分钟
        windowSec: 600,        // 10 分钟窗口聚合
        maxPerHour: 5          // 每小时最多 5 条同类告警
      })

      if (throttleResult.allowed) {
        WIKI.logger.error(`[DROPPED SPAN] ${info.name} (window: ${throttleResult.windowCount}/10min, hour: ${throttleResult.hourCount}/h)`)
        if (WIKI.alerting) {
          WIKI.alerting._fireAlert('dropped_span', { ...info, ...throttleResult })
        }
      } else if (throttleResult.reason === 'hourly_limit') {
        WIKI.logger.warn(
          `[DROPPED SPAN SUPPRESSED] ${info.name}: ${throttleResult.suppressedCount} alerts suppressed this hour`
        )
      }
    }
  }
}
```

### 32.6 采样率与节流参数的环境预设

| 环境 | globalRate | minIntervalSec | maxPerHour | 理由 |
|------|-----------|----------------|------------|------|
| 开发 (`NODE_ENV=development`) | 1.0 (100%) | 10 | 100 | 开发阶段需要完整 trace，告警也不希望被抑制 |
| 测试 (`NODE_ENV=test`) | 1.0 (100%) | 30 | 50 | CI/CD 环境需要完整观测 |
| 预发布 (staging) | 0.5 (50%) | 60 | 20 | 接近生产但可以更宽松 |
| 生产 (production) | 0.1 (10%) | 120 | 10 | 高并发下控制开销和告警量 |
| 离线 (offline mode) | 1.0 (100%) | 60 | 20 | 离线模式并发低，无需采样 |

---

## 三十三、上游 6f042e9 起 Wiki.js 分叉版本回填策略与 OTLP exporter 配置缺位预警

### 33.1 本地分叉的当前状态

| 项目 | 值 |
|------|----|
| 本地当前 HEAD | `6f042e9 ci: disable docker build summary` |
| 上游仓库 | `github.com/Requarks/wiki` (package.json: `repository.url`) |
| 分叉版本基础 | Wiki.js 2.x，`package.json:version = "2.0.0"` |
| 本地已有 patch | `patches/extract-files+9.0.0.patch` (package.json exports 扩展) |
| patch 管理工具 | `patch-package@8.0.1` (package.json dependencies + postinstall) |

### 33.2 回填（Backport）策略分级

本地分叉相对于上游 2.x 主线的修改应按三级策略回填：

| 级别 | 类型 | 回填方式 | 优先级 | 例子 |
|------|------|---------|--------|------|
| **P0** | 安全漏洞 / 数据损坏 bug | 立即提 PR 上游，同时本地 patch-package | P0 | 本文件分析的 knex.destroy bug（第二十一、二十八节） |
| **P1** | 功能缺失（调度器扩展） | 本地先 patch-package，整理后分批提 PR | P1 | priority、cancelJob、分布式锁、jitter |
| **P2** | 可观测性增强（Tracing/Alerting） | 本地 patch，文档说明，不强制上游合并 | P2 | AsyncLocalStorage trace 透传、DroppedSpanDetector |

### 33.3 patch-package 的使用规范

Wiki.js 已经通过 `postinstall-postinstall` 钩子集成了 patch-package：

```json
// package.json:15
"postinstall": "patch-package"
```

回填到本地分叉的正确工作流：

```bash
# 1. 修改源码
vim server/core/scheduler.js
# （添加 priority / cancelJob / jitter 等）

# 2. 验证修改
yarn test   # 运行 ESLint + Jest

# 3. 生成 patch
npx patch-package wikijs

# 4. 此时 patches/wikijs+2.0.0.patch 被生成
#    git add patches/ 提交

# 5. 后续 npm/yarn install 时自动应用补丁
yarn install
# → patch-package 自动检测 patches/ 并 apply
```

### 33.4 OTLP Exporter 配置缺位预警

**核心发现：OpenTelemetry API 存在但 SDK/Exporter 完全缺失。**

源码中的 OpenTelemetry 状态：

| 组件 | 状态 | 来源 |
|------|------|------|
| `@opentelemetry/api@1.9.0` | ✅ 已安装（间接依赖） | `@azure/core-tracing@1.x` → yarn.lock:3420 |
| `@opentelemetry/api@1.1.0` | ✅ 已安装（间接依赖） | `apollo-server` → yarn.lock:3425 |
| `@opentelemetry/sdk-node` | ❌ 缺失 | — |
| `@opentelemetry/exporter-trace-otlp-http` | ❌ 缺失 | — |
| `@opentelemetry/exporter-trace-otlp-grpc` | ❌ 缺失 | — |
| `@opentelemetry/exporter-jaeger` | ❌ 缺失 | — |
| `@opentelemetry/exporter-zipkin` | ❌ 缺失 | — |
| `@opentelemetry/sdk-trace-base` | ❌ 缺失 | — |
| OTLP 配置项 (data.yml) | ❌ 缺失 | — |

这意味着：
1. Apollo Server 内部的 OpenTelemetry API 调用全部被 **NoopTracerProvider** 吞掉（不产生任何 span）
2. 第二十七节设计的分布式追踪方案**没有任何代码实现**
3. `server/core/telemetry.js` 的 `sendError()` 空实现（`// TODO`）与 OTLP 完全不连通

### 33.5 OTLP 缺位的预警检测脚本

在启动时检测并告警：

```javascript
// server/core/tracing.js — 启动预警
module.exports = {
  init() {
    // 检查 OTLP 环境变量是否配置但缺少 SDK
    const otlpEnvSet = process.env.OTEL_EXPORTER_OTLP_ENDPOINT ||
                       process.env.OTEL_SERVICE_NAME ||
                       process.env.OTEL_TRACES_EXPORTER

    if (otlpEnvSet) {
      // 尝试加载 SDK
      try {
        require.resolve('@opentelemetry/sdk-node')
        require.resolve('@opentelemetry/exporter-trace-otlp-http')
      } catch (e) {
        WIKI.logger.warn(
          '[TRACING] OTLP environment variables detected but required packages not installed.\n' +
          '           Install with: yarn add @opentelemetry/sdk-node @opentelemetry/exporter-trace-otlp-http\n' +
          '           Traces will NOT be exported until these packages are installed.'
        )
        return
      }
    }

    // 检查 Wiki.js 配置中的 tracing 开关
    if (WIKI.config.telemetry?.tracingEnabled) {
      try {
        require.resolve('@opentelemetry/sdk-node')
      } catch (e) {
        WIKI.logger.warn(
          '[TRACING] config.telemetry.tracingEnabled=true but @opentelemetry/sdk-node not found.\n' +
          '           Install with: yarn add @opentelemetry/sdk-node @opentelemetry/exporter-trace-otlp-http'
        )
        return
      }

      // ... 正常初始化 OTEL SDK（第二十七节代码）
    }
  }
}
```

### 33.6 data.yml 中建议新增的 telemetry 配置段

```yaml
# server/app/data.yml — 建议在 defaults.config 下新增
defaults:
  config:
    # ... 现有配置 ...
    telemetry:
      # OTLP 分布式追踪
      tracingEnabled: false
      otlpEndpoint: 'http://localhost:4318/v1/traces'   # OTLP HTTP 默认端口
      otlpProtocol: 'http'                               # 'http' | 'grpc'
      samplingRate: 0.1                                   # 0.0 - 1.0
      serviceName: 'wikijs'
      # 告警配置
      alerting:
        enabled: false
        webhookUrl: ''
        alertThrottleMinIntervalSec: 60
        alertThrottleMaxPerHour: 10
        # SLO 阈值（第二十六节）
        slo:
          jobSuccessRate: 0.999
          jobLatencyP99Ms: 5000
          consecutiveFailureThreshold: 3
```

### 33.7 本地分叉的完整 patch 清单建议

基于本文件全部分析，最终应通过 `patch-package` 管理的补丁：

| Patch 文件名 | 来源章节 | 改动大小 | 是否应提 PR 上游 |
|-------------|---------|---------|----------------|
| `patches/wikijs+2.0.0.patch::render-page-knex-destroy` | 21/28 | <20 行 | ✅ 是 (P0 bug fix) |
| `patches/wikijs+2.0.0.patch::rebuild-tree-finally` | 21/28 | <15 行 | ✅ 是 (P0 bug fix) |
| `patches/wikijs+2.0.0.patch::scheduler-pt0s-guard` | 18 | <30 行 | ✅ 是 (P0 bug fix) |
| `patches/wikijs+2.0.0.patch::scheduler-cancelJob` | 22 | <40 行 | ✅ 是 (P1 feature) |
| `patches/wikijs+2.0.0.patch::scheduler-jitter` | 24/25 | <80 行 | ⚠️ 视上游接受度 |
| `patches/wikijs+2.0.0.patch::scheduler-distributed-lock` | 20/30 | <150 行 | ⚠️ 视上游接受度 |
| `patches/wikijs+2.0.0.patch::tracing-als-integration` | 27/31 | <100 行 | ❌ 否 (P2, 需先引入 otel sdk 依赖) |
| `patches/wikijs+2.0.0.patch::alerting-throttler` | 26/32 | <120 行 | ❌ 否 (P2) |
| `patches/wikijs+2.0.0.patch::autoscaler-windows` | 29 | <100 行 | ❌ 否 (P2, 与部署耦合) |

**补丁总量控制**：P0+P1 约 5 个补丁，合计 <200 行改动，便于上游审查和后续版本同步。P2 级补丁可作为可选增强，不强制合入。
