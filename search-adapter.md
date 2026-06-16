# 多种搜索引擎统一接口接入 —— 代码实现梳理

## 1. 整体架构分层

搜索模块在代码中分为 **三个核心层次**，自顶向下依次是：

| 层级 | 职责 | 关键文件 |
|------|------|----------|
| **GraphQL API 层** | 暴露搜索管理、搜索查询接口，做权限与参数校验 | `server/graph/schemas/search.graphql`、`server/graph/schemas/page.graphql`、`server/graph/resolvers/search.js`、`server/graph/resolvers/page.js` |
| **统一引擎调度层** | 引擎元信息扫描、动态加载、激活/切换、生命周期管理，将 `WIKI.data.searchEngine` 指向当前生效 provider | `server/models/searchEngines.js` |
| **Provider 适配层** | 针对每种搜索引擎实现统一契约接口，完成字段映射、查询翻译、索引同步 | `server/modules/search/<engine-key>/engine.js` + `definition.yml` |

调用方向：
```
用户查询 ──► GraphQL (PageQuery.search)
                  │
                  ▼
        WIKI.data.searchEngine.query(q, opts)   ◄── 统一调度层挂在全局对象上
                  │
                  ▼
        具体 provider 的 engine.js 实现
                  │
                  ▼
        搜索引擎原生 SDK / HTTP API
```

页面变更事件触发方向：
```
Page.createPage / updatePage / movePage / deletePage
                  │
                  ▼
        WIKI.data.searchEngine.created / updated / renamed / deleted
                  │
                  ▼
        具体 provider 同步到搜索引擎索引
```

---

## 2. 统一接口契约

所有 Provider（位于 `server/modules/search/*/engine.js`）必须导出以下 **9 个方法**，由调度层按同名约定调用（无抽象基类，纯 Duck Typing）。

### 2.1 生命周期方法

| 方法 | 调用时机 | 说明 |
|------|----------|------|
| `activate()` | 引擎被用户启用时（`initEngine({activate:true})`） | 做前置环境检查；失败可抛 `WIKI.Error.SearchActivationFailed` 触发回退到 `db` 引擎 |
| `deactivate()` | 切换到其他引擎之前 | 清理资源（如 postgres 会删自建表） |
| `init()` | 引擎加载后立即执行 | 初始化客户端、创建索引、定义字段映射 |
| `rebuild()` | 管理员手动触发「重建索引」 | 全量遍历 pages 表并重新写入搜索引擎 |

### 2.2 查询方法

| 方法 | 签名 | 返回值规范 |
|------|------|------------|
| `query(q, opts)` | `q:String` 搜索关键词；`opts:{locale?, path?}` 过滤条件 | **必须返回** `{ results: Array, suggestions: Array, totalHits: Number }` |

**`results` 每条记录规范字段**（所有 Provider 必须映射到这套字段）：
```js
{
  id: String,          // 对应 page.hash
  locale: String,      // 对应 page.localeCode
  path: String,        // 对应 page.path
  title: String,
  description: String,
  tags?: Array<String> // 仅部分引擎提供，用于上层权限校验
}
```

### 2.3 索引变更事件

| 方法 | 触发点 (pages.js) | 入参 page 对象关键字段 |
|------|-------------------|------------------------|
| `created(page)` | `createPage()` 成功渲染后 (L341) | `hash, localeCode, path, title, description, safeContent` |
| `updated(page)` | `updatePage()` 成功渲染后 (L452) | 同上 |
| `renamed(page)` | `movePage()` 时 (L744) | 额外附带 `destinationPath, destinationLocaleCode, destinationHash` |
| `deleted(page)` | `deletePage()` 时 (L821) | `hash, localeCode, path` |

> 设计要点：**上层只发事件，不关心底层搜索引擎怎么实现**。Provider 内部决定是单条写入、批量缓冲、还是删旧建新。

---

## 3. Provider 适配实现全景

项目共内置 **9 个 search provider**，位于 `server/modules/search/`：

