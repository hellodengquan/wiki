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

---

## 9. 运行时动态注册 provider 的安全审计

### 9.1 实际加载时机与路径

`refreshSearchEnginesFromDisk()` **并非运行时热加载**，而是在两个固定时机被调用：
- 系统启动时：`server/core/kernel.js:78`（postBootMaster 顺序第 7 步）
- 初始化安装时：`server/setup.js:282`

**没有运行时动态注册新 provider 的 API 或机制**。要新增 provider 必须部署新的 `server/modules/search/<key>/` 目录后重启服务。

### 9.2 安全控制点

| 控制点 | 位置 | 说明 |
|--------|------|------|
| **文件系统边界** | `searchEngines.js:44` | `fs.readdir(path.join(WIKI.SERVERPATH, 'modules/search'))` 只扫描固定目录，不可通过配置改变 |
| **配置写入权限** | `search.graphql:21,31` | 搜索管理接口 `searchEngines` / `updateSearchEngines` 均要求 `@auth(requires: ["manage:system"])` |
| **激活失败回退** | `searchEngines.js:109-113` | `activate()` 抛 `SearchActivationFailed` 时，自动：①禁用当前引擎 ②启用 `db` 引擎 ③递归 `initEngine()` |
| **配置反序列化** | `search.js:50-53` | 配置值通过 `JSON.parse(value.value).v` 读取，`_.set` 写入，无 eval 风险 |
| **异常吞噬** | 各 provider `query()` 内 | `try/catch` 打 `WIKI.logger.warn` 但不抛异常，防止搜索故障拖垮整个应用 |

### 9.3 审计日志缺失

当前代码**没有专门的搜索安全审计日志**：
- 引擎切换、配置更新没有 audit log 事件
- 仅普通 `WIKI.logger.info/warn` 记录初始化与错误
- 错误类型定义了 `SearchActivationFailed (code 4002)` 和 `SearchGenericError (code 4001)`（`helpers/error.js:204-211`），但未用于审计

---

## 10. 复合字段与拆字段映射场景

### 10.1 输入侧：拆字段与清洗

页面对象的 `render` 字段（渲染后的完整 HTML）在进入搜索引擎前会经过 **两级拆解与清洗**：

**第一级：`cleanHTML()` 全局净化**（`pages.js:1152`，所有 provider 共享）
```js
static cleanHTML(rawHTML = '') {
  return striptags(rawHTML, [], ' ')          // 去 HTML 标签
    .replace(emojiRegex(), '')                // 去 emoji
    |> he.decode(...)                           // HTML 实体解码
    |> .replace(punctuationRegex, ' ')         // 标点转空格
    |> .replace(/\s\s+/g, ' ')                 // 合并空格
    |> .split(' ').filter(w => w.length > 1)   // 过滤单字符
    |> .join(' ').toLowerCase()                // 统一小写
}
```
结果存入 `page.safeContent`（`pages.js:340/451/743`），作为所有搜索引擎的 `content` 字段输入。

**第二级：Provider 内部进一步拆解**

| Provider | 拆字段逻辑 | 位置 |
|----------|-----------|------|
| **elasticsearch** | `buildSuggest(page)` 将 `title`/`description`/`safeContent` 按空格拆分为 token 数组，各自带权重（10/3/1），存入 `suggest` 复合字段 | `elasticsearch/engine.js:207-221` |
| **elasticsearch** | `buildTags(id)` 单独查关联表 `withGraphJoined('tags')`，将 tag.title 拼成字符串数组 | `elasticsearch/engine.js:198-203` |
| **postgres** | `setweight(to_tsvector(...), 'A') || setweight(..., 'B') || setweight(..., 'C')` 将三个字段分别加权后合并为单个 `TSVECTOR` | `postgres/engine.js:109-112` |
| **postgres** | 单独建 `pagesWords` 物化表，从 `pagesVector` 的三个字段中 `ts_stat` 抽取所有单词，用于拼写建议 | `postgres/engine.js:46-50` |

