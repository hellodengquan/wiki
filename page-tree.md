# Wiki.js Page 树构建与路径解析

## 概述

Wiki.js 使用 **预计算的物化路径树** 来管理页面的层级结构。核心设计是维护一张独立的 `pageTree` 表，存储所有节点（包括文件夹和页面）的完整树结构信息，以空间换时间，实现高效的层级查询和路径解析。

---

## 一、数据库设计

### 1.1 pages 表（主表）

存储实际的页面内容和元数据：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | integer | 主键，自增 |
| path | string | 页面路径（如 `docs/guide/install`） |
| hash | string | 由 locale + path + privateNS 生成的 SHA1 哈希，用于缓存键 |
| title | string | 页面标题 |
| localeCode | string | 语言代码（如 `en`, `zh-CN`） |
| isPrivate | boolean | 是否私有页面 |
| isPublished | boolean | 是否已发布 |
| contentType | string | 内容类型（markdown, html, asciidoc 等） |
| content | text | 原始内容 |
| render | text | 渲染后的 HTML |

> 源码位置：`server/models/pages.js:33-55`

### 1.2 pageTree 表（树表）

预计算的树结构表，每个节点（文件夹或页面）占一行：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | integer | 树节点 ID（重建时重新生成，非自增） |
| path | string | 节点完整路径 |
| depth | integer | 节点深度（根目录为 1） |
| title | string | 节点标题（文件夹用路径段名，页面用页面标题） |
| isFolder | boolean | 是否为文件夹 |
| isPrivate | boolean | 是否私有（仅页面有意义） |
| privateNS | string | 私有命名空间 |
| parent | integer | 父节点 ID，根节点为 null |
| pageId | integer | 关联的页面 ID（文件夹为 null） |
| localeCode | string | 语言代码 |
| ancestors | json | 祖先节点 ID 数组（JSON 字符串） |

> 源码位置：`server/db/migrations/2.0.0.js:174-183`（初始表）
> `server/db/migrations/2.3.23.js:3-5`（新增 ancestors 字段）

---

## 二、URL 到节点的映射

### 2.1 路径解析流程

当用户访问 `/en/docs/guide/install` 时，解析过程如下：

```
请求 URL → pageHelper.parsePath() → 提取 locale 和 path → 查询 pages/pageTree 表
```

### 2.2 parsePath 函数详解

**文件**：`server/helpers/page.js:24-67`

核心处理逻辑：

1. **清理路径**：去除首尾斜杠、不安全字符、连续斜杠、`..` 遍历
2. **提取 locale**：检查首段是否匹配 locale 格式（如 `en`, `zh-CN`）
3. **去除扩展名**：可选参数 `stripExt`，用于识别页面 vs 资源
4. **返回对象**：包含 locale、path、private、explicitLocale 等

```javascript
parsePath(rawPath, opts = {}) {
  // 1. 清理路径
  rawPath = _.trim(qs.unescape(rawPath))
  rawPath = rawPath.replace(unsafeCharsRegex, '')
  rawPath = rawPath.replace(/\\/g, '').replace(/\/\//g, '').replace(/\.\.+/ig, '')

  // 2. 拆分路径段
  let pathParts = _.filter(_.split(rawPath, '/'), p => { ... })

  // 3. 识别 locale
  if (localeSegmentRegex.test(pathParts[0])) {
    pathObj.locale = pathParts[0]
    pathObj.explicitLocale = true
    pathParts.shift()
  }

  // 4. 去除扩展名（可选）
  if (opts.stripExt && pathParts.length > 0) { ... }

  pathObj.path = _.join(pathParts, '/')
  return pathObj
}
```

### 2.3 控制器路由

**文件**：`server/controllers/common.js`

主要路由入口：

| 路由前缀 | 用途 | 关键函数 |
|----------|------|----------|
| `/a/*` | 管理后台 | - |
| `/e/*` | 编辑器（创建/编辑） | `pageHelper.parsePath` + `getPageFromDb` |
| `/h/*` | 页面历史 | `pageHelper.parsePath` + `getPageFromDb` |
| `/i/:id` | 按 ID 重定向 | `findById` + 重定向到路径 |
| `/s/*` | 查看源码 | `pageHelper.parsePath` + `getPageFromDb` |
| `/*` | 页面浏览/资源 | `pageHelper.parsePath` + `getPage` |

页面浏览的核心流程（`common.js:417-582`）：

```javascript
router.get('/*', async (req, res, next) => {
  // 1. 判断是页面还是资源（通过扩展名）
  const stripExt = _.some(WIKI.config.pageExtensions, ...)
  const pageArgs = pageHelper.parsePath(req.path, { stripExt })

  if (isPage) {
    // 2. locale 命名空间重定向（如需要）
    // 3. 从缓存/数据库获取页面
    const page = await WIKI.models.pages.getPage({
      path: pageArgs.path,
      locale: pageArgs.locale,
      ...
    })
    // 4. 权限检查
    // 5. 渲染页面
  } else {
    // 资源处理
  }
})
```

---

## 三、树的构建（重建）

### 3.1 触发时机

树的重建在以下情况触发：

- 创建页面后：`pages.js:336`
- 更新页面且路径变化时：`pages.js:472-477`
- 移动页面后：`pages.js:739`
- 删除页面后：`pages.js:818`
- 手动触发：通过 GraphQL mutation `rebuildTree`

### 3.2 重建算法

**文件**：`server/jobs/rebuild-tree.js`

算法核心是**遍历所有页面，逐层展开路径段**：

```javascript
const pages = await WIKI.models.pages.query()
  .select('id', 'path', 'localeCode', 'title', 'isPrivate', 'privateNS')
  .orderBy(['localeCode', 'path'])

for (const page of pages) {
  const pagePaths = page.path.split('/')  // 拆分路径
  let currentPath = ''
  let depth = 0
  let parentId = null
  let ancestors = []

  for (const part of pagePaths) {
    depth++
    const isFolder = (depth < pagePaths.length)  // 判断是否是文件夹
    currentPath = currentPath ? `${currentPath}/${part}` : part

    // 检查该路径节点是否已存在
    const found = _.find(tree, { localeCode: page.localeCode, path: currentPath })

    if (!found) {
      // 不存在则创建新节点
      pik++  // 全局递增 ID
      tree.push({
        id: pik,
        localeCode: page.localeCode,
        path: currentPath,
        depth: depth,
        title: isFolder ? part : page.title,
        isFolder: isFolder,
        parent: parentId,
        pageId: isFolder ? null : page.id,
        ancestors: JSON.stringify(ancestors)
      })
      parentId = pik
    } else if (isFolder && !found.isFolder) {
      // 已存在但之前标记为页面，需要升级为文件夹
      found.isFolder = true
      parentId = found.id
    } else {
      parentId = found.id
    }
    ancestors.push(parentId)
  }
}
```

**关键特性**：

1. **按 locale 隔离**：不同语言的树完全独立
2. **虚拟文件夹**：即使没有文件夹页面，只要有子页面就会自动生成文件夹节点
3. **节点升级**：如果一个路径先作为页面存在，后发现有子页面，会升级为文件夹（isFolder=true）
4. **祖先链**：`ancestors` 字段存储所有祖先节点 ID，用于快速查询面包屑

### 3.3 批量插入

为避免 SQL 参数限制，分块插入：

```javascript
// Postgres/MySQL/MSSQL: 每块 100 条
// SQLite: 每块 60 条
for (const chunk of _.chunk(tree, 100)) {
  await WIKI.models.knex.table('pageTree').insert(chunk)
}
```

---

## 四、层级懒加载

### 4.1 后端查询接口

**GraphQL Query**：`pages.tree`

**文件**：`server/graph/resolvers/page.js:249-295`

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| path | String | 页面路径（与 parent 二选一） |
| parent | Int | 父节点 ID（与 path 二选一） |
| mode | PageTreeMode | `FOLDERS` / `PAGES` / `ALL` |
| locale | String | 语言代码 |
| includeAncestors | Boolean | 是否包含祖先节点 |

**查询逻辑**：