| 目录 key | 标题 | 实现完整度 | 特点 |
|----------|------|------------|------|
| `db` | Database - Basic | ✅ 完整 | 直接查 `pages` 表用 `LIKE/ILIKE`，零依赖默认引擎 |
| `postgres` | Database - PostgreSQL | ✅ 完整 | 利用 PG `tsvector` + `tsquery` + `pg_trgm`，自建 `pagesVector` / `pagesWords` 表 |
| `elasticsearch` | Elasticsearch | ✅ 完整 | 同时兼容 6.x / 7.x / 8.x 三个大版本客户端，bulk + stream 重建 |
| `algolia` | Algolia | ✅ 完整 | SaaS 搜索，按单/批量 `saveObjects`，有 10KB 单文档限制 |
| `azure` | Azure Cognitive Search | ✅ 完整 | 官方 SDK + `scoringProfiles` 权重，suggestion 走原始 HTTP |
| `aws` | AWS CloudSearch | ✅ 完整 | 需先定义 `IndexField` / `AnalysisScheme` / `Suggester`，再 `indexDocuments` |
| `solr` | Solr | ⚪ 空壳 | 所有方法空实现，占位 |
| `manticore` | Manticore Search | ⚪ 空壳 | 所有方法空实现，占位 |
| `sphinx` | Sphinx | ⚪ 空壳 | 所有方法空实现，占位 |

### 3.1 Provider 的声明文件 definition.yml

每个 provider 根目录下的 `definition.yml` 用于：
- 描述元信息（`key`、`title`、`description`、`logo`、`website`、`author`）
- 用 `props` 声明可配置项（类型、默认值、枚举、顺序、提示文案）

示例（elasticsearch）：
```yaml
key: elasticsearch
title: Elasticsearch
props:
  apiVersion:
    type: String
    enum: ['8.x', '7.x', '6.x']
    default: '7.x'
    order: 1
  hosts:
    type: String
    order: 2
  indexName:
    type: String
    default: wiki
    order: 5
```

调度层 `refreshSearchEnginesFromDisk()`（`searchEngines.js:38`）会扫描该目录并：
1. YAML 解析后写入 `WIKI.data.searchEngines`
2. 数据库不存在则插入默认配置（按 `props.default` 赋值）
3. 数据库已有则补齐新增字段的默认值

### 3.2 引擎激活与切换流程

代码入口：`searchEngines.js:98` `initEngine({activate})`

```
1. 查 searchEngines 表 where isEnabled = true
2. require(`../modules/search/${key}/engine`) → 挂载到 WIKI.data.searchEngine
3. WIKI.data.searchEngine.key = key
4. WIKI.data.searchEngine.config = DB 中存的 config 对象
5. if (activate) 调用 searchEngine.activate()
   ├─ 成功 → 继续
   └─ 抛 SearchActivationFailed → 强制切换回 db 引擎并递归 initEngine()
6. 调用 searchEngine.init()
```

切换引擎时（resolver `updateSearchEngines`）：
1. 先对旧引擎调用 `deactivate()`
2. 再调用 `initEngine({activate: true})` 加载新引擎

---

## 4. 字段映射详解

### 4.1 统一「页面对象」→「搜索引擎文档」映射表

不同 provider 对同一逻辑字段的命名/类型略有差异，但语义一致：

| 逻辑字段 | 来源 (page 对象) | db (SQL) | postgres | elasticsearch | algolia | azure | aws |
|----------|-------------------|----------|----------|---------------|---------|-------|-----|
| **文档 ID** | `page.hash` | 未索引用主键 | `path+locale` 唯一 | `_id = page.hash` | `objectID` | `id (Edm.String key)` | `id (literal)` |
| **locale** | `page.localeCode` | `localeCode` | `locale` | `locale (keyword)` | `locale` | `locale` | `locale (literal)` |
| **path** | `page.path` | `path` | `path` | `path (text)` | `path` | `path` | `path (literal)` |
| **title** | `page.title` | `title` | `title (weight A)` | `title (text, boost 10~20x)` | `title` | `title (weight 4)` | `title (text)` |
| **description** | `page.description` | `description` | `description (weight B)` | `description (text, boost 3x)` | `description` | `description (weight 3)` | `description (text)` |
| **正文内容** | `page.safeContent` (cleanHTML 后) | 不参与索引 | `content (weight C)` | `content (text, boost 1x)` | `content` | `content` | `content (text, ReturnEnabled=false)` |
| **标签** | `page.tags` | 未映射 | 未映射 | `tags (text, boost 8x)` | 未映射 | 未映射 | 未映射 |
| **suggest** | 分词聚合 | 无 | `pagesWords` 表 | `suggest (completion)` | 无 | `suggester` | `default_suggester` |

