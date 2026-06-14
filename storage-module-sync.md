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

## 12. 关键设计模式总结

### 12.1 防循环写入

所有从外部存储 → DB 的写入操作均传入 `skipStorage: true`：
- `commonDisk.processPage()` → `updatePage({ skipStorage: true })` / `createPage({ skipStorage: true })`
- `git.processFiles()` → `movePage({ skipStorage: true })` / `deletePage({ skipStorage: true })`

这确保 Pull 路径不会触发 Push 路径的 `Storage.pageEvent()`，避免无限循环。

### 12.2 事件驱动 Push

所有 DB → 存储的写入通过事件模型：
```
Page Model 操作 → Storage.pageEvent({ event, page }) → 遍历所有 target → target.fn[event](page)
```
每个存储模块只需实现 `created / updated / deleted / renamed` 四个接口即可自动接收 DB 变更。

### 12.3 增量 Diff 同步

Git 模块的 `sync()` 不是全量比对，而是基于 commit hash 的增量 diff：
1. 记录 sync 前的 HEAD hash
2. pull rebase 后获取新的 HEAD hash
3. `diffSummary(oldHash, newHash)` 只处理两次 sync 之间的变更
4. 无变更则跳过处理

### 12.4 编辑器冲突的乐观锁模式

```
用户A打开编辑 → checkoutDate = page.updatedAt
用户B保存修改 → page.updatedAt 更新
用户A保存时 → checkConflicts(checkoutDate) → updatedAt > checkoutDate → 冲突!
```

这是典型的**乐观并发控制（OCC）**模式，用时间戳代替版本号作为一致性标记。

---

## 13. 涉及的关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `server/core/kernel.js` | 启动时序，初始化存储模块 |
| `server/core/scheduler.js` | 定时任务调度器 |
| `server/core/db.js` | PG LISTEN/NOTIFY 多实例缓存失效 |
| `server/jobs/sync-storage.js` | sync-storage job 入口 |
| `server/models/storage.js` | Storage 模型：target 管理、事件分发 |
| `server/models/pages.js` | Page 模型：CRUD、parseMetadata |
| `server/models/pageHistory.js` | 页面版本历史快照、版本链查询 |
| `server/models/assets.js` | 资产上传流程、缓存/存储写入 |
| `server/helpers/page.js` | 路径解析、frontmatter 注入/提取、hash 生成 |
| `server/modules/storage/git/storage.js` | Git 存储模块（唯一完整双向同步实现） |
| `server/modules/storage/git/definition.yml` | Git 模块配置定义 |
| `server/modules/storage/disk/storage.js` | Disk 存储模块（push + 备份） |
| `server/modules/storage/disk/common.js` | Disk/Git 共享的 processPage / processAsset 逻辑 |
| `server/modules/storage/sftp/storage.js` | SFTP 单向 push 实现 |
| `server/modules/storage/azure/storage.js` | Azure Blob 单向 push 实现 |
| `server/modules/storage/s3/common.js` | S3 兼容存储 单向 push 实现 |
| `server/graph/resolvers/page.js` | GraphQL: checkConflicts / conflictLatest |
| `server/graph/schemas/page.graphql` | GraphQL Schema: PageConflictLatest 类型 |
| `server/graph/resolvers/storage.js` | GraphQL: 存储管理接口 |
| `client/components/editor.vue` | 编辑器主组件：冲突检测轮询 + 保存流程 |
| `client/components/editor/editor-modal-conflict.vue` | Markdown/Code 编辑器的冲突解决 UI |
| `client/components/editor/ckeditor/conflict.vue` | CKEditor 的冲突解决 UI |
| `client/libs/codemirror-merge/diff-match-patch.js` | 文本差异计算库 |
