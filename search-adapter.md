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

## 17. ProviderRegistry 审计加权限分级

### 17.1 当前注册表实现状态

代码中**没有独立的 ProviderRegistry 类**，注册与加载逻辑散落在 `server/models/searchEngines.js` 一个 Model 文件里：
- 「扫描 + 入库」在 `refreshSearchEnginesFromDisk()`（L38）
- 「加载 + 激活」在 `initEngine()`（L98）
- 「内存元信息」挂在 `WIKI.data.searchEngines`（纯数组，无索引/版本/生命周期钩子）

这与典型的 Registry 模式（注册表类、注册/注销钩子、实例生命周期管理）有较大差距。

### 17.2 现有权限分级

`searchEngines` 表 schema 中预留了 `level` 字段（`searchEngines.js:25`），但**整个代码库无任何地方读写此字段**。目前唯一的权限控制在 GraphQL 层：

| 操作 | GraphQL 权限要求 | 位置 |
|------|-----------------|------|
| 查看引擎配置列表 | `manage:system` | `search.graphql:21` |
| 更新引擎配置/切换激活 | `manage:system` | `search.graphql:31` |
| 重建索引 | `manage:system` | `search.graphql:33` |
| 执行搜索查询 | 无（公共接口） | `page.graphql:29-33` |

搜索查询接口 `PageQuery.search` **完全没有权限装饰器**，任何登录/未登录用户都可调用，权限靠 resolver 内部 `_.filter(checkAccess)` 后置过滤（详见第 16 章）。

### 17.3 审计日志现状

**完全没有结构化审计日志**：
- `refreshSearchEnginesFromDisk()` 只有 `logger.info('Loaded N new search engines')`
- `initEngine()` 失败只有 `logger.warn(err)`
- `updateSearchEngines` resolver（`search.js:41`）成功时返回 `generateSuccess(...)`，但**不记录是谁、在什么时间、改了什么**
- `rebuildIndex` 同理，无审计

数据库 `searchEngines` 表也**没有 `updatedBy`/`updatedAt` 字段**（只有 `key/isEnabled/level/config`），无法事后追溯配置变更。

### 17.4 引入 ProviderRegistry + 审计 + 分级权限的演进建议

1. **Registry 抽象**：抽出 `ProviderRegistry` 类，暴露 `register/unregister/list/get` 方法，替代直接操作 `WIKI.data.searchEngines` 数组
2. **权限分级落地**：激活 `level` 字段，定义 `core（内置不可禁用）/ official（官方维护）/ community（第三方）` 三级；非 core 引擎的 activate 需额外安全检查（如代码签名校验、沙箱执行）
3. **审计事件**：在 Registry 的 `activate/deactivate/configure/rebuild` 方法中 emit `search:audit` 事件，写入结构化审计表（包含 user、timestamp、action、diff、clientIP）
4. **查询限流**：对 `PageQuery.search` 公共接口增加 `@rateLimit`（目前无任何限流，搜索接口可被滥用打垮搜索引擎）

---

## 18. FieldMapper 复合字段冲突优先级

### 18.1 当前字段映射机制

**没有独立的 FieldMapper 抽象**。字段映射以**每个 provider 内部硬编码**的方式存在，散落在：
- `init()` 中的索引创建/mapping 定义（es 的 `indices.create`、azure 的 `createIndex`、aws 的 `defineIndexField`）
- `created/updated/rebuild` 中的文档组装（`buildDocument` 类函数）

各 provider 对同一逻辑字段的定义是**平行且独立**的，没有集中映射表。

### 18.2 复合字段拆解与隐式优先级

以「title/description/content」三个文本字段为例，它们在不同 provider 中存在**隐式的字段优先级冲突**：

| Provider | 字段优先级表达机制 | 冲突处理方式 |
|----------|-------------------|-------------|
| **elasticsearch 6/7.x** | mapping 中 `properties.title.boost = 20` 静态声明 | 8.x 不支持 mapping boost，静默失效，改在 query 层 `fields: ['title^20']` 重新声明 |
| **postgres** | `setweight(to_tsvector(title), 'A')` 写入 `TSVECTOR` | A/B/C/D 四级权重，无冲突 |
| **algolia** | `searchableAttributes: ['title', 'description', 'content']` 数组顺序即权重 | 靠前优先，无冲突 |
| **azure** | `scoringProfiles.fieldWeights: { title:4, description:3, content:1 }` | 与 `searchFields` 声明的字段列表需一致，不一致时 azure 会忽略权重 |
| **db** | SQL `title ILIKE OR description ILIKE OR content ILIKE` | 三个字段 OR 平等，无权重概念 |

### 18.3 实际冲突场景

代码中已存在但未被显式处理的冲突：

1. **elasticsearch 版本冲突**（`elasticsearch/engine.js:117-174`）：
   - 6.x/7.x 在 `mappings.properties` 静态声明 `boost`
   - 8.x 客户端去掉了 mapping boost 支持，代码在 8.x 分支里**依然尝试设置**，但 es 8.x 会忽略该字段
   - 代码通过 `fields: ['title^20', 'description^3', 'content^1', 'tags^8']` **在 query 层做了二次补偿**，但这两份权重声明是**分散维护**的，改一处容易漏改另一处

2. **tags 字段的优先级缺失**：
   - elasticsearch 索引了 `tags` 并给了 `boost:8`，但 postgres/algolia/azure/aws/db **完全没有索引 tags 字段**
   - 同一逻辑字段在不同引擎中的权重差异可能导致「换引擎后搜索结果排序剧变」

3. **suggest 字段定义冲突**：
   - elasticsearch 把 suggest 定义为 `completion` 类型（专门的 suggest 数据结构）
   - postgres 用独立的 `pagesWords` 表 + `similarity()` 函数实现
   - azure/aws 走独立的 `/autocomplete` HTTP 接口
   - 三者在同一搜索请求中触发条件不一致（azure/aws 是「结果不足 5 条 fallback」，elasticsearch 是「每次必查」）

### 18.4 FieldMapper 集中化的演进建议

1. 抽出 `FieldMapper` 配置层：`{ logicalField -> { type, analyzers, weights: { elasticsearch, postgres, algolia }, suggest: true/false } }`
2. Provider 初始化时从 FieldMapper 读取自己的字段定义，不再硬编码
3. 定义冲突检测规则：同一逻辑字段在不同引擎的权重差异超过阈值（如 2x）时警告
4. suggest 字段统一触发策略（是每次查还是结果不足时 fallback）由 FieldMapper 配置控制

---

## 19. 深嵌套查询性能上限

### 19.1 当前状态：无深嵌套

如第 11 章所述，当前所有 provider 的文档模型都是**扁平的**：
- 没有 nested / object / join / parent-child 类型字段
- `tags` 在 es 中是 `text` 字符串数组，不是 `nested`
- GraphQL 查询接口只有扁平的 `query/path/locale` 三个参数

因此深嵌套查询的性能问题在当前代码中**尚未暴露**，但这是一个重要的架构约束边界。

### 19.2 如果引入深嵌套查询的性能上限预测

基于各 provider 的原生能力分析：

| Provider | nested 查询支持 | 预期性能上限 | 主要瓶颈 |
|----------|----------------|-------------|---------|
| **elasticsearch** | ✅ 原生 `nested` 类型 + `nested` query | 中等：嵌套深度 ≤ 3，每个 nested 字段 cardinality < 1000 时性能可接受；超过后 `inner_hits` 开销急剧上升 | nested query 执行时是子文档独立打分再聚合，内存开销与嵌套文档数成正比 |
| **postgres** | ⚠️ 需用 `jsonb_path_ops` GIN 索引模拟 | 较低：JSONB 嵌套查询在层级 > 2 且数据量 > 100 万行时明显变慢 | jsonb 解析 + 倒排合并，无法利用 tsvector 权重 |
| **algolia** | ❌ 不支持 nested object 查询 | N/A | 只能扁平字段，需在应用层做 join |
| **azure** | ❌ 不支持 nested | N/A | 同上，Edm.ComplexType 只能过滤不能全文 |
| **aws CloudSearch** | ❌ 不支持 nested | N/A | 仅 literal/text/int 等标量数组 |
| **db (basic)** | ❌ 不支持 | N/A | 纯 LIKE 查询，无任何结构化查询能力 |

### 19.3 硬限制参考（按各引擎官方文档）

- **elasticsearch nested 字段上限**：默认 `index.mapping.nested_fields.limit = 50`，`index.mapping.nested_objects.limit = 10000`
- **elasticsearch inner_hits 性能**：单次 inner_hits 返回 > 100 条时建议改走应用层 join
- **postgres jsonb 嵌套**：超过 5 层嵌套后 `@>` 操作符的选择性下降明显，需配合 expression index
- **通用教训**：一旦引入 nested 查询，**分页成本将从 O(N) 上升到 O(N × avg_nested_count)**，当前硬编码的 `size:50` 需要重新评估

### 19.4 演进约束建议

如果未来需要支持结构化嵌套查询：
1. 优先在应用层做**反规范化（denormalization）**，将嵌套字段拍平为前缀字段（如 `tags.0.title → tag_0_title`），避免走引擎原生 nested
2. 如必须用 nested，限制最大嵌套深度为 2，单个文档嵌套对象数 ≤ 100
3. nested 查询在 FieldMapper 中声明为「高级特性」，db/algolia/azure/aws provider 可抛出 `SearchFeatureNotSupported` 异常

---

## 20. 不同 Provider 排序稳定性差异

### 20.1 排序稳定性定义

排序稳定性 = **相同查询 + 相同文档集 + 相同引擎配置，多次查询结果顺序是否一致**。对分页、UI 体验和结果可复现性至关重要。

### 20.2 各 Provider 的排序稳定性分析