### 10.2 输出侧：字段归约

查询返回时，各 provider 将引擎原生字段映射回统一格式（详见第 4 章映射表）。典型差异：
- elasticsearch 8.x 从 `hits.hits[]._source` 取值，6.x/7.x 从 `body.hits.hits[]._source` 取值
- aws CloudSearch 从 `fields.title[0]` 取第一个值（数组）
- postgres 直接返回行对象

---

## 11. nested 与 has_child 结构化查询编排

### 11.1 当前实现状态

**代码中完全没有 nested / has_child / parent-child 等结构化查询**。所有 provider 的文档模型都是**扁平的**：

- `tags` 在 elasticsearch 中是 `text` 类型的字符串数组，不是 `nested` 类型
- 没有父子文档关联（如「页面」→「评论」一对多查询）
- GraphQL schema 中的 `PageSearchResult` 也是扁平结构（`id/title/description/path/locale`）

### 11.2 查询能力现状

| 查询类型 | 支持度 | 说明 |
|----------|--------|------|
| 关键词全文检索 | ✅ 全部 | `title/description/content` 多字段 OR 查询 |
| 精确值过滤 | ⚠️ 部分 | `locale/path` 仅 db/postgres 实现，es/algolia/azure/aws 忽略 |
| 标签过滤 | ❌ 无 | tags 仅 es 索引了但未提供查询入口，上层权限校验用 |
| 范围查询 | ❌ 无 | 无日期/数字范围查询接口 |
| 聚合统计 | ❌ 无 | 无 facet / aggregation 接口 |
| 嵌套对象查询 | ❌ 无 | 无 nested 字段，无 inner_hits |
| 父子关联查询 | ❌ 无 | 无 join 类型，无 has_child/has_parent |

### 11.3 扩展路径观察

如果未来要支持结构化查询，需要：
1. GraphQL schema 扩展 `PageQuery.search` 参数（增加 `tags:[String]`, `updatedFrom:Date` 等）
2. 统一查询 DSL 抽象层（当前 `opts` 只有 `locale/path`）
3. 各 provider 适配翻译到对应引擎的查询 DSL（es 的 `bool.filter`、algolia 的 `facetFilters` 等）

---

## 12. 结果排序与分页统一抽象

### 12.1 排序：无统一抽象，各引擎自行其是

| Provider | 排序逻辑 | 可配置性 |
|----------|---------|----------|
| **db** | 无 `ORDER BY`，返回数据库物理顺序 | ❌ 不可配置 |
| **postgres** | `ORDER BY ts_rank(tokens, query) DESC`，按文本相似度打分 | ❌ 不可配置 |
| **elasticsearch** | 引擎默认打分（BM25）+ 字段加权（`title^20` 等） | ❌ 不可配置 |
| **algolia** | 引擎默认打分 + `searchableAttributes` 顺序权重 | ❌ 不可配置 |
| **azure** | `scoringProfile: 'fieldWeights'` 预定义权重 | ❌ 不可配置 |
| **aws** | CloudSearch 默认打分 | ❌ 不可配置 |

**没有** `orderBy` 参数暴露给 GraphQL 搜索接口。GraphQL schema 中的 `PageQuery.search` 只有 `query/path/locale` 三个参数（`page.graphql:29-33`）。

### 12.2 分页：硬编码上限，无游标

分页能力非常基础：
- **硬编码上限**：`WIKI.config.search.maxHits = 100`（`server/app/data.yml:114`），仅 `db` provider 实际使用了这个值
- **固定页长**：外部搜索引擎硬编码 `size:50` / `top:50` / `hitsPerPage:50`
- **无翻页参数**：没有 `from` / `offset` / `page` 参数，`PageSearchResponse` 也没有 `totalPages` / `hasMore` 字段
- **无游标/快照**：不支持深度分页的 search_after / scroll

