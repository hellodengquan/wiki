# 外部存储模块与数据库双向同步 — 代码走向分析

## 1. 整体架构概览

本项目的存储同步体系由三层构成：

```
┌──────────────────────────────────────────────────────────┐
│  客户端（编辑器 + 冲突解决 UI）                            │
│  editor.vue / editor-modal-conflict.vue / ckeditor/conflict.vue │
└───────────────────────┬──────────────────────────────────┘
                        │ GraphQL (checkConflicts / conflictLatest / update)
┌───────────────────────▼──────────────────────────────────┐
│  服务端调度与模型层                                        │
│  scheduler → sync-storage job → Storage.initTargets()     │
│  Storage.pageEvent() → pages.createPage / updatePage     │
└───────────────────────┬──────────────────────────────────┘
                        │ target.fn.sync() / target.fn.created() / updated()...
┌───────────────────────▼──────────────────────────────────┐
│  存储模块实现层（Strategy 模式）                            │
│  git/storage.js  |  disk/storage.js  |  sftp/storage.js  │
│  (双向同步+diff)     (仅push+备份)      (仅push)           │
│  dropbox/gdrive/onedrive/box/azure/s3 → 占位桩            │
└──────────────────────────────────────────────────────────┘
```

---

## 2. 同步驱动（Sync Driver）— 启动与调度

### 2.1 启动链路

```
kernel.postBootMaster()
  → Storage.refreshTargetsFromDisk()    // 从磁盘 yml 读取存储模块定义，写入 DB
  → Storage.initTargets()               // 激活已启用的存储目标，注册定时任务
  → WIKI.scheduler.start()             // 启动全局调度器
```

**关键代码位置：**
- `server/core/kernel.js:79-87` — 启动时序
- `server/models/storage.js:38-112` — `refreshTargetsFromDisk()`
- `server/models/storage.js:117-178` — `initTargets()`

### 2.2 initTargets() 详解

```
initTargets()
  ① 从 DB 查询 isEnabled=true 的 storage targets
  ② 停止旧的 sync-storage 定时任务
  ③ 对每个 target：
     a. require(`../modules/storage/${target.key}/storage`)  加载模块
     b. 将 target.config 和 target.mode 注入模块实例
     c. 调用 target.fn.init()   初始化存储连接
     d. 注册定时同步任务：
        - 用户自定义间隔 → WIKI.scheduler.registerJob({ name:'sync-storage', schedule, repeat:true }, targetKey)
        - 模块内部调度间隔 → 同上，使用 internalSchedule
  ④ init 失败时将状态写回 DB (status: 'error')
```

**关键点：** `mode` 字段决定同步方向：
- `push` — 仅 DB → 外部存储（单向）
- `pull` — 仅外部存储 → DB（单向）
- `sync` — 双向

只有 `git` 模块完整实现了 `sync` 模式（`definition.yml:9-11`），其他模块仅实现 `push`。

### 2.3 调度器机制

**代码位置：** `server/core/scheduler.js`

```javascript
class Job {
  constructor({ name, immediate, schedule, repeat, worker }, queue)
  start(data)       // 推入队列，immediate=true 立即执行，否则按 schedule 延迟
  enqueue(data)     // setTimeout(this.invoke, schedule.asMilliseconds(), data)
  invoke(data)      // worker=true 时 fork 子进程，否则 require(`../jobs/${name}`)(data)
  stop()            // clearTimeout + 从队列移除
}

registerJob(opts, data) → new Job(opts, this).start(data)
```

**定时同步调用链：**

```
scheduler 按 syncInterval 周期触发
  → jobs/sync-storage.js(targetKey)
    → 在 WIKI.models.storage.targets 中查找 target
    → target.fn.sync()   // 调用具体存储模块的 sync 方法
    → 成功: patch state { status:'operational' }
    → 失败: patch state { status:'error', message:err.message }
```

**代码位置：** `server/jobs/sync-storage.js:1-35`

---

## 3. 双向同步核心 — Git 模块 sync()

**代码位置：** `server/modules/storage/git/storage.js:120-188`

### 3.1 sync() 执行流程

```
sync()
  │
  ├─ ① 记录当前 commit hash (currentCommitLog)
  │     git.log(['-n', '1', branch, '--'])
  │
  ├─ ② 【PULL 阶段】mode 为 'sync' 或 'pull' 时
  │     git.pull('origin', branch, ['--rebase'])
  │     - 使用 rebase 策略拉取远程变更
  │     - 远程变更与本地未推送的 commit 由 git rebase 自动合并
  │
  ├─ ③ 【PUSH 阶段】mode 为 'sync' 或 'push' 时
  │     git.push('origin', branch, ['--signed=if-asked'])
  │     - mode='push' 时追加 --force 强制推送
  │
  └─ ④ 【DIFF 处理阶段】mode 为 'sync' 或 'pull' 时
        git.log(['-n', '1', branch, '--'])  → 获取 latestCommitLog
        git.diffSummary(['-M', currentHash, latestHash])
        │
        ├─ diff.files.length > 0 → 有变更
        │   ├─ 解析文件名（处理 git rename 格式 `{old => new}`）
        │   ├─ 构造 filesToProcess 数组
        │   └─ processFiles(filesToProcess, rootUser)
        │
        └─ diff.files.length === 0 → 无变更，跳过
```

### 3.2 三种模式的行为差异

| 阶段 | `push` | `pull` | `sync` |
|------|--------|--------|--------|
| Pull rebase | ✗ | ✓ | ✓ |
| Push | ✓ (带 --force) | ✗ | ✓ |
| Diff 处理 | ✗ | ✓ | ✓ |

- **push 模式**：只把本地 DB 内容推到远程，不关心远程变更
- **pull 模式**：只从远程拉取变更写入 DB，不推送本地
- **sync 模式**：先 pull 再 push，然后处理 diff

---

## 4. 差异比对（Diff）环节

### 4.1 Git Diff 比对

**代码位置：** `server/modules/storage/git/storage.js:145-188`

```
git.diffSummary(['-M', currentCommitHash, latestCommitHash])
```

- `-M` 标志：启用重命名检测
- 比较的是 sync 开始前与 pull 后的 HEAD commit 之间的差异
- 结果格式：`{ files: [{ file, binary, before, after, deletions, insertions }] }`

**文件名解析**（处理 git rename 标记）：

```javascript
const filePattern = /(.*?)(?:{(.*?))? => (?:(.*?)})?(.*)/
// 匹配格式如：`path/{old => new}/file.md`
// 或简单路径：`path/file.md`
```

解析后得到 `oldPath`（变更前路径）和 `relPath`（变更后路径），用于判断是 rename 还是普通修改。

### 4.2 processFiles() — 差异分类与处理

**代码位置：** `server/modules/storage/git/storage.js:195-291`

```
processFiles(files, user)
  │
  ├─ 对每个 file 判断 contentType = pageHelper.getContentType(relPath)
  │
  ├──【页面文件】(!item.binary && contentType 存在)
  │   │
  │   ├─ 条件1: fileExists && relPath !== oldPath
  │   │   → 文件被重命名
  │   │   → pages.movePage({ skipStorage: true })
  │   │   → 然后继续执行 processPage 更新内容
  │   │
  │   ├─ 条件2: !fileExists && deletions>0 && insertions===0
  │   │   → 文件被删除
  │   │   → pages.deletePage({ skipStorage: true })
  │   │   → continue（不再处理）
  │   │
  │   └─ 默认: commonDisk.processPage()
  │       → 读取文件 → 解析 frontmatter → 更新或创建页面
  │
  └──【资产文件】(binary 或 contentType 不存在)
      │
      ├─ 条件1: fileExists && (before===after || deletions===0 && insertions===0)
      │   → 资产被重命名
      │   → 更新 DB 中的 filename 和 hash
      │
      ├─ 条件2: !fileExists && (before>0 && after===0 || deletions>0 && insertions===0)
      │   → 资产被删除
      │   → 从 DB 删除资产记录和资产数据
      │
      └─ 默认: commonDisk.processAsset()
          → 导入资产到 DB
```

### 4.3 commonDisk.processPage() — 页面内容差异合并

**代码位置：** `server/modules/storage/disk/common.js:73-115`

这是外部存储内容写入 DB 的核心逻辑：

```
processPage({ user, fullPath, relPath, contentType, moduleName })
  │
  ├─ ① 从磁盘读取文件内容
  │     fs.readFile(path.join(fullPath, relPath))
  │
  ├─ ② 解析 frontmatter 元数据
  │     pages.parseMetadata(itemContents, contentType)
  │     - markdown: 解析 ---yaml--- 格式
  │     - html: 解析 <!-- yaml --> 格式
  │
  ├─ ③ 查询 DB 中是否已存在该页面
  │     pages.getPageFromDb({ path, locale })
  │
  ├──【已存在】→ 标记为 modified
  │   pages.updatePage({
  │     title: pageData.title || currentPage.title,
  │     description: pageData.description || currentPage.description,
  │     tags: pageData.tags || currentPage.tags,
  │     isPublished: pageData.isPublished || currentPage.isPublished,
  │     content: pageData.content,        // ← 内容直接覆盖
  │     skipStorage: true                  // ← 防止循环回写存储
  │   })
  │
  └──【不存在】→ 标记为 new
      pages.createPage({
        path, locale, title, description, tags,
        isPublished: pageData.isPublished || true,
        content: pageData.content,
        skipStorage: true
      })
```

**关键设计：** `skipStorage: true` 参数防止"存储 → DB → 存储"的无限循环。当从外部存储拉取内容更新 DB 时，不会触发 `Storage.pageEvent()` 回写存储。

---

## 5. Push 方向 — DB 变更写入外部存储

### 5.1 事件驱动模型

DB 中的页面增删改操作会触发 `Storage.pageEvent()`，通知所有已启用的存储模块：

```
pages.createPage() / updatePage() / deletePage() / movePage()
  └→ WIKI.models.storage.pageEvent({ event, page })
       └→ for each target: target.fn[event](page)
           // 即 target.fn.created / updated / deleted / renamed
```

**代码位置：** `server/models/storage.js:180-189`

### 5.2 Git 模块的 Push 实现

| 事件 | 行为 | 代码位置 |
|------|------|---------|
| `created(page)` | 写文件 → `git add` → `git commit` | git/storage.js:297-312 |
| `updated(page)` | 覆写文件 → `git add` → `git commit` | git/storage.js:319-334 |
| `deleted(page)` | `git rm` → `git commit` | git/storage.js:341-354 |
| `renamed(page)` | `fs.move` → `git rm` → `git add` → `git commit` | git/storage.js:361-384 |

每次 commit 的 author 信息取自当前操作的 page 作者，fallback 到 defaultName/defaultEmail。

**注意：** Push 方向的 commit 不会立即 push 到远程。远程推送发生在下一次 `sync()` 调用的 Push 阶段。

---

## 6. 编辑冲突检测与解决

此项目存在**两层冲突处理**机制：

### 6.1 存储层冲突（Git Rebase）

在 `sync()` 的 Pull 阶段，使用 `git pull --rebase` 策略：

```
git.pull('origin', branch, ['--rebase'])
```

- 如果远程和本地修改了同一文件，git rebase 会尝试自动合并
- 如果自动合并失败（真正的冲突），`git.pull` 会抛出异常
- 异常被 `sync-storage.js` 捕获，写入 `state: { status: 'error', message }` 到 DB
- 管理员需要手动介入解决（如使用 `purge` 操作重建本地仓库）

### 6.2 编辑器层冲突（乐观锁 + 时间戳比对）

这是面向终端用户的并发编辑冲突检测机制，独立于存储同步。

#### 检测逻辑

**服务端（GraphQL resolver）：** `server/graph/resolvers/page.js:354-368`

```javascript
async checkConflicts(obj, args, context) {
  let page = await pages.query().select('updatedAt').findById(args.id)
  return page.updatedAt > args.checkoutDate   // DB中的更新时间 > 用户开始编辑的时间
}
```

- `checkoutDate`：用户打开编辑器时记录的页面 `updatedAt` 时间戳
- 如果 DB 中的 `updatedAt` 比用户的 `checkoutDate` 更新，说明有其他人已修改了页面

#### 客户端流程

**代码位置：** `client/components/editor.vue`

```
① 编辑器初始化
   checkoutDateActive = checkoutDate   (editor.vue:232)

② 定时轮询冲突检测（5秒间隔）
   apollo: { isConflict: { pollInterval: 5000, ... } }   (editor.vue:557-577)
   调用 pages.checkConflicts(id, checkoutDate: checkoutDateActive)
   仅在 mode !== 'create' && !isSaving && isDirty 时运行

③ 保存时前置检测
   save() → pages.checkConflicts(id, checkoutDate: checkoutDateActive)
   (editor.vue:377-394)
   如果返回 true → 抛出 'editor:conflict.warning' 错误

④ 冲突触发
   $root.$emit('saveConflict')  →  显示冲突解决 UI

⑤ 保存成功后
   checkoutDateActive = resp.page.updatedAt   // 重置基线时间戳
   isConflict = false
```

#### 冲突解决 UI