| Provider | 打分算法 | 排序稳定性 | 破坏稳定性的因素 |
|----------|---------|-----------|-----------------|
| **elasticsearch** | BM25 + 字段加权 | ⚠️ **有条件稳定** | ① 分片数 > 1 时，BM25 的 IDF 是分片级统计的，不同分片 IDF 不同 → 同文档在不同分片打分不同；② 8.x 默认 `track_total_hits: false` 可能跳过部分分片 |
| **postgres** | `ts_rank` / `ts_rank_cd` | ✅ **稳定** | 基于全局统计的 tsvector，单库单实例，排序完全确定 |
| **algolia** | 自定义 Tie-breaking + BM25 | ✅ **稳定** | SaaS 平台内部做了全局统计同步，相同查询排序一致 |
| **azure** | BM25 + `scoringProfile` | ⚠️ **有条件稳定** | 免费层服务可能跨副本漂移；相同搜索服务实例内稳定 |
| **aws CloudSearch** | BM25 变体 | ✅ **稳定** | 实例稳定后排序一致 |
| **db (basic)** | 无排序（物理顺序） | ❌ **不稳定** | `SELECT * FROM pages WHERE ... ILIKE` 无 ORDER BY，返回顺序取决于数据库页分裂、VACUUM、并发写入等不可控因素 |

### 20.3 代码中已有的稳定性隐患

1. **elasticsearch 未启用 `track_scores`/`search_after`**：
   - 代码查询（`elasticsearch/engine.js:154`）使用 `simple_query_string` 但**未设置 `track_scores: true`**，在某些 filter-only 查询场景下引擎可能跳过打分
   - 未设置 `preference=_primary_first` 或固定分片路由，多副本环境下同一查询可能路由到不同副本，IDF 统计有细微差异

2. **db 引擎完全无排序**：
   - `db/engine.js:40` 的查询 `select(...).from('pages').where(...).limit(...)` **没有 ORDER BY 子句**
   - 这意味着同样的关键词搜索两次，结果顺序可能完全不同，且前端分页会出现重复/遗漏

3. **无统一二级排序键（tie-breaker）**：
   - 所有 provider 在打分相同的情况下都**没有配置 tie-breaker**（如 `_score` 相同则按 `updatedAt desc` 再按 `path asc`）
   - 这导致评分相同的文档顺序在各引擎中是随机的（依赖写入顺序/文档 ID）

### 20.4 排序统一抽象的演进建议

1. 抽出 `SortNormalizer` 层，为所有 provider 注入默认 tie-breaker：`_score DESC, updatedAt DESC, path ASC`
2. elasticsearch 强制加 `preference` 参数（用 user.id 或 session.id 哈希），保证同一用户看到的结果一致
3. db 引擎补 `ORDER BY (CASE WHEN title ILIKE ... THEN 1 ELSE 0 END) DESC, updatedAt DESC` 显式排序
4. 对外暴露的 `PageSearchResult` 增加 `score` 字段，便于前端调试和用户反馈排序问题

---

## 21. 多 Provider 并发查询的超时与部分失败回填占位

### 21.1 当前状态：单引擎无并发

第 13 章已详细说明，当前架构是**单 active 引擎模型**：
- 全局只有一个 `WIKI.data.searchEngine`
- 查询只打一个引擎
- 失败则返回空结果（无 fallback）

没有多 provider 并发查询、没有超时控制、没有占位回填。

### 21.2 引入多 Provider 并发的架构设计空间

如果未来演进为多引擎混合召回（如「es 做主搜 + algolia 做拼写纠错 + db 做保底」），需要解决三个问题：超时、部分失败、结果合并。

### 21.3 超时控制设计

```js
// 伪代码：带超时的并发查询编排
async function multiProviderQuery(q, opts) {
  const providers = [
    { key: 'elasticsearch', timeout: 500,  weight: 1.0, critical: false },
    { key: 'algolia',       timeout: 800,  weight: 0.8, critical: false },
    { key: 'db',            timeout: 2000, weight: 0.3, critical: true  }
  ]

  const results = await Promise.all(providers.map(async p => {
    try {
      return await Promise.race([
        WIKI.data.providers[p.key].query(q, opts),
        new Promise((_, rej) => setTimeout(() => rej(new Error('timeout')), p.timeout))
      ])
    } catch (err) {
      WIKI.logger.warn(`[SEARCH/${p.key}] failed:`, err.message)
      return p.critical ? getPlaceholderResult(q, opts) : null
    }
  }))

  return mergeResults(results, providers)
}
```

### 21.4 占位结果（Placeholder）设计

占位结果是**保底结果**，用于 critical provider 失败时防止 UI 出现空白。可选占位策略：

| 占位策略 | 实现方式 | 适用场景 |
|---------|---------|---------|
| **空结果占位** | `{ results: [], suggestions: [], totalHits: 0, partial: true }` | UI 友好，提示「部分搜索服务不可用」 |
| **数据库保底** | 调用 `db` provider 的 `query()`（SQL LIKE 查询） | 牺牲性能换可用性，适合用户可接受慢查询的场景 |
| **热门结果缓存** | 返回最近 N 条热门/编辑推荐页面（从 Redis 或内存缓存取） | 不保证与 query 相关，但保证 UI 有内容 |
| **上次结果复用** | 同一 user + query 的上次成功结果（带 TTL） | 个性化场景，短时间内重复查询可用 |

当前代码中**没有任何占位机制**，provider 返回 `undefined` 时前端看到的就是空白搜索结果。

### 21.5 部分失败的用户可见性

`PageSearchResponse` GraphQL 类型应增加：
```graphql
type PageSearchResponse {
  results: [PageSearchResult]!
  suggestions: [String]!
  totalHits: Int!
  partial: Boolean!       # 是否部分 provider 失败
  failedProviders: [String]  # 失败的 provider key 列表，前端可展示提示
}
```

当前类型（`page.graphql:257`）中没有这两个字段。

---

## 22. Cache Eviction：LRU 与 LFU 选型

### 22.1 当前缓存的 Eviction 机制

当前页面渲染缓存的 eviction **完全缺失**：

| 特性 | 当前实现 | 位置 |
|------|---------|------|
| 写入 | `fs.outputFile(cache/${hash}.bin, avsc.encode(page))` | `pages.js:1052-1077` |
| 读取 | `fs.readFile(cache/${hash}.bin)`，不存在返回 `false` | `pages.js:1086-1106` |
| 手动失效 | `fs.remove(cache/${hash}.bin)`（单条）/ `fs.emptyDir(cache)`（全部） | `pages.js:1114/1121` |
| 容量限制 | ❌ 无 | — |
| 自动 eviction | ❌ 无 | — |
| TTL | ❌ 无 | — |
| 访问统计 | ❌ 无 | — |

缓存文件会**无限增长**，直到管理员手动调用 `flushCache` 或磁盘写满。没有任何基于容量/时间的淘汰。

### 22.2 LRU vs LFU 选型分析

针对 Wiki.js 这种文档型应用的页面渲染缓存场景：

| 维度 | LRU（最近最少使用） | LFU（最不经常使用） | W-TinyLFU（现代混合算法） |
|------|---------------------|---------------------|--------------------------|
| 核心思想 | 淘汰最久没访问的 | 淘汰访问频率最低的 | 频率+时效的加权混合，Caffeine/Tk-Cache 默认 |
| 命中率预期 | 中等 | 较高（Wiki 有稳定的「常读页」集） | 最高 |
| 对「一次性批量访问」的抗污染 | ❌ 差（批量扫描会把热点挤出） | ✅ 好（频率低的不会挤掉高频） | ✅ 好（窗口衰减机制） |
| 冷启动问题 | 无 | ⚠️ 新页面频率为 0，需等多次访问才被保留 | ✅ 小 LRU 窗口处理新项 |
| 内存开销 | 低（链表） | 中（频率计数器） | 中-高（频率草图 + LRU 窗口） |
| 实现复杂度 | 低（`Map` 即 LRU） | 中 | 高 |
| Wiki 场景适配 | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

### 22.3 搜索查询缓存的 Eviction 额外考虑

如果未来引入「搜索查询 → 结果」缓存（当前没有），还需要考虑：
- 搜索结果的时效性比页面渲染更强（新页面创建后应立即可搜到）
- 因此查询缓存的 TTL 应更短（如 5~30 秒），且 `created/updated/deleted` 事件触发时应做前缀失效（所有包含该页面可能命中的关键词的查询缓存都应失效）
- 纯 LRU/LFU 不足以处理这种事件驱动的失效，需配合「Tag-based Invalidation」（Tag = page.hash）

### 22.4 演进建议

1. **短期**：为文件缓存加基于 `mtime` 的 cron 清理（如每天扫一次，删除 > 7 天未访问的文件），先用操作系统文件系统元信息解决无限增长问题
2. **中期**：抽出 `PageCache` 类替代直接 `fs` 调用，实现 LFU eviction（容量阈值 = 磁盘占用或文件数），访问时 `utimes` 更新 mtime，eviction 时按 atime/mtime 排序删除
3. **长期**：如命中率仍不理想，切换到 W-TinyLFU（可引入 `lru-cache` v7+ 或 `tiny-lfu` npm 包），并考虑将缓存从磁盘迁移到 Redis 以支持多节点共享 eviction

---

## 23. SynonymExpander 同义词环图检测

### 23.1 当前同义词能力状态

第 15 章已确认：**所有 provider 均未实现同义词扩展**，具体表现为：
- definition.yml 中无同义词词典配置项
- elasticsearch 未配置 `synonym` / `synonym_graph` token filter
- postgres 未创建自定义同义词字典
- 应用层无查询改写（同义词替换）逻辑

### 23.2 同义词环（Synonym Cycle）的定义与危害

同义词环指词典中存在 `A → B, B → C, C → A` 或 `A → B, B → A` 之类的循环引用。如果不对环做检测，会导致：
- **查询展开爆炸**：`A → B → C → A → ...` 无限递归
- **索引体积膨胀**：文档被分词时每个词被循环替换，term 数急剧增大
- **打分异常**：同一词在查询中被多次加权，BM25 打分失真

### 23.3 环图检测算法

同义词词典本质是**有向图**，每条同义词规则 `A, B, C`（等价组）是 A↔B、B↔C、A↔C 的双向边；`A => B`（单向替换）是 A→B 的单向边。

环检测的实现：
```js
// 伪代码：SynonymExpander 环检测
function detectSynonymCycles(rules) {
  // 1. 构建邻接表
  const adj = new Map()
  for (const rule of rules) {
    for (const [from, to] of rule.expandToEdges()) {
      if (!adj.has(from)) adj.set(from, [])
      adj.get(from).push(to)
    }
  }
  // 2. DFS 三色法检测环（White=未访问, Gray=访问中, Black=已完成）
  const color = new Map()
  const WHITE = 0, GRAY = 1, BLACK = 2
  function dfs(node) {
    color.set(node, GRAY)
    for (const next of (adj.get(node) || [])) {
      if (color.get(next) === GRAY) throw new Error(`Synonym cycle detected: ${node} -> ${next}`)
      if (color.get(next) === WHITE) dfs(next)
    }
    color.set(node, BLACK)
  }
  for (const node of adj.keys()) {
    if (color.get(node) === WHITE) dfs(node)
  }
}
```