### 12.3 totalHits 统计问题

`totalHits` 在不同 provider 中语义不一致：
- postgres/db：返回 **过滤后** 的实际条数（因为 SQL 查询时就应用了 locale/path 过滤）
- es/algolia/azure/aws：返回 **引擎侧召回的总数**，不扣减上层权限过滤掉的结果
→ 用户看到的 `totalHits` 可能大于实际 `results.length`

---

## 13. 多 provider 并发查询、超时与部分失败 fallback

### 13.1 架构现状：单 active 引擎模型

**代码中完全没有多 provider 并发查询的设计**：
- `searchEngines` 表设计上允许多条 `isEnabled=true`，但 `initEngine()` 用 `findOne('isEnabled', true)` 只取第一个（`searchEngines.js:99`）
- 全局只有一个 `WIKI.data.searchEngine` 实例
- `updateSearchEngines` resolver 逻辑是「遍历 args.engines，如果有 isEnabled 则记录为 newActiveEngine」，实际上**最后一个 isEnabled 的会覆盖前面的**

### 13.2 单个 provider 的失败处理

**查询路径的错误处理**（所有 provider 的 `query()` 内部）：
```js
async query(q, opts) {
  try {
    // ... 调用搜索引擎 SDK
    return { results, suggestions, totalHits }
  } catch (err) {
    WIKI.logger.warn('Search Engine Error:', err)
    // ← 没有 return！隐式返回 undefined
  }
}
```
上层 resolver 接收 `undefined` 后走 fallback 分支（`page.js:53`）：
```js
if (WIKI.data.searchEngine) {
  const resp = await WIKI.data.searchEngine.query(...)
  // resp 可能是 undefined
  return { ...resp, results: _.filter(resp.results, ...) }
} else {
  return { results: [], suggestions: [], totalHits: 0 }
}
```
→ 实际表现：搜索引擎挂掉时，页面不会报错，但搜索结果永远为空。

**激活路径的错误处理**（`searchEngines.js:105-114`）：
```js
try {
  await WIKI.data.searchEngine.activate()
} catch (err) {
  if (err instanceof WIKI.Error.SearchActivationFailed) {
    await WIKI.models.searchEngines.query().patch({ isEnabled: false }).where('key', searchEngine.key)
    await WIKI.models.searchEngines.query().patch({ isEnabled: true }).where('key', 'db')
    await WIKI.models.searchEngines.initEngine()  // 递归重试
  }
  throw err
}
```
→ 这是**唯一的 fallback 机制**：激活失败强制切回 `db` 引擎。

### 13.3 超时控制缺失

所有 provider 调用均**未设置超时**：
- elasticsearch 客户端默认超时依赖 SDK 配置（未显式设置）
- algolia/azure/aws SDK 同理
- 没有 `AbortController` / `Promise.race([query, timeout])` 包装
→ 搜索引擎网络故障时，请求会挂到 TCP 超时（可能几分钟）

---

## 14. Cache Invalidation 策略

### 14.1 两层独立缓存体系

搜索模块涉及**两套互不关联的缓存**：

| 缓存层 | 用途 | 失效策略 | 代码位置 |
|--------|------|----------|----------|
| **页面渲染缓存** | 缓存 `page.render`（HTML），避免每次重复渲染 | ✅ 精细失效 | `server/models/pages.js:1047-1122` |
| **搜索引擎索引** | 搜索引擎自身的倒排索引 | ✅ 事件驱动更新 | 各 provider `created/updated/deleted/renamed` |

**注意：搜索查询结果本身不做应用层缓存**，每次查询直接打搜索引擎。

### 14.2 页面渲染缓存的失效机制

缓存文件：`cache/${page.hash}.bin`（avsc 二进制序列化）