```javascript
async tree (obj, args, context, info) {
  // 1. 如果传了 path，先查 pageTree 获取 parent
  if (args.path && !args.parent) {
    curPage = await WIKI.models.knex('pageTree').first('parent', 'ancestors').where({
      path: args.path,
      localeCode: args.locale
    })
    args.parent = curPage.parent || 0
  }

  // 2. 查询子节点
  const results = await WIKI.models.knex('pageTree').where(builder => {
    builder.where('localeCode', args.locale)
    
    // 按 mode 过滤
    switch (args.mode) {
      case 'FOLDERS': builder.andWhere('isFolder', true); break
      case 'PAGES': builder.andWhereNotNull('pageId'); break
    }

    // 按 parent 过滤（根节点 parent 为 null）
    if (!args.parent || args.parent < 1) {
      builder.whereNull('parent')
    } else {
      builder.where('parent', args.parent)
      // 可选：包含祖先节点
      if (args.includeAncestors && curPage) {
        builder.orWhereIn('id', JSON.parse(curPage.ancestors))
      }
    }
  }).orderBy([{ column: 'isFolder', order: 'desc' }, 'title'])

  // 3. 权限过滤
  return results.filter(r => WIKI.auth.checkAccess(...))
}
```

### 4.2 前端懒加载实现

#### 侧边栏导航（nav-sidebar.vue）

**文件**：`client/themes/default/components/nav-sidebar.vue`

**懒加载逻辑**：

```javascript
async fetchBrowseItems (item) {
  // 查询指定 parent 下的所有节点
  const resp = await this.$apollo.query({
    query: gql`
      query ($parent: Int, $locale: String!) {
        pages {
          tree(parent: $parent, mode: ALL, locale: $locale) {
            id, path, title, isFolder, pageId, parent, locale
          }
        }
      }
    `,
    variables: { parent: item.id, locale: this.locale }
  })
  this.currentItems = _.get(resp, 'data.pages.tree', [])
}
```

**初始化加载当前页面路径**：

```javascript
async loadFromCurrentPath() {
  // 使用 includeAncestors 一次性获取所有祖先节点
  const resp = await this.$apollo.query({
    query: gql`
      query ($path: String, $locale: String!) {
        pages {
          tree(path: $path, mode: ALL, locale: $locale, includeAncestors: true) {
            ...
          }
        }
      }
    `,
    variables: { path: this.path, locale: this.locale }
  })
  // 构建面包屑和当前目录内容
}
```

#### 页面选择器（page-selector.vue）

**文件**：`client/components/common/page-selector.vue`

使用 Vuetify 的 `v-treeview` 组件的 `load-children` 特性实现按需加载：

```vue
v-treeview(
  :items='tree'
  :load-children='fetchFolders'
  item-id='path'
  item-text='title'
)
```

```javascript
async fetchFolders (item) {
  const resp = await this.$apollo.query({
    query: gql`...`,  // 同上
    variables: { parent: item.id, mode: 'ALL', locale: this.currentLocale }
  })
  const items = _.get(resp, 'data.pages.tree', [])
  // 分离文件夹和页面
  const itemFolders = _.filter(items, ['isFolder', true]).map(f => ({...f, children: []}))
  const itemPages = _.filter(items, i => i.pageId > 0)
  item.children = itemFolders.length > 0 ? itemFolders : undefined
}
```

---

## 五、移动/改名后路径重写

### 5.1 movePage 函数

**文件**：`server/models/pages.js:665-783`

当页面移动或改名时，执行以下步骤：

```
1. 权限检查（源页面 + 目标路径）
2. 检查目标路径是否已存在
3. 创建历史版本快照
4. 更新页面的 path, localeCode, title, hash
5. 清除旧缓存
6. 重建 pageTree（整棵树重建）
7. 更新搜索索引
8. 更新存储（如 Git 存储）
9. 重新连接页面链接
```

**核心代码**：

```javascript
static async movePage(opts) {
  // 1. 查找源页面
  const page = await WIKI.models.pages.query().findById(opts.id)

  // 2. 验证目标路径合法性
  // 3. 权限检查
  // 4. 检查目标路径冲突

  // 5. 创建历史版本
  await WIKI.models.pageHistory.addVersion({ ...page, action: 'moved' })

  // 6. 计算新的 hash
  const destinationHash = pageHelper.generateHash({
    path: opts.destinationPath,
    locale: opts.destinationLocale,
    privateNS: opts.isPrivate ? 'TODO' : ''
  })

  // 7. 更新页面（如果标题是路径名，自动更新标题）
  const destinationTitle = (page.title === _.last(page.path.split('/'))
    ? _.last(opts.destinationPath.split('/'))
    : page.title)
  
  await WIKI.models.pages.query().patch({
    path: opts.destinationPath,
    localeCode: opts.destinationLocale,
    title: destinationTitle,
    hash: destinationHash
  }).findById(page.id)

  // 8. 清除缓存
  await WIKI.models.pages.deletePageFromCache(page.hash)

  // 9. 重建整棵树
  await WIKI.models.pages.rebuildTree()

  // 10. 更新搜索索引
  await WIKI.data.searchEngine.renamed({ ... })

  // 11. 更新存储
  await WIKI.models.storage.pageEvent({ event: 'renamed', ... })

  // 12. 重新连接链接（更新所有指向旧路径的链接）
  await WIKI.models.pages.reconnectLinks({ mode: 'move', ... })
  await WIKI.models.pages.reconnectLinks({ mode: 'create', ... })
}
```

### 5.2 链接重连（reconnectLinks）

**文件**：`server/models/pages.js:850-915`

移动页面后，需要更新所有其他页面中指向该页面的内部链接：

```javascript
static async reconnectLinks (opts) {
  const pageHref = `/${opts.locale}/${opts.path}`
  let replaceArgs = { from: '', to: '' }

  switch (opts.mode) {
    case 'create':
      // 新页面创建：无效链接变有效链接
      replaceArgs.from = `<a href="${pageHref}" class="is-internal-link is-invalid-page">`
      replaceArgs.to = `<a href="${pageHref}" class="is-internal-link is-valid-page">`
      break
    case 'move':
      // 页面移动：旧路径链接更新为新路径
      const prevPageHref = `/${opts.sourceLocale}/${opts.sourcePath}`
      replaceArgs.from = `<a href="${prevPageHref}" class="is-internal-link is-valid-page">`
      replaceArgs.to = `<a href="${pageHref}" class="is-internal-link is-valid-page">`
      break
    case 'delete':
      // 页面删除：有效链接变无效链接
      replaceArgs.from = `<a href="${pageHref}" class="is-internal-link is-valid-page">`
      replaceArgs.to = `<a href="${pageHref}" class="is-internal-link is-invalid-page">`
      break
  }

  // 批量更新 render 字段中的链接
  await WIKI.models.pages.query()
    .patch({
      render: WIKI.models.knex.raw('REPLACE(??, ?, ?)', ['render', replaceArgs.from, replaceArgs.to])
    })
    .whereIn('pages.id', function () {
      this.select('pageLinks.pageId').from('pageLinks').where({
        'pageLinks.path': opts.path,
        'pageLinks.localeCode': opts.locale
      })
    })

  // 清除受影响页面的缓存
  for (const hash of affectedHashes) {
    await WIKI.models.pages.deletePageFromCache(hash)
  }
}
```

### 5.3 整树重建 vs 增量更新

**注意**：Wiki.js 采用的是**整树重建**策略，而非增量更新。每次页面增删改都会触发完整的树重建：

- **优点**：实现简单，不会出现数据不一致
- **缺点**：页面数量多时可能性能较差

重建是异步执行的（通过 worker job），不会阻塞用户请求。

---

## 六、页面缓存与 hash

### 6.1 Hash 生成

**文件**：`server/helpers/page.js:71-73`

```javascript
generateHash(opts) {
  return crypto.createHash('sha1')
    .update(`${opts.locale}|${opts.path}|${opts.privateNS}`)
    .digest('hex')
}
```

Hash 由三个字段组成：`locale + | + path + | + privateNS`

### 6.2 缓存机制

- 缓存以 hash 为文件名，存储为二进制文件（使用 js-binary 序列化）
- 移动页面后，旧 hash 的缓存会被删除
- 新路径生成新的 hash，缓存失效

---

## 七、关键文件索引

| 文件 | 作用 |
|------|------|
| `server/models/pages.js` | Page 模型，CRUD、movePage、reconnectLinks |
| `server/helpers/page.js` | 路径解析、hash 生成、元数据注入 |
| `server/jobs/rebuild-tree.js` | 树重建任务 |
| `server/graph/resolvers/page.js` | GraphQL resolvers（tree 查询） |
| `server/graph/schemas/page.graphql` | GraphQL schema 定义 |
| `server/controllers/common.js` | HTTP 路由控制器 |
| `client/themes/default/components/nav-sidebar.vue` | 侧边栏导航（树浏览） |
| `client/components/common/page-selector.vue` | 页面选择器对话框 |
| `client/graph/common/common-pages-query-tree.gql` | 树查询 GraphQL 片段 |