### 23.4 应用层 vs 引擎层同义词的边界

| 层级 | 实现位置 | 环检测时机 | 适用场景 |
|------|---------|-----------|---------|
| **应用层**（SynonymExpander） | 查询时在 `query()` 之前对 `q` 做展开，文档写入时对 `safeContent` 做展开 | 服务启动加载词典时 + 管理员更新词典时 | 简单等价替换（如「CPU = 中央处理器」） |
| **引擎层**（es token filter / pg 词典） | 引擎配置中定义，分词时生效 | 引擎加载配置时（es 启动、postgres `CREATE TEXT SEARCH CONFIGURATION`） | 复杂形态还原、多语言、与 analyzer 深度耦合 |

### 23.5 演进建议

1. **起步阶段**：在应用层实现 `SynonymExpander`，支持简单等价组，启动时做 DFS 环检测，通过后才能激活
2. **进阶阶段**：为 elasticsearch provider 增加 `synonyms` 配置项，将同义词写入 es 的 `synonyms.txt` 并配置 `synonym_graph` filter；同时在应用层保留环检测作为前置校验（es 自己不检测环，只会日志警告）
3. **注意**：环检测不能只在管理员保存词典时做——`refreshSearchEnginesFromDisk()` 从磁盘加载时也必须重跑检测，防止手工编辑 definition.yml 引入环

---

## 24. PermissionInterceptor 与 Adapter 内部 RLS 双重生效边界

### 24.1 当前架构：只有应用层拦截

第 16 章已详细说明：
- 权限过滤在 **resolver 层**通过 `_.filter(results, r => checkAccess(user, ['read:pages'], {path, locale, tags}))` 后置执行
- 各 provider **内部没有任何行级安全（RLS）** 实现
- 唯一例外：`db` provider 查询时加了 `andWhere('isPrivate', false)` 和 `andWhere('isPublished', true)`（`db/engine.js:39-41`），这是**内容属性过滤**而非**用户权限过滤**

### 24.2 RLS（行级安全）的定义

RLS 指在搜索引擎/数据库**查询层**就把用户不可见的文档过滤掉，而不是召回后再过滤。典型实现：
- PostgreSQL：`CREATE POLICY page_policy ON pages FOR SELECT USING (check_access(current_user, path, locale))`
- Elasticsearch：查询时注入 `bool.filter.term`，结合 x-pack 的 Document Level Security (DLS)
- Algolia：`filters: "NOT path:private/*"` 作为强制过滤条件

### 24.3 双重生效的边界问题

如果同时启用「应用层 PermissionInterceptor 后置过滤」和「Provider 内部 RLS 前置过滤」，需要明确边界：

| 维度 | RLS（引擎层） | PermissionInterceptor（应用层） |
|------|--------------|-------------------------------|
| **执行顺序** | 先（查询时） | 后（结果返回时） |
| **过滤粒度** | 粗（路径前缀、locale、是否私有） | 细（用户组 pageRules、标签匹配、时间窗口） |
| **对 totalHits 的影响** | ✅ 返回的就是过滤后的真实 totalHits | ❌ totalHits 仍然是召回总数，需修正 |
| **对分页的影响** | ✅ 分页准确（因为过滤在 limit/offset 之前） | ❌ 分页可能失效（前 50 条被过滤掉 40 条，用户只看到 10 条） |
| **性能影响** | ✅ 低（引擎索引高效过滤） | ⚠️ 高（召回越多过滤越慢，且已浪费引擎计算资源） |
| **能否覆盖所有权限维度** | ❌ 不能（用户组动态规则、标签复杂表达式难以推到引擎 DSL） | ✅ 能（任意 JS 逻辑） |

### 24.4 当前代码中的边界风险

当前架构没有 RLS，因此存在 **「召回 + 后置过滤」** 的天然缺陷：

1. **分页错位**：es 返回前 50 条结果，经过滤后可能只剩 5 条用户可见，前端以为只有 5 条结果
2. **totalHits 虚高**：`totalHits = 1000`，但用户实际只能看到其中 50 条
3. **性能浪费**：用户只能看 `/engineering/*`，但 es 依然在全索引 100 万文档上打分排序，90% 的计算浪费在用户不可见的文档上

### 24.5 双重生效的正确分层设计

```
查询请求
  │
  ▼
[1] PermissionInterceptor.preQuery(user, query, opts)
  │  ├─ 提取用户可见的路径前缀集合（如 ['/engineering/*', '/public/*']）
  │  ├─ 提取用户可见的 locale 集合
  │  └─ 返回 filterContext = { allowedPaths, allowedLocales, allowedTags }
  │
  ▼
[2] Provider.query(q, opts, filterContext)  ◄── Provider 内部 RLS 下推
  │  ├─ es: filterContext.allowedPaths 注入到 bool.filter
  │  ├─ postgres: filterContext 拼接 SQL AND
  │  ├─ algolia: filterContext 转为 facetFilters
  │  └─ 返回 { results, suggestions, totalHits }
  │
  ▼
[3] PermissionInterceptor.postQuery(user, results, filterContext)
     └─ 对 RLS 无法覆盖的维度（复杂标签规则、用户组动态规则）做最终精确过滤
        └─ 修正 totalHits = results.length
```

关键原则：
- **RLS 只负责过滤粗粒度、引擎 DSL 能表达的规则**（路径前缀、locale、isPrivate）
- **Interceptor 负责精过滤**（细粒度标签、AND/OR 复杂组合、时间窗口）
- **两层同时启用时，Interceptor 必须始终跑**（作为安全兜底，防止 RLS 翻译遗漏导致越权）
- **绝对不能只依赖 RLS**：RLS 翻译逻辑有 bug 时会造成信息泄露，应用层 Interceptor 是最后一道防线

### 24.6 演进路径

1. **第一步**：抽出 `PermissionInterceptor` 高阶函数（替代 6 处重复的 `_.filter`），并在 post-filter 后修正 `totalHits`
2. **第二步**：在 provider 的 `query()` 入参中增加 `filterContext`，`db` 和 `postgres` 两个 SQL 系引擎先实现 RLS 下推（它们已经在用 locale/path 过滤了，只是目前是硬编码不是从 filterContext 读取）
3. **第三步**：elasticsearch provider 实现将 `allowedPaths` 翻译为 `wildcard` query，`allowedLocales` 翻译为 `terms` query
4. **第四步**：审计——当 Interceptor 过滤掉的条数 > 总召回数的 50% 时，打 warn 日志提示 RLS 可能未生效，需要排查

---

## 25. 再补充关键代码坐标

| 关注点 | 文件 : 行号 |
|--------|-------------|
| 搜索引擎 Model（含未使用的 level 字段） | `server/models/searchEngines.js:1-125` |
| searchEngines GraphQL 管理接口权限 | `server/graph/schemas/search.graphql:21,31,33` |
| updateSearchEngines resolver（无审计） | `server/graph/resolvers/search.js:41-70` |
| 页面缓存 save/get/delete/flush | `server/models/pages.js:1047-1123` |
| 页面缓存 avsc schema 定义 | `server/models/pages.js:137-163` |
| algolia searchableAttributes 字段优先级 | `server/modules/search/algolia/engine.js:26-30` |
| elasticsearch 简单查询字符串（无 track_scores） | `server/modules/search/elasticsearch/engine.js:154` |
| db 引擎查询（无 ORDER BY） | `server/modules/search/db/engine.js:40-64` |
| db 引擎硬编码 isPrivate/isPublished 过滤 | `server/modules/search/db/engine.js:39-41` |

---

## 26. RBAC 自定义权限组合的扩展点

### 26.1 当前 RBAC 架构三层模型

Wiki.js 的权限系统是 **三层 RBAC 叠加**，每层都有可扩展点：

```
┌─────────────────────────────────────────────────┐
│ Layer 1: 全局权限 (Global Permissions)           │
│   perms: ['read:pages', 'write:pages', ...]      │
│   存储: groups.permissions (JSON 数组)           │
├─────────────────────────────────────────────────┤
│ Layer 2: 页面规则 (Page Rules)                   │
│   rules: [{match, path, deny, roles, locales}]  │
│   存储: groups.pageRules (JSON 数组)             │
├─────────────────────────────────────────────────┤
│ Layer 3: 系统管理员豁免                           │
│   perms.includes('manage:system') → 直接放行      │
└─────────────────────────────────────────────────┘
```

### 26.2 权限字符串硬编码，无集中定义

权限字符串（`manage:system`、`read:pages`、`write:pages` 等）在代码中**完全硬编码**，没有集中定义文件或枚举：

| 出现位置 | 权限使用方式 |
|----------|-------------|
| `auth.js:225,335,347` | `if (_.includes(userPermissions, 'manage:system'))` 直接字符串比较 |
| `auth.js:230` | `_.intersection(userPermissions, permissions).length < 1` 数组交集 |
| `search.graphql:21,31,33` | `@auth(requires: ["manage:system"])` GraphQL 指令 |
| `group.graphql:43` | `permissions: [String]!` 直接字符串数组输入 |
| `data.yml:143` | `defaultPermissions: ['read:pages', 'read:assets', ...]` |

没有类似 `Permissions.READ_PAGES = 'read:pages'` 的常量定义，新增权限需全局 grep 修改。

### 26.3 Page Rules 的匹配优先级算法（可扩展点）

`_applyPageRuleSpecificity()`（`auth.js:368-388`）是 page rules 的核心冲突解决算法，按以下优先级决定哪条规则生效：

```
1. 规则路径长度优先（长路径覆盖短路径）：
   rule.path.length > checkState.specificity.length → 覆盖

2. 同长度时，按匹配类型优先级（高→低）：
   EXACT > TAG > REGEX > END > START
   由 each case 的 higherPriority 参数决定：
   - START:  higherPriority=['END','REGEX','EXACT','TAG']
   - END:    higherPriority=['REGEX','EXACT','TAG']
   - REGEX:  higherPriority=['EXACT','TAG']
   - TAG:    higherPriority=['EXACT']
   - EXACT:  higherPriority=[]

3. 同长度同类型时：
   - DENY 规则不被 ALLOW 规则覆盖（拒绝优先）
   - 后遍历到的规则覆盖前面的（取决于 groups 遍历顺序）
```