**失效触发点**（全部是 `deletePageFromCache(hash)` + emit 事件）：
- 创建页面：不涉及（新页面无缓存）
- 更新页面：`pages.js:447` → `deletePageFromCache(page.hash)` + `emit('deletePageFromCache', page.hash)`
- 移动页面：`pages.js:735-736` → 删旧 hash 缓存
- 删除页面：`pages.js:814-815` → 删 hash 缓存
- 手动 flush：`PageMutation.flushCache` → `fs.emptyDir(cache)`

**HA 多节点同步**（`pages.js:1163-1173`）：
```js
static subscribeToEvents() {
  WIKI.events.inbound.on('deletePageFromCache', hash => {
    WIKI.models.pages.deletePageFromCache(hash)
  })
  WIKI.events.inbound.on('flushCache', () => {
    WIKI.models.pages.flushCache()
  })
}
```
→ 通过事件总线跨节点同步失效。

### 14.3 搜索引擎索引的失效机制

索引更新完全**事件驱动**，与页面生命周期同步：
- `created(page)` → 新增文档
- `updated(page)` → 全量覆盖文档
- `renamed(page)` → 删旧 + 建新（es/algolia/azure/aws）或 UPDATE（postgres）
- `deleted(page)` → 删除文档
- 管理员 `rebuildIndex` → 删全索引 + 流式重建

**注意**：索引更新与页面 DB 事务**不保证原子性**。如果 DB 事务成功但搜索引擎调用失败，会产生不一致。当前无补偿/重试机制。

---

## 15. Query Rewriting 与同义词扩展

### 15.1 Query Rewriting

**只有 postgres provider 做了有限的查询重写**，其余引擎直接透传用户输入：

**postgres：`pg-tsquery` 库**（`postgres/engine.js:1`）
```js
const tsquery = require('pg-tsquery')()
// ...
let qryParams = [this.config.dictLanguage, tsquery(q), `%${q.toLowerCase()}%`]
```
`pg-tsquery` 的作用：
- 转义特殊字符（`!` `&` `|` `:` `*` `(` `)` 等）
- 将用户输入的空格转换为 `&`（AND 操作）
- 支持前缀匹配 `foo:*`
- 防止语法错误导致 SQL 异常

**elasticsearch**：用 `simple_query_string` 查询类型（`elasticsearch/engine.js:154`），自带有限的操作符解析（`+` 强制、`-` 排除、`"` 短语、`*` 前缀），但这是 es 引擎原生能力，不是应用层 query rewriting。

**其余引擎**（algolia/azure/aws/db）：`q` 直接传给 SDK，无任何改写。

### 15.2 同义词扩展

**所有 provider 均未实现同义词扩展**：
- 没有配置同义词词典的 UI 入口（definition.yml 中无相关 props）
- elasticsearch 没有配置 `synonym` token filter
- postgres 没有创建自定义同义词字典（`CREATE TEXT SEARCH DICTIONARY`）
- 没有应用层同义词替换逻辑

### 15.3 查询预处理一览

| Provider | 查询预处理 |
|----------|-----------|
| **db** | 无，直接 `%${q}%` 拼接 |
| **postgres** | `tsquery(q)` 转义 + `%${q.toLowerCase()}%` path 模糊匹配 |
| **elasticsearch** | 前后加 `*` 通配（`*${q}*`）+ `simple_query_string` 解析 |
| **algolia** | 无，直接 `q` 透传 |
| **azure** | 无，直接 `q` 透传 |
| **aws** | 无，直接 `q` 透传 + `partial:true` |

---

## 16. 各 adapter 权限过滤抽出统一拦截器

### 16.1 现状：分散重复代码，未抽出拦截器

权限过滤逻辑**在 resolver 层多处重复**，没有统一为中间件/拦截器：

| 位置 | 代码模式 |
|------|---------|
| `page.js:57-62` (search) | `_.filter(resp.results, r => WIKI.auth.checkAccess(user, ['read:pages'], {path, locale, tags}))` |
| `page.js:135-139` (list) | `_.filter(results, r => WIKI.auth.checkAccess(user, ['read:pages'], {path, locale}))` |
| `page.js:208-211` (tags) | 同上 |
| `page.js:238-242` (searchTags) | 同上 |
| `page.js:285-289` (tree) | 同上 |
| `page.js:328-330` (links) | 双重 checkAccess（源页面 + 目标页面） |