有两种实现，对应不同编辑器：

| 组件 | 编辑器 | 代码位置 |
|------|--------|---------|
| `editor-modal-conflict.vue` | Markdown / Code | `client/components/editor/editor-modal-conflict.vue` |
| `ckeditor/conflict.vue` | CKEditor (WYSIWYG) | `client/components/editor/ckeditor/conflict.vue` |

**Markdown/Code 编辑器的冲突解决流程：**

```
editor-modal-conflict.vue mounted()
  │
  ├─ 调用 pages.conflictLatest(id) 获取 DB 中最新版本
  │
  ├─ 使用 CodeMirror.MergeView 展示双栏对比
  │   左栏 (value):  当前编辑器内容（可编辑）
  │   右栏 (orig):   DB 最新版本（只读）
  │   依赖: diff-match-patch.js 库计算文本差异
  │
  └─ 用户选择：
      ├─ "Use Local" (useLocal)
      │   → 保留当前编辑器内容
      │   → editor/content = cm.edit.getValue()
      │   → checkoutDateActive = latest.updatedAt
      │   → overwriteEditorContent / resetEditorConflict
      │
      └─ "Use Remote" (useRemote)  // 需二次确认
          → 使用 DB 最新版本
          → editor/content = latest.content
          → checkoutDateActive = latest.updatedAt
          → overwriteEditorContent / resetEditorConflict
```

**CKEditor 的冲突解决流程更简略：**
- 不提供双栏 diff 视图
- 仅展示最新版本的作者和时间信息
- 提供链接打开最新版本页面预览
- 同样提供 Use Local / Use Remote 两个选择

---

## 7. 完整同步流程图（Git sync 模式）

```
定时调度 / 手动触发
       │
       ▼
  sync-storage job
       │
       ▼
  git.sync()
       │
       ├─ 1. 记录 currentCommitHash
       │
       ├─ 2. git pull --rebase ← ─ ─ ─ 远程 Git 仓库
       │     └─ rebase 冲突? → 记录 error 状态，人工介入
       │
       ├─ 3. git push ─ ─ ─ ─ ─ ─ ─ → 远程 Git 仓库
       │     (推送步骤2之前本地已有的commit)
       │
       └─ 4. diffSummary(currentHash, latestHash)
             │
             ▼
       有变更的文件列表
             │
       ┌─────┴─────┐
       │            │
    页面文件      资产文件
       │            │
  ┌────┼────┐   ┌───┼───┐
  │    │    │   │   │   │
 重命名 删除 修改 重命名 删除 修改
  │    │    │   │   │   │
  ▼    ▼    ▼   ▼   ▼   ▼
move  del  process  DB   DB  import
Page  Page Page   upd  del  Asset
  │    │    │   │   │   │
  │    │  ┌───┴───┐│   │   │
  │    │  │       ││   │   │
  │    │ 已存在  不存在│   │   │
  │    │  │       ││   │   │
  │    │ update  create│  │
  │    │  Page    Page │  │
  │    │  (skip   (skip│  │
  │    │ Storage) Storage) │
  │    │  │       ││   │   │
  ▼    ▼  ▼       ▼▼   ▼   ▼
       DB 更新完成
```

---

## 8. 各存储模块能力矩阵

| 模块 | Push (DB→存储) | Pull (存储→DB) | Sync (双向) | 定时调度 | 导入全量 | 冲突处理 |
|------|:-:|:-:|:-:|:-:|:-:|:-:|
| **git** | ✓ | ✓ | ✓ | PT5M | importAll / syncUntracked | git rebase |
| **disk** | ✓ | ✗ | ✗ | 可配 | importAll / dump | 无 |
| **sftp** | ✓ | ✗ | ✗ | — | exportAll | 无 |
| **s3** | ✓ | ✗ | ✗ | — | — | 无 |
| **azure** | ✓ | ✗ | ✗ | — | — | 无 |
| **dropbox** | 占位 | 占位 | 占位 | — | — | — |
| **gdrive** | 占位 | 占位 | 占位 | — | — | — |
| **onedrive** | 占位 | 占位 | 占位 | — | — | — |
| **box** | 占位 | 占位 | 占位 | — | — | — |

> `占位` = 模块文件存在但所有方法体为空，尚未实现。

---

## 9. 网络波动与断连恢复：增量同步 vs 全量补偿

### 9.1 断连场景与状态追踪

每次 `sync-storage` job 执行都会更新存储目标的状态字段：

```
state = {
  status: 'operational' | 'error' | 'pending',
  message: '错误详情',          // 失败时写入
  lastAttempt: '2025-06-14T10:30:00.000Z'  // 每次执行更新
}
```

**代码位置：**
- `sync-storage.js:15-31` — 成功/失败均写回 `lastAttempt`
- `initTargets()` 中 target.init() 成功写 `status:'operational'`，失败写 `status:'error'`

**断连不会自动触发状态切换为"等待恢复"，而是依赖下一次定时调度自然重试。** 调度器是连续的，无论上一次成功还是失败，下一个 `syncInterval` 周期到时都会再次执行。

### 9.2 恢复策略：默认增量同步

**Git 模块的增量同步机制**依赖本地仓库和远程仓库的 commit 历史，天然具备断连后自动增量恢复的能力：

```
断连期间：
  本地DB变更 → pageEvent() → git commit（本地仓库累积）
  远程仓库变更 → 由其他人推送，与本地无关

恢复后（网络恢复，下一次 sync() 触发）：
  ① git.pull --rebase
     - 拉取断连期间远程新增的所有 commit
     - 将本地累积的 commit rebase 到远程 HEAD 之后
     - 由 git 自动处理文件级别的合并
  ② git.push
     - 推送 rebase 后本地独有的 commit 到远程
  ③ diffSummary(oldHash, newHash)
     - oldHash: 本次 sync 开始前本地 HEAD（断连前最后一次 sync 的位置）
     - newHash: rebase 后的新 HEAD
     - 两者之间的 diff 就是远程断连期间发生的所有变更
     - 逐个文件写入 DB（增量）
```

**增量判断依据**：不是时间戳，而是两次 sync 的 commit hash 区间。

- 如果断连了 1 天，期间远程有 50 个 commit，`diffSummary(oldHash, newHash)` 会汇总这 50 个 commit 的所有文件变更为一个文件级别 diff
- 同一个文件如果在断连期间被修改了多次，diff 只反映最终差异（不保留中间过程）
- 断连期间本地有 commit 时，rebase 后 `oldHash` 与 `newHash` 都已前移，仍能正确计算增量

### 9.3 何时触发全量补偿

增量机制在以下情况失效，需要**全量补偿**：

| 场景 | 触发方式 | 补偿操作 |
|------|---------|---------|
| 本地仓库损坏（rebase冲突无法自动解决、.git 目录损坏） | 管理员手动触发 | `purge()` → 清空本地仓库 → `init()` 重新 clone → `importAll()` 全量写入 DB |
| 本地仓库与远程分叉过深（非 fast-forward 无法 rebase） | 管理员手动触发 | `purge()` 同上 |
| 初始激活 Git 模块（仓库为全新空仓库） | 自动触发 | `syncUntracked()` → 将 DB 中所有非私有页面/资产 → 写入本地仓库 → git add → commit |
| DB 数据重建（从 Git 恢复） | 管理员手动触发 | `importAll()` → 遍历本地仓库所有文件 → 逐个调用 `processFiles([{ importAll: true }])` → 写入/更新 DB |

#### purge() — 彻底重建

**代码位置：** `server/modules/storage/git/storage.js:517-522`

```javascript
async purge() {
  await fs.emptyDir(this.repoPath)           // 清空本地目录
  await this.init()                          // 重新 clone 远程
  // init() 内部会：git.clone → checkout branch → config user → 若有 sshKey 则设置
}
```

#### importAll() — 全量导入

**代码位置：** `server/modules/storage/git/storage.js:432-469`

```
importAll()
  → klaw 遍历 repoPath（排除 .git 目录）
  → 对每个文件调用 processFiles([{ importAll: true, ... }])
  → importAll: true 的作用：
     ① processFiles 中跳过 rename 和 delete 的判断
     ② 所有文件默认走 commonDisk.processPage / processAsset（统一视为"新增/修改"）
     ③ 在 getPageFromDb 中已存在则 update，不存在则 create
  → clearFolderCache() 清除磁盘路径缓存
```

#### syncUntracked() — 全量导出

**代码位置：** `server/modules/storage/git/storage.js:470-516`

```
syncUntracked()
  ① 从 pages 表流式查询所有非私有页面 (isPrivate=false)
     → 构造 filePath = locale/path.ext
     → fs.outputFile(injectPageMetadata(page))
     → git add
  ② 从 assets / assetData 联表流式查询所有资产
     → 解析 assetFolders 得到路径
     → fs.outputFile
     → git add
  ③ git commit("docs: add all untracked content")
```

### 9.4 Disk 模块的全量策略

Disk 模块不支持双向同步，因此全量操作更直接：

```
disk.dump()      // 全量导出：DB → 磁盘（相当于 syncUntracked）
disk.importAll() // 全量导入：磁盘 → DB（与 Git importAll 相同的 klaw 遍历）
```

### 9.5 Cloud/SFTP 模块的断连行为

对于只有单向 `push` 的模块（s3/azure/sftp/s3/dropbox/gdrive/box/onedrive）：

- 断连不影响 DB 正常写入（pageEvent 正常执行但网络异常会失败）
- **缺陷**：`sync-storage.js` 捕获错误后，DB 已写入的记录不会回滚到存储中
- 恢复后下一次调度只执行 `sync()`（但这些模块的 sync() 是空函数或仅做健康检查），**丢失期间的 push 操作**
- 管理员可使用 `exportAll()`（sftp/azure/s3 已实现）手动补偿

---

## 10. 同步事务保护与部分提交回滚

### 10.1 事务使用范围

本项目使用的事务机制来自 **Objection.js + knex**：

```javascript
const trx = await WIKI.models.Objection.transaction.start(WIKI.models.knex)
try {
  // ... 多步操作 ...
  await trx.commit()
} catch (err) {
  await trx.rollback()
}
```

**全局事务仅出现在模型元数据刷新阶段**：

| 位置 | 事务保护内容 | 代码位置 |
|------|-------------|---------|
| Storage | `refreshTargetsFromDisk()` 中批量 upsert 存储模块配置 | `storage.js:88-109` |
| SearchEngines | 批量 upsert 搜索引擎配置 | `searchEngines.js:80-93` |
| Renderers | 批量 upsert 渲染器配置 | `renderers.js:84-105` |
| Loggers | 批量 upsert 日志器配置 | `loggers.js:81-94` |
| Editors | 批量 upsert 编辑器配置 | `editors.js:79-92` |
| CommentProviders | 批量 upsert 评论提供者配置 | `commentProviders.js:83-96` |
| Analytics | 批量 upsert 分析器配置 | `analytics.js:79-92` |

### 10.2 页面同步链路的事务保护

**关键发现：页面 CRUD + 存储同步链路全程没有显式数据库事务。**

#### Push 方向（DB → 存储）事务行为

```
pages.updatePage(opts)           // NO transaction
  ├─ 查询原始页面
  ├─ pageHistory.addVersion()     // 插入历史版本记录（独立 insert）
  ├─ query().patch(pageFields)    // 更新 pages 表
  ├─ renderPage() → patch render  // 更新渲染缓存
  ├─ deletePageFromCache()        // 清除缓存事件
  ├─ data.searchEngine.updated()  // 写入搜索索引
  └─ skipStorage ? skip : storage.pageEvent({ event:'updated', page })
       └─ for each target: target.fn.updated(page)
            ├─ Git: fs.write → git add → git commit
            ├─ S3/Azure: putObject
            └─ SFTP: sftp.writeFile

// 如果 storage.pageEvent 中某个 target.fn.updated() 抛出异常：
//   - pages 表的 update 已提交（没有事务回滚）
//   - pageHistory 记录已存在（已 commit）
//   - 其他已执行成功的 target 的更改无法撤销
//   - sync-storage job 捕获异常，state.status = 'error'
```

**结论：Push 是"尽力而为"的 at-most-once 语义。** DB 写入成功但存储写入失败时，数据不一致，依赖下一次 `sync()`（Git 模块）或管理员手动补偿。

#### Pull 方向（存储 → DB）事务行为

```
git.processFiles(files)          // NO transaction
  ├─ for each file:
       ├─ movePage({ skipStorage: true })
       │    └─ pageHistory.addVersion() + patch
       ├─ deletePage({ skipStorage: true })
       │    └─ query().deleteById() + $beforeDelete hook 删除评论
       └─ commonDisk.processPage()
            ├─ fs.readFile() → parseMetadata()
            ├─ getPageFromDb()
            ├─ updatePage({ skipStorage: true }) / createPage(...)
            │    └─ 历史 + 表更新 + 搜索 + 缓存失效
            └─ 失败时 catch(err) + WIKI.logger.error

// 如果处理第 N 个文件时抛出异常：
//   - 前 N-1 个文件的 update/create/delete 都已 commit
//   - 第 N 个文件及其后续文件不会被处理
//   - 整个 sync 结果被标记为 error
//   - 下次 sync 时 diffSummary 仍基于旧的 hash 区间，已成功的文件会重新
//     被 diff 出来（因为 commit hash 没变），再次 update（幂等）
```