groups 遍历顺序是 `user.groups.forEach`，即用户加入组的顺序，**非确定性**。如果用户在两个组里有同优先级冲突的规则，结果取决于组 ID 顺序。

### 26.4 搜索场景的自定义权限组合扩展点

将 RBAC 权限与搜索结合，存在以下可扩展但未实现的点：

1. **自定义搜索权限**：目前只有 `manage:system`（管搜索引擎）和 `read:pages`（搜结果可见），没有细粒度的「允许搜索特定路径」「允许搜索草稿」等权限
2. **权限组合作为 search filter**：`checkAccess` 的 pageRules 逻辑（START/END/REGEX/EXACT/TAG）可以直接翻译为搜索引擎的 filter DSL，但目前只在后置过滤用
3. **自定义 roles 扩展**：pageRules 的 `roles` 字段是字符串数组，目前硬编码为 `['read:pages', 'write:pages']` 等，可扩展为自定义角色（如 `search:advanced` 允许使用范围/标签查询）
4. **`defaultPermissions` / `defaultPageRules`**（`data.yml:143-158`）：已有默认权限模板，但搜索模块没有专属默认规则

### 26.5 演进建议

1. 抽出 `Permissions` 常量模块，所有权限字符串从硬编码改为引用常量
2. 在 pageRules schema 中增加 `priority: Number` 字段，用于同长度同类型时的确定性冲突解决，替代当前的「后遍历覆盖」
3. 搜索 PermissionInterceptor 的 `preQuery()` 直接调用 `_applyPageRuleSpecificity` 提取用户可见的路径集合，下推为搜索引擎 filter
4. 新增 `search:query-advanced` 权限（范围查询、标签过滤、聚合统计）和 `search:rebuild` 权限（单独的索引重建权限，从 manage:system 中拆分）

---

## 27. 同 weight 字段冲突的稳定性兜底

### 27.1 当前 weight 声明的冲突状态

各 provider 中 title/description/content 的权重是**分散声明且无冲突检测**的：

| Provider | title 权重 | description 权重 | content 权重 | 声明方式 |
|----------|-----------|-----------------|-------------|---------|
| es 6/7.x mapping | 10.0 | 3.0 | 1.0 | `properties.title.boost`（静态） |
| es 8.x query 层 | 20 | 3 | 1 | `fields: ['title^20', ...]`（动态） |
| postgres | A（最高） | B | C | `setweight(..., 'A'/'B'/'C')` |
| algolia | 第 1 位 | 第 2 位 | 第 3 位 | `searchableAttributes` 顺序 |
| azure | 4 | 3 | 1 | `scoringProfiles.fieldWeights` |
| db | 无权重 | 无权重 | 无权重 | OR 平等 |

**同 weight 冲突的定义**：不同来源（mapping vs query、不同 provider、同一 provider 不同版本）对同一逻辑字段声明了不同权重值，导致搜索结果不可预测。

### 27.2 已存在的实际冲突案例

**elasticsearch 内部冲突（mapping boost vs query boost）**：
- es 6.x/7.x 在 mapping 中声明了 `title.boost: 10.0`（`engine.js:73`）
- 同时 query 时又声明了 `fields: ['title^20']`（`engine.js:158`）
- es 实际生效是**两者相乘**（10 × 20 = 200），但代码作者可能以为是覆盖关系
- es 8.x 去掉了 mapping boost，同样的 query 在 8.x 只有 20 倍权重，导致**跨版本排序不一致**

**跨 provider 同权重语义不一致**：
- postgres 的 A/B/C/D 是 4 档离散值，实际数值由 `ts_rank` 的 `weights` 参数决定（默认 {0.1, 0.2, 0.4, 1.0}）
- azure 的 fieldWeights 是线性整数（1~4）
- es 的 boost 是浮点数乘数（1~20）
- 三者的「title 权重最高」在数值上完全不对等，换引擎后 title 命中的文档排序位置会跳变

### 27.3 稳定性兜底策略建议

1. **单一真相来源**：在 FieldMapper 集中声明逻辑权重（如 `title: 100, description: 30, content: 10`），各 provider 按比例映射到自己的权重系统
2. **冲突检测**：`refreshSearchEnginesFromDisk()` 时校验各 provider 的权重声明与 FieldMapper 偏差，超过阈值（如 1.5x）告警
3. **es boost 归一化**：删除 mapping 层 boost（es 8.x 已不支持），统一在 query 层声明，避免双重乘
4. **同分数 tie-breaker**：见第 28 章，权重相同时由二级排序字段兜底，保证结果稳定

---

## 28. 嵌套上限暴露配置项

### 28.1 当前状态：无配置、无嵌套

如第 11 章和第 19 章所述，当前所有 provider：
- 无 nested / parent-child / join 类型字段
- 所有 definition.yml 中无 `maxNestedDepth`、`maxNestedObjects` 等配置项
- GraphQL schema 无嵌套查询参数

即使未来引入嵌套字段，用户也无法在 UI 上调整上限。

### 28.2 各引擎原生嵌套上限参考（应映射为配置项）

| 引擎 | 原生限制 | 默认值 | 应暴露的配置项 |
|------|---------|-------|--------------|
| **elasticsearch** | `index.mapping.nested_fields.limit` | 50 | `maxNestedFields`（全局索引级） |
| **elasticsearch** | `index.mapping.nested_objects.limit` | 10000 | `maxNestedObjectsPerDoc`（单文档级） |
| **elasticsearch** | `index.mapping.depth.limit` | 20 | `maxNestedDepth`（字段路径深度） |
| **postgres jsonb** | 无硬限制，但深度 >5 性能恶化 | — | `maxJsonbDepth`（建议 ≤ 3） |
| **postgres tsvector** | 不支持嵌套 | — | N/A |
| **algolia/azure/aws** | 不支持 nested | — | N/A |

### 28.3 配置项设计建议

在 `FieldMapper` 的 nested 字段声明中增加：
```yaml
# definition.yml 示例（嵌套场景）
props:
  nestedSearchEnabled:
    type: Boolean
    default: false
    order: 10
  maxNestedDepth:
    type: Number
    default: 2
    min: 1
    max: 5
    order: 11
  maxNestedObjectsPerDoc:
    type: Number
    default: 100
    min: 1
    max: 1000
    order: 12
```
同时 provider 的 `init()` 中根据配置设置引擎参数（如 es 的 `indices.putSettings`），`activate()` 中校验用户配置不超过引擎硬限制。

---

## 29. score 相同 tie-breaker 的 user-defined 字段优先级

### 29.1 当前状态：无 tie-breaker

第 20 章已确认，所有 provider 都**没有配置二级排序键**。BM25/ts_rank 打分相同的文档，返回顺序依赖：
- es：文档内部 `_doc`（插入顺序），多分片时不确定
- postgres：SQL 无 ORDER BY 时是物理存储顺序
- db：完全随机
- algolia/azure/aws：引擎内部默认顺序（通常是 `objectID` / 文档 ID 字典序）

### 29.2 tie-breaker 设计层级

合理的 tie-breaker 应分三层，优先级从高到低：

| 层级 | 来源 | 字段示例 | 可配置性 |
|------|------|---------|---------|
| Layer 1: 用户自定义 | `PageQuery.search(orderBy: [...])` | `updatedAt DESC`, `viewCount DESC`, `path ASC` | ✅ 每个查询可选 |
| Layer 2: 全局默认 | admin 配置（definition.yml） | `_score DESC, updatedAt DESC, path ASC` | ✅ 站点级配置 |
| Layer 3: 最终兜底 | 代码硬编码保证确定性 | `id ASC` / `hash ASC` | ❌ 不可配置，保证永远有确定性排序 |

### 29.3 各 provider 的 tie-breaker 实现差异

| Provider | 如何添加二级排序 |
|----------|----------------|
| **elasticsearch** | `sort: [{ _score: 'desc' }, { updatedAt: 'desc' }, { path: 'asc' }, { _id: 'asc' }]` |
| **postgres** | `ORDER BY ts_rank(...) DESC, "updatedAt" DESC, path ASC, id ASC` |
| **algolia** | `index.search(q, { sortFacetValuesBy: ['count', 'alpha'] })` （algolia 排序在索引配置中，查询级 tie-breaker 支持有限） |
| **azure** | `$orderby=search.score() desc, updatedAt desc, path asc, id asc` |
| **aws** | `sort: '_score desc, updatedAt desc, path asc'` |
| **db** | `orderByRaw('CASE WHEN title ILIKE ? THEN 1 ELSE 0 END DESC, "updatedAt" DESC, path ASC, id ASC')` |

### 29.4 GraphQL schema 扩展点

当前 `PageQuery.search`（`page.graphql:29-33`）只有 `query/path/locale`，可扩展：
```graphql
type PageQuery {
  search(
    query: String!
    path: String
    locale: String
    orderBy: [PageSearchOrderByInput]   # 新增：user-defined 排序
    first: Int = 50                      # 新增：分页
    after: String                         # 新增：游标
  ): PageSearchResponse
}

input PageSearchOrderByInput {
  field: PageSearchOrderField!   # SCORE / UPDATED_AT / CREATED_AT / PATH / TITLE
  direction: OrderDirection!     # ASC / DESC
}
```
PermissionInterceptor 需要校验：用户是否有 `search:advanced` 权限才能使用自定义排序。

---

## 30. PlaceholderResult 与真实结果合并时歧义

### 30.1 歧义场景定义

多 provider 并发查询时（第 21 章），PlaceholderResult 与真实结果合并会产生以下歧义：

| 歧义场景 | 问题描述 |
|---------|---------|
| **去重歧义** | es 返回了页面 A，db 占位结果也返回了页面 A，合并时按什么字段去重？（id/hash/path+locale?） |
| **排序歧义** | es 页面 A score=5.2，db 页面 A score=0.8（占位无真实分），合并后 A 的最终排序位置？ |
| **来源歧义** | 用户看到页面 A，不知道是真实搜索结果还是占位，反馈问题时无法定位 |
| **总数歧义** | es totalHits=1000，db 占位 totalHits=50，合并后 totalHits 应该是多少？ |
| **建议歧义** | es suggestions=['javascript']，db 占位 suggestions=[]，合并后取并集还是只取真实引擎？ |
| **分页歧义** | es 返回前 50 条（过滤剩 5 条可见），db 占位返回 50 条，合并后下一页从哪里开始？ |

