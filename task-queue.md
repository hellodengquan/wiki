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