### 4.2 权重设计模式

- **elasticsearch 6.x/7.x**：在 mapping `properties.title.boost` 静态声明；8.x 不支持 mapping boost，改在 query 时 `fields: ['title^20', ...]` 动态加权
- **postgres**：`setweight(..., 'A'/'B'/'C')` 写入 `TSVECTOR`
- **azure**：`scoringProfiles.text.weights: { title:4, description:3, content:1 }`
- **aws**：未设置字段级权重，依赖 CloudSearch 默认打分
- **algolia**：通过 `searchableAttributes` 顺序隐式权重（靠前优先级高）
- **db**：无权重，返回顺序与数据库行顺序一致

---

## 5. 查询编排衔接

### 5.1 一次用户搜索的完整链路

```
前端 GraphQL 调用 Query.pages.search(query, path?, locale?)
        │
        ▼
server/graph/resolvers/page.js PageQuery.search (L52)
        │
        ├─ 1. 调用 WIKI.data.searchEngine.query(args.query, args)
        │       └─► 具体 provider 返回 { results, suggestions, totalHits }
        │
        ├─ 2. 权限过滤：_.filter(results, r =>
        │       WIKI.auth.checkAccess(user, ['read:pages'], {
        │         path: r.path, locale: r.locale, tags: r.tags
        │       }))
        │
        └─ 3. 返回过滤后的 PageSearchResponse
                { results: [...], suggestions: [...], totalHits }
```

> **重要**：权限过滤在上层 resolver 做，**不在 provider 里做**。Provider 只管「召回」，不管「授权」。这也是为什么 `db` provider 返回结果里显式 `withGraphJoined('tags')` —— 上层校验需要。

### 5.2 Provider 内部查询编排差异

不同 provider 对同一查询参数的翻译方式：

| Provider | `query(q, opts)` 内部实现要点 |
|----------|-------------------------------|
| **db** | Knex builder：`title ILIKE %q% OR description ILIKE %q% OR path ILIKE %q%`，同时 `andWhere(isPublished=true)`，`opts.locale/path` 叠加 `andWhere`。无 suggestion。 |
| **postgres** | `to_tsquery(dictLanguage, tsquery(q)) @@ tokens`，`ts_rank` 排序；结果 <5 条时 fallback 查 `pagesWords` 用 `similarity` 做拼写建议。 |
| **elasticsearch** | `simple_query_string` + `*${q}*` 通配 + 字段加权，同时返回 `completion suggest`；`opts.locale/path` **未使用**（只做关键词检索）。 |
| **algolia** | SDK `index.search(q, {hitsPerPage:50})`，suggestions 空数组；`opts` **未使用**。 |
| **azure** | SDK `search({search:q, scoringProfile:'fieldWeights', queryType:simple, top:50})`；结果 <5 条时走 HTTP `/docs/autocomplete` 获取建议。 |
| **aws** | `clientDomain.search({query:q, partial:true, size:50})`；结果 <5 条时调 `suggest()` 返回建议。 |

> 差异点：`opts.locale` 和 `opts.path` 过滤 **只有 db 和 postgres 两个 SQL 系引擎真正实现了**，elasticsearch/algolia/azure/aws 都忽略了这两个参数，权限和路径范围完全靠上层 resolver 后置过滤。

---

## 6. 索引重建 (rebuild) 的通用模式

所有完整实现的 provider 的 `rebuild()` 共享一套 **流式 + 分批** 的架构，防止一次性加载所有页面内存溢出：

