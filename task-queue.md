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

### 4.1 优先级排序

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

### 4.2 超时回收

**设计结论：无主动超时回收机制**

- **没有**为任务执行设置超时时间（如 `Promise.race` + `setTimeout`）
- Worker 子进程无限等待，直到自然退出或崩溃
- 依赖子进程自身的错误处理和退出码判断

**潜在风险**：
- 死锁或无限循环的任务会导致子进程僵尸化
- 资源泄漏风险

**代码证据**：
```javascript
// invoke() 方法中 await this.finished 没有超时包装
await this.finished
// 仅监听 exit 事件，没有超时定时器
proc.on('exit', (code, signal) => { ... })
```

### 4.3 失败重试

**设计结论：无内置重试机制**

- 任务失败仅记录警告日志：`WIKI.logger.warn(err)`
- **没有**重试计数、退避算法（exponential backoff）
- 周期任务的"重试"是假象：失败后仍会按 schedule 触发下一次

```javascript
try {
  await this.finished
} catch (err) {
  WIKI.logger.warn(err)  // 仅记录日志
}
// 无论成功失败，只要 repeat=true 就继续调度
if (this.repeat && this.queue.jobs.includes(this)) {
  this.enqueue(data)
}
```

[server/core/scheduler.js:83-91](server/core/scheduler.js#L83-L91)

### 4.4 任务可见性与持久化

**设计结论：纯内存队列，无持久化**

| 特性 | 实现方式 | 局限性 |
|------|----------|--------|
| 任务存储 | 内存数组 `this.jobs` | 进程重启丢失 |
| 任务状态 | 仅存在 `timeout` 句柄 | 无法查询历史任务 |
| 持久化 | 无数据库表 | 崩溃后任务状态丢失 |
| 多实例协调 | 无 | 多实例部署会重复执行 |

**代码证据**：
```javascript
// 数据库迁移文件中无任何 job/queue 相关表
// 仅在 data.yml 中定义周期任务，启动时重新注册
module.exports = {
  jobs: [],  // 内存数组
  // ...
  registerJob(opts, data) {
    const job = new Job(opts, this)
    job.start(data)
    return job
  }
}
```

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