**结论：Pull 方向是**"部分成功 + 下次重试自动补全"**的机制，依赖文件级幂等更新。

#### 资产上传的事务风险

**代码位置：** `server/models/assets.js:81-163`

```
assets.upload(opts)              // NO transaction
  ├─ 查询是否存在同 hash 资产
  ├─ (SVG时) 注册 sanitize-svg job
  ├─ 存在 ? patch assets row
           : insert assets row
  ├─ insert/update assetData (knex 原生 SQL，非同一事务)
  ├─ fs.move/copy 到 cache 目录
  └─ skipStorage ? skip : storage.assetEvent({ event:'uploaded', asset })
```

风险窗口：
- ① assets 表 insert 成功，但 assetData 表 insert 失败 → 资产记录存在但无数据
- ② assetData 成功但 fs.move 失败 → DB 有数据，本地缓存缺失（下次读取时会从 assetData 重新生成）
- ③ 全部成功但 storage.assetEvent 失败 → DB 有数据，外部存储缺失

### 10.3 Git 存储层的原子性保障

Git 模块通过**git commit 的原子性**提供"本地文件系统"层面的事务（不是数据库事务）：

```
storage.pageEvent → git.updated(page)
  ├─ fs.outputFile(filePath, content)   // 覆盖写磁盘
  ├─ git.add(file)                      // 暂存区
  └─ git.commit(`docs: update ${path}`)  // 原子提交
```

- 一个 page 的修改 = 一个独立的 commit
- commit 是 git 的原子单位：要么整个 commit 写入 `.git/objects`，要么什么都不变
- 多个 page 的修改之间没有 batch commit 原子性：前一个 commit 成功后一个失败，前一个已存在
- `sync()` 中的 push 也是原子的：整个 push 要么全部到远程，要么都不到（git 协议级保证）

### 10.4 rename 操作的补偿风险

`storage.renamed(page)` 和 `storage.assetRenamed(asset)` 在 Azure/S3 模块中使用两步操作：

```
// Azure storage.js:62-79
1. destBlockBlobClient.syncCopyFromURL(sourceBlobURL)   // 复制到新位置
2. sourceBlockBlobClient.delete(...)                     // 删除旧位置
```

如果步骤 1 成功但步骤 2 失败，会出现**数据重复**而非数据丢失（新旧位置都存在相同内容）。这是一种"安全的部分提交"——最坏情况是多一份冗余副本，不会有数据丢失。

SFTP 模块的 renamed 类似：先 sftp.readFile 再 sftp.unlink 再 sftp.writeFile（`storage.js:133-147`），中间任一步失败都可能产生残留。

### 10.5 回滚策略总结

| 层级 | 机制 | 回滚能力 | 失败后果 |
|------|------|---------|---------|
| 存储模块配置 upsert | DB Transaction | 完整回滚 | 整个 batch 取消，模型定义不更新 |
| Page CRUD | 无事务 | 不可回滚 | 已执行的 DB 写入保留；history/搜索/缓存 可能与 page 表不一致 |
| Pull 多文件处理 | 无事务 | 不可回滚 | 前 N-1 个文件成功已保留；下次 retry 自动幂等重放 |
| Git 本地 commit | git 原子性 | 单 commit 级原子 | 一个 commit 内的文件要么全部入 git，要么都不 |
| Git push / pull | git 协议 | 连接级原子 | push/pull 整体成功或失败 |
| Cloud rename | 复制+删除 | 不可回滚 | 可能产生冗余副本（数据安全，需清理） |
| Asset 上传 | 无事务 | 不可回滚 | 可能产生 assets 行存在但 assetData 缺失的不一致 |

---

## 11. 多用户同时编辑同一页的合并策略

### 11.1 架构前提：无实时协作

本项目**没有提供实时协作编辑（CRDT/OT/WebSocket 协同）**：

- 代码中无 WebSocket/Socket.IO 协同服务
- 无 BroadcastChannel（跨标签页也仅用于 UI 事件，非内容同步）
- 无 CRDT（Yjs/Automerge 等）库引用
- `subscribeToNotifications` 仅支持多实例部署时的缓存失效广播，不涉及页面内容合并

多用户编辑冲突的**唯一处理机制是：编辑器层乐观锁检测 + Last-Writer-Wins 变体（用户手动选择）**。

### 11.2 场景模拟：A 和 B 同时编辑同一页

```
T0: 页面 P 的 updatedAt = 10:00:00, content = "Version 0"

T1: 用户A打开编辑器 → checkoutDateActive = 10:00:00
T2: 用户B打开编辑器 → checkoutDateActive = 10:00:00
    (两人都看到 Version 0)

T3: 用户A开始编辑："Version 0 + A的修改" → 编辑器内容变更，isDirty = true
T4: 用户B开始编辑："Version 0 + B的修改" → isDirty = true

T5: Apollo 5秒轮询检查冲突
    对A：DB updatedAt (10:00:00) == checkoutDate (10:00:00) → 无冲突
    对B：同上 → 无冲突

T6: 用户B点保存
    ① save() 内部先 checkConflicts(id, checkoutDate: 10:00:00)
       → DB updatedAt(10:00:00) 不 > 10:00:00 → 无冲突
    ② pages.updatePage()
       → DB updatedAt 更新为 10:01:00
       → pageHistory.addVersion() 记录快照 B
       → storage.pageEvent('updated', page) → git commit + push
    ③ 保存成功 → B 的 checkoutDateActive = 10:01:00

T7: A 的下一次 5秒轮询触发
    checkConflicts(id, checkoutDate: 10:00:00)
    → DB updatedAt(10:01:00) > 10:00:00 → 返回 true
    → isConflict = true → 显示冲突提示条

T8: A 点击保存（如果没注意到冲突提示）
    save() → checkConflicts 返回 true
    → throw 'editor:conflict.warning' 错误
    → this.$root.$emit('saveConflict')
    → 弹出 editor-modal-conflict.vue
```

### 11.3 合并策略：用户手动二选一

弹出的冲突解决 UI 提供 **Last Writer Wins / First Writer Wins 两种选择**，但不提供自动文本 diff-merge：

```
CodeMirror.MergeView 界面：
  ┌───────────────────────────────┬───────────────────────────────┐
  │  左侧：当前编辑器内容（A编辑版）│  右侧：DB 最新版本（B编辑版） │
  │  （可编辑）                     │  （只读）                     │
  │  "Version 0 + A的修改"          │  "Version 0 + B的修改"         │
  └───────────────────────────────┴───────────────────────────────┘
  [ Use Local ]   [ Use Remote ]
```

#### 选项 1：Use Local（Last Writer = A）

**代码位置：** `editor-modal-conflict.vue:79-89`

```
cm.edit.getValue() → 获取左侧（A修改+可能的手工合并）内容
  → editor/content = 该内容
  → checkoutDateActive = latest.updatedAt (10:01:00)
  → overwriteEditorContent
  → resetEditorConflict → isConflict = false
  → 用户再次手动点击保存 → checkConflicts(10:01:00 == DB 10:01:00 → 通过)
  → updatePage → DB updatedAt = 10:02:00
  → B 的修改完全丢失（仅在 pageHistory 中作为历史版本存在）
```

**策略本质：A 的版本覆盖 B 的版本。B 的贡献保留在历史版本中，可通过 pageHistory 回滚。**

#### 选项 2：Use Remote（First Writer = B Wins）

**代码位置：** `editor-modal-conflict.vue:93-106`

```
确认弹窗（$confirm "confirm overwrite your local changes?"）
  → Yes：
     editor/content = latest.content （B 的版本）
     checkoutDateActive = latest.updatedAt (10:01:00)
     overwriteEditorContent
     resetEditorConflict
     → A 的所有修改立即丢失（不写入任何历史！）
```

**⚠️ 重要隐患：选择 Use Remote 时，A 本地的修改被直接丢弃，不会被记录到 pageHistory 中**，因为此时还没有调用 updatePage（addVersion 只在 updatePage 时触发）。如果 A 尚未手动保存过任何中间版本，这些修改永久丢失。

#### 选项 3：手动合并（隐式）

MergeView 左栏是可编辑的，用户可以：
1. 参考右栏差异，在左栏中手动整合 A 和 B 的修改
2. 选 Use Local → 左栏当前编辑内容被作为新内容
3. 再次保存，作为新版本覆盖写入

这是唯一能同时保留双方修改的方式，但完全依赖用户手动操作。

### 11.4 pageHistory 版本链

**代码位置：** `server/models/pageHistory.js:91-243`

每次 `updatePage`/`createPage`/`movePage` 都会插入一条 pageHistory 记录：

```
pageHistory 表字段：
  id           主键
  pageId       → pages.id
  path, title, description, content, contentType   // 完整内容快照
  isPrivate, isPublished, publishStartDate, publishEndDate
  authorId     // 操作作者
  editorKey    // 编辑器类型
  localeCode   // 语言
  hash         // 路径 hash
  action: 'created' | 'updated' | 'deleted' | 'moved' | 'restored'
  versionDate  // 快照对应的原始页面 updatedAt
  createdAt    // 这条历史记录写入时间
  tags         // (ManyToMany) pageHistoryTags → tags
```

版本链查询 (`getHistory`)：
```
ORDER BY versionDate DESC，相邻版本比较 path 差异，自动标注 actionType：
  initial  — 最早的版本（页面创建）
  move     — path 变化了（页面移动/重命名）
  edit     — path 没变（内容修改）
```

### 11.5 多用户场景下的 pageHistory 链

回到场景 T8 之后的结果：

```
pageHistory for Page P:
  ┌────┬─────────┬──────────┬──────────┬──────────────┐
  │ id │ version │ action   │ author   │ content      │
  ├────┼─────────┼──────────┼──────────┼──────────────┤
  │ 3  │ 10:02   │ updated  │ 用户A    │ A的最终版本   │  ← 覆盖了B
  │ 2  │ 10:01   │ updated  │ 用户B    │ B的修改      │  ← 保留
  │ 1  │ 10:00   │ initial  │ 原作者   │ Version 0    │  ← 保留
  └────┴─────────┴──────────┴──────────┴──────────────┘
```

管理员或用户可以通过 GraphQL `pages.version()` 取回 id=2 的版本内容，手动恢复 B 的修改。

### 11.6 多实例部署下的一致性

**代码位置：** `server/core/db.js:231-288`

启用 HA 模式且使用 PostgreSQL 时，通过 Postgres `LISTEN/NOTIFY` 实现跨实例缓存失效：

```
实例A: updatePage(page) → deletePageFromCache(page.hash)
         → WIKI.events.outbound.emit('deletePageFromCache', hash)
         → notifyViaDB → pg NOTIFY 'wiki', payload={ source:INSTANCE_A, event:'deletePageFromCache', value:hash }

实例B: pg LISTEN 'wiki' → 收到 payload (source != INSTANCE_B)
         → WIKI.events.inbound.emit('deletePageFromCache', hash)
         → pages.subscribeToEvents → 清除本地内存中的 page 缓存
```

**效果**：实例B下一次渲染同一页时，不会从缓存拿到旧内容，保证跨实例的读一致性。但**不会**主动通知实例B上正在编辑的用户（没有 WebSocket 推送到前端），冲突仍依赖实例B的5秒轮询 `checkConflicts` 检测。

### 11.7 多用户策略总结

| 层面 | 策略 | 能力 |
|------|------|------|
| **实时协同** | 未实现 | ❌ 无法做到 A 输入 B 实时可见 |
| **冲突检测** | 乐观锁 + `updatedAt` 时间戳 + 5秒轮询 | ✅ 5秒内发现冲突（保存时再检查一次） |
| **冲突解决** | 二选一：Local Wins / Remote Wins + 可选手动合并 | ⚠️ 无自动 3-way merge，选 Remote 时本地修改不进历史 |
| **版本保留** | pageHistory 完整快照，每次写入都生成新版本 | ✅ 保存成功的修改都可回溯 |
| **跨实例** | PG LISTEN/NOTIFY 缓存失效 | ⚠️ 仅保证读缓存一致，前端编辑器靠轮询 |
| **Git 层** | git rebase 自动合并非冲突行 | ✅ 存储层自动合并，只在真正冲突时中断 |

**设计哲学**：Wiki 作为低频协作文档系统，假设多人同时编辑同一页的概率较低，因此采用了轻量的乐观锁方案，而不是复杂度高的 CRDT/OT 协同方案。

---

## 12. 跨集群多节点同步的冲突仲裁

### 12.1 架构前提：无分布式锁/选主

本项目的多实例（HA）部署基于 **PostgreSQL LISTEN/NOTIFY**，但**没有实现分布式锁、选主或冲突仲裁机制**：