---

## 八、总结

Wiki.js 的页面树系统采用 **「物化路径 + 预计算树表」** 的设计模式：

1. **路径即主键**：`path` 字段直接表示层级关系，无需递归查询
2. **预计算树表**：`pageTree` 表提前展开所有节点，查询效率高
3. **整树重建**：简化实现，以异步重建换取一致性
4. **懒加载查询**：前端按需加载子节点，减少初始数据量
5. **链接追踪**：通过 `pageLinks` 表维护反向链接，支持移动时批量更新

这种设计适合中小规模的 Wiki 站点，实现简单且查询性能良好。对于超大规模站点，可能需要考虑增量更新树结构或使用嵌套集（Nested Set）模型。

---

## 九、rebuild-tree 锁与一致性

### 9.1 进程隔离：独立 worker 进程

**文件**：`server/core/scheduler.js:53-79`、`server/core/worker.js`

rebuild-tree 被注册为 `worker: true` 的 job，这意味着它**不在主进程中执行**，而是通过 `childProcess.fork` 启动一个全新的 Node.js 子进程：

```javascript
// scheduler.js:55-79
if (this.worker) {
  const proc = childProcess.fork(`server/core/worker.js`, [
    `--job=${this.name}`,
    `--data=${data}`
  ], { cwd: WIKI.ROOTPATH, stdio: ['inherit', 'inherit', 'pipe', 'ipc'] })
  this.finished = new Promise((resolve, reject) => {
    proc.on('exit', (code, signal) => {
      if (code === 0) { resolve(data) }
      else { reject(new Error(`Error when running job ${this.name}`)) }
    })
  })
}
```

子进程 `worker.js` 会独立初始化 DB 连接：

```javascript
// worker.js
WIKI.models = require('../core/db').init()
await WIKI.configSvc.loadFromDb()
await require(`../jobs/${args.job}`)(args.data)
process.exit(0)
```

### 9.2 无显式锁机制

**关键发现**：rebuild-tree **没有使用任何数据库锁或分布式锁**。它采用的是「truncate + insert」的原子替换策略：

```javascript
// rebuild-tree.js:57-68
await WIKI.models.knex.table('pageTree').truncate()
if (tree.length > 0) {
  for (const chunk of _.chunk(tree, 100)) {
    await WIKI.models.knex.table('pageTree').insert(chunk)
  }
}
```

**一致性风险**：

1. **truncate 和 insert 之间存在时间窗口**：在 `truncate()` 完成后、`insert()` 完成前，所有树查询会返回空结果。但由于 rebuild-tree 在独立 worker 进程中运行，主进程的请求仍会读到旧数据（不同 DB 连接），实际影响有限。

2. **并发重建**：如果两个页面几乎同时被创建/移动/删除，`rebuildTree()` 会被调用两次。但由于 `scheduler.registerJob` 会将两个 job 都推入队列，而它们各自 fork 的子进程**互相不感知**，可能出现以下竞态：

   ```
   Job A: truncate → insert(tree_A)     // tree_A 不包含页面 B
   Job B:              truncate → insert(tree_B)  // tree_B 不包含页面 A
   ```

   后完成的 job 会覆盖前者的结果。但由于每次 rebuild-tree 都从 pages 表全量重建，最终结果是正确的——只是中间状态可能短暂不一致。

3. **无事务包裹**：truncate + 多次 insert 不在同一个事务中。如果中途失败（如 OOM），pageTree 表会被清空但不完整，需要等下一次重建任务来修复。

### 9.3 启动时自动重建

```yaml
# server/app/data.yml:136-141
rebuildTree:
  onInit: true
  offlineSkip: false
  repeat: false
  immediate: true
  worker: true
```

系统启动时会自动触发一次 rebuild-tree，确保 pageTree 表与 pages 表一致，弥补了无事务保护的风险。

---

## 十、GraphQL 树查询的权限继承

### 10.1 两层权限模型

Wiki.js 的权限检查分为两层：

**第一层：GraphQL 指令级权限**（`server/graph/directives/auth.js`）

```javascript
// auth.js:33-48 — @auth 指令包装 resolver
field.resolve = async function (...args) {
  const requiredScopes = field._requiredAuthScopes || objectType._requiredAuthScopes
  if (!requiredScopes) { return resolve.apply(this, args) }
  const context = args[2]
  if (!_.some(context.req.user.permissions, pm => _.includes(requiredScopes, pm))) {
    throw new Error('Forbidden')
  }
  return resolve.apply(this, args)
}
```

tree 查询声明了 `@auth(requires: ["manage:system", "read:pages"])`，只要用户拥有其中任一全局权限，即可调用该接口。

**第二层：页面级 Page Rules**（`server/core/auth.js:221-295`）

```javascript
// auth.js:221 — checkAccess
checkAccess(user, permissions = [], page = false) {
  // 1. manage:system 系统管理员直接通过
  if (_.includes(userPermissions, 'manage:system')) { return true }
  // 2. 全局权限检查
  if (_.intersection(userPermissions, permissions).length < 1) { return false }
  // 3. 如果没有 page 上下文，直接返回 true
  if (!page) { return true }
  // 4. 检查 Page Rules（按组遍历）
  user.groups.forEach(grp => {
    _.get(WIKI.auth.groups, `${grpId}.pageRules`, []).forEach(rule => {
      // 匹配规则：START/END/REGEX/TAG/EXACT
    })
  })
  return (checkState.match && !checkState.deny)
}
```

### 10.2 树查询中的权限过滤

**文件**：`server/graph/resolvers/page.js:285-294`

```javascript
return results.filter(r => {
  return WIKI.auth.checkAccess(context.req.user, ['read:pages'], {
    path: r.path,
    locale: r.localeCode
  })
})
```

**重要特性**：

1. **每个节点独立检查**：tree 查询返回的每个节点都独立进行 `checkAccess` 过滤，而非继承父节点的权限。

2. **文件夹节点无独立权限**：文件夹节点的 `isPrivate` 始终为 `false`，`pageId` 为 `null`。权限检查时传入的是 `{ path, locale }`，Page Rules 的匹配依据是路径前缀。

3. **路径前缀继承效果**：Page Rules 支持 `START` 匹配模式（`server/core/auth.js:255-257`），例如规则 `deny: true, match: START, path: secret` 会拒绝所有以 `secret` 开头的路径。这在效果上实现了「父目录拒绝 → 子页面全部拒绝」的继承语义。

4. **无显式继承逻辑**：代码中没有「如果父文件夹被拒绝则子节点也被拒绝」的逻辑。继承完全依赖 Page Rules 的 `START` 匹配模式。如果管理员配置了 `EXACT` 规则只拒绝父文件夹，子页面仍可见。

### 10.3 特异性（Specificity）算法

当多条 Page Rules 匹配同一个页面时，通过特异性决定最终结果：

```javascript
// auth.js:368-388
_applyPageRuleSpecificity ({ rule, checkState, higherPriority = [] }) {
  if (rule.path.length === checkState.specificity.length) {
    // 同长度：不覆盖更高优先级的匹配类型，不覆盖已有的 DENY
    if (_.includes(higherPriority, checkState.match)) { return checkState }
    if (rule.match === checkState.match && checkState.deny && !rule.deny) { return checkState }
  } else if (rule.path.length < checkState.specificity.length) {
    // 更短路径：不覆盖更长（更具体）的规则
    return checkState
  }
  return { deny: rule.deny, match: rule.match, specificity: rule.path }
}
```

优先级排序：`EXACT > TAG > REGEX > END > START`。路径越长（越具体）优先级越高。DENY 规则一旦匹配，不会被同级别的 ALLOW 规则覆盖。

---

## 十一、treeview 大目录性能

### 11.1 后端查询无分页

**文件**：`server/graph/resolvers/page.js:266-284`

tree 查询**没有分页参数**，每次请求返回指定 parent 下的**全部子节点**：

```javascript
const results = await WIKI.models.knex('pageTree').where(builder => {
  builder.where('localeCode', args.locale)
  // ... mode 过滤
  builder.where('parent', args.parent)
}).orderBy([{ column: 'isFolder', order: 'desc' }, 'title'])
```

**风险**：如果一个目录下有数千个页面/文件夹，一次查询会返回全部数据。

### 11.2 前端内存限制

**侧边栏（nav-sidebar.vue）**：