### 30.2 去重主键选择

当前 provider 的结果 id 策略不统一（第 8 章第 6 点）：
- es/algolia/azure/aws：`id = page.hash`
- postgres：无明确 id，返回的是 `id`（自增主键）+ `path` + `locale`

合并去重的**唯一可靠主键是 `(path, locale)` 二元组**，因为：
- `hash` 在 rename 时会变，且 postgres 没返回
- `id`（自增）在不同环境可能不一致
- `(path, locale)` 是业务上真正唯一的页面标识

### 30.3 合并策略建议

```
1. 分类标记
   - 真实结果每条标记 source: 'elastic'/'algolia'
   - 占位结果每条标记 source: 'placeholder'，score=null

2. 去重
   - key = (path, locale)
   - 如果同 key 既有真实结果又有占位，丢弃占位，保留真实结果
   - 如果同 key 多个真实引擎返回，按引擎权重取最高分

3. 排序
   - 真实结果按 _score DESC，占位结果追加在最后（按 updatedAt DESC）
   - 保证占位结果永远不会排在真实结果前面

4. 统计
   - totalHits = 真实结果的 totalHits（忽略占位的 totalHits）
   - partial = 有任何 provider 失败
   - failedProviders = 失败的引擎 key 列表

5. suggestions
   - 取所有成功引擎 suggestions 的去重并集
   - 占位引擎的 suggestions（通常为空）忽略

6. 前端可见性
   - 占位结果在 UI 上用灰色 / 「兜底结果」标签标记
```

---

## 31. LFU-Aging 混合策略实现

### 31.1 纯 LFU 的问题

纯 LFU（Least Frequently Used）有两个已知问题：
1. **冷启动问题**：新页面频率为 0，即使是热点内容也会被立即淘汰
2. **历史污染问题**：曾经热门但现在没人看的页面（如「2020 年度报告」）历史频率很高，长期占据缓存

### 31.2 LFU-Aging 混合策略

LFU-Aging 是对纯 LFU 的改进：每个缓存项维护 `(frequency, last_access_time)`，eviction 时按以下公式计算得分：

```
score = frequency * decay_factor^(current_time - last_access_time)
```

其中 `decay_factor ∈ (0, 1)`（通常 0.9 ~ 0.99），时间窗口单位 = 1 天。

效果：
- 30 天前访问 100 次的页面 = 100 × 0.95^30 ≈ 21 分
- 昨天访问 10 次的新页面 = 10 × 0.95^1 ≈ 9.5 分
- 1 年前访问 1000 次的页面 = 1000 × 0.95^365 ≈ 0.05 分（几乎为 0）

既保留了频率偏好，又让历史热点自然「冷却」。

### 31.3 落地到 Wiki.js 页面缓存

当前页面缓存是磁盘文件 `cache/${hash}.bin`，可以**利用文件系统元信息**低成本实现 Aging：

| 缓存属性 | 文件系统对应字段 | 用法 |
|---------|----------------|------|
| frequency | 自定义扩展属性 `user.freq`（`fs.setxattr`）或独立 SQLite/JSON 索引 | 每次 `getPageFromCache()` 命中时 +1 |
| last_access_time | `atime`（文件访问时间）或 `mtime`（修改时间，若 mount noatime） | 每次命中时 `fs.utimes()` 更新 |
| size | 文件 `stat().size` | 容量统计 |

eviction 触发时机（cron job 或写入时超过阈值）：
```
1. 扫描 cache/ 目录所有 .bin 文件
2. 对每个文件读取 atime + xattr.freq
3. 计算 score = freq * decay^(days_since_atime)
4. 按 score 升序排序
5. 删除得分最低的 N 个文件，直到总大小 < 容量阈值
```

### 31.4 与 W-TinyLFU 的对比与选型

| 策略 | Wiki.js 适配度 | 实现成本 | 命中率预期 |
|------|--------------|---------|----------|
| **纯 LRU** | ⭐⭐（批量扫描污染严重） | 低 | 中 |
| **纯 LFU** | ⭐⭐⭐（历史污染） | 中 | 中高 |
| **LFU-Aging** | ⭐⭐⭐⭐（利用文件系统元信息即可实现） | 中 | 高 |
| **W-TinyLFU** | ⭐⭐⭐⭐⭐（Caffeine 同款最优） | 高（需引入计数草图） | 最高 |

**推荐分阶段演进**：
- 短期：LFU-Aging（文件 xattr + atime），50 行代码即可
- 长期：如需迁移到 Redis/内存缓存，切换到 W-TinyLFU（引入 `tiny-lfu` npm 包）

---

## 32. 同义词环 cluster 拆分与业务反馈

### 32.1 同义词环检测后的处理：不仅仅是报错

第 23 章的 DFS 环检测发现环后，应**自动拆分 cluster** 而非直接拒绝加载。

同义词词典的图结构中，一个环是一个 **Strongly Connected Component（SCC，强连通分量）**。对每个 SCC：

```
1. 用 Tarjan / Kosaraju 算法找出所有 SCC
2. 对每个 SCC：
   a. 如果大小 = 1（单个节点无自环）→ 正常
   b. 如果大小 > 1（真实环）：
      i. 收集业务反馈（各词在搜索日志中的点击量、曝光量）
      ii. 选点击量最高的词作为「主词」
      iii. 切断环内其他词指向主词的反向边（只保留主词 → 其他词）
      iv. 打 warn 日志记录自动拆分决策
```

### 32.2 业务反馈数据源

Wiki.js 目前**没有搜索行为日志**，但可以定义以下反馈信号：

| 反馈信号 | 获取方式 | 用途 |
|---------|---------|------|
| 搜索曝光量 | 记录 `PageQuery.search` 调用参数 | 哪些词被搜得多 |
| 搜索点击率 | 前端埋点：搜索结果点击事件 | 哪些词的结果被用户实际点开 |
| 页面浏览量 | `pageViews` 表（如果存在）或 access log | 同义词目标页面的热度 |
| 管理员标注 | 后台 UI 手工指定某词为主词 | 最高优先级，覆盖算法决策 |

### 32.3 拆环示例

同义词规则：
```
CPU, 中央处理器, 处理器    # 等价组（双向边，形成 SCC 大小=3）
GPU → 图形处理器          # 单向替换
```

DFS 检测到 `CPU ↔ 中央处理器 ↔ 处理器 ↔ CPU` 是环。

假设业务反馈：
- 「CPU」搜索日志曝光量 10000 次，点击率 12%
- 「中央处理器」曝光量 500 次，点击率 3%
- 「处理器」曝光量 2000 次，点击率 5%

算法选择「CPU」为主词，自动拆环为：
```
CPU → 中央处理器
CPU → 处理器
GPU → 图形处理器
```
环被打开为树结构，查询「中央处理器」展开为「CPU 中央处理器」，不会循环。

### 32.4 演进建议

1. 短期：先实现 DFS 环检测 + 报错 + 管理员手动修复
2. 中期：增加搜索行为日志（曝光/点击），实现 SCC 检测 + 基于曝光量的自动拆环
3. 长期：管理员 UI 可视化同义词图，标红 SCC 环，支持手动选主词/切边

---

## 33. 异步拦截 race condition

### 33.1 当前搜索相关的异步 race condition 场景

搜索链路中有多处异步操作存在竞态风险：

#### 场景 1：页面更新 vs 索引更新（最严重）

```
pages.js:447                WIKI.events.outbound.emit('deletePageFromCache', page.hash)
pages.js:452                await WIKI.data.searchEngine.updated(page)

时间线：
  T0: 用户 A 提交更新，开始事务
  T1: DB UPDATE pages SET ...（事务提交）
  T2: emit('deletePageFromCache')  ← 缓存失效事件
  T3: await searchEngine.updated(page)  ← 开始调用 es
  T4: 用户 B 搜索，es 仍返回旧版本（T3 还在传输中）
  T5: es 写入完成
  T6: 用户 C 搜索，返回新版本
```

**问题**：T4 时刻存在 **DB 已更新但搜索引擎未更新** 的不一致窗口。窗口大小 = 搜索引擎 RTT + 写入延迟（通常几十到几百毫秒，高负载时可达秒级）。

#### 场景 2：rebuild 期间的增量更新冲突

```
管理员 T0: 触发 rebuildIndex() → 删除索引 → 流式重建（预计 10 分钟）
用户 T1: 创建新页面 → searchEngine.created(page) → 写入新索引
          ↑ 新文档可能在流式重建的不同阶段被覆盖或重复
```

**问题**：`rebuild()` 和 `created()/updated()` 同时写入同一索引，可能：
- 流式重建先写 v1，增量更新写 v2，然后流式重建又覆盖回 v1
- 同文档被写入两次

#### 场景 3：权限组变更 vs 权限过滤

```
管理员 T0: 修改用户组的 pageRules
管理员 T1: WIKI.events.outbound.emit('reloadGroups')
          → 各节点 WIKI.auth.reloadGroups()（异步，无同步等待）
用户 T2: 搜索 → PermissionInterceptor 仍用旧规则
节点 A T3: reloadGroups 完成
节点 B T4: reloadGroups 完成
```

**问题**：T2~T4 窗口不同节点权限规则不一致，同一用户在 A 节点可见的内容在 B 节点可能不可见。

#### 场景 4：多节点 HA 的缓存失效广播

```
pages.js:447  WIKI.events.outbound.emit('deletePageFromCache', page.hash)
              ↓ 事件总线广播到所有节点
              ↓ 每个节点 subscribeToEvents() 接收
              ↓ 各节点独立 deletePageFromCache(hash)
```

**问题**：节点网络抖动导致事件丢失时，该节点的缓存永远不失效，持续返回旧版本。当前无重试/确认机制。

### 33.2 Race Condition 防护建议