| 分布式协调功能 | 实现状态 | 说明 |
|---------------|---------|------|
| Leader 选举 | ❌ 未实现 | 无 Raft/ZooKeeper/Redis 选主，所有节点都是对等的 |
| 分布式锁 | ❌ 未实现 | 无 `pg_advisory_lock` / Redis Redlock / 数据库行级锁 `FOR UPDATE` |
| 全局序列号 | ❌ 未实现 | 无逻辑时钟（Lamport）、向量时钟或 HLC |
| 事务冲突仲裁 | ❌ 未实现 | 没有 Last-Writer-Wins 的自动决策逻辑在跨节点层面 |
| 缓存失效 | ✅ PG LISTEN/NOTIFY | 节点间通过 Postgres pub/sub 清除缓存 |

**代码位置：**
- `server/core/db.js:229-288` — HA 模式下的缓存失效通知
- `server/index.js:17` — `IS_MASTER: true` 是硬编码的，所有节点都认为自己是 master
- `server/core/db.js:210-216` — 只有 `IS_MASTER=true` 的节点运行数据库迁移

### 12.2 多实例冲突场景

**场景：3 节点集群（A、B、C）都启用了 Git 双向同步**

```
T0: 三个节点各自独立运行 sync-storage job，默认每 5 分钟一次

T1: 节点A执行 sync()
    → git pull --rebase → 拉取远程变更 → git push → 处理 diff → updatePage
    → updatePage 更新 DB updatedAt = 10:00:01
    → WIKI.events.outbound.emit('deletePageFromCache', page.hash)
    → notifyViaDB() → pg NOTIFY 'wiki' { source: INSTANCE_A, event:'deletePageFromCache' }

T2: 节点B和节点C收到通知
    → pg LISTEN 'wiki' → payload.source !== INSTANCE_B → WIKI.events.inbound.emit
    → pages.subscribeToEvents → 清除本节点内存中的 page 缓存

T3: 几乎同时，节点B也触发 sync()
    → git pull --rebase（正常，因为节点A刚 push，远程 HEAD 已更新）
    → 本地仓库与远程一致，diffSummary 为空 → 不处理任何页面
    → 无事发生（幂等）

T4: 极端冲突：节点A和节点B在完全相同的时间点开始 sync()
    节点A: git pull --rebase（远程 HEAD = X）→ 本地 commit → git push
    节点B: git pull --rebase（远程 HEAD = X，因为A还没 push）
         → 本地 commit → git push（失败！因为远程 HEAD 已被节点A推进）
         → 抛出 "non-fast-forward" 错误
         → sync-storage.js 捕获 → state.status = 'error'
         → 下次调度时重试，自动恢复
```

### 12.3 现有保护机制

#### 12.3.1 Git 协议级保护

Git 本身提供了**写保护**：
- `git push` 默认是 fast-forward 检查，无法覆盖他人已推送的 commit
- 节点B在 T4 的 push 失败不会影响节点A已成功的 push
- 失败节点下一次调度会自动 `git pull --rebase` 拉取节点A的 commit，然后重试
- **最终一致性**：只要有一个节点 push 成功，下次 sync 时所有节点都会同步

#### 12.3.2 DB 层面乐观锁

页面更新时使用 `updatedAt` 时间戳作为乐观锁：
- `checkConflicts(id, checkoutDate)` → 检测 DB 中的 `updatedAt` 是否比本地 `checkoutDate` 更新
- 这在跨节点场景下也生效，因为所有节点共享同一个 PostgreSQL 数据库
- 节点A的 `updatePage` 更新了 `updatedAt`，节点B上正在编辑的用户会通过 Apollo 轮询检测到冲突

#### 12.3.3 事件源排除（source 过滤）

PG NOTIFY 的 payload 中包含 `source: WIKI.INSTANCE_ID`：
```javascript
// db.js:251
if (_.has(payload, 'event') && payload.source !== WIKI.INSTANCE_ID) {
  WIKI.events.inbound.emit(payload.event, payload.value)
}
```
- 节点只处理其他节点发出的事件，避免自己发的事件自己又处理（循环广播）
- 保证缓存失效不会形成广播风暴

### 12.4 关键缺陷与风险

| 风险场景 | 后果 | 严重程度 |
|---------|------|---------|
| 多个节点同时执行 `syncUntracked()` | 重复 git add + git commit → 多个空 commit 或冲突 | 中 |
| 多个节点同时执行 `importAll()` | 重复 `updatePage` → 页面内容后写入的覆盖先写入的（无事务保护） | 高 |
| 节点A执行 `importAll()` 中途，节点B执行 `sync()` | 部分导入的页面被节点B的 sync 重新从 Git 拉取 → 可能丢失 DB 未提交的变更 | 高 |
| 多个节点同时 purge() | 多个节点同时清空并 clone 同一仓库 → 无功能影响但浪费资源 | 低 |

**重要：所有同步操作（sync/importAll/syncUntracked/purge）在多节点环境下都是并发执行的，没有互斥保护。** 管理员在多实例部署下手动触发这些操作时，必须确保只有一个节点执行。

### 12.5 IS_MASTER 的真实含义

`server/index.js:17` 中 `IS_MASTER: true` 是**每节点独立的硬编码标记**，不是集群选举结果：

- 所有节点都 `IS_MASTER: true`
- 仅在 `db.js:210` 中用于决定是否运行数据库迁移任务
- 所有节点都会：注册 sync-storage job、执行同步、处理 pageEvent
- 没有"工作节点/主节点"的角色区分

---

## 13. 同步任务的优先级调度与限流

### 13.1 调度器设计

**代码位置：** `server/core/scheduler.js`

本项目的调度器是一个**极简的 setTimeout 包装器**，不支持优先级队列。

#### Job 类结构

```javascript
class Job {
  constructor({
    name,
    immediate = false,     // 是否立即执行
    schedule = 'P1D',      // 首次执行延迟，或两次执行之间的间隔
    repeat = false,        // 是否重复调度
    worker = false         // 是否在子进程中执行
  }, queue)
```

#### 调度方式分类

| 配置 | 触发时机 | 适用场景 |
|------|---------|---------|
| `immediate: true, repeat: false` | 注册后立即执行一次，执行完退出 | 一次性任务：rebuildTree |
| `immediate: true, repeat: true` | 注册后立即执行，执行完按 schedule 间隔重复 | 启动即执行：syncGraphLocales |
| `immediate: false, repeat: false` | 延迟 schedule 时间后执行一次 | 单次延迟任务 |
| `immediate: false, repeat: true` | 延迟 schedule 时间后首次执行，之后按间隔重复 | 定时轮询：sync-storage（Git PT5M） |

### 13.2 系统内置任务优先级

**代码位置：** `server/app/data.yml:120-141`

系统启动时自动注册 4 个内置任务：

```yaml
jobs:
  purgeUploads:           # 清理上传临时文件
    onInit: true          # 启动时立即执行
    schedule: PT15M       # 每 15 分钟
    repeat: true
  syncGraphLocales:       # 同步多语言图数据
    onInit: true
    schedule: P1D         # 每天一次
    offlineSkip: true     # 离线模式跳过
    repeat: true
  syncGraphUpdates:       # 同步图更新
    onInit: true
    schedule: P1D
    offlineSkip: true
    repeat: true
  rebuildTree:            # 重建页面导航树
    onInit: true
    immediate: true
    worker: true          # 在独立子进程中执行
    repeat: false
```

此外，每个存储目标独立注册自己的 `sync-storage` job，间隔由模块 `definition.yml` 的 `schedule` 字段或用户配置的 `syncInterval` 决定。

### 13.3 限流与并发控制

#### 13.3.1 无全局并发限制

**当前实现不提供任何限流机制：**

- 调度器的 `queue` 只是一个数组 `jobs: []`，用于存引用方便 stop，没有实际队列逻辑
- 没有 `maxConcurrent` / `concurrency` 限制
- 没有任务去重：同一个存储目标重复调用 `initTargets()` 会注册多个相同的 sync 任务
- 没有背压（backpressure）：如果一次 sync 执行了 6 分钟，而下一个 5 分钟间隔已到，**会并发执行**

**风险场景：**
- Git 远程仓库响应慢，一次 sync 执行超过 5 分钟 → 下一次 sync 启动，并发执行
- 两次并发的 `git pull --rebase` 操作同一本地仓库 → git 锁冲突（`.git/index.lock`）
- 第二次 sync 失败 → `state.status = 'error'`，但下次调度仍按 5 分钟间隔启动

#### 13.3.2 Worker 子进程隔离

**代码位置：** `scheduler.js:55-79`

`worker: true` 的任务会通过 `childProcess.fork` 在独立子进程中执行：

```javascript
const proc = childProcess.fork(`server/core/worker.js`, [
  `--job=${this.name}`,
  `--data=${data}`
], { cwd: WIKI.ROOTPATH, stdio: ['inherit', 'inherit', 'pipe', 'ipc'] })
```

**作用：**
- CPU 密集型任务（如 rebuildTree）不阻塞主事件循环
- 内存密集型任务执行完进程退出，自动释放内存
- 但**不限制并发子进程数**，多个 worker 任务可以同时 fork

**当前使用 worker 的任务：**
- `rebuildTree` — 重建导航树（大量路径计算）
- `sanitize-svg` — SVG 安全扫描（`assets.js:110` 手动注册）
- `render-page` — 页面渲染（手动注册）

#### 13.3.3 存储模块的隐式互斥

Git 模块通过 git 自身的锁机制（`.git/index.lock`）提供**隐式并发保护**：
- 如果前一次 sync 还在进行（本地仓库被锁），后一次 sync 执行 `git pull` 时会抛出 "Another git process seems to be running"
- 异常被捕获 → sync 标记为 error
- 下次调度自动重试

**SFTP/Azure/S3 模块没有任何隐式互斥**，并发 push 时可能对同一文件执行多次 putObject/rename。

#### 13.3.4 离线模式限流

`offlineSkip: true` 配置（`scheduler.js:111-114`）：
```javascript
if (WIKI.config.offline && queueParams.offlineSkip) {
  WIKI.logger.warn(`Skipping job ${queueName} because offline mode is enabled. [SKIPPED]`)
  return
}
```
在离线模式下自动跳过需要联网的任务（syncGraphLocales、syncGraphUpdates）。

### 13.4 sync-storage 调度细节

**代码位置：** `server/models/storage.js:145-153`

```javascript
if (targetDef.schedule && target.syncInterval !== `P0D`) {
  const schedule = target.syncInterval || targetDef.schedule
  WIKI.scheduler.registerJob({
    name: `sync-storage`,
    schedule: schedule,
    repeat: true
  }, target.key)
}
```

- 每个存储目标**独立注册**自己的 sync-storage job
- 任务名称都是 `sync-storage`，通过传入的 `target.key` 区分执行目标
- 多个存储目标并行执行，互不影响
- 如果配置了 `internalSchedule`（Git 没有），则使用内部调度间隔

### 13.5 优先级缺失的实际影响

由于没有优先级队列，任务调度具有以下特性：

1. **FIFO 但并行**：`registerJob` 顺序决定 `setTimeout` 注册顺序，但 setTimeout 到期后都是并行执行
2. **无抢占**：正在执行的任务不会被高优先级任务打断
3. **无动态优先级调整**：网络波动后不会自动提高 sync 频率加速恢复
4. **手动触发优先级最高**：管理员通过 GUI 触发的 Force Sync 立即执行，不排队

---

## 14. 冷启动全量初始化的性能优化

冷启动包括两种场景：
1. **首次激活**：存储模块从无到有建立同步
2. **故障恢复**：purge 之后重新初始化

### 14.1 首次激活 Git 存储的流程

```
管理员启用 Git 模块并保存配置
  ↓
Storage.initTargets()
  ├─ require git/storage.js
  ├─ target.fn.init()
  │   ├─ 检查本地 repoPath 目录是否存在
  │   ├─ 不存在 → fs.ensureDir → git.clone(remoteUrl)
  │   ├─ 存在且非空 → 检查是否为有效 git 仓库
  │   ├─ git.checkout(branch)
  │   ├─ git.addConfig('user.name', config.defaultName)
  │   └─ git.addConfig('user.email', config.defaultEmail)
  └─ 注册 sync-storage job（PT5M）

首次 sync() 触发（5分钟后，或管理员 Force Sync）
  ├─ git.pull --rebase
  ├─ git.push
  ├─ diffSummary（空，因为刚 clone）
  └─ 发现 diff 为空，跳过处理

此时 DB 中有页面，但 Git 仓库中没有 → 需要管理员手动触发 syncUntracked
  └─ syncUntracked() → 流式导出所有页面和资产到 git → git add → git commit
```

### 14.2 流式处理（Stream + Pipeline）

所有全量操作（`importAll`、`syncUntracked`、`exportAll`、`dump`）都使用 **Node.js Stream + Transform + pipeline** 模式，避免一次性加载所有数据到内存。

**代码模式：** `git/storage.js:437-464`