```javascript
data() {
  return {
    currentItems: [],   // 当前目录的所有子项
    loadedCache: [],    // 已加载过的目录 ID
  }
}
```

侧边栏每次只展示一个目录的内容（`currentItems`），每次点击文件夹时重新查询。虽然每个目录无分页，但懒加载机制确保只加载用户实际展开的层级。

**页面选择器（page-selector.vue）**：

```javascript
async fetchFolders (item) {
  const items = _.get(resp, 'data.pages.tree', [])
  const itemFolders = _.filter(items, ['isFolder', true]).map(f => ({...f, children: []}))
  const itemPages = _.filter(items, i => i.pageId > 0)
  item.children = itemFolders.length > 0 ? itemFolders : undefined
  this.pages = _.unionBy(this.pages, itemPages, 'id')
  this.all = _.unionBy(this.all, items, 'id')
}
```

页面选择器将所有已访问过的页面累积在 `this.pages` 和 `this.all` 中，**不会释放**。如果用户在大目录中来回浏览，内存占用会持续增长。

### 11.3 v-treeview 的渲染性能

Vuetify 的 `v-treeview` 会对每个节点创建 Vue 组件实例。当目录中包含数千个文件夹节点时，每个节点都有展开箭头和 `children: []` 占位，会导致：

- 大量 Vue 组件实例化开销
- DOM 节点过多
- 无虚拟滚动支持

**实际缓解**：`fetchFolders` 中只有文件夹节点才设置 `children: []`（触发懒加载），页面节点不设置 `children`，因此只有文件夹节点是可展开的。

### 11.4 Apollo 缓存策略差异

| 组件 | fetchPolicy | 说明 |
|------|------------|------|
| nav-sidebar | `cache-first` | 优先读缓存，减少重复请求 |
| page-selector | `network-only` | 每次都发请求，确保数据最新 |

nav-sidebar 使用 `cache-first`，但在 locale 切换时会强制重置树（`treeViewCacheId += 1`），通过 `:key` 强制重建组件。

---

## 十二、movePage 并发冲突解决

### 12.1 目标路径冲突检测

**文件**：`server/models/pages.js:709-716`

```javascript
const destPage = await WIKI.models.pages.query().findOne({
  path: opts.destinationPath,
  localeCode: opts.destinationLocale
})
if (destPage) {
  throw new WIKI.Error.PagePathCollision()
}
```

如果目标路径已被占用，抛出 `PagePathCollision`（错误码 6006），拒绝移动。

### 12.2 创建时的重复检测

**文件**：`server/models/pages.js:266-269`

```javascript
const dupCheck = await WIKI.models.pages.query()
  .select('id')
  .where('localeCode', opts.locale)
  .where('path', opts.path).first()
if (dupCheck) {
  throw new WIKI.Error.PageDuplicateCreate()
}
```

### 12.3 竞态条件分析

**检查-执行间隙（TOCTOU）**：冲突检测和实际写入之间没有事务或行锁保护。两个并发请求可能同时通过冲突检测，然后都尝试写入：

```
请求 A: findOne(空) → 通过检查 → patch(path=A)
请求 B: findOne(空) → 通过检查 → patch(path=A) → 数据覆盖
```

**实际影响**：
- `createPage` 使用 `insert`，如果 `(localeCode, path)` 有唯一约束，第二次 insert 会抛出数据库唯一约束错误
- `movePage` 使用 `patch`，后执行的 patch 会覆盖先执行的结果

**没有乐观锁**：`updatePage` 和 `movePage` 都没有使用版本号或 `updatedAt` 做乐观并发控制。`checkConflicts` 查询（`server/graph/resolvers/page.js:354-368`）只在前端编辑器中使用，对比 `updatedAt` 与 `checkoutDate`，但这个检查**不在后端修改流程中强制执行**。

### 12.4 路径合法性校验

**文件**：`server/models/pages.js:242-249`、`680-692`

```javascript
// 创建和移动都执行相同的校验
if (opts.path.includes('.') || opts.path.includes(' ') ||
    opts.path.includes('\\') || opts.path.includes('//')) {
  throw new WIKI.Error.PageIllegalPath()
}
if (opts.path.endsWith('/')) { opts.path = opts.path.slice(0, -1) }
if (opts.path.startsWith('/')) { opts.path = opts.path.slice(1) }
```

这防止了路径中的 `.`（可导致扩展名混淆）、空格、反斜杠和双斜杠。

---

## 十三、reconnectLinks 与外部链接降级

### 13.1 内部链接的渲染时标注

**文件**：`server/modules/rendering/html-core/renderer.js:43-126`

页面渲染时，渲染器会为每个 `<a>` 标签添加 CSS 类：

| 类名 | 含义 | 条件 |
|------|------|------|
| `is-internal-link` | 内部链接 | href 不含 `://`，不含 `.`，非系统路径 |
| `is-external-link` | 外部链接 | href 含 `://` |
| `is-system-link` | 系统路径 | 匹配 `/x/` 前缀（如 `/a/`, `/e/`） |
| `is-asset-link` | 资源链接 | href 含 `.`（文件扩展名） |
| `is-valid-page` | 内部链接-页面存在 | pages 表中找到对应记录 |
| `is-invalid-page` | 内部链接-页面不存在 | pages 表中未找到对应记录 |

### 13.2 pageLinks 反向索引

渲染器在标注链接状态的同时，维护 `pageLinks` 反向索引：

```javascript
// renderer.js:132-196
const pastLinks = await this.page.$relatedQuery('links')

// 添加新链接
const missingLinks = _.differenceWith(internalRefs, pastLinks, ...)
await WIKI.models.pageLinks.query().insert(missingLinks.map(...))

// 删除过期链接
const outdatedLinks = _.differenceWith(pastLinks, internalRefs, ...)
await WIKI.models.pageLinks.query().delete().whereIn('id', _.map(outdatedLinks, 'id'))
```

`pageLinks` 表结构（`server/models/pageLinks.js`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | integer | 主键 |
| pageId | integer | 源页面 ID（外键 → pages.id，CASCADE 删除） |
| path | string | 链接目标路径 |
| localeCode | string | 链接目标语言 |

### 13.3 reconnectLinks 的操作范围

**文件**：`server/models/pages.js:850-915`

reconnectLinks **只处理内部链接**（`is-internal-link` 类），不处理外部链接。具体来说，它通过字符串替换 `render` 字段中的 HTML 来更新链接状态：

```javascript
case 'move':
  replaceArgs.from = `<a href="${prevPageHref}" class="is-internal-link is-valid-page">`
  replaceArgs.to = `<a href="${pageHref}" class="is-internal-link is-valid-page">`
  break
```

### 13.4 外部链接降级场景

**场景一：页面移动后，外部站点的链接失效**

reconnectLinks 只更新 Wiki 内部页面的 render 字段，不会通知外部站点。外部站点指向旧路径的链接会收到 404。

Wiki.js **没有**实现旧路径到新路径的 301 重定向。移动后旧路径直接失效。

**场景二：页面删除后，内部链接变为无效**

```javascript
case 'delete':
  replaceArgs.from = `<a href="${pageHref}" class="is-internal-link is-valid-page">`
  replaceArgs.to = `<a href="${pageHref}" class="is-internal-link is-invalid-page">`
```

删除后，所有引用该页面的内部链接的 CSS 类从 `is-valid-page` 变为 `is-invalid-page`，前端会通过样式（通常是红色或删除线）提示用户链接已失效。链接本身**不会被删除**，仍可点击，只是目标页面不存在。

**场景三：REPLACE 字符串匹配失败**

reconnectLinks 使用精确的 HTML 字符串做 REPLACE，如果渲染管线在未来版本修改了 HTML 结构（如属性顺序变化、class 名修改），REPLACE 将匹配不到任何内容，**静默失败**而不报错。这是一个脆弱的设计。

**场景四：pageLinks 表不同步**

reconnectLinks 通过 `pageLinks` 表查找受影响的页面。如果 `pageLinks` 表数据不完整（如渲染失败未写入），部分引用页面不会被更新，导致链接状态不一致。

---

## 十四、分块批量插入与数据库行锁

### 14.1 分块策略

**文件**：`server/jobs/rebuild-tree.js:57-68`

```javascript
await WIKI.models.knex.table('pageTree').truncate()
if (tree.length > 0) {
  if ((WIKI.config.db.type !== 'sqlite')) {
    for (const chunk of _.chunk(tree, 100)) {
      await WIKI.models.knex.table('pageTree').insert(chunk)
    }
  } else {
    for (const chunk of _.chunk(tree, 60)) {
      await WIKI.models.knex.table('pageTree').insert(chunk)
    }
  }
}
```