| 场景 | 防护策略 |
|------|---------|
| **DB ↔ 索引不一致** | ① 搜索结果中增加 `_version`（page.updatedAt），前端检测版本过旧时自动重试查询；② PermissionInterceptor 过滤后从 DB 二次校验关键页面 |
| **rebuild + 增量更新冲突** | rebuild 期间 `created/updated/deleted` 写入临时队列，rebuild 完成后重放队列；或 rebuild 使用临时索引 + alias 原子切换 |
| **权限组 reload 竞态** | 权限结果加 `groupsVersion` 号，reload 完成后原子自增，Interceptor 比较版本号决定是否等待 reload |
| **缓存失效丢失** | 失效事件加递增 id，每个节点记录已处理到的最大 id，定期对账补齐 |
| **通用防护** | 所有异步拦截（PermissionInterceptor.postQuery、缓存失效、索引更新）都应是**幂等**的，重复执行不产生错误结果 |

### 33.3 幂等性实现要点

- `searchEngine.updated(page)` 应使用 upsert（覆盖写）而非 append，重复调用 N 次结果一致
- `deletePageFromCache(hash)` 重复删除同一路径无副作用
- PermissionInterceptor 对同 results 数组过滤多次，结果相同（纯函数 filter）
- 同义词展开对同 query 展开 N 次结果相同

---

## 34. 最终补充关键代码坐标

| 关注点 | 文件 : 行号 |
|--------|-------------|
| checkAccess 三层 RBAC 主逻辑 | `server/core/auth.js:221-295` |
| _applyPageRuleSpecificity 规则优先级算法 | `server/core/auth.js:368-388` |
| getEffectivePermissions 页面权限聚合 | `server/core/auth.js:496-521` |
| 默认权限 defaultPermissions/defaultPageRules | `server/app/data.yml:143-158` |
| 全局权限字符串硬编码示例 | `server/core/auth.js:225,335,510-518` |
| 页面更新 → 缓存失效 → 索引更新顺序 | `server/models/pages.js:447-452` |
| es 6/7.x mapping boost 声明 | `server/modules/search/elasticsearch/engine.js:73-78` |
| es 8.x query 层 fields boost 声明 | `server/modules/search/elasticsearch/engine.js:158` |
| postgres setweight A/B/C 权重 | `server/modules/search/postgres/engine.js:109-112` |
| algolia searchableAttributes 顺序权重 | `server/modules/search/algolia/engine.js:26-30` |
| azure scoringProfiles fieldWeights | `server/modules/search/azure/engine.js:71-79` |
| HA 多节点缓存失效事件订阅 | `server/models/pages.js:1166-1172` |
| 权限组 reload 事件订阅 | `server/core/auth.js:479-481` |

---

## 35. 复杂 boolean 表达式的解析栈深度上限

### 35.1 当前查询解析：无自定义解析器，完全依赖各引擎原生能力

搜索链路中**没有独立的 Boolean 表达式解析器**。用户输入的 `q` 直接透传给各引擎，由引擎自行解析：

| Provider | 内部 boolean 解析机制 | 深度控制 |
|----------|----------------------|---------|
| **elasticsearch** | `simple_query_string` 查询（`engine.js:154`），支持 `+`/`-`/`"`/`*`/`()`/`|`/`~` 操作符 | es 原生 `indices.query.bool.max_clause_count` 默认 1024，应用层未配置 |
| **postgres** | `pg-tsquery` 库（`postgres/engine.js:1,70`），将用户输入转义为合法 tsquery | pg-tsquery 内部无栈深限制，postgres 原生 `max_stack_depth`（默认 2MB，约递归 2000 层） |
| **db (basic)** | 纯 Knex builder 嵌套 `.andWhere/.orWhere`（`db/engine.js:29-47`），仅一层 OR（title ILIKE OR description ILIKE OR path ILIKE） | 固定深度 2，无动态嵌套 |
| **algolia** | 原生 query 解析，引擎侧限制 | SaaS 平台侧控制 |
| **azure** | `simple` query type（`azure/engine.js:98`），SDK 转义后传给 Azure | Azure Search `maxClauseCount` 默认 1024 |
| **aws** | `partial:true` 文本查询，无复杂 boolean | CloudSearch 默认限制 |

### 35.2 实际存在的栈深度风险

**elasticsearch `simple_query_string` 的括号嵌套风险**：
- es 的 query parser 是递归下降实现，括号嵌套过深会触发 `StackOverflowError`
- 虽然 es 默认 `indices.query.bool.max_clause_count=1024`，但这是子句数限制而非括号深度
- 恶意输入 `((((((...query...))))))`（500+ 层括号）可能打爆 es 节点 JVM 栈
- **当前代码未对用户输入做括号深度检查或扁平化预处理**

**postgres `pg-tsquery` 的特殊字符注入风险**：
- `pg-tsquery` 主要做转义（`!` `&` `|` `:` `*` 等），而非解析嵌套结构
- 但 postgres 执行 `to_tsquery` 时，如果输入被构造出极深的 `&`/`|` 嵌套树，同样可能栈溢出
- 当前代码只调用了 `tsquery(q)` 做转义，**未做 AST 深度检查**

### 35.3 演进建议

1. 抽出 `QuerySanitizer` 中间件，在用户 `q` 到达各 provider 之前：
   - 括号深度计数，超过阈值（如 32 层）截断或报错
   - boolean 子句数计数，超过 `maxClauseCount`（如 256）报错
   - 对 `simple_query_string` 的特殊操作符（`+ - " * | ~`）做白名单过滤
2. elasticsearch provider 的 `init()` 中调用 `indices.putSettings({ 'indices.query.bool.max_clause_count': 256 })` 收紧默认值
3. postgres provider 在 `query()` 中对 `tsquery(q)` 结果额外做长度校验，超长拒绝

---

## 36. Unicode collation 在 lexicographic 兜底的选型

### 36.1 当前 lexicographic 排序实现

如第 20/29 章所述，当前 tie-breaker 的最终兜底需要 lexicographic（字典序）排序对 `path` / `title` 字段做稳定排序。但**当前代码完全未涉及 Unicode collation**，不同 provider 的默认排序规则差异极大：

| Provider | 默认字符串排序规则 | 对 Unicode 的处理 |
|----------|------------------|----------------|
| **elasticsearch** | 字段 keyword 类型使用 Unicode 代码点排序（UCA 默认），取决于底层 ICU 默认值 | 依赖 es 的 `icu_collation` 插件（未配置），否则为简单 UTF-16 code unit 比较 |
| **postgres** | 数据库 `LC_COLLATE` 决定（安装时指定，通常 `en_US.UTF-8` 或 `C`） | `C` 排序 = 字节序；`en_US.UTF-8` = glibc locale 排序 |
| **db (MySQL/SQLite)** | 数据库 collation 决定（通常 `utf8mb4_general_ci` 或 `BINARY`） | MySQL `_ci` 不区分大小写，`_bin` 区分大小写 |
| **algolia** | `sortFacetValuesBy: ['count', 'alpha']`，alpha 按引擎内部 Unicode 规则 | SaaS 固定规则，不可定制 |
| **azure/aws** | 各自平台默认 collation | 不可定制 |

### 36.2 实际问题场景

以中文、德文、法文混合标题为例：
- `café` vs `cafe`：glibc `en_US.UTF-8` 视为相同，es 无 ICU plugin 视为不同
- `ä` vs `a`：德文 collation ä 在 a 之后，字节序排序 ä 在 z 之后
- `中文标题` vs `English Title`：代码点排序中文（U+4E00+）永远在英文（U+0041+）之后

如果用户 A 在 postgres（`de_DE.UTF-8`）搜到的排序与用户 B 在 es（无 ICU）搜到的排序完全不同，前端分页会产生错乱。

### 36.3 Collation 选型建议

| 方案 | 适用场景 | 实现方式 |
|------|---------|---------|
| **DUCET（默认 Unicode 排序）** | 多语言混合 Wiki，无特殊需求 | es 安装 `analysis-icu` 插件，定义 `icu_collation` keyword 字段；postgres 用 `collate "en_US.UTF-8"`；应用层 `Intl.Collator` 兜底 |
| **CLDR 根排序器** | 需要跨平台完全一致 | es: `"type": "icu_collation", "language": "", "country": ""`；pg: `CREATE COLLATION ... (provider = icu, locale = '')` |
| **特定语言排序** | 单语言站点（如德文 Wiki） | es/az/pg 全部指定 `de-DE@collation=phonebook` 等对应 locale |
| **不区分重音/大小写** | 搜索友好，用户不关心大小写 | 所有引擎 collation 设置 `strength=primary`（仅比较基础字母） |

**关键原则**：所有 provider + 应用层 JS 兜底必须使用**同一份 collation 规则**，否则 cross-engine 结果合并时排序错乱无解。

---

## 37. max_nesting_depth 热更新路径

### 37.1 当前状态：无嵌套、无配置、无热更新

第 19/28 章已确认：当前无 nested 字段，无 `maxNestedDepth` 配置项暴露，更无热更新机制。所有配置变更需走 `updateSearchEngines` resolver → DB 写入 → 重新 `initEngine()` 流程，涉及：
- `deactivate()` 旧引擎
- `require()` 新实例
- `activate()` + `init()`

这是**冷重启**而非热更新。

### 37.2 嵌套上限热更新的约束条件

不同引擎的 nested 限制参数热更新能力不同：

| 参数 | es 是否支持热更新 | pg 是否支持热更新 | 其他引擎 |
|------|-----------------|-----------------|---------|
| `index.mapping.nested_fields.limit` | ✅ `indices.putSettings` 动态更新（无需重建索引） | N/A | algolia/azure/aws 不支持 nested |
| `index.mapping.nested_objects.limit` | ✅ 同上动态更新 | N/A | N/A |
| `index.mapping.depth.limit` | ✅ 同上动态更新 | N/A | N/A |
| 自定义应用层 `maxNestedDepth`（前端传入校验） | ✅ 应用层内存变量即时生效 | ✅ 同左 | ✅ 同左 |
| nested 字段 mapping 本身（新增/删除字段） | ❌ 需 `_reindex` 重建 | ⚠️ jsonb 索引需 REINDEX | N/A |

### 37.3 热更新路径设计

```
管理员更新 maxNestedDepth
        │
        ▼
[1] GraphQL Mutation.updateSearchEngines
        │  └─ 更新 DB config JSON
        ▼
[2] ProviderRegistry.setConfig(key, newConfig)
        │  └─ 内存中的 provider.config 即时更新
        ▼
[3] provider.onConfigChange(diff) 钩子
        │
        ├─ 对 es：调用 client.indices.putSettings({
        │      index: config.indexName,
        │      body: { 'index.mapping.nested_fields.limit': newVal }
        │    })
        │
        ├─ 对 pg：无需操作（无硬限制）
        │
        └─ 对不支持的 provider：打 warn 日志（配置变更下次 init 生效）
        ▼
[4] emit('search:config-updated', { key, diff }) 通知集群其他节点
```

