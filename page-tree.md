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
