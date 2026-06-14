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

## 9. 关键设计模式总结

### 9.1 防循环写入

所有从外部存储 → DB 的写入操作均传入 `skipStorage: true`：
- `commonDisk.processPage()` → `updatePage({ skipStorage: true })` / `createPage({ skipStorage: true })`
- `git.processFiles()` → `movePage({ skipStorage: true })` / `deletePage({ skipStorage: true })`

这确保 Pull 路径不会触发 Push 路径的 `Storage.pageEvent()`，避免无限循环。

### 9.2 事件驱动 Push

所有 DB → 存储的写入通过事件模型：
```
Page Model 操作 → Storage.pageEvent({ event, page }) → 遍历所有 target → target.fn[event](page)
```
每个存储模块只需实现 `created / updated / deleted / renamed` 四个接口即可自动接收 DB 变更。

### 9.3 增量 Diff 同步

Git 模块的 `sync()` 不是全量比对，而是基于 commit hash 的增量 diff：
1. 记录 sync 前的 HEAD hash
2. pull rebase 后获取新的 HEAD hash
3. `diffSummary(oldHash, newHash)` 只处理两次 sync 之间的变更
4. 无变更则跳过处理

### 9.4 编辑器冲突的乐观锁模式

```
用户A打开编辑 → checkoutDate = page.updatedAt
用户B保存修改 → page.updatedAt 更新
用户A保存时 → checkConflicts(checkoutDate) → updatedAt > checkoutDate → 冲突!
```

这是典型的**乐观并发控制（OCC）**模式，用时间戳代替版本号作为一致性标记。

---

## 10. 涉及的关键文件索引

| 文件路径 | 职责 |
|---------|------|
| `server/core/kernel.js` | 启动时序，初始化存储模块 |
| `server/core/scheduler.js` | 定时任务调度器 |
| `server/jobs/sync-storage.js` | sync-storage job 入口 |
| `server/models/storage.js` | Storage 模型：target 管理、事件分发 |
| `server/models/pages.js` | Page 模型：CRUD、parseMetadata |
| `server/helpers/page.js` | 路径解析、frontmatter 注入/提取、hash 生成 |
| `server/modules/storage/git/storage.js` | Git 存储模块（唯一完整双向同步实现） |
| `server/modules/storage/git/definition.yml` | Git 模块配置定义 |
| `server/modules/storage/disk/storage.js` | Disk 存储模块（push + 备份） |
| `server/modules/storage/disk/common.js` | Disk/Git 共享的 processPage / processAsset 逻辑 |
| `server/graph/resolvers/page.js` | GraphQL: checkConflicts / conflictLatest |
| `server/graph/schemas/page.graphql` | GraphQL Schema: PageConflictLatest 类型 |
| `server/graph/resolvers/storage.js` | GraphQL: 存储管理接口 |
| `client/components/editor.vue` | 编辑器主组件：冲突检测轮询 + 保存流程 |
| `client/components/editor/editor-modal-conflict.vue` | Markdown/Code 编辑器的冲突解决 UI |
| `client/components/editor/ckeditor/conflict.vue` | CKEditor 的冲突解决 UI |
| `client/libs/codemirror-merge/diff-match-patch.js` | 文本差异计算库 |