```javascript
const { pipeline } = require('node:stream/promises')
const { Transform } = require('node:stream')

await pipeline(
  // 源：knex query stream → 从数据库流式读取，每次一批
  WIKI.models.knex.column('id', 'path', ...).select().from('pages').where({...}).stream(),

  // 转换：Transform 流，每次处理一行
  new Transform({
    objectMode: true,
    transform: async (page, enc, cb) => {
      // 处理单个页面：injectMetadata + fs.outputFile + git add
      const filePath = path.join(this.repoPath, `${page.path}.md`)
      await fs.outputFile(filePath, pageHelper.injectPageMetadata(page))
      await this.git.add(`./${page.path}.md`)
      cb()
    }
  })
)
```

**优化点：**
- `knex().stream()` 使用数据库游标，每次 fetch 一批（默认 100 行），内存占用 O(batch_size)
- `pipeline` 自动处理背压（Transform 处理不过来时，自动暂停上游读取）
- `objectMode: true` 允许在流中传递 JS 对象而非 Buffer
- 出错时 `pipeline` 自动销毁所有流，防止资源泄漏

### 14.3 全量导入的幂等性

**代码位置：** `git/storage.js:452-459`

`importAll()` 传入 `importAll: true` 标志：

```javascript
await this.processFiles([{
  ...
  deletions: 0,
  insertions: 0,
  importAll: true      // ← 关键标志
}], rootUser)
```

**`importAll: true` 对 processFiles 的影响：**
- 跳过 rename 检测逻辑（`git/storage.js:202, 216, 245, 261`）
- 跳过 delete 检测逻辑
- 所有文件统一走 `commonDisk.processPage` / `processAsset`
- `commonDisk.processPage` 中 `getPageFromDb` 检查是否存在：
  - 已存在 → `updatePage`（内容覆盖）
  - 不存在 → `createPage`（新增）

**性能特性：**
- 遍历仓库所有文件（包括 `.git` 之外的全部）
- 对每个文件执行 DB 查询（`getPageFromDb`）
- 无论内容是否变化都会执行 `updatePage`（即使内容相同）
- 每次 `updatePage` 都会：`addVersion` → `patch` → `renderPage` → `search.updated`
- **这是一个 O(N) 的"暴力"全量覆盖操作**，在页面数量大时（>10000）可能耗时较长

### 14.4 批处理优化（仅在搜索模块实现）

批量写入的优化模式在搜索模块中实现（`elasticsearch/engine.js:320-399`），但**同步模块没有采用**：

```javascript
// Elasticsearch 批量优化（参考模式）
const MAX_INDEXING_BYTES = 10 * 1024 * 1024  // 10MB
const MAX_INDEXING_COUNT = 500

let chunks = []
let bytes = 0

const flushBuffer = async () => {
  await this.client.bulk({ body: flatten(chunks) })
  chunks.length = 0
  bytes = 0
}

// 每条记录累积，达到大小或数量阈值时 flush
```

**同步模块为什么不批量：**
- 每个页面处理涉及：fs 读写 + git add + DB 查询 + DB 更新 + 搜索索引更新
- 多个页面之间没有依赖，可以流式串行处理
- 但**没有并行处理**（`Transform` 是串行的，`async (page, enc, cb)` 中 `await` 完成才调用 `cb()`）
- 搜索模块的批量模式是因为 Elasticsearch bulk API 有显著的性能收益，而同步模块没有对应的批量接口

### 14.5 缓存机制

- **磁盘路径缓存**：`commonDisk.clearFolderCache()` 在 `importAll()` 完成后清除
- **页面缓存**：NodeCache 内存缓存，HA 模式下通过 LISTEN/NOTIFY 失效
- **Git 对象缓存**：由 git 自身管理（`.git/objects`），无需应用层干预

### 14.6 性能瓶颈与优化空间

| 操作 | 时间复杂度 | 主要瓶颈 | 潜在优化 |
|------|-----------|---------|---------|
| `sync()` 增量 | O(changed_files) | git pull/push 网络 RTT | 可接受，5分钟一次 |
| `importAll()` | O(total_files × DB_latency) | 逐文件 getPageFromDb 查询 | 改为批量查询（WHERE path IN (...)） |
| `syncUntracked()` | O(total_pages + total_assets) | fs.outputFile 单文件写入 | 可并行写入（Promise.all 分批） |
| `purge()` | O(1) | git clone 网络 RTT + 仓库大小 | 可接受，手动操作 |
| `dump()`（disk） | O(total_pages + total_assets) | fs.outputFile 串行写入 | 可并行 |

### 14.7 冷启动性能数据估算

假设 10000 个页面，平均每个页面 5KB：

| 操作 | 估算耗时 | 说明 |
|------|---------|------|
| 首次 `git clone`（100MB 仓库） | 5-30s | 取决于网络 |
| `syncUntracked()`（10000 页面） | 50-100s | 5-10ms per page（fs + git add）+ 一次 commit |
| `importAll()`（10000 页面） | 100-200s | 10-20ms per page（fs read + frontmatter parse + DB query + DB update + search index）|
| 首次 `sync()` 空 diff | < 1s | 几乎无成本 |

---

## 15. 存储模块容量预估与监控告警

### 15.1 容量预估：各存储后端的空间消耗

Wiki.js 的存储内容主要由**页面文本**和**资产二进制**两部分组成，不同存储后端的额外开销不同：

| 存储后端 | 内容存储 | 元数据开销 | 历史版本开销 | 冗余副本 | 典型空间倍率（相对 DB） |
|---------|---------|-----------|------------|---------|----------------------|
| **PostgreSQL DB**（主存储） | pages.content + assetData (bytea) | 索引 + 表结构 | pageHistory 全量快照 | 0（单副本，依赖 DB 备份） | 1×（基线） |
| **Git** | 与 DB 相同的明文文件 | frontmatter（~100B per file） | `.git/objects` 打包压缩所有历史 commit | 本地 1 份 + 远程 1 份 | 1.5-3×（历史越多倍率越高） |
| **Local Disk** | 与 DB 相同的明文文件 | frontmatter（~100B per file） | 可选 daily backup tar.gz（`createDailyBackups`） | 本地 1 份（可挂 NFS 多副本） | 1-10×（备份保留 30 天） |
| **S3** | 每个对象独立存储 | 对象元数据（HTTP headers） | 可选 S3 Versioning（Wiki.js 不主动开启） | 配置的 replication factor（默认 3×） | 3-10×（含云厂商复制） |
| **Azure Blob** | 每个 blob 独立存储 | blob 属性 + 存储层元数据 | 可选 Blob Versioning（Wiki.js 不主动开启） | 3× LRS / 12× GRS | 3-15× |
| **SFTP** | 明文文件传输 | 无额外元数据 | 无内置版本控制 | 远端 1 份（依赖远端策略） | 1× |

#### 容量估算公式

```
单页面存储大小估算：
  Git/Disk/SFTP:  frontmatter(100B) + content_size + fs_block_overhead(4KB 取整)
  S3/Azure:       content_size + object_overhead(≈200B)

资产存储大小估算：
  所有后端: asset_binary_size（资产不携带 frontmatter）

Git 额外开销：
  .git 目录 ≈ (总内容大小 × 0.3 压缩比) × 历史 commit 数量因子
  粗略估算：每 1000 次 commit 增加 0.1-0.5× 当前内容大小
```

**估算示例：** 10000 页面，平均 5KB；1000 资产，平均 500KB
- DB: ≈ 10000×5KB + 1000×500KB = 50MB + 500MB ≈ **550MB**
- Git（本地 + 远程，含历史）: ≈ 550MB × 2 × 1.5 ≈ **1.6GB**
- S3（3× 复制）: ≈ 550MB × 3 ≈ **1.6GB**
- Disk（含 30 天每日备份）: ≈ 550MB + 550MB×30（压缩比 0.5）≈ **8.8GB**

### 15.2 监控机制：三态状态机

存储目标的状态通过 `state` JSONB 字段追踪，暴露在 GraphQL `storage.status` 查询中：

```javascript
// models/storage.js → storage 表 state 字段
state: {
  status: 'pending' | 'operational' | 'error',
  message: '错误详情字符串',
  lastAttempt: 'ISO 8601 时间戳'
}
```

**状态流转：**

```
        initTargets() → init()
  ┌────────────── pending ──────────────┐
  │                                      │
  │  init()成功 / sync()成功             │ sync()失败 / init()失败
  │                                      │
  ▼                                      ▼
operational ──────────────────────► error
  ▲   │                                │
  │   │ 下一次 sync()成功              │ 管理员 Force Sync 成功
  │   │                                │
  │   └────────────────────────────────┘
  │
  │ updateTargets() → 重置为 pending
  └───────────────────────────────────────
```

**代码位置：**
- `sync-storage.js:15-31` — 每次 sync 更新 state（成功 operational / 失败 error + message）
- `resolvers/storage.js:40-52` — `StorageQuery.status` 查询暴露给管理后台
- `resolvers/storage.js:55-89` — `StorageMutation.updateTargets` 重置 state 为 pending

### 15.3 管理后台监控面板

**代码位置：** `client/components/admin/admin-storage.vue:37-80`

管理后台 `Storage` 页实时展示：

| 状态 | 图标 | 颜色 | 显示内容 |
|------|-----|------|---------|
| `pending` | `mdi-clock-outline` | 紫色 | 状态 + 进度环 |
| `operational` | `mdi-check-circle` | 绿色 | "最后同步于 X 时间前"（moment from） |
| `error` | `mdi-close-circle-outline` | 红色 | "最后一次尝试 X 时间前" + 详情按钮弹出错误 message |

**刷新机制：** Apollo `pollInterval` 定时轮询 GraphQL `storage.status` 查询，不是 WebSocket 推送。

### 15.4 告警机制

**当前实现：无主动告警推送。**

| 告警能力 | 实现状态 | 说明 |
|---------|---------|------|
| 管理后台可视化告警 | ✅ | 红色状态 + 错误详情 |
| 日志记录 | ✅ | `WIKI.logger.warn(err)` + `sync-storage.js` 写 message 到 DB |
| 邮件告警 | ❌ | 未实现 |
| Webhook 告警 | ❌ | 未实现 |
| Prometheus 指标 | ❌ | 未暴露 `/metrics` endpoint |
| 容量告警 | ❌ | 无磁盘/空间/配额监控 |
| 失败次数阈值告警 | ❌ | 不统计连续失败次数，没有"连续失败 N 次自动禁用"逻辑 |

管理员只能通过登录管理后台查看状态，或扫描服务器日志发现同步失败。

### 15.5 Telemetry（遥测）

**代码位置：** `server/core/telemetry.js`

WIKI.telemetry 存在但用途极有限：
- 仅在 `sendInstanceEvent('STARTUP'/'INSTALL')` 时向官方遥测端点发送匿名系统信息（OS、DB 类型、CPU、RAM、Node 版本）
- `sendEvent()` 和 `sendError()` 方法体为空（`// TODO`）
- 不追踪存储模块的任何运行数据
- 可在管理后台开关（`system.js:80`）

---

## 16. 不同存储后端的能力差异

### 16.1 配置定义层面的差异

各存储后端的 `definition.yml` 定义了模块的能力边界：

| 特性 | Git | Local Disk | S3 | Azure Blob | SFTP | 云端占位* |
|------|:-:|:----------:|:--:|:----------:|:----:|:--------:|
| `isAvailable` | ✅ true | ✅ true | ✅ true | ✅ true | ✅ true | ✅ true |
| `supportedModes` | `sync/push/pull` | `push` | `push` | `push` | `push` | 仅 `push` |
| `defaultMode` | `sync` | `push` | `push` | `push` | `push` | `push` |
| `schedule` | PT5M | false | false | false | false | false |
| `internalSchedule` | — | P1D | — | — | — | — |
| 模块文件实现完整度 | 100% | 90% | 80% | 80% | 80% | 5%（空桩） |

> *云端占位 = dropbox / gdrive / onedrive / box，`definition.yml` 存在但 `storage.js` 方法体全部为空或 return null

### 16.2 核心接口能力矩阵

| 接口 | 职责 | Git | Disk | S3 | Azure | SFTP |
|------|------|:-:|:----:|:--:|:-----:|:----:|
| `init()` | 初始化连接/仓库 | ✅ | ✅ | ✅ | ✅ | ✅ |
| `sync()` | 双向同步调度 | ✅ | — | — | — | — |
| `created(page)` | 页面创建推送 | ✅ | ✅ | ✅ | ✅ | ✅ |
| `updated(page)` | 页面更新推送 | ✅ | ✅ | ✅ | ✅ | ✅ |
| `deleted(page)` | 页面删除推送 | ✅ | ✅ | ✅ | ✅ | ✅ |
| `renamed(page)` | 页面重命名推送 | ✅ | ✅ | ✅ | ✅ | ✅ |
| `assetUploaded(asset)` | 资产上传推送 | ✅ | ✅ | ✅ | ✅ | ✅ |
| `assetDeleted(asset)` | 资产删除推送 | ✅ | ✅ | ✅ | ✅ | ✅ |
| `assetRenamed(asset)` | 资产重命名推送 | ✅ | ✅ | ✅ | ✅ | ✅ |
| `getLocalLocation(asset)` | 获取资产本地访问路径 | ✅（读本地仓库） | ✅（读本地备份目录） | ⚠️（空字符串，需走 HTTP URL） | ⚠️（空字符串） | ⚠️（空字符串） |
| `dump()` | DB→存储 全量导出 | ✅（含 git commit） | ✅（tar.gz 备份） | — | — | — |
| `backup()` | 创建本地备份归档 | — | ✅ | — | — | — |
| `syncUntracked()` | DB→仓库 增量补全 | ✅ | — | — | — | — |
| `importAll()` | 存储→DB 全量导入 | ✅ | ✅ | — | — | — |
| `exportAll()` | DB→存储 全量覆盖 | — | — | ✅ | ✅ | ✅ |
| `purge()` | 销毁并重建本地仓库 | ✅ | — | — | — | — |