关键注意：
- **区分「可动态参数」vs「需重建索引参数」**：`maxNestedDepth`（应用层校验）可即时生效，`nested_fields.limit`（引擎级）可热更新，新增 nested 字段 mapping 需重建索引
- 当前 `initEngine()` 每次 `require` 新实例，不保留状态；热更新要求 provider 实例是**常驻单例**，这与现有架构冲突
- 热更新后需要对**进行中的查询**做版本标记（`queryId -> configVersion`），防止半途中配置变更导致结果不一致

---

## 38. SortAxis 同名字段冲突解决

### 38.1 SortAxis 的定义与冲突场景

SortAxis = 排序字段的完整表示（如 `updatedAt DESC, title ASC`）。当用户自定义排序与全局默认 tie-breaker 或 provider 内部强制排序存在**同名字段但方向不同**时，产生冲突。

示例冲突场景：
```
全局默认 tie-breaker: ['_score DESC', 'updatedAt DESC', 'path ASC']
用户自定义 orderBy:  [{ field: 'updatedAt', direction: 'ASC' }, { field: 'title', direction: 'DESC' }]
合并后：updatedAt 出现两次，一次 DESC 一次 ASC → 歧义
```

### 38.2 当前状态：无 SortAxis 抽象，无冲突解决

第 29 章已确认当前甚至没有 tie-breaker，更谈不上冲突解决。所有排序在各 provider 中硬编码且互不相同：
- postgres：`ORDER BY ts_rank(...) DESC`（`postgres/engine.js:69`）
- db：**完全无 ORDER BY**（`db/engine.js:40`）
- es：未指定 `sort` 参数，默认 `_score DESC`（`elasticsearch/engine.js:150-175`）
- algolia：索引 `ranking` 配置
- azure/aws：默认打分排序

没有统一的 SortAxis → DSL 翻译层，也没有冲突检测。

### 38.3 冲突解决策略（优先级从高到低）

```
策略 1: 用户显式排序覆盖全局默认（User-Defined > Default）
   用户指定了 updatedAt ASC → 删除默认中的 updatedAt DESC，保留用户版本

策略 2: 同名字段取首次出现（First-Wins）
   如果同一字段在同一优先级层出现两次 → 取第一次，忽略后续，打 warn 日志

策略 3: 不可冲突字段强制追加（Force-Append）
   如 `hash ASC` 是最终确定性兜底，永远追加在最后，与用户排序不冲突（用户不能指定 hash 排序）

策略 4: 提供名空间消除同名字段歧义（Namespace）
   逻辑字段名 vs 物理字段名区分：user:title（用户指定 title） vs sys:_score（系统内部打分）
```

### 38.4 合并算法伪代码

```js
function mergeSortAxis(userOrderBy, defaultOrderBy, forcedOrderBy) {
  const seen = new Set()
  const result = []

  // Layer 1: 用户自定义（最高优先级）
  for (const axis of userOrderBy) {
    const key = axis.field.toLowerCase()
    if (!seen.has(key)) {
      seen.add(key)
      result.push(axis)
    } else {
      WIKI.logger.warn(`Duplicate sort field ${axis.field} in user orderBy, ignoring duplicate`)
    }
  }

  // Layer 2: 全局默认（用户未指定的才追加）
  for (const axis of defaultOrderBy) {
    const key = axis.field.toLowerCase()
    if (!seen.has(key)) {
      seen.add(key)
      result.push(axis)
    }
  }

  // Layer 3: 强制兜底（永远追加，不检查冲突——因为命名空间隔离）
  for (const axis of forcedOrderBy) {
    result.push(axis)
  }

  return result
}
```

---

## 39. mergeResolver 自定义合并函数注入接口

### 39.1 mergeResolver 的定义

多 provider 并发查询时（第 21/30 章），需要将多个引擎返回的结果合并为单个 `PageSearchResponse`。默认合并策略（第 30 章）是：按 `(path, locale)` 去重、真实引擎排占位前面、真实引擎按 `_score` 排序。

mergeResolver = 可替换的合并策略函数，允许管理员/插件自定义合并逻辑。

### 39.2 当前状态：无合并，无注入点

单 active 引擎模型，完全不需要 merge。代码中**没有任何合并逻辑**、没有 resolver 注入框架、没有插件系统钩子。

### 39.3 注入接口设计

```js
// mergeResolver 契约
type MergeResult = {
  results: PageSearchResult[]
  suggestions: string[]
  totalHits: number
  partial: boolean
  failedProviders: string[]
}

interface MergeContext {
  query: string
  opts: Object
  user: User
  providerResults: Array<{
    providerKey: string
    providerWeight: number
    ok: boolean
    data?: PageSearchResponse   // ok=true
    error?: Error               // ok=false
    latencyMs: number
  }>
}

type MergeResolver = (ctx: MergeContext) => MergeResult | Promise<MergeResult>
```

**内置 mergeResolver 列表**：
| 名称 | 策略 | 适用场景 |
|------|------|---------|
| `weighted-rrf` | Reciprocal Rank Fusion，按各引擎排名倒数加权求和融合 | 多引擎质量相近 |
| `primary-fallback` | 主引擎结果优先，主引擎失败/不足时才用备引擎填充 | 主搜 + 保底 |
| `intersection` | 只返回所有成功引擎都出现的结果 | 高精度要求 |
| `union-by-score` | 按真实 score 归一化后合并（第 30 章默认策略） | 简单场景 |

**注入路径**：
```yaml
# definition.yml 配置项
props:
  mergeStrategy:
    type: String
    enum: ['weighted-rrf', 'primary-fallback', 'intersection', 'union-by-score', 'custom']
    default: 'union-by-score'
    order: 20
  customMergeResolverPath:
    type: String
    hint: 'Absolute path to a JS module exporting a MergeResolver function (mergeStrategy=custom时生效)'
    default: ''
    order: 21
```

Registry 加载时：
```js
if (config.mergeStrategy === 'custom' && config.customMergeResolverPath) {
  const fn = require(config.customMergeResolverPath)
  if (typeof fn !== 'function') throw new SearchActivationFailed('Custom merge resolver must export a function')
  ProviderRegistry.setMergeResolver(key, fn)
}
```

**安全边界**：`customMergeResolverPath` 必须限制在 `WIKI.ROOTPATH/data/search-resolvers/` 目录内，防止任意文件读取/代码执行；加载时需沙箱化（`vm` 模块或 isolated-vm）。

---

## 40. decay-coefficient 自适应学习率

### 40.1 decay-coefficient 的定义与当前状态

第 31 章 LFU-Aging 策略中的 `decay_factor ∈ (0, 1)` 就是 decay-coefficient，控制历史频率随时间衰减的速度。

当前：**无缓存 eviction → 无 decay-coefficient 概念**。

### 40.2 固定 decay 的问题

固定 decay（如 0.95/天）在不同业务模式下表现差异大：
- **活跃 Wiki**（日均编辑 1000+ 页）：衰减太慢，旧热点占位；decay 应调高（如 0.9/天，衰减更快）
- **静态 Wiki**（文档库，月更 < 10 页）：衰减太快，有效内容被淘汰；decay 应调低（如 0.99/天，衰减更慢）
- **突发热点**（如发布新版本时访问量激增）：固定 decay 无法快速响应新热点

### 40.3 自适应学习率算法

```
初始 decay = 0.95
观察窗口 = 7 天
评估指标 = cache_hit_ratio（7 日滑动平均）

每 24 小时调整一次：
  if hit_ratio < target_low (e.g. 0.5):
      decay = clamp(decay * 0.98, 0.80, 0.995)   // 降低衰减（乘更小系数 → 更慢衰减）
  elif hit_ratio > target_high (e.g. 0.8):
      decay = clamp(decay * 1.01, 0.80, 0.995)   // 提高衰减
  else:
      keep
```

其中：
- `hit_ratio = cache_hits / (cache_hits + cache_misses)`，由 `getPageFromCache()` 返回 false 时计数
- `clamp` 保证 decay 永远在合理范围（不低于 0.80，不高于 0.995）
- 调整幅度小（±1~2%），防止震荡

### 40.4 代码落地位置与边界

| 组件 | 职责 | 建议位置 |
|------|------|---------|
| **Hit/Miss 计数器** | 每次 `getPageFromCache()` 命中 +1 hit，未命中 +1 miss | `pages.js:1086 getPageFromCache()` 内 |
| **滑动窗口存储** | 最近 N 天每小时 hit/miss 数（Redis ZSET 或 SQLite 表） | 新模块 `server/core/cache-metrics.js` |
| **自适应调节器** | 每日 cron 计算 hit_ratio，调整 decay 并存入 DB config | `server/jobs/cache-decay-adjust.js`（新 job） |
| **eviction 执行器** | 读取当前 decay 计算 score，淘汰低分文件 | `server/jobs/cache-evict.js`（新 job） |

**防止自适应调整失效的边界**：
- 新站点前 7 天 hit_ratio 无意义，使用初始 decay 0.95
- 流量突降（节假日）时暂停调整（当 PV < baseline 的 30%）
- decay 变更后 `emit('cache:decay-updated', newDecay)` 广播到 HA 节点

---

## 41. cycleFeedbackHook 异步反馈延迟容忍

### 41.1 cycleFeedbackHook 的定义

第 32 章同义词环自动拆分需要业务反馈（搜索曝光量、点击率）来决定主词。cycleFeedbackHook = 将前端/业务层的搜索行为数据异步回传给同义词引擎的钩子。

**当前状态**：无搜索行为日志 → 无反馈 → 无 hook。

### 41.2 反馈延迟容忍度分析

反馈数据从产生到生效的延迟对拆环质量的影响：