**为什么分块**：

- **Postgres**：单查询最多约 35,000 个参数（65535 限制），每条 tree 记录约 10 个字段，100 条 = 1000 参数，安全
- **MSSQL**：单查询最多约 2,000 个参数，100 条 = 1000 参数，安全
- **SQLite**：单查询最多约 999 个参数，60 条 × 10 字段 = 600 参数，安全

### 14.2 无事务包裹的风险

truncate 和后续的多次 insert **不在同一个事务中**。如果 insert 过程中失败（如进程被 kill）：

1. `truncate()` 已完成 → pageTree 表为空
2. 部分 `insert()` 完成 → pageTree 表只有部分数据
3. 树查询会返回不完整的结果

**补救措施**：系统启动时的 `onInit: true` 会自动触发一次完整的 rebuild-tree。

### 14.3 行锁与并发查询

rebuild-tree 在独立 worker 进程中运行，使用独立的 DB 连接。主进程的查询和 worker 进程的写入通过数据库自身的 MVCC/锁机制协调：

- **Postgres**：truncate 需要 ACCESS EXCLUSIVE 锁，会阻塞所有并发查询，但持锁时间很短（只删数据不加条件）
- **MySQL/InnoDB**：truncate 等价于 DROP + CREATE，会短暂阻塞
- **SQLite**：使用文件级锁，truncate 和 insert 期间整个数据库不可写

**实际影响**：由于 truncate 执行很快（毫秒级），而 insert 是分批执行，每批之间有间隙，并发查询可以在批次间隙中执行，不会长时间阻塞。

### 14.4 pageTree 表的外键约束

```javascript
// migrations/2.0.0.js:304-311
table.integer('parent').unsigned().references('id').inTable('pageTree').onDelete('CASCADE')
table.integer('pageId').unsigned().references('id').inTable('pages').onDelete('CASCADE')
table.string('localeCode', 5).references('code').inTable('locales')
```

- `parent → pageTree.id`：自引用外键，CASCADE 删除（但 rebuild-tree 先 truncate 整表，不会触发）
- `pageId → pages.id`：CASCADE 删除，删除页面时自动清理 pageTree 中对应行（但 rebuild-tree 也会重建，所以这里的外键更像是安全网）
- MSSQL 不支持 self-cascade-delete（`dbCompat.selfCascadeDelete`），对 parent 字段只建普通整数列

---

## 十五、虚拟文件夹命名冲突

### 15.1 冲突产生场景

Wiki.js 的文件夹是**虚拟的**——由页面路径隐式生成。当两个不同 locale 的页面共享同一路径前缀，或一个页面路径恰好是另一个页面的前缀时，就会产生命名冲突。

**场景一：页面与文件夹同名**

```
页面 A: path = "guide"       → 生成节点: guide (isFolder=false, pageId=A.id)
页面 B: path = "guide/intro" → 需要节点: guide (isFolder=true)
```

rebuild-tree 的处理：

```javascript
// rebuild-tree.js:47-49
} else if (isFolder && !found.isFolder) {
  found.isFolder = true
  parentId = found.id
}
```

节点 `guide` 先作为页面节点创建（isFolder=false），当处理 `guide/intro` 时发现该路径已存在但不是文件夹，于是**升级为文件夹**。此时 `found.isFolder = true`，但 `found.pageId` 仍保留为页面 A 的 ID。

**结果**：节点同时是文件夹和页面——侧边栏中它既可展开又可点击浏览。

**场景二：不同页面共享中间路径**

```
页面 A: path = "docs/guide"
页面 B: path = "docs/api"
```

两个页面共享 `docs` 前缀，rebuild-tree 只创建一个 `docs` 文件夹节点（isFolder=true, pageId=null）。

**场景三：标题冲突**

文件夹的 title 直接取路径最后一段的原始文本（`part` 变量），不保证唯一：

```javascript
title: isFolder ? part : page.title
```

如果路径是 `API/reference`，文件夹 `API` 的标题就是 `API`。如果后来有路径 `api/usage`（小写），会生成另一个 `api` 文件夹节点，与 `API` 是**不同的节点**（path 不同）。

### 15.2 同路径不同 locale

不同 locale 的树完全隔离（通过 `localeCode` 区分），不会产生跨 locale 的命名冲突：

```javascript
const found = _.find(tree, { localeCode: page.localeCode, path: currentPath })
```

### 15.3 潜在的数据丢失

如果先有页面 `guide`（isFolder=false, pageId=A.id），后有页面 `guide/intro`，`guide` 节点被升级为文件夹。但如果是反向操作——删除 `guide/intro` 后重建树，`guide` 节点**不会再被升级为文件夹**（因为已无子页面），而是恢复为普通页面节点。这个过程是幂等的，不会丢失数据。

---

## 十六、history 快照与回滚

### 16.1 快照创建时机

**文件**：`server/models/pageHistory.js:91-109`

| 操作 | action 标记 | 触发位置 |
|------|------------|----------|
| 创建页面 | `updated` | pages.js:319（隐含在 insert 中，无显式快照） |
| 更新页面 | `updated` | pages.js:391-396 |
| 移动页面 | `moved` | pages.js:719-723 |
| 删除页面 | `deleted` | pages.js:806-810 |
| 转换页面 | `updated` | pages.js:633-639（仅当 shouldConvert=true） |

**注意**：创建页面时**不会**创建 history 快照，第一条快照产生于第一次更新。

### 16.2 快照数据结构

```javascript
static async addVersion(opts) {
  await WIKI.models.pageHistory.query().insert({
    pageId: opts.id,
    authorId: opts.authorId,
    content: opts.content,
    contentType: opts.contentType,
    description: opts.description,
    editorKey: opts.editorKey,
    hash: opts.hash,
    isPrivate: opts.isPrivate,
    isPublished: opts.isPublished,
    localeCode: opts.localeCode,
    path: opts.path,
    publishEndDate: opts.publishEndDate || '',
    publishStartDate: opts.publishStartDate || '',
    title: opts.title,
    action: opts.action || 'updated',
    versionDate: opts.versionDate
  })
}
```

快照保存的是**操作前的状态**（即 `...page` 展开的是修改前的数据），`versionDate` 是页面的 `updatedAt` 时间戳。

### 16.3 历史轨迹的 action 推断

**文件**：`server/models/pageHistory.js:158-231`

history 表的 `action` 字段只存储 `updated`/`moved`/`deleted`，但前端展示时需要区分 `initial`/`edit`/`move`。`getHistory` 方法通过比较相邻版本的 `path` 来推断操作类型：

```javascript
static async getHistory({ pageId, offsetPage = 0, offsetSize = 100 }) {
  // ...
  let prevPh = null
  _.reduce(_.reverse(history.results), (res, ph) => {
    let actionType = 'edit'
    if (!prevPh && history.total < upperLimit) {
      actionType = 'initial'           // 最早的记录 = 初始版本
    } else if (_.get(prevPh, 'path', '') !== ph.path) {
      actionType = 'move'              // 路径变了 = 移动
      valueBefore = _.get(prevPh, 'path', '')
      valueAfter = ph.path
    }
    prevPh = ph
    // ...
  })
}
```

### 16.4 版本回滚机制

**文件**：`server/graph/resolvers/page.js:576-608`

```javascript
async restore (obj, args, context) {
  const page = await WIKI.models.pages.query().select('path', 'localeCode').findById(args.pageId)
  // 权限检查
  if (!WIKI.auth.checkAccess(context.req.user, ['write:pages'], { path: page.path, locale: page.localeCode })) {
    throw new WIKI.Error.PageRestoreForbidden()
  }
  const targetVersion = await WIKI.models.pageHistory.getVersion({ pageId: args.pageId, versionId: args.versionId })
  // 用历史版本的数据调用 updatePage
  await WIKI.models.pages.updatePage({
    ...targetVersion,
    id: targetVersion.pageId,
    user: context.req.user,
    action: 'restored'
  })
}
```

**回滚的本质是「用旧数据创建新版本」**：

1. 获取指定版本的完整数据（content, title, description, path, locale 等）
2. 调用 `updatePage()` 将当前页面更新为旧版本的内容
3. `updatePage()` 内部会先创建当前状态的快照（`action: 'updated'`），然后应用旧版本内容
4. 回滚操作本身也会产生一条 history 记录