### 16.3 管理操作（Actions）差异

每个模块在 `definition.yml` 的 `actions` 中暴露给管理员可手动执行的操作：

| 操作 | Git | Disk | S3 | Azure | SFTP |
|------|:-:|:----:|:--:|:-----:|:----:|
| Force Sync（立即同步） | ✅ sync | — | — | — | — |
| Add Untracked（补全 DB→仓库） | ✅ syncUntracked | — | — | — | — |
| Import Everything（存储→DB） | ✅ importAll | ✅ importAll | — | — | — |
| Purge Local Repository | ✅ purge | — | — | — | — |
| Dump all content to disk | — | ✅ dump | — | — | — |
| Create Backup | — | ✅ backup | — | — | — |
| Export All（DB→存储全量） | — | — | ✅ exportAll | ✅ exportAll | ✅ exportAll |

### 16.4 内容格式差异

所有模块输出的页面文件格式基本一致，但细节有差异：

| 特性 | Git / Disk | S3 / Azure / SFTP |
|------|:----------:|:-----------------:|
| Frontmatter 注入 | ✅ 完整 `injectPageMetadata` | ✅ 完整 |
| 内容类型 | `.md` / `.html` 按 contentType 区分 | ✅ 相同 |
| 路径命名空间 | `locale/path.ext`（可配置 alwaysNamespace） | ✅ 相同 |
| 资产路径 | `_assets/filename.ext`（含 folder 层级） | ✅ 相同 |
| 实时版本控制 | ✅ git commit 自动记录 | ❌ 对象覆盖写 |
| 每日归档 | — | ✅ Disk 可选 tar.gz（保留 30 天） | ❌ — |
| 文件系统访问 | ✅ 本地路径可用（`getLocalLocation` 返回实际路径） | ❌ 需走云存储 URL |

### 16.5 认证与安全

| 后端 | 支持的认证方式 | 敏感字段加密 |
|------|--------------|------------|
| **Git** | SSH Key（path 或 content） / Basic（用户名+密码/PAT） | SSH key 内容、密码：DB 中 `config.*` 未加密，仅前端展示打码为 `********` |
| **Local Disk** | 操作系统文件权限 | 无 |
| **S3** | Access Key ID + Secret Access Key | Secret：同上，DB 明文存储 |
| **Azure Blob** | Account Name + Account Key | Account Key：同上，DB 明文存储 |
| **SFTP** | 私钥（+ Passphrase） / 密码 | 私钥内容、密码、passphrase：同上，DB 明文存储 |

> **安全隐患**：`sensitive: true` 字段仅在前端 GraphQL 响应中打码（`resolvers/storage.js:31`），DB 中存储明文。如果攻击者获取到数据库，所有存储凭证即泄露。

### 16.6 多模块并存策略

系统支持**同时启用多个存储目标**：

```
pageEvent(page) → for each enabled target:
                    target.fn.created(page)  // 并发执行
```

- 多个模块并行推送，互不干扰
- 一个模块推送失败不影响其他模块（`pageEvent` 的 `try/catch` 在循环外，一个失败将中断后续所有模块）
  - ⚠️ 注意：`storage.js:179-189` 中的 for/of + try/catch 包在循环外层，单个 target 抛异常将跳过剩余 targets
- sync 方向（双向）与 push 方向（单向）模块可并存

---

## 17. 存储失败后回退到只读模式的链路

### 17.1 架构前提：无全局只读模式

**本项目没有实现存储失败后自动回退到"系统只读"的功能。** 代码中不存在任何 read-only / degraded / maintenance 模式的实现：

```bash
$ grep -rn "readOnly\|read-only\|readonly\|maintenance\|failover\|degraded" server/
# 无任何匹配结果
```

### 17.2 存储失败后的实际行为

存储模块失败（同步失败、推送失败）时的真实行为链路如下：

```
场景 A：用户保存页面时存储推送失败

  pages.updatePage(opts)
    ├─ ✅ DB updatePage 成功（写入 pages 表）
    ├─ ✅ pageHistory.addVersion() 成功（写入历史版本）
    ├─ ✅ renderPage() 成功（更新 render 缓存）
    ├─ ✅ searchEngine.updated() 成功（写入搜索索引）
    └─ ❌ storage.pageEvent({ event:'updated', page })
         └─ for (let target of this.targets) {
              await target.fn.updated(page)   ← 此处抛异常
              // 注意：try/catch 在 for 循环外层！
            }
         └─ catch (err) {
              WIKI.logger.warn(err)           // 仅写日志
              throw err                        // 重新抛出！
            }
    └─ ❌ updatePage 整体抛出异常给 GraphQL resolver
         └─ graphHelper.generateError(err)
              → 返回给前端：保存失败的错误消息
              → DB 已提交的更改不回滚！（无事务）
```

**结果：**
- 页面内容已保存到 DB、已更新历史、已更新搜索、已更新缓存（全部成功）
- 但外部存储（Git/S3/Azure/SFTP）未更新，产生 DB ↔ 存储不一致
- 前端用户看到"保存失败"错误提示，但实际上数据已写入主存储
- 用户再次点击保存时，可能因 DB 已最新但用户无感知而产生困惑

```
场景 B：定时 sync 失败（网络波动、远程仓库不可达）

  sync-storage.js job
    ├─ target.fn.sync() → 抛出异常
    ├─ await WIKI.models.storage.query().patch({
         state: { status: 'error', message: err.message, lastAttempt: now }
       })
    └─ WIKI.logger.warn(err)
  → 下一次 sync 调度到时自动重试
  → 用户读写页面完全不受影响（因为 DB 是主存储）
```

**结果：**
- 存储模块状态标记为 error，管理后台显示红色告警
- 前台用户完全感知不到，Wiki 继续正常读写
- DB ↔ 外部存储差距逐渐扩大，直到 sync 恢复正常后增量补偿

### 17.3 不一致的风险窗口

| 操作路径 | 存储失败对用户可见性 | 数据一致性风险 |
|---------|-------------------|-------------|
| **用户保存页面时** pageEvent 抛异常 | ⚠️ 前端提示"保存失败"但实际 DB 已保存 | 低（DB 有最新，外部存储滞后，下次 sync 补偿） |
| **定时 sync 失败** | ❌ 用户不可见，仅管理员可见 | 中（Git 模块：断连期间远程变更不会同步到 DB） |
| **资产上传时** assetEvent 抛异常 | ⚠️ 前端提示上传失败但 DB assetData 已写入 | 中（资产 DB 记录存在但外部存储缺失） |
| **页面删除时** storage.deleted 抛异常 | ⚠️ 前端提示删除失败但 DB 已删除 | 高（DB 中页面已不存在，但外部存储中残留文件） |
| **页面重命名时** storage.renamed 抛异常 | ⚠️ 前端提示失败但 DB 已重命名 | 高（DB 路径已变，外部存储中残留旧路径文件，新路径缺失） |

### 17.4 隐含的降级模式

虽然没有显式的只读模式，但系统存在一些**隐式降级行为**：

#### 17.4.1 DB 是唯一可信源（Source of Truth）

所有用户读写操作都直接操作 DB：
- 读页面：`pages.getPage()` → 直接查 DB，完全不经过存储模块
- 写页面：`pages.updatePage()` → 写 DB，存储推送是附加操作
- 资产访问：`Storage.getLocalLocations()` 只在资产需要读取本地路径时调用存储模块，assetData 存于 DB

**这意味着：存储模块完全不可用时，Wiki 仍可 100% 正常读、写、搜索页面。** 外部存储只是"备份/同步副本"，不是主路径。

#### 17.4.2 资产降级路径

当资产的外部存储路径不可用时，系统可从 DB assetData 表恢复：
```
访问资产时：
  1. 优先从本地文件缓存读取（cache/${asset.folder}/${asset.filename}）
  2. 缓存不存在时，尝试从已启用的存储模块 getLocalLocation 获取
  3. 都不可用时，从 DB assetData 表读取 bytea 二进制数据写入缓存
```
**实际代码中第 3 步的降级逻辑仅在资产上传时启用（用于生成缓存文件），资产下载时主要依赖本地缓存路径。**

#### 17.4.3 离线模式

**代码位置：** `config.js:27`、`scheduler.js:111-114`

系统有一个全局 `offline` 模式配置：
```yaml
# config.yml
offline: true
```
- 禁用遥测（telemetry）
- 所有标记了 `offlineSkip: true` 的调度任务（syncGraphLocales、syncGraphUpdates）自动跳过
- 但存储模块的 sync-storage job **不**受 offline 配置影响，继续执行

### 17.5 pageEvent 异常传播分析

**代码位置：** `server/models/storage.js:178-199`

```javascript
static async pageEvent({ event, page }) {
  try {
    for (let target of this.targets) {
      await target.fn[event](page)   // 串行！一个失败，后续全部跳过
    }
  } catch (err) {
    WIKI.logger.warn(err)
    throw err   // ← 重新抛出给调用方
  }
}
```

**两个问题：**
1. **串行执行**：target1 推送 300ms、target2 推送 300ms → total 600ms。没有 `Promise.all` 并行
2. **第一个失败即中断全部**：target1 失败，target2（即使是另一个完全独立的存储）不会被尝试。对于启用了多个存储目标（如 Git + S3 同时推送）的部署，这会导致所有目标状态不一致

### 17.6 理论上的只读模式实现路径（基于现有代码的可行改造）

如果需要实现存储失败后自动切换只读模式，可基于以下现有组件扩展：

```
建议的改造链路：

1. 在 sync-storage job 中增加连续失败计数：
   state: { status, message, lastAttempt, consecutiveFailures: N }

2. 当 consecutiveFailures > 阈值（如 3 次）：
   - 通过 PG NOTIFY 广播 "storageDegraded" 事件
   - 在 WIKI 全局状态中标记 isStorageDegraded = true

3. 在 pages.createPage / updatePage / deletePage 入口增加检查：
   if (WIKI.isStorageDegraded && WIKI.config.storage.failoverToReadOnly) {
     throw new Error('系统处于只读维护模式，请稍后再试')
   }

4. 页面渲染时增加 banner 提示：
   "当前处于只读模式，部分功能暂不可用"
```

**但在当前代码中，以上逻辑均未实现。** 存储失败后没有任何自动故障切换机制。

---

## 18. 存储迁移工具与零停机切换

### 18.1 迁移工具：Wiki.js 1.x → 2.x 导入

系统提供了从 Wiki.js 1.x 版本迁移到 2.x 的官方导入工具，管理后台路径：**Administration → Utilities → Import from Wiki.js 1.x**。

**代码位置：**
- 前端：`client/components/admin/admin-utilities-importv1.vue`
- 后端：复用现有的 storage 模块 `importAll()` / sync 机制

#### 迁移内容选项

| 可迁移内容 | 数据源 | 实现方式 |
|-----------|-------|---------|
| **Content + Uploads**（页面+资产） | Git 仓库 或 本地磁盘文件夹 | 配置对应 storage 模块后执行 `importAll()` |
| **Users**（用户） | Wiki.js 1.x MongoDB 数据库 | 直接连接 MongoDB 读取用户数据写入 PG |

#### Git 迁移流程（推荐）

```
管理员选择 "Import from Git Connection" → 填写 Git 配置（与 Git 存储模块配置相同）
  ↓
前端调用 storage.updateTargets mutation
  ├─ 将 Git 模块配置写入 DB（替换任何已有的 Git 配置）
  └─ 将 Git 模块 isEnabled 设为 true
  ↓
Storage.initTargets()
  ├─ 初始化 Git 模块 → git clone 远程仓库到本地 repoPath
  └─ 注册 sync-storage 定时任务
  ↓
轮询 storage.status 直到状态变为 operational
  ↓
调用 storage.executeAction('git', 'importAll')
  └─ git.importAll() → 遍历所有文件 → processFiles(importAll:true) → 逐个写入 DB
  ↓
迁移完成 → 页面/资产出现在 Wiki 中
```

**注意事项（管理后台明确提示）：**
- 新的本地 repoPath 必须为空或不存在，**不要指向旧的 1.x 仓库文件夹**
- v1 和 v2 可以共用同一个远程 Git 仓库，但**不应同时编辑相同页面**
- 迁移过程中，原有 Git 配置会被替换
- 迁移用户时，必须先删除目标系统中要迁移的同名用户

#### 本地磁盘迁移流程