| 延迟区间 | 影响 | 容忍度 |
|---------|------|--------|
| **< 1 秒**（实时） | 几乎无影响，用户点击后立即影响同义词权重 | ✅ 理想，但成本高 |
| **1 秒 ~ 1 分钟**（准实时） | 轻微影响，突发热点同义词识别稍慢 | ✅ 可接受 |
| **1 分钟 ~ 1 小时**（近线） | 新热点词可能延迟识别为环的主词 | ⚠️ 可接受 |
| **1 小时 ~ 24 小时**（离线批量） | 环拆分决策严重滞后，可能持续数小时错误排序 | ❌ 不可接受（用户体验差） |
| **> 24 小时** | 完全失效，不如不用业务反馈 | ❌ 不可用 |

### 41.3 容忍延迟的实现策略

采用**分层缓冲 + 批量异步写入**架构：

```
前端点击事件（page.js 埋点）
        │ POST /api/search/feedback
        ▼
[1] API Layer（server/routes/search.js）
     ├─ 校验 user/query/page 非空
     ├─ 写入内存环形缓冲区（ring buffer，容量 10000）
     └─ 立即返回 202 Accepted（不等待下游）
        │
        ▼  每 10 秒或缓冲区满 1000 条触发
[2] Flush Worker
     ├─ 批量写入 ClickHouse/PostgreSQL search_feedback 表
     ├─ 去重（同一 user + query + page 在 5 分钟窗口内只记 1 次）
     └─ 更新内存中的热词计数器（增量曝光/点击）
        │
        ▼  每 15 分钟触发
[3] CycleFeedbackHook
     ├─ 读取最近 N 小时反馈数据
     ├─ 运行 SCC 检测 + 主词重算
     ├─ 如果主词变化 > 阈值（如 20% 的环主词变更）
     │    └─ 更新同义词配置 → reloadSynonyms() → 广播集群
     └─ 否则保持不变
```

### 41.4 延迟异常处理

| 异常场景 | 降级策略 |
|---------|---------|
| **缓冲区满（突发热点）** | 丢弃最旧数据，保留高 10% 点击率数据；打 warn 日志 |
| **DB 写入失败** | 数据暂存本地磁盘 spill 文件，后台重试指数退避（1s/2s/4s/.../60s） |
| **Hook 执行超时** | 跳过本轮，使用上一版同义词配置；连续 3 轮超时告警 |
| **节点重启丢失内存数据** | 缓冲区持久化到 WAL（write-ahead log），启动时重放 |

关键原则：**反馈链路永远不能阻塞搜索主链路**。所有 hook 必须异步执行，失败不影响用户搜索体验。

---

## 42. Mutex 大量并发吞吐降级

### 42.1 当前并发控制：完全无 Mutex

代码中搜索链路相关操作**完全没有锁**：
- `rebuildIndex()` 无互斥，管理员连点两次会启动两个并行流式重建，互相覆盖
- `searchEngine.updated()` 无互斥，同一页面被快速编辑两次可能后发起的请求先完成（网络/引擎抖动），导致索引中版本落后于 DB
- `reloadGroups()` / `refreshSearchEnginesFromDisk()` 无互斥，并发 reload 可能写坏内存状态
- 缓存 `savePageToCache()` / `deletePageFromCache()` 无互斥，并发读写同一 hash.bin 文件可能读到半写文件

### 42.2 Mutex 引入对吞吐的影响

引入锁必然降低吞吐。不同锁策略的吞吐降级：

| 锁策略 | 典型场景 | 吞吐降级幅度 | 实现复杂度 |
|--------|---------|------------|----------|
| **全局单写互斥**（1 把大锁） | rebuildIndex 全局排它锁 | 严重：rebuild 期间所有 created/updated 阻塞 | 低 |
| **Key 级分片锁**（按 page.hash 分片） | 单文档更新互斥 | 轻微：不同页面完全并发，同页面连续编辑串行化 | 中 |
| **读写锁（RWLock）** | rebuild=写锁，查询=读锁 | 中等：rebuild 阻塞所有写但不阻塞读 | 中高 |
| **乐观锁（版本号）** | 索引更新冲突检测 | 极微：无锁，仅冲突时重试 | 高 |
| **无锁队列 + 单消费者** | 所有索引变更入有序队列，单 worker 串行消费 | 轻微：写入吞吐 = worker 消费能力，不阻塞用户请求 | 中 |

### 42.3 Wiki.js 搜索场景的推荐策略

按操作粒度分层设计：

```
┌──────────────────────────────────────────────────────────────┐
│ rebuildIndex → 全局 RWLock.writeLock（重建期间拒绝新的 rebuild │
│                但 created/updated 不阻塞，入增量队列待重放）      │
├──────────────────────────────────────────────────────────────┤
│ created/updated/deleted → Key 级乐观锁 + 幂等 upsert           │
│   冲突条件：ES _version < DB page.updatedAt → 重试 2 次         │
│   最终兜底：冲突 → 放弃写入，10 秒后由补偿任务重新同步            │
├──────────────────────────────────────────────────────────────┤
│ savePageToCache → 文件原子写（先写 tmpfile + fs.rename）         │
│   不需要 Mutex，rename 在 POSIX 上是原子操作                     │
├──────────────────────────────────────────────────────────────┤
│ reloadGroups/refreshSearchEngines → 内存对象读写用单例 Promise   │
│   维护 loadingPromise，并发调用时复用同一个 Promise              │
└──────────────────────────────────────────────────────────────┘
```

### 42.4 高并发下的吞吐降级保护

当持锁请求排队长度超过阈值（如队列长度 > 1000）时，触发**降级策略**：

1. **快速失败**：对非关键路径（如 suggestions 查询）直接返回空，避免阻塞主链路
2. **批量合并**：对同一 page 的多个 updated 请求合并为最后一次写入（去抖 = debounce 500ms）
3. **写入节流**：用户态搜索索引写入 QPS 限制（如 ≤ 100/s），超额请求排队或返回 `Retry-After`
4. **优先级剥离**：管理员触发的 rebuild 优先级低于用户创建/更新页面操作；用户页面操作永远不被 rebuild 阻塞

---

## 43. 最终补充关键代码坐标

| 关注点 | 文件 : 行号 |
|--------|-------------|
| es simple_query_string 查询（无栈深检查） | `server/modules/search/elasticsearch/engine.js:154-159` |
| postgres pg-tsquery 转义（无 AST 深度检查） | `server/modules/search/postgres/engine.js:1,70` |
| db 引擎 Knex builder 嵌套 where（固定深度 2） | `server/modules/search/db/engine.js:29-47` |
| postgres ts_rank ORDER BY（无 Unicode collation 配置） | `server/modules/search/postgres/engine.js:69` |
| db 引擎完全无 ORDER BY（排序完全随机） | `server/modules/search/db/engine.js:40-64` |
| es 查询未指定 sort（默认 _score DESC） | `server/modules/search/elasticsearch/engine.js:150-175` |
| es 8.x _source 字段列表（title/description/path/locale） | `server/modules/search/elasticsearch/engine.js:163` |
| 页面更新 → 缓存失效 → 索引更新顺序（无互斥） | `server/models/pages.js:447-452` |
| 缓存 savePageToCache（fs.outputFile 非原子写） | `server/models/pages.js:1052-1077` |
| 缓存 getPageFromCache（无 hit/miss 计数） | `server/models/pages.js:1086-1106` |
| initEngine 冷重启（require 新实例，无热更新） | `server/models/searchEngines.js:98-124` |
| updateSearchEngines 配置更新入口 | `server/graph/resolvers/search.js:41-70` |
| _applyPageRuleSpecificity 规则冲突算法（可参考作 SortAxis 冲突算法） | `server/core/auth.js:368-388` |

---

## 44. 演进观察（最终完整汇总）

1. **安全审计薄弱**：引擎切换、配置修改无审计日志；`query()` 错误仅打 warn 不抛异常，静默失败可能掩盖攻击。`level` 字段预留但未使用，缺少 provider 可信分级。
2. **无超时控制**：所有搜索引擎调用均未设置超时，网络故障会长时间阻塞请求；搜索查询接口无 rate limit，可被滥用。
3. **单引擎模型限制**：架构上不支持多引擎组合（如主搜+备搜、混合召回），查询失败时只能返回空结果，无占位回填；占位与真实结果合并的歧义未定义；无 mergeResolver 自定义注入接口。
4. **排序/分页能力原始**：无自定义排序、无 SortAxis 同名字段冲突解决、无 user-defined tie-breaker、db 引擎完全无 ORDER BY、es 未设置 `preference`、无 Unicode collation 统一规则、无深度分页、totalHits 语义不统一。
5. **同义词/查询重写缺失**：除 postgres 基础转义外，无查询理解层（QUL）；未考虑 boolean 表达式栈深上限、同义词环检测、SCC 自动拆环、cycleFeedbackHook 异步反馈延迟容忍。
6. **权限过滤重复代码**：6 处 resolver 重复相同的 `_.filter + checkAccess` 模式，未抽出统一拦截器；provider 内部无 RLS 下推，召回集浪费严重；权限字符串全库硬编码无常量定义。
7. **缓存与索引不一致风险**：页面渲染缓存与搜索引擎索引是两套独立失效机制，无分布式事务保证；页面缓存无 eviction（无限增长）、无 TTL、无 LRU/LFU/LFU-Aging 策略、无 decay-coefficient 自适应学习率、无 hit/miss 指标。
8. **字段映射无集中抽象**：各 provider 字段定义、权重声明、优先级策略硬编码且分散维护；es 6/7.x mapping boost 与 8.x query boost 存在双重乘冲突；跨 provider 同权重语义不对等导致换引擎排序剧变。
9. **结构化查询为零**：无 nested/has_child/聚合/范围查询能力；tags 字段仅 es 索引但未提供查询入口；嵌套上限未暴露配置项；无 max_nesting_depth 热更新路径。
10. **Provider 注册非 Registry 模式**：扫描/加载/激活散落在单个 Model 文件中，无注册钩子、无生命周期管理、无版本控制、无配置热更新。
11. **RBAC 扩展点未利用**：pageRules 的 `_applyPageRuleSpecificity` 优先级算法可直接翻译为搜索引擎 filter，但目前仅用于后置过滤；无 `search:advanced` 等细粒度搜索权限。
12. **异步竞态未防护**：DB ↔ 搜索引擎更新窗口、rebuild + 增量更新冲突、多节点权限 reload 不一致、HA 缓存失效事件丢失均存在 race condition；无 Mutex/乐观锁/队列串行化控制；异步操作幂等性未显式保证；高并发吞吐无降级策略。