**重要限制**：

- 回滚**不会恢复页面路径**：即使旧版本的 `path` 不同，`updatePage` 中路径变化会触发 `movePage`，但 `getVersion` 返回的 `path` 是**快照时的路径**。如果此时路径已被其他页面占用，move 会因 `PagePathCollision` 失败
- 回滚**不恢复 tags**：`getVersion` 返回 `tags: []`（`pageHistory.js:148`），所以回滚后页面的标签会被清空
- 回滚**不恢复 isPrivate/isPublished**：`updatePage` 中这两个字段由前端传入，`targetVersion` 中有值但可能不是用户期望的

### 16.5 历史清理

```javascript
// pageHistory.js:238-242
static async purge (olderThan) {
  const dur = Duration.fromISO(olderThan)
  const olderThanISO = DateTime.utc().minus(dur)
  await WIKI.models.pageHistory.query().where('versionDate', '<', olderThanISO.toISO()).del()
}
```

通过 GraphQL mutation `purgeHistory(olderThan: "P90D")` 可以删除 90 天前的所有历史版本。此操作不可逆，且**不受外键保护**——pageHistory 表没有 CASCADE 关联到其他表。

---

## 十七、综合架构图

```
用户请求 URL
  │
  ▼
Express Router (common.js)
  │
  ├─ pageHelper.parsePath() ──→ 提取 locale + path
  │
  ├─ pages.getPage() ──→ 缓存(hash.bin) ──→ DB(pages 表)
  │
  └─ 渲染 HTML 响应

GraphQL Tree 查询
  │
  ▼
page.js resolver
  │
  ├─ pageTree 表查询 (parent/ancestors)
  │
  └─ auth.checkAccess() 逐节点过滤 ──→ Page Rules (START/END/REGEX/EXACT/TAG)

页面创建/更新/删除/移动
  │
  ▼
pages.js 模型方法
  │
  ├─ pageHistory.addVersion()    ← 快照
  ├─ pages.query().patch()       ← 写入
  ├─ pages.rebuildTree()         ← 注册 worker job
  │     │
  │     ▼
  │   scheduler.js ──→ fork worker.js ──→ rebuild-tree.js
  │     │                                    │
  │     │                                    ├─ DB: truncate pageTree
  │     │                                    └─ DB: chunk insert pageTree
  │     │
  │     └─ rebuildJob.finished (Promise)
  │
  ├─ searchEngine.renamed/deleted/created  ← 搜索索引
  ├─ storage.pageEvent()                   ← 外部存储
  └─ pages.reconnectLinks()                ← 链接重连
        │
        ├─ pageLinks 表查受影响页面
        ├─ REPLACE(render, 旧href, 新href)
        └─ 清除受影响页面缓存

页面渲染管线
  │
  ▼
render-page.js (worker job)
  │
  └─ html-core/renderer.js
       │
       ├─ 标注链接类型 (is-internal/external/system/asset-link)
       ├─ 查询 pages 表验证链接有效性 (is-valid/invalid-page)
       ├─ 更新 pageLinks 反向索引
       └─ 生成 TOC
```

---

## 十八、truncate 重试退避

### 18.1 rebuild-tree 本身无重试机制

**文件**：`server/jobs/rebuild-tree.js`

`rebuild-tree.js` 被整个 `try-catch` 包裹，但**没有任何重试逻辑**：

```javascript
try {
  // ... 构建 tree 数组 ...
  await WIKI.models.knex.table('pageTree').truncate()  // 无重试
  // ... 分块插入 ...
  WIKI.logger.info('Page tree rebuild: [ COMPLETED ]')
} catch (err) {
  WIKI.logger.error(err)
  WIKI.logger.error('Page tree rebuild: [ FAILED ]')
  process.exit(1)  // 直接退出 worker 进程
}
```

如果 truncate 失败（如锁超时、连接断开），worker 进程直接退出码 1，**任务失败且不重试**。scheduler 会收到 `exit code !== 0` 的信号，调用 `reject(err)`，但不会自动重调度。

### 18.2 数据库连接层的重试

**文件**：`server/core/db.js:179-195`

只有在系统启动时的初始数据库连接才有重试：

```javascript
try {
  await self.knex.raw('SELECT 1 + 1;')
} catch (err) {
  if (conAttempts < 10) {
    WIKI.logger.warn(`Will retry in 3 seconds... [Attempt ${++conAttempts} of 10]`)
    await new Promise(resolve => setTimeout(resolve, 3000))  // 固定 3 秒间隔，无指数退避
    await initTasks.connect()
  } else {
    throw err
  }
}
```

**连接重试特性**：
- 固定间隔 3 秒，无指数退避
- 最多 10 次尝试
- 仅在启动时的初始连接使用，运行时连接断开不会自动重连

### 18.3 手动恢复策略

如果 rebuild-tree 失败，系统提供两种手动恢复方式：

1. **GraphQL mutation**：`rebuildTree`（需 `manage:system` 权限）
2. **重启系统**：启动时自动触发一次 rebuild-tree（`onInit: true`）

没有自动重试的设计考量：
- rebuild-tree 是幂等操作，失败了随时可以重试
- 自动重试可能导致多个重建任务并发，反而加剧问题
- 树数据不一致的影响相对可控（侧边栏显示异常，但页面仍可通过 URL 直接访问）

---

## 十九、自引用外键的递归深度

### 19.1 应用层无递归深度限制

**文件**：`server/jobs/rebuild-tree.js`

树的深度由页面路径的分段数决定，**没有硬编码的最大深度限制**：

```javascript
for (const page of pages) {
  const pagePaths = page.path.split('/')  // 拆分路径段
  for (const part of pagePaths) {         // 遍历每一段
    depth++                                // 深度递增
    // ... 创建节点 ...
  }
}
```

理论上，如果有一个页面路径有 1000 段（虽然极其不现实），rebuild-tree 会创建 1000 层深度的节点。

### 19.2 数据库层的自引用递归限制

虽然应用层无限制，但数据库本身可能有递归查询限制：

| 数据库 | 递归查询深度限制 | 说明 |
|--------|----------------|------|
| Postgres | 默认无限制 | `max_stack_depth` 控制递归深度 |
| MySQL | 1000 层 | `cte_max_recursion_depth` 变量 |
| SQL Server | 100 层 | `MAXRECURSION` 提示，默认 100 |
| SQLite | 无明确限制 | 受内存和栈限制 |

**重要**：Wiki.js **从未使用递归 CTE 查询** pageTree 表。所有查询都是单层 `WHERE parent = ?` 或 `WHERE path = ?`，因此数据库的递归深度限制不会成为问题。

### 19.3 实际场景中的深度限制

操作系统和文件系统的限制间接限制了路径深度：

- **URL 长度限制**：大多数浏览器和服务器限制 URL 长度在 2000-8000 字符
- **磁盘路径长度**：Linux 文件名最长 255 字节，路径总长通常无限制
- **浏览器历史栈深度**：通常不限制，但极深路径影响 UX

Wiki.js 的路径校验（`pages.js:242-249`）也间接限制了单段长度（不能包含 `.`、空格等），但没有显式的总长度或深度校验。

### 19.4 潜在风险

极深的路径（如 50+ 层）可能导致：

1. **rebuild-tree 内存占用**：每一层都创建节点对象，极深路径会增加内存使用
2. **页面选择器 UX 问题**：v-treeview 需要多次点击才能展开到深层
3. **URL 冗长**：复制/分享不便

---

## 二十、MSSQL 自引用 CASCADE 不支持的兼容回退

### 20.1 问题根源

**文件**：`server/db/migrations/2.0.0.js:4-7`

```javascript
const dbCompat = {
  selfCascadeDelete: WIKI.config.db.type !== 'mssql'
}
```

SQL Server **不支持自引用外键的 CASCADE DELETE**。如果尝试创建这样的约束，MSSQL 会报错：

```
Introducing FOREIGN KEY constraint '...' on table 'pageTree' may cause cycles or multiple cascade paths.
```

MSSQL 认为 `parent → pageTree.id` 是一个潜在的循环引用路径，拒绝创建 CASCADE 约束。

### 20.2 兼容回退方案

**文件**：`server/db/migrations/2.0.0.js:304-309`

```javascript
.table('pageTree', table => {
  if (dbCompat.selfCascadeDelete) {
    table.integer('parent').unsigned().references('id').inTable('pageTree').onDelete('CASCADE')
  } else {
    table.integer('parent').unsigned()  // MSSQL: 只建普通整数列，无外键约束
  }
  table.integer('pageId').unsigned().references('id').inTable('pages').onDelete('CASCADE')
})
```