```
管理员选择 "Import from local folder" → 填写 contentPath（1.x content 目录绝对路径）
  ↓
启用 Disk 存储模块，mode = push，path = contentPath
  ↓
执行 disk.importAll() → klaw 遍历所有文件 → 写入 DB
```

### 18.2 存储后端切换（存储迁移）

**场景**：从 Local Disk 切换到 Git，或从 Git 切换到 S3。

系统**没有提供专门的"一键切换"工具**，但可以通过以下手动步骤实现接近零停机的切换：

```
零停机切换步骤（示例：Disk → Git）：

1. 在管理后台启用 Git 模块（不要禁用 Disk）
   → 此时写入会同时推送到 Disk 和 Git
   → 已有内容通过 Force Sync 或 syncUntracked 全量同步到 Git

2. 等待 Git 同步完成，状态变为 operational
   → 验证 Git 仓库内容完整性

3. 在管理后台禁用 Disk 模块（只保留 Git）
   → 读写不中断（DB 始终是主存储）
   → 新的写入只会推送到 Git

4. （可选）执行一次 purge + importAll 从 Git 回导到 DB
   → 确保 DB 与新存储完全一致
```

**零停机的核心保证：DB 是唯一的 Source of Truth**
- 所有读写操作直接走 DB，存储模块只做异步备份/同步
- 存储模块的启用/禁用/切换完全不影响用户的读写操作
- 多个存储模块可以同时启用并行推送

### 18.3 迁移风险与保护

| 风险场景 | 现有保护 | 手动建议 |
|---------|---------|---------|
| 迁移中途网络中断，Git clone 失败 | ❌ 无断点续传，下次重新 clone | 网络稳定时执行，大仓库建议预下载 |
| importAll 中途失败，部分页面已导入 | ✅ 幂等：已导入页面下次会 update 而非重复 | 重新执行 importAll 即可 |
| 迁移期间用户修改页面 → 冲突 | ⚠️ Git importAll 会用 Git 内容覆盖 DB 最新 | 迁移期间建议设置公告，限制编辑 |
| 迁移完成后旧存储残留数据 | ❌ 无自动清理 | 手动删除旧存储中的文件 |

### 18.4 其他迁移工具

| 工具 | 用途 | 代码位置 |
|------|------|---------|
| **Locale Migration** | 将页面从一个语言批量迁移到另一个语言 | `pages.migrateToLocale()` in `models/pages.js:1133` |
| **Content Export**（System → Utilities） | 将内容/用户/配置导出为 JSON 文件 | `core/system.js:100-450`（全量流式导出） |
| **Content Import**（System → Utilities） | 将 JSON 备份文件导入回系统 | 导入功能在 `core/system.js` 中以 TODO 标记 |

> ⚠️ 注意：Content Import 功能（从 JSON 备份恢复）在当前代码中**尚未实现**，方法体仅包含 TODO 注释。

### 18.5 零停机架构本质

Wiki.js 的存储架构天然支持零停机切换，关键设计决策：

1. **DB 中心化**：PostgreSQL 是唯一可信源，存储模块是附加层
2. **事件驱动异步推送**：pageEvent 不阻塞主操作，失败可重试
3. **多模块并行**：支持同时启用多个存储目标，并行推送
4. **幂等操作**：importAll / syncUntracked 可重复执行，不会产生重复数据

---

## 19. 附件大文件流式处理与分片上传

### 19.1 上传流程总览

附件（资产）上传的完整链路：

```
客户端浏览器
  │  POST /u  multipart/form-data
  ▼
server/controllers/upload.js
  ├─ multer 中间件接收文件到临时目录
  │   dest: ./data/uploads
  │   limits: { fileSize: maxFileSize, files: maxFiles }
  │   一次只能上传一个文件（req.files.length > 1 拒绝）
  ├─ 权限检查（write:assets / manage:system）
  ├─ 路径权限检查（checkAccess）
  └─ 调用 assets.upload(opts)
       │
       ▼
server/models/assets.js upload()
  ├─ 计算 fileHash = assetHelper.generateHash(assetPath)
  ├─ SVG 安全扫描（可选，worker 子进程）
  ├─ fs.readFile(opts.path) → 读取整个文件到内存 Buffer
  ├─ DB：UPSERT assets 表 + assetData 表（bytea 存完整二进制）
  ├─ fs.move/copy 到缓存目录 ./data/cache/${hash}.dat
  └─ skipStorage ? skip : storage.assetEvent('uploaded', asset)
       └─ for each target: target.fn.assetUploaded(asset)
            ├─ Git: fs.writeFile + git add + git commit
            ├─ S3: s3.putObject({ Body: asset.data })
            ├─ Azure: blockBlobClient.upload(asset.data, asset.data.length)
            └─ SFTP: sftp.put(fileBuffer, remotePath)
```

**代码位置：**
- `controllers/upload.js:1-99` — HTTP 上传入口
- `models/assets.js:81-167` — 上传处理核心逻辑

### 19.2 文件大小限制

**配置项**（管理后台 → Settings → Uploads）：

| 配置 | 默认值 | 作用位置 |
|------|--------|---------|
| `maxFileSize` | 未明确（由 multer 限制） | `multer({ limits: { fileSize } })` — 超过时 MulterError: File too large |
| `maxFiles` | 1 | `multer({ limits: { files } })` — 一次请求最多上传文件数 |
| `allowedExtensions` | 可配置列表 | 前端校验，不在列表中的文件不能选择 |
| `scanSVG` | true | SVG 文件上传后自动触发 sanitize-svg job 扫描恶意内容 |
| `forceDownload` | 可配置 | 非图片类扩展名强制下载而非浏览器打开 |

**硬限制**：`upload.js:32-36` 硬编码 `req.files.length > 1` 拒绝多文件上传，即使配置 maxFiles > 1 也无效。

### 19.3 流式处理 vs 全量加载

**当前实现：全量加载到内存**

```javascript
// models/assets.js:120
const fileBuffer = await fs.readFile(opts.path)  // ← 整个文件读入内存 Buffer

// S3 上传
await this.s3.putObject({ Key: asset.path, Body: asset.data }).promise()
// asset.data 是完整 Buffer → 内存占用 = 文件大小

// Azure 上传
await blockBlobClient.upload(asset.data, asset.data.length, { tier: this.config.storageTier })
// 同上，完整 Buffer 上传

// Git 本地写入
await fs.outputFile(filePath, asset.data)
// 完整 Buffer 写磁盘
```

**问题**：上传 500MB 视频文件时，至少需要 **500MB 内存**用于存放 Buffer，加上 assetData 表写入和多个存储目标的推送，峰值内存可能达到 **文件大小 × (1 + 存储目标数)**。

### 19.4 各存储后端的流式能力

虽然资产上传使用全量 Buffer，但不同存储 SDK 本身支持流式上传，只是 Wiki.js 没有利用：

| 后端 | SDK 流式上传能力 | Wiki.js 当前用法 |
|------|-----------------|-----------------|
| **Git** | 不适用（本地文件系统） | `fs.outputFile` 可接受 stream，但使用完整 Buffer |
| **Local Disk** | `fs.createWriteStream()` | 同上，使用完整 Buffer |
| **S3** | `upload({ Body: ReadableStream })` 自动分片 | `putObject({ Body: Buffer })` 全量 |
| **Azure Blob** | `uploadStream()` / `uploadFile()` 断点续传 | `upload(Buffer, length)` 全量 |
| **SFTP** | `sftp.createWriteStream()` + pipe | `sftp.put(fileBuffer, ...)` 全量 |

**S3 的 `upload()` 方法（未使用）**：
```javascript
// AWS SDK 支持的流式分片上传（Wiki.js 未用）
await s3.upload({
  Key: asset.path,
  Body: fs.createReadStream(localPath)  // ← 流式，内存占用 O(chunk_size)
}, {
  partSize: 5 * 1024 * 1024,  // 5MB 分片
  queueSize: 4                // 并发上传分片数
}).promise()
```

这会自动处理分片、并发、失败重试，内存占用仅 ~20MB（4×5MB），无论文件多大。

### 19.5 资产下载的流式优化

**资产下载链路**使用了更优化的流式处理：

```javascript
// models/assets.js:196-233
getAsset(assetPath, res)
  ├─ 尝试本地缓存：res.sendFile(cachePath)
  │  → Express 自动使用流式传输，不占用服务器内存
  ├─ 尝试存储模块本地路径：res.sendFile(location.path)
  │  → 同上，流式
  └─ 降级从 DB 读取：res.send(assetData.data)
     → 一次性发送完整 Buffer（但 assetData 存于 DB，已加载到内存）
     → 同时写入本地缓存：fs.outputFile(cachePath, assetData.data)
```

**关键优化**：
- `res.sendFile()` 使用 `send` 库，内部通过 `fs.createReadStream()` + `pipe(res)` 实现零拷贝传输
- HTTP 响应自动设置 `Content-Length`、`Accept-Ranges`，支持断点续传
- 本地缓存命中时，服务器内存占用 ≈ 0

### 19.6 下载的三级回退策略

```
getAsset(assetPath)
  │
  ├─ Level 1: 本地文件缓存（./data/cache/${hash}.dat）
  │   → 最快，零内存占用
  │   → 由 purgeUploads 定时任务每 15 分钟清理
  │
  ├─ Level 2: 存储模块本地路径（Git/Disk 本地副本）
  │   → 次快，零内存占用
  │   → S3/Azure/SFTP 返回空字符串，跳过此级
  │
  └─ Level 3: DB assetData 表（bytea 字段）
      → 最慢，占用内存 = 文件大小
      → 成功后写入 Level 1 缓存，下次访问提速
```

**代码位置：** `models/assets.js:169-194`

### 19.7 分片上传实现状态

| 功能 | 实现状态 | 说明 |
|------|---------|------|
| 客户端分片上传（tus 协议 / resumable.js） | ❌ 未实现 | 整个文件一次性上传，大文件网络中断需重传 |
| 服务端接收分片 | ❌ 未实现 | multer 接收整个文件到磁盘 |
| 云存储 SDK 分片上传（S3 multipart / Azure block blob） | ❌ 未利用 | SDK 支持但 Wiki.js 使用全量 Buffer 上传 |
| 断点续传 | ❌ 未实现 | 上传中断需从头开始 |
| 并发上传多个文件 | ❌ 前端限制 | 一次只能上传一个文件 |

### 19.8 大文件上传性能瓶颈

| 阶段 | 性能瓶颈 | 内存占用 | 可优化方向 |
|------|---------|---------|-----------|
| 1. HTTP 接收 | multer 写临时文件 | ≈ 0（流式写磁盘） | 已优化 |
| 2. 读入内存 Buffer | `fs.readFile(opts.path)` | = 文件大小 | 改为流式处理，传递 Stream |
| 3. 写入 DB | `knex('assetData').insert({ data: fileBuffer })` | 2× 文件大小（PG 协议 + 驱动 Buffer） | PG large object API / 禁用 bytea hex 转义 |
| 4. 写入缓存 | `fs.move/copy` → `./data/cache/${hash}.dat` | ≈ 0（fs.move 是 rename 系统调用） | 已优化 |
| 5. 推送至各存储 | `for...of await target.fn.assetUploaded()` | 文件大小 × (1 + 存储目标数) | 并行 Promise.all + 传递 Stream |

---

## 20. 存储加密 At-Rest 链路

### 20.1 加密现状总览

Wiki.js 的 at-rest 加密（静态数据加密）**严重依赖底层基础设施**，应用层本身实现的加密非常有限。

| 层级 | 加密实现 | 密钥管理 |
|------|---------|---------|
| **DB 连接加密**（传输中） | ✅ 可选 | 由 PostgreSQL/MariaDB 客户端 SSL 配置控制 |
| **DB 数据加密**（at-rest） | ❌ 应用层未实现 | 依赖数据库透明数据加密（TDE）或磁盘加密 |
| **资产数据加密**（at-rest） | ❌ 应用层未实现 | assetData bytea 明文存储 |
| **存储模块数据加密**（at-rest） | ❌ 应用层未实现 | 依赖云存储 SSE（服务端加密）或磁盘加密 |
| **敏感配置加密**（存储凭证） | ❌ DB 明文存储 | 仅前端展示打码 |
| **会话 Cookie 加密** | ✅ AES-256-CBC | `sessionSecret` 作为 passphrase |
| **内部证书加密** | ✅ AES-256-CBC | `sessionSecret` 作为 passphrase |

### 20.2 传输加密（In-Transit）

#### PostgreSQL 连接加密

**代码位置：** `core/db.js:60-136`

```javascript
if (WIKI.config.db.ssl) {
  if (WIKI.config.db.type === 'postgres') {
    if (WIKI.config.db.ssl === true) {
      dbConfig.ssl = { rejectUnauthorized: true }
    } else {
      // CA 证书配置，支持从环境变量读取 64 字符分段的 CA
      const chunks = []
      for (let i = 0; i < process.env.DB_SSL_CA.length; i += 64) {
        chunks.push(process.env.DB_SSL_CA.substring(i, i + 64))
      }
      dbConfig.ssl = {
        rejectUnauthorized: true,
        ca: '-----BEGIN CERTIFICATE-----\n' + chunks.join('\n') + '\n-----END CERTIFICATE-----\n'
      }
    }
    _.set(dbConfig, 'options.encrypt', true)
  }
  // ... MSSQL 支持 encrypt 选项
}
```