```
1. 清空/删除旧索引
2. 创建新索引（含字段映射、analyzer、suggester 等）
3. WIKI.models.knex.select(...).from('pages')
        .where({isPublished:true, isPrivate:false})
        .stream()   ◄── 数据库流式游标
        │
        ▼
4. new Transform({ objectMode:true, transform })
        │
        ├─ 对每条 page.cleanHTML(render) → safeContent
        ├─ 按字节数 (MAX_INDEXING_BYTES ~10MB) 和条数 (MAX_INDEXING_COUNT ~1000) 双阈值缓冲
        └─ 达到阈值时 flush
        │
        ▼
5. 搜索引擎批量写入 API
        ├─ elasticsearch: bulk({ body: [action, doc, action, doc, ...] })
        ├─ algolia: saveObjects([...])
        ├─ aws: uploadDocuments({ documents: JSON.stringify([...]) })
        ├─ azure: createIndexingStream() (直接 pipe)
        └─ postgres: 逐条 INSERT (无需批量)
```

核心复用代码：`node:stream/promises.pipeline` + `node:stream.Transform`，在 elasticsearch、algolia、aws、azure、postgres 中均以相同模式出现。

---

## 7. 关键代码坐标速查

| 关注点 | 文件 : 行号 |
|--------|-------------|
| 引擎扫描与 DB 同步 | `server/models/searchEngines.js:38` `refreshSearchEnginesFromDisk()` |
| 引擎加载与激活 | `server/models/searchEngines.js:98` `initEngine()` |
| 用户搜索查询入口 | `server/graph/resolvers/page.js:52` `PageQuery.search` |
| 管理端引擎配置保存 | `server/graph/resolvers/search.js:41` `updateSearchEngines` |
| 管理端重建索引入口 | `server/graph/resolvers/search.js:71` `rebuildIndex` |
| 页面创建触发索引 | `server/models/pages.js:341` `searchEngine.created(page)` |
| 页面更新触发索引 | `server/models/pages.js:452` `searchEngine.updated(page)` |
| 页面移动触发索引 | `server/models/pages.js:744` `searchEngine.renamed(...)` |
| 页面删除触发索引 | `server/models/pages.js:821` `searchEngine.deleted(page)` |
| 统一搜索响应 GraphQL 类型 | `server/graph/schemas/page.graphql:257` `PageSearchResponse` / `PageSearchResult` |
| Provider 统一契约（参考完整实现） | `server/modules/search/db/engine.js` 或 `server/modules/search/elasticsearch/engine.js` |

---

## 8. 设计归纳与演进观察

1. **Duck Typing 契约而非 OOP 继承**：没有抽象基类/接口，所有 provider 只要导出约定方法即可，适合插件化扩展但缺乏编译期约束（solr/manticore/sphinx 三个空实现即能通过）。

2. **「召回」与「授权」分离**：Provider 只做搜索召回，权限过滤统一在上层 resolver 后置执行。这保证了接入新引擎时不需要重复实现权限逻辑，但也带来「实际可见数远小于 totalHits」的统计偏差风险。

3. **过滤器参数 (`locale`/`path`) 未全量落地**：仅 SQL 系的 db/postgres 在查询层应用了 locale 和 path 过滤；外部搜索引擎（es/algolia/azure/aws）忽略这两个参数，靠上层过滤。对大型站点来说，召回集可能被过滤掉大部分，产生性能浪费。

4. **字段权重通过「多处分散声明」**：不同引擎用不同机制表达同一套权重语义（title > description > content），无集中权重配置。新增引擎时需在 mapping/scoringProfile/searchableAttributes 中人工对齐。

5. **suggestion 触发策略不统一**：postgres 用相似度、es 用 completion suggester、azure/aws 是「结果不足 5 条时 fallback 查询」、algolia/db 直接返回空。上层 UI 对 suggestions 的渲染因此呈现不一致。

6. **文档 ID 策略不统一**：大多数引擎用 `page.hash` 作主键，postgres 则用 `path + locale` 作为去重键。这导致 rename 操作时，es/algolia/azure/aws 需要「先删旧 hash 再建新 hash」，而 postgres 只需 UPDATE 行。