**MSSQL 回退特性**：

1. **无外键约束**：`parent` 列只是普通整数，没有引用完整性保证
2. **无 CASCADE 删除**：删除父节点时，子节点的 `parent` 字段不会被自动清理
3. **数据一致性依赖应用层**：完全由 rebuild-tree 的 `truncate + insert` 保证一致性

### 20.3 为什么这个回退是安全的

rebuild-tree 的工作方式使得这个回退几乎没有实际影响：

1. **总是全量重建**：每次都是先 `truncate` 整张表，再 `insert` 所有节点。不存在「删除单个父节点」的操作场景。
2. **CASCADE 从未被触发**：即使在 Postgres/MySQL 上，`parent` 的 CASCADE DELETE 约束也从未被实际触发过，因为从来不会 delete 单个节点。
3. **外键仅作为文档**：在非 MSSQL 数据库上，外键更多是文档性质和额外的安全网，而非功能依赖。

### 20.4 MSSQL 的其他兼容问题

MSSQL 还有其他兼容性处理：

```javascript
// migrations/2.5.122.js:4-6
const dbCompat = {
  blobLength: (WIKI.config.db.type === `mysql` || WIKI.config.db.type === `mariadb`),  // MySQL 需要 LONGBLOB
  charset: (WIKI.config.db.type === `mysql` || WIKI.config.db.type === `mariadb`)       // MySQL 需要 utf8mb4
}
```

MSSQL 不需要这些特殊处理，使用标准的 `binary` 和数据库默认字符集。

---

## 二十一、虚拟文件夹命名规范

### 21.1 路径校验规则

**文件**：`server/models/pages.js:242-249`（创建时）、`680-692`（移动时）

```javascript
if (opts.path.includes('.') || opts.path.includes(' ') ||
    opts.path.includes('\\') || opts.path.includes('//')) {
  throw new WIKI.Error.PageIllegalPath()
}
if (opts.path.endsWith('/')) { opts.path = opts.path.slice(0, -1) }
if (opts.path.startsWith('/')) { opts.path = opts.path.slice(1) }
```

**禁止的字符**：

| 字符 | 禁止原因 |
|------|----------|
| `.` | 防止与文件扩展名混淆（`parsePath` 用 `.` 识别资源） |
| 空格 | 防止 URL 编码问题和路径歧义 |
| `\` | 防止 Windows 路径分隔符注入 |
| `//` | 防止空路径段 |

**自动清理**：
- 开头的 `/` 自动去除
- 结尾的 `/` 自动去除

### 21.2 文件夹标题的生成规则

**文件**：`server/jobs/rebuild-tree.js:40`

```javascript
title: isFolder ? part : page.title
```

虚拟文件夹的 `title` 直接取路径段的**原始文本**（`part` 变量），**不做任何转换**：

1. **大小写敏感**：`API` 和 `api` 是不同的路径，生成两个不同的文件夹节点
2. **保留特殊字符**：路径段中的 Unicode 字符、数字、连字符、下划线都原样保留
3. **不做 slugify**：不会将中文自动转为拼音，不会将空格替换为连字符（因为路径校验已经禁止了空格）

### 21.3 允许的路径段

合法的路径段示例：

| 路径 | 说明 |
|------|------|
| `docs/guide/install` | 标准 ASCII 路径 |
| `文档/指南/安装` | 纯中文路径（完全支持） |
| `api/v2/users` | 含版本号 |
| `my-guide_2024/chapter-1` | 含下划线和连字符 |
| `α/β/γ` | Unicode 希腊字母 |
| `1/2/3` | 纯数字 |

### 21.4 命名冲突的处理

**大小写冲突**：`Docs/Guide` 和 `docs/guide` 会生成两个独立的文件夹树，因为 path 字段是大小写敏感的字符串比较。

**同路径不同 locale**：通过 `localeCode` 区分，不会冲突。

**节点升级冲突**：当页面路径恰好是另一页面的前缀时，节点会被升级为文件夹（同时保留 pageId），这是设计允许的行为，不是冲突。

---

## 二十二、history 快照存储成本

### 22.1 快照数据结构

**文件**：`server/db/migrations/2.0.0.js:115-131`

```javascript
.createTable('pageHistory', table => {
  table.increments('id').primary()
  table.string('path').notNullable()
  table.string('hash').notNullable()
  table.string('title').notNullable()
  table.string('description')
  table.boolean('isPrivate').notNullable().defaultTo(false)
  table.boolean('isPublished').notNullable().defaultTo(false)
  table.string('action').defaultTo('updated')
  table.integer('pageId').unsigned()
  table.text('content')           // 完整的原始内容，无压缩
  table.string('contentType').notNullable()
  table.string('createdAt').notNullable()
})
```

### 22.2 存储成本分析

**每次快照存储的内容**：

| 字段 | 典型大小 | 是否冗余 |
|------|----------|----------|
| path | 50-200 字节 | 多次更新时重复存储 |
| title | 10-100 字节 | 多次更新时重复存储 |
| content | **1KB-1MB+** | 完整存储，无差异压缩 |
| 其他元数据 | ~100 字节 | - |

**无增量存储**：每次更新都保存完整的 `content` 副本，即使只修改了一个字。

**无压缩**：`content` 字段是普通的 `TEXT` 类型，数据库层面可能做透明压缩（如 Postgres TOAST、MySQL InnoDB 页压缩），但应用层未做任何压缩。

### 22.3 成本估算公式

```
总存储 = 页面数 × 平均每次更新内容大小 × 平均每页面更新次数 × 1.3（索引开销）
```

**典型场景估算**：

| 场景 | 页面数 | 平均内容 | 更新次数 | 估算存储 |
|------|--------|----------|----------|----------|
| 小型 Wiki | 100 | 10 KB | 10 次 | ~13 MB |
| 中型 Wiki | 1,000 | 20 KB | 50 次 | ~1.3 GB |
| 大型 Wiki | 10,000 | 50 KB | 100 次 | **~65 GB** |

### 22.4 pageHistoryTags 的额外开销

**文件**：`server/db/migrations/2.0.0.js:258-263`

```javascript
.createTable('pageHistoryTags', table => {
  table.integer('pageId').unsigned().references('id').inTable('pageHistory').onDelete('CASCADE')
  table.integer('tagId').unsigned().references('id').inTable('tags').onDelete('CASCADE')
})
```

- 每个历史版本的每个标签占用一行（约 16 字节 + 索引）
- 每次 purgeHistory 时，通过 CASCADE 自动清理对应的标签关联

### 22.5 存储成本的缓解措施

1. **定期 purge**：通过 `purgeHistory(olderThan: "P90D")` 清理旧版本
2. **关闭历史记录**：可以在配置中关闭（实际代码中未提供开关，但可以通过权限限制 `write:pages` 用户）
3. **数据库压缩**：依赖数据库的透明压缩功能

---

## 二十三、回滚的部分恢复策略

### 23.1 回滚是「全量恢复」

**文件**：`server/graph/resolvers/page.js:576-608`

```javascript
async restore (obj, args, context) {
  const targetVersion = await WIKI.models.pageHistory.getVersion({ ... })
  await WIKI.models.pages.updatePage({
    ...targetVersion,    // 展开完整的版本数据
    id: targetVersion.pageId,
    user: context.req.user,
    action: 'restored'
  })
}
```

`restore` 操作是**全量恢复**——将目标版本的所有字段（path, title, description, content, contentType, isPrivate, isPublished, publishStartDate, publishEndDate 等）一次性应用到当前页面。

### 23.2 「部分不恢复」的特例

虽然设计上是全量恢复，但有两处硬编码的「部分不恢复」：

**特例一：Tags 不恢复**

**文件**：`server/models/pageHistory.js:148`

```javascript
return {
  ...version,
  updatedAt: version.createdAt || null,
  tags: []  // 硬编码为空数组
}
```

`getVersion` 方法总是返回 `tags: []`，导致回滚后页面的标签被清空。

**原因分析**：历史标签存储在 `pageHistoryTags` 关联表中，但 `getVersion` 没有 join 这个表。可能是设计疏漏，也可能是有意为之（标签变化不视为内容变化）。

**特例二：editorKey 映射变化**

```javascript
{
  versionId: 'pageHistory.id',
  editor: 'pageHistory.editorKey',  // 字段重命名
  locale: 'pageHistory.localeCode'
}
```