**支持的数据库 SSL 模式**：
- PostgreSQL：`ssl: true`（验证服务端证书）或自定义 CA 证书
- MySQL/MariaDB：不支持（代码中仅处理 postgres 和 mssql）
- MSSQL：`encrypt: true`（Azure 要求）
- SQLite：不适用（本地文件）

#### HTTPS / SSL

**代码位置：** `core/servers.js:1-100`、`core/letsencrypt.js`

- 支持 Let's Encrypt 自动申请和续期证书
- 支持自定义 SSL 证书（key + cert + chain）
- 支持 HTTP/2（spdy 模块）
- 支持 HSTS（Strict-Transport-Security）头

### 20.3 应用层加密使用

应用层仅在两个地方使用了加密，都与存储无关：

#### 1. 安装时生成的内部证书

**代码位置：** `setup.js:155-167`

```javascript
const certs = crypto.generateKeyPairSync('rsa', {
  modulusLength: 2048,
  publicKeyEncoding: { type: 'pkcs1', format: 'pem' },
  privateKeyEncoding: {
    type: 'pkcs1',
    format: 'pem',
    cipher: 'aes-256-cbc',           // ← AES-256-CBC 加密
    passphrase: WIKI.config.sessionSecret  // ← 使用 sessionSecret 作为密码
  }
})
```

**用途**：SAML 认证的签名/解密、JWT 签名验证。

#### 2. 认证策略重生成证书

**代码位置：** `core/auth.js:414-426`

与安装时相同的 AES-256-CBC 加密，用途相同。

> **重要**：这两处加密保护的是**内存中的私钥导出**，不是存储在 DB 中的数据。私钥加密后存储在 `WIKI.config.certs.private`，但这个配置对象整体以明文 JSON 存储在 DB 的 `config` 表中。

### 20.4 存储后端的 SSE（服务端加密）

虽然应用层不加密数据，但可以通过云存储的**服务端加密（Server-Side Encryption, SSE）**实现 at-rest 加密，这需要在云存储控制台配置，Wiki.js 不需要改动：

| 后端 | SSE 选项 | Wiki.js 是否需要配置 |
|------|---------|-------------------|
| **Amazon S3** | SSE-S3（AES-256，由 S3 管理密钥）<br>SSE-KMS（AWS KMS 管理密钥）<br>SSE-C（客户提供密钥） | SSE-S3/SSE-KMS：在 Bucket 策略中配置默认加密，Wiki.js 无需改动<br>SSE-C：Wiki.js 当前不支持（每次请求需传密钥） |
| **Azure Blob** | Azure Storage Service Encryption（AES-256，默认启用）<br>客户管理密钥（CMK） | 所有新存储账户默认启用，无需 Wiki.js 配置 |
| **Git** | 不支持（明文文件） | 依赖仓库所在磁盘加密（LUKS / BitLocker）或使用 git-crypt / git-remote-gcrypt 透明加密 |
| **Local Disk** | 不支持（明文文件） | 依赖操作系统磁盘加密（LUKS / BitLocker / FileVault） |
| **SFTP** | 不支持（明文文件） | 依赖远程服务器磁盘加密 |
| **PostgreSQL** | 不支持应用层字段加密 | 依赖 PG TDE（透明数据加密，仅商业版支持）或 pgcrypto 扩展（需应用层代码改动） |

### 20.5 S3 SDK 的加密能力（未启用）

AWS SDK 支持客户端加密，但 Wiki.js 的 S3 上传代码完全没有启用：

```javascript
// 当前用法（明文）
await this.s3.putObject({
  Key: asset.path,
  Body: asset.data          // ← 明文 Buffer
}).promise()

// 可启用但未启用的 SSE（服务端加密）
await this.s3.putObject({
  Key: asset.path,
  Body: asset.data,
  ServerSideEncryption: 'AES256',  // ← SSE-S3
  // 或 SSEKMSKeyId: 'arn:aws:kms:...'  // SSE-KMS
}).promise()
```

**Azure Blob** 类似，SDK 支持 `customerProvidedKey` 选项（客户端提供加密密钥），但 Wiki.js 未使用。

### 20.6 敏感数据存储安全

| 数据类型 | 存储位置 | 加密状态 | 风险 |
|---------|---------|---------|------|
| Git SSH 私钥 / 密码 | storage.config JSONB | ❌ 明文 | DB 泄露即全泄露 |
| S3 Secret Access Key | storage.config JSONB | ❌ 明文 | DB 泄露即全泄露 |
| Azure Account Key | storage.config JSONB | ❌ 明文 | DB 泄露即全泄露 |
| SFTP 私钥 / 密码 | storage.config JSONB | ❌ 明文 | DB 泄露即全泄露 |
| 资产二进制数据 | assetData.bytea | ❌ 明文 | DB 备份泄露即所有文件泄露 |
| 页面内容 | pages.content | ❌ 明文 | DB 备份泄露即所有内容泄露 |
| 用户密码哈希 | users.password | ✅ bcrypt | 相对安全 |
| 会话密钥 | config.sessionSecret | ❌ 明文（config 表） | 泄露可伪造所有会话 |

**前端打码保护（非加密）**：
```javascript
// resolvers/storage.js:31
value: (configData.sensitive && value.length > 0) ? '********' : value
```
仅在 GraphQL 响应中对敏感字段打码，**DB 中仍然是明文**。

### 20.7 应用层加密的可行改造路径

如果需要在应用层实现 at-rest 加密，可按以下方式扩展（当前均未实现）：

```
建议的加密架构：

1. 新增 KMS 配置
   - 支持 AWS KMS / Azure Key Vault / HashiCorp Vault
   - 主密钥（CMK）存于 KMS，数据加密密钥（DEK）由 KMS 生成并加密保存
   - 每个资产/页面使用独立 DEK

2. 资产上传加密链路
   assetUploaded(asset)
     ├─ 生成随机 DEK
     ├─ 使用 DEK + AES-256-GCM 加密 asset.data
     ├─ 使用 CMK 加密 DEK，存储在 assets.metadata
     └─ 上传加密后的密文到存储模块

3. 资产下载解密链路
   getAsset(assetPath)
     ├─ 从 DB 读取加密的 DEK + 密文
     ├─ 调用 KMS 解密 DEK
     ├─ 使用 DEK 解密内容
     └─ 返回明文给用户

4. 数据库加密
   - 对 pages.content 和 assetData.data 字段使用 pgcrypto 扩展
   - 或切换到支持 TDE 的 PG 商业版
```

### 20.8 加密总结

**当前实现的加密：**
- ✅ DB 连接 SSL/TLS（可配置）
- ✅ HTTPS / HTTP/2（可配置）
- ✅ 内部私钥 AES-256-CBC 加密（使用 sessionSecret）
- ✅ 用户密码 bcrypt 哈希

**未实现但基础设施可补的加密：**
- ⚠️ DB 静态数据加密：依赖 PG TDE 或磁盘加密
- ⚠️ 云存储静态数据加密：依赖 S3 SSE / Azure Storage Encryption
- ⚠️ 本地文件加密：依赖操作系统磁盘加密

**完全未实现的加密：**
- ❌ 存储凭证加密（DB 明文）
- ❌ 资产数据应用层加密
- ❌ 页面内容应用层加密
- ❌ 客户端分片上传加密

---

## 21. 关键设计模式总结

### 21.1 防循环写入

所有从外部存储 → DB 的写入操作均传入 `skipStorage: true`：
- `commonDisk.processPage()` → `updatePage({ skipStorage: true })` / `createPage({ skipStorage: true })`
- `git.processFiles()` → `movePage({ skipStorage: true })` / `deletePage({ skipStorage: true })`

这确保 Pull 路径不会触发 Push 路径的 `Storage.pageEvent()`，避免无限循环。

### 21.2 事件驱动 Push

所有 DB → 存储的写入通过事件模型：
```
Page Model 操作 → Storage.pageEvent({ event, page }) → 遍历所有 target → target.fn[event](page)
```
每个存储模块只需实现 `created / updated / deleted / renamed` 四个接口即可自动接收 DB 变更。

### 21.3 增量 Diff 同步

Git 模块的 `sync()` 不是全量比对，而是基于 commit hash 的增量 diff：
1. 记录 sync 前的 HEAD hash
2. pull rebase 后获取新的 HEAD hash
3. `diffSummary(oldHash, newHash)` 只处理两次 sync 之间的变更
4. 无变更则跳过处理

### 21.4 编辑器冲突的乐观锁模式

```
用户A打开编辑 → checkoutDate = page.updatedAt
用户B保存修改 → page.updatedAt 更新
用户A保存时 → checkConflicts(checkoutDate) → updatedAt > checkoutDate → 冲突!
```

这是典型的**乐观并发控制（OCC）**模式，用时间戳代替版本号作为一致性标记。

---

## 22. 涉及的关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `server/core/kernel.js` | 启动时序，初始化存储模块 |
| `server/core/scheduler.js` | 定时任务调度器 |
| `server/core/db.js` | PG LISTEN/NOTIFY 多实例缓存失效 + DB SSL 配置 |
| `server/core/cache.js` | 内存缓存（NodeCache） |
| `server/core/config.js` | 配置加载，WIKI.data = appdata 含jobs定义 |
| `server/core/telemetry.js` | 匿名遥测（仅系统信息，不追踪存储状态） |
| `server/core/worker.js` | 子进程 worker 执行器 |
| `server/core/auth.js` | 认证策略，证书 AES-256-CBC 加密 |
| `server/core/servers.js` | HTTPS/HTTP2 服务 + Let's Encrypt |
| `server/core/letsencrypt.js` | Let's Encrypt 证书自动申请续期 |
| `server/core/system.js` | Content Export 全量 JSON 导出 |
| `server/jobs/sync-storage.js` | sync-storage job 入口 |
| `server/jobs/rebuild-tree.js` | 导航树重建（chunk 分批插入参考） |
| `server/jobs/sanitize-svg.js` | SVG 安全扫描 worker |
| `server/models/storage.js` | Storage 模型：target 管理、事件分发、状态追踪 |
| `server/models/pages.js` | Page 模型：CRUD、parseMetadata、migrateToLocale |
| `server/models/pageHistory.js` | 页面版本历史快照、版本链查询 |
| `server/models/assets.js` | 资产上传/下载流程、缓存/存储写入 |
| `server/helpers/page.js` | 路径解析、frontmatter 注入/提取、hash 生成 |
| `server/helpers/asset.js` | 资产路径解析、hash 生成 |
| `server/controllers/upload.js` | HTTP 上传入口（multer） |
| `server/modules/storage/git/storage.js` | Git 存储模块（唯一完整双向同步实现） |
| `server/modules/storage/git/definition.yml` | Git 模块配置定义（含 actions 列表） |
| `server/modules/storage/disk/storage.js` | Disk 存储模块（push + 备份） |
| `server/modules/storage/disk/definition.yml` | Disk 模块配置定义 |
| `server/modules/storage/disk/common.js` | Disk/Git 共享的 processPage / processAsset 逻辑 |
| `server/modules/storage/sftp/storage.js` | SFTP 单向 push 实现 |
| `server/modules/storage/sftp/definition.yml` | SFTP 模块配置定义 |
| `server/modules/storage/azure/storage.js` | Azure Blob 单向 push 实现 |
| `server/modules/storage/azure/definition.yml` | Azure 模块配置定义 |
| `server/modules/storage/s3/common.js` | S3 兼容存储 单向 push 实现 |
| `server/modules/storage/s3/definition.yml` | S3 模块配置定义 |
| `server/modules/search/elasticsearch/engine.js` | Elasticsearch 批量优化模式参考 |
| `server/graph/resolvers/page.js` | GraphQL: checkConflicts / conflictLatest / restoreVersion |
| `server/graph/schemas/page.graphql` | GraphQL Schema: PageConflictLatest 类型 |
| `server/graph/resolvers/storage.js` | GraphQL: 存储 targets 查询 + status 状态 + 更新 + 执行 action |
| `server/graph/resolvers/system.js` | GraphQL: 系统配置、telemetry 开关 |
| `server/app/data.yml` | 系统内置任务配置（jobs 定义） |
| `server/setup.js` | 安装向导，内部证书 AES-256-CBC 加密生成 |
| `client/components/editor.vue` | 编辑器主组件：冲突检测轮询 + 保存流程 |
| `client/components/editor/editor-modal-conflict.vue` | Markdown/Code 编辑器的冲突解决 UI |
| `client/components/editor/ckeditor/conflict.vue` | CKEditor 的冲突解决 UI |
| `client/components/admin/admin-storage.vue` | 管理后台存储配置 + 状态监控面板 |
| `client/components/admin/admin-utilities-importv1.vue` | Wiki.js 1.x → 2.x 迁移工具 UI |
| `client/libs/codemirror-merge/diff-match-patch.js` | 文本差异计算库 |