每次的 `_.filter` + `WIKI.auth.checkAccess` 调用都是**复制粘贴**的模式。

### 16.2 checkAccess 内部实现

`WIKI.auth.checkAccess(user, permissions, page)`（`auth.js:221-237`）三层检查：
```
1. 系统管理员豁免：user.permissions 包含 'manage:system' → return true
2. 全局权限检查：_.intersection(user.permissions, permissions).length < 1 → return false
3. 页面规则检查：遍历用户所属 groups 的 pageRules，按路径前缀 / 标签匹配
```

### 16.3 统一拦截器的可行演进方向

当前架构可抽出一个 `withPageAccessFilter` 高阶函数：
```js
// 伪代码：统一拦截器
function withPageAccessFilter(fn, permission = 'read:pages') {
  return async (obj, args, context, info) => {
    const result = await fn(obj, args, context, info)
    if (result && Array.isArray(result.results)) {
      result.results = _.filter(result.results, r =>
        WIKI.auth.checkAccess(context.req.user, [permission], {
          path: r.path, locale: r.locale, tags: r.tags
        })
      )
      // 可选：修正 totalHits = result.results.length
    }
    return result
  }
}
```

但需注意：
- 目前 `checkAccess` 是同步函数，性能开销可接受
- 对于 es/algolia 等不支持 locale/path 下推的引擎，filter 之前的 results 可能包含大量用户不可见的页面，需要在拦截器中**同步修正 totalHits**，否则前端分页统计会出错

---

## 17. 补充关键代码坐标

| 关注点 | 文件 : 行号 |
|--------|-------------|
| 搜索配置默认值（maxHits） | `server/app/data.yml:114` |
| cleanHTML 内容净化 | `server/models/pages.js:1152` |
| checkAccess 权限校验实现 | `server/core/auth.js:221` |
| elasticsearch buildSuggest 复合字段 | `server/modules/search/elasticsearch/engine.js:207` |
| elasticsearch buildTags 关联查询 | `server/modules/search/elasticsearch/engine.js:198` |
| postgres tsquery 重写 | `server/modules/search/postgres/engine.js:1,70` |
| SearchActivationFailed 错误定义 | `server/helpers/error.js:204` |
| 页面缓存失效事件订阅 | `server/models/pages.js:1166` |
| 启动时 provider 扫描顺序 | `server/core/kernel.js:72-86` |
| GraphQL 搜索接口参数定义 | `server/graph/schemas/page.graphql:29-33` |
| GraphQL 搜索接口类型定义 | `server/graph/schemas/page.graphql:257-269` |

---

## 18. 演进观察（补充）

7. **安全审计薄弱**：引擎切换、配置修改无审计日志；`query()` 错误仅打 warn 不抛异常，静默失败可能掩盖攻击。
8. **无超时控制**：所有搜索引擎调用均未设置超时，网络故障会长时间阻塞请求。
9. **单引擎模型限制**：架构上不支持多引擎组合（如主搜+备搜、混合召回），查询失败时只能返回空结果。
10. **排序/分页能力原始**：无自定义排序、无深度分页、totalHits 语义不统一。
11. **同义词/查询重写缺失**：除 postgres 基础转义外，无查询理解层（QUL），搜索质量完全依赖搜索引擎原生能力。
12. **权限过滤重复代码**：6 处 resolver 重复相同的 `_.filter + checkAccess` 模式，未抽出统一拦截器，新增查询接口时易遗漏。
13. **缓存与索引不一致风险**：页面渲染缓存与搜索引擎索引是两套独立失效机制，无分布式事务保证，极端情况下搜索结果链接可能 404。