`editorKey` 被重命名为 `editor`，但 `updatePage` 期望的是 `editorKey` 字段。如果 `updatePage` 内部使用 `opts.editorKey`，这个字段会是 `undefined`，可能导致编辑器类型被重置为默认值。

### 23.3 路径恢复的限制

回滚时的路径恢复不是原子操作：

1. `getVersion` 返回历史版本的 `path`（快照时的路径）
2. `updatePage` 检测到 `opts.path !== currentPage.path`，调用 `movePage`
3. `movePage` 检查目标路径是否已被占用
4. 如果已占用，抛出 `PagePathCollision`，回滚失败

这意味着：如果页面被移动后，旧路径被另一个页面占用，就无法回滚到旧版本（因为路径冲突）。

### 23.4 部分恢复的实现思路

当前代码不支持细粒度的部分恢复（如「只恢复内容不恢复标题」）。如果需要，可以通过以下方式扩展：

1. 在 `restore` mutation 中增加可选参数，指定要恢复的字段
2. 只展开 `targetVersion` 中指定的字段到 `updatePage`
3. 但 `updatePage` 本身的设计假设传入完整数据，需要调整

---

## 二十四、purgeHistory 审计

### 24.1 权限控制

**文件**：`server/graph/schemas/page.graphql:163-165`

```graphql
purgeHistory (
  olderThan: String!
): DefaultResponse @auth(requires: ["manage:system"])
```

只有拥有 `manage:system` 权限的用户（系统管理员）才能调用 purgeHistory。

### 24.2 操作实现

**文件**：`server/graph/resolvers/page.js:612-620`

```javascript
async purgeHistory (obj, args, context) {
  try {
    await WIKI.models.pageHistory.purge(args.olderThan)
    return {
      responseResult: graphHelper.generateSuccess('Page history purged successfully.')
    }
  } catch (err) {
    return graphHelper.generateError(err)
  }
}
```

**关键缺失**：

1. **无审计日志**：没有记录「谁执行了 purge、在什么时间、删除了多少条记录」
2. **无操作前备份**：直接 DELETE，不可逆
3. **无确认机制**：mutation 没有 `confirm: true` 参数或二次确认
4. **无影响范围预览**：无法先查询「将删除多少条记录、哪些页面的历史」再执行

### 24.3 purge 实际执行的 SQL

**文件**：`server/models/pageHistory.js:238-242`

```javascript
static async purge (olderThan) {
  const dur = Duration.fromISO(olderThan)
  const olderThanISO = DateTime.utc().minus(dur)
  await WIKI.models.pageHistory.query().where('versionDate', '<', olderThanISO.toISO()).del()
}
```

生成的 SQL 大致是：

```sql
DELETE FROM "pageHistory" WHERE "versionDate" < '2024-01-01T00:00:00.000Z';
```

`pageHistoryTags` 表中的关联记录通过 `onDelete('CASCADE')` 自动清理。

### 24.4 潜在风险

1. **误操作无法恢复**：管理员误操作 purge 了过短的时间窗口（如 `P1D` 而非 `P90D`），所有昨天及之前的历史版本永久丢失
2. **无法审计追责**：如果 purge 被恶意执行，无法追溯操作者和时间
3. **长事务锁表**：如果 pageHistory 表很大（百万行级），DELETE 可能持锁很长时间，阻塞其他操作
4. **无进度反馈**：purge 是同步执行的，大表删除可能超时

### 24.5 审计改进建议

生产环境建议增加以下审计措施：

1. **操作日志**：在执行 purge 前写入审计日志，记录 userId、timestamp、olderThan 参数
2. **影响预览**：先执行 `count()` 返回将删除的记录数，让用户确认
3. **软删除**：增加 `isPurged` 标记，而非物理删除
4. **定期自动 purge**：通过 cron job 定期执行，避免人工误操作

---

## 二十五、CASCADE 级联深度限制

### 25.1 数据库级联链分析

让我们梳理 Wiki.js 中所有的 CASCADE 删除关系：

#### pageTree 表
```
pages.id ──CASCADE──→ pageTree.pageId
```
- 深度：**1 层**
- 说明：删除页面时，pageTree 中对应的行被自动删除（但 rebuild-tree 会重建，所以不重要）

#### pageLinks 表
```
pages.id ──CASCADE──→ pageLinks.pageId
```
- 深度：**1 层**
- 说明：删除页面时，该页面引用的所有链接记录被自动删除

#### pageHistory 表
```
users.id ──→ pageHistory.authorId  (无 CASCADE)
editors.key ──→ pageHistory.editorKey  (无 CASCADE)
```
- 无 CASCADE 删除，删除用户不会级联删除其编辑的历史版本

#### pageHistoryTags 表
```
pageHistory.id ──CASCADE──→ pageHistoryTags.pageId
tags.id ──CASCADE──→ pageHistoryTags.tagId
```
- 深度：**2 层**（pages → pageHistory → pageHistoryTags）
- 说明：删除页面 → 删除 pageHistory → 删除 pageHistoryTags

#### pageTags 表
```
pages.id ──CASCADE──→ pageTags.pageId
tags.id ──CASCADE──→ pageTags.tagId
```
- 深度：**1 层**

#### userGroups 表
```
users.id ──CASCADE──→ userGroups.userId
groups.id ──CASCADE──→ userGroups.groupId
```
- 深度：**1 层**

#### commentProviders 等其他表
均为单层 CASCADE 或无 CASCADE。

### 25.2 最长级联链

整个数据库中最长的 CASCADE 删除链是 **2 层**：

```
pages.id
    ↓ CASCADE
pageHistory.id
    ↓ CASCADE
pageHistoryTags
```

当删除一个页面时：
1. `pages.id` CASCADE 删除 `pageHistory` 中所有 `pageId` 匹配的行
2. `pageHistory.id` CASCADE 删除 `pageHistoryTags` 中所有 `pageId` 匹配的行

### 25.3 级联深度限制

各数据库的级联删除深度限制：

| 数据库 | 级联深度限制 | 说明 |
|--------|------------|------|
| Postgres | 无明确限制 | 受 `max_stack_depth` 间接限制 |
| MySQL | 15 层 | 超过时报错 "Too many tables in cascade delete" |
| SQL Server | 无明确限制 | 受事务日志和锁超时限制 |
| SQLite | 无明确限制 | 受内存限制 |

Wiki.js 的最长级联链是 2 层，**远低于所有数据库的限制**，不存在级联深度超限问题。

### 25.4 CASCADE 与 pageTree 自引用

如前所述，pageTree 的自引用外键：

```
pageTree.id ──CASCADE──→ pageTree.parent
```

在 MSSQL 上被禁用（见第二十章），在其他数据库上启用但**从未实际触发**，因为：
1. 从未单独删除某个 pageTree 节点（总是 truncate 整表）
2. truncate 不触发 CASCADE（truncate 是 DDL，不是 DML DELETE）

因此这个自引用 CASCADE 只是"摆设"，不构成实际的级联链。

### 25.5 pageTree.parent 的 CASCADE 触发场景

理论上，以下操作会触发自引用 CASCADE（仅非 MSSQL）：

```sql
DELETE FROM "pageTree" WHERE id = 123;  -- 假设 id=123 有子节点
```

但 Wiki.js 的代码中**从来不会执行这样的 DELETE**。所有 pageTree 表的修改都是：
- `truncate()` → 全表清空，不触发 CASCADE
- `insert()` → 批量插入

因此，这个自引用 CASCADE 约束更多是防御性设计，防止手动操作数据库时留下孤儿节点。

---

## 二十六、深度风险总结

| 风险点 | 严重程度 | 影响范围 | 触发条件 |
|--------|----------|----------|----------|
| rebuild-tree 失败无重试 | 中 | 侧边栏、页面选择器 | 数据库锁冲突、网络闪断 |
| movePage 无乐观锁 | 中 | 页面内容 | 两个用户同时编辑同一页面 |
| reconnectLinks 字符串匹配失败 | 高 | 所有内部链接 | 渲染管线修改了 HTML 结构 |
| 大目录 treeview 性能 | 中 | 前端交互 | 单目录 1000+ 节点 |
| purgeHistory 无审计 | 高 | 历史版本 | 管理员误操作、账号泄露 |
| 回滚不恢复 tags | 中 | 页面标签 | 任何回滚操作 |
| MSSQL 无自引用外键 | 低 | pageTree 数据一致 | 手动操作数据库误删除 |
| 级联深度超限 | 无 | - | 不可能触发 |
