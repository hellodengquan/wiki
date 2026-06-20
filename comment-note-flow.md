# 评论与注释功能——完整代码链路分析

本文档追踪 Wiki.js 中评论（Comment）功能从创建、权限校验到页面呈现的完整代码链路。
代码分散在 GraphQL 层、Model 层、Provider 模块、SSR 控制器、Vuex Store、Vue 组件共 6 个位置，逻辑不直白，下面逐层拆解。

---

## 1. 整体架构一图

```
用户浏览器                          服务端
──────────                          ──────────────────────────────────────────────
comments.vue                        GraphQL Schema (comment.graphql)
  │  Apollo mutation/create ─────────▶ @auth 指令（全局权限门控）
  │  (isAuthenticated? 显示guest字段) │ @rateLimit（15s/次 IP 级限流）
  │                                   ▼
  │                                 Resolver (comment.js)
  │                                   │  WIKI.models.comments.postNewComment()
  │                                   ▼
  │                                 Comment Model (comments.js)
  │                                   │  user.id===2? 校验guestName/Email
  │                                   │  WIKI.auth.checkAccess()（页面级权限）
  │                                   ▼
  │                                 CommentProvider (default/comment.js)
  │                                   │  ├─ Markdown 渲染(html:false + emoji)
  │                                   │  ├─ DOMPurify XSS 清洗
  │                                   │  ├─ Akismet 反垃圾(role=guest/user/admin)
  │                                   │  ├─ minDelay 频率控制(Guest 全局共享)
  │                                   │  └─ 硬删除 / 直接覆盖(无版本历史)
  │                                   ▼
  │                                 DB INSERT (Objection.js → comments 表)
  │
  │     ⚠️  以下功能均未实现：
  │     ❌ 无 event emit → 无通知 / Activity log / Webhook
  │     ❌ 无点赞 / 收藏表 → 无社交互动数据
  │     ❌ 无 @mention 解析 → 仅纯文本，无自动补全
  │
  ├─ Apollo query/list ─────────────▶ Resolver.list()
  │                                   │  查 pages 表 → checkAccess → 查 comments 表
  │                                   ▼
  │                                 返回 [CommentPost]

page.vue (主题层)                     SSR Controller (common.js)
  │  读取 effectivePermissions ◀────── WIKI.auth.getEffectivePermissions()
  │  决定是否渲染评论区               │  → page.pug → 传入 commentsEnabled / effectivePermissions
  │                                   ▼
page.pug (SSR 模板)                 → 传入 <page :comments-enabled :effective-permissions>
  │                                   → <template slot='comments'> 嵌入 comments.main
  ▼
comments.vue 挂载
  │  v-intersect → fetch()
  │  根据 permissions.write / permissions.manage 控制UI
  │  (注意：作者本人也需 manage:comments 才能编辑/删除)
```

---

## 2. 评论创建链路

### 2.1 前端入口：`client/components/comments.vue`

- 用户在 `<v-textarea>` 输入评论内容，点击"Post Comment"按钮
- `postComment()` 方法（:223）执行：
  1. **前端校验**：用 `validate.js` 检查内容长度≥2；未登录用户还要求 `guestName`（2~255字符）和 `guestEmail`（合法邮箱）
  2. **Apollo Mutation**：调用 `comments.create`，传入 `pageId`、`replyTo`、`content`、`guestName`、`guestEmail`
  3. 成功后调用 `fetch()` 刷新列表，并自动滚动到新评论

### 2.2 GraphQL Schema 层：`server/graph/schemas/comment.graphql`

```graphql
type CommentMutation {
  create(
    pageId: Int!
    replyTo: Int
    content: String!
    guestName: String
    guestEmail: String
  ): CommentCreateResponse
    @auth(requires: ["write:comments", "manage:system"])
    @rateLimit(limit: 1, duration: 15)
}
```

**两道指令门控**：
- `@auth(requires: [...])`：全局权限，要求用户拥有 `write:comments` 或 `manage:system` 之一
- `@rateLimit(limit: 1, duration: 15)`：同一 IP 15秒内只允许 1 次创建

### 2.3 @auth 指令实现：`server/graph/directives/auth.js`

- 使用 `SchemaDirectiveVisitor` 包装字段的 `resolve` 函数
- 在执行 resolver 前，检查 `context.req.user.permissions` 是否包含 `requiredScopes` 中的任意一个
- 不满足则抛出 `Forbidden`

### 2.4 @rateLimit 指令实现：`server/graph/directives/rate-limit.js`

- 基于 `graphql-rate-limit-directive`，以 `IP:父类型.字段名` 为 key 做窗口限流

### 2.5 Resolver 层：`server/graph/resolvers/comment.js`

`CommentMutation.create`（:102）：
```js
async create(obj, args, context) {
  const cmId = await WIKI.models.comments.postNewComment({
    ...args,
    user: context.req.user,
    ip: context.req.ip
  })
  return { responseResult: graphHelper.generateSuccess('...'), id: cmId }
}
```

职责：透传参数 + 当前用户 + 请求 IP 到 Model 层。

### 2.6 Model 层：`server/models/comments.js`

`Comment.postNewComment()`（:63）——核心业务逻辑：

1. **访客信息校验**（:65-89）：如果 `user.id === 2`（Guest 用户），校验 `guestEmail` 和 `guestName`
2. **内容校验**（:92-95）：`content` trim 后长度必须 ≥ 2
3. **加载页面**（:98）：`WIKI.models.pages.getPageFromDb(pageId)` 获取页面及 tags
4. **页面级权限校验**（:100-106）：`WIKI.auth.checkAccess(user, ['write:comments'], { path, locale, tags })`
5. **委托给 CommentProvider**（:112）：`WIKI.data.commentProvider.create(...)` → 走具体提供者

### 2.7 Provider 层：`server/modules/comments/default/comment.js`

`create()`（:64）——默认内置提供者：

1. **Markdown 渲染 + XSS 清洗**（:68）：`DOMPurify.sanitize(mkdown.render(content))`
2. **Akismet 反垃圾**（:78-107）：若配置了 Akismet Key，调用 `akismetClient.checkSpam()` 检测垃圾评论
3. **最小发言间隔**（:110-115）：若配置了 `minDelay`，检查同一用户最近一条评论的时间间隔
4. **写入数据库**（:118）：`WIKI.models.comments.query().insert(newComment)` → Objection.js → `comments` 表
5. **返回评论 ID**

---

## 2.8 匿名 vs 已登录用户：差异化路径与限制策略

评论功能对匿名（Guest）和已登录用户采用不同的代码路径和限制策略，差异贯穿前端 UI、后端校验、反垃圾评分、频率控制四个层面。

### 2.8.1 Guest 用户的标识

系统用 **`user.id === 2`** 标识 Guest 用户，定义在 `server/models/users.js:879-889`：

```js
static async getGuestUser() {
  const user = await WIKI.models.users.query().findById(2).withGraphJoined('groups')
  // ...
  user.permissions = user.getGlobalPermissions()
  return user
}
```

Guest 是一个系统预置账户（`isSystem: true`），默认属于 `id=2` 的 Guest 组。未登录用户访问时，`auth.js:170-176` 会将 `req.user` 设置为该 Guest 对象。

### 2.8.2 前端差异化（`client/components/comments.vue`）

| 差异点 | 已登录用户（`isAuthenticated=true`） | 匿名用户（`!isAuthenticated`） | 代码位置 |
|--------|-----------------------------------|-----------------------------|----------|
| 额外输入字段 | 无 | 显示 `guestName` + `guestEmail` 文本框 | `comments.vue:17-42` |
| 身份显示 | 显示 "Posting as **用户名**" | 无 | `comments.vue:47-49` |
| 前端校验规则 | 仅校验 `content` 长度≥2 | 额外校验 `name`（2~255字符）和 `email`（合法邮箱格式） | `comments.vue:234-250` |
| Apollo 变量 | `guestName`/`guestEmail` 传 `''` | 传用户输入值 | `comments.vue:298-300` |

### 2.8.3 后端差异化（`server/models/comments.js`）

`postNewComment()` 方法（:65-89）中针对 Guest 用户做额外校验：

```js
if (user.id === 2) {
  const validation = validate({
    email: _.toLower(guestEmail),
    name: guestName
  }, {
    email: { email: true, length: { maximum: 255 } },
    name: { presence: { allowEmpty: false }, length: { minimum: 2, maximum: 255 } }
  }, { format: 'flat' })
  if (validation && validation.length > 0) {
    throw new WIKI.Error.InputInvalid(validation[0])
  }
}
```

**数据层差异**（`postNewComment()` :116-123）：传入 Provider 时，Guest 用户的 name/email 会被 `guestName`/`guestEmail` 覆盖：

```js
user: {
  ...user,
  ...(user.id === 2) ? { name: guestName, email: guestEmail } : {},
  ip
}
```

### 2.8.4 反垃圾评分差异化（`default/comment.js:78-84`）

Akismet 反垃圾检查中，根据用户组标识角色：

```js
let userRole = 'user'
if (user.groups.indexOf(1) >= 0) {
  userRole = 'administrator'
} else if (user.groups.indexOf(2) >= 0) {
  userRole = 'guest'   // ← Guest 组用户标记为 guest，反垃圾评分更严格
}
```

### 2.8.5 频率控制差异化

`definition.yml:21` 的 `minDelay` 配置注释明确说明：

> "Minimum delay (in seconds) between comments per account. Note that **all guests are considered as a single account**."

`default/comment.js:110-115` 实现：

```js
if (WIKI.data.commentProvider.config.minDelay > 0) {
  const lastComment = await WIKI.models.comments.query()
    .select('updatedAt')
    .findOne('authorId', user.id)   // ← 所有 Guest 都是 id=2，共享同一个延迟计数器
    .orderBy('updatedAt', 'desc')
  // ...
}
```

关键含义：**所有匿名用户共享同一个 `minDelay` 计数器**，因为他们的 `authorId` 都是 2。这是防止匿名用户刷屏的重要策略。

---

## 2.9 评论编辑与删除：无状态机、无版本历史

### 2.9.1 数据库结构分析

`comments` 表的字段演进（来自三次迁移）：

1. **初始建表**（`migrations/2.0.0.js:63-69`）：
   ```js
   table.increments('id').primary()
   table.text('content').notNullable()
   table.string('createdAt').notNullable()
   table.string('updatedAt').notNullable()
   ```

2. **2.4.36** 新增：`render`、`name`、`email`、`ip`
3. **2.4.61** 新增：`replyTo`

**关键观察**：没有 `isDeleted`、`deletedAt`、`version`、`status` 等字段，也没有 `commentHistory` 或 `commentVersions` 表。

### 2.9.2 更新操作：直接覆盖，无版本记录

`default/comment.js:126-133` 的 `update()` 方法：

```js
async update ({ id, content, user }) {
  const renderedContent = DOMPurify.sanitize(mkdown.render(content))
  await WIKI.models.comments.query().findById(id).patch({
    content,              // 直接覆盖
    render: renderedContent
  })
  return renderedContent
}
```

- 没有保存历史版本
- 没有变更日志
- 仅 `updatedAt` 自动更新（`comments.js:52-54` 的 `$beforeUpdate()` 钩子）

前端在 `comments.vue:89` 通过比较 `createdAt` 和 `updatedAt` 显示 "Modified" 提示，但无法恢复旧版本。

### 2.9.3 删除操作：硬删除，不可恢复

`default/comment.js:137-139` 的 `remove()` 方法：

```js
async remove ({ id, user }) {
  return WIKI.models.comments.query().findById(id).delete()
}
```

- **物理删除**（`DELETE` 语句），不是软删除
- 删除后数据不可恢复
- 没有回收站/撤销机制
- 没有 `deleteComment` 审计日志

### 2.9.4 权限要求

| 操作 | 所需权限 | 代码位置 |
|------|----------|----------|
| 编辑评论 | `manage:comments` 或 `manage:system` | `comment.graphql:50` |
| 删除评论 | `manage:comments` 或 `manage:system` | `comment.graphql:54` |

注意：**评论作者本人也不能编辑/删除自己的评论**，除非拥有 `manage:comments` 权限。这是因为 Schema 层和 Model 层的权限校验只检查权限位，不比较 `authorId`。

---

## 2.10 Markdown 渲染管道、转义和反垃圾过滤

评论内容的处理走**独立的轻量级管道**，与 Wiki 页面的完整渲染管道（Renderer 系统）完全分离。

### 2.10.1 渲染管道架构

`default/comment.js:1-25` 初始化独立的 Markdown 实例：

```js
const md = require('markdown-it')
const { full: mdEmoji } = require('markdown-it-emoji')
const { JSDOM } = require('jsdom')
const createDOMPurify = require('dompurify')

const window = new JSDOM('').window
const DOMPurify = createDOMPurify(window)

const mkdown = md({
  html: false,        // 禁止原始 HTML（关键安全措施）
  breaks: true,       // 换行转 <br>
  linkify: true,      // 自动识别链接
  highlight(str, lang) {
    return `<pre><code class="language-${lang}">${_.escape(str)}</code></pre>`
  }
})

mkdown.use(mdEmoji)   // 启用 Emoji 支持
```

**与页面渲染管道的差异**：

| 特征 | 评论渲染 | 页面渲染 |
|------|---------|---------|
| HTML 支持 | `html: false`，完全禁止 | 部分支持，依赖 renderer |
| 插件系统 | 仅 `markdown-it-emoji` | 完整插件链（PlantUML, KaTeX, Kroki 等） |
| 代码高亮 | 简单转义，无高亮 | Prism.js 完整语法高亮 |
| XSS 防护 | DOMPurify + `html:false` 双重防护 | DOMPurify + 插件链各自防护 |
| 自定义扩展 | 无 | 支持自定义 renderer 模块 |

### 2.10.2 WYSIWYG 支持情况

**评论功能不支持 WYSIWYG 编辑器**，仅支持纯 Markdown 文本输入。证据：

1. `comments.vue:3-16` 的输入框是 `<v-textarea>`，没有富文本编辑器
2. `comments.vue:44-45` 明确提示 "Markdown format" 并显示 Markdown 图标
3. 代码库中搜索不到 `wysiwyg` 或 `editor.*comment` 相关实现

### 2.10.3 完整处理流程

```
用户输入 content
    │
    ├─ 前端校验（validate.js）
    │    ├─ 长度 ≥ 2
    │    └─ Guest 额外校验 name/email
    │
    ▼
┌─ @rateLimit（IP 级别：15 秒 1 次）
│
├─ @auth 权限校验（write:comments）
│
├─ Model 层 checkAccess（页面级权限）
│
└─ Provider 层处理（default/comment.js）
     ├─ 1. Markdown 渲染：mkdown.render(content)
     │       ├─ html: false → 所有 HTML 标签转义
     │       ├─ breaks: true → 换行转 <br>
     │       ├─ linkify: true → URL 自动转链接
     │       └─ emoji 插件 → :emoji: 转 Unicode
     │
     ├─ 2. XSS 防护：DOMPurify.sanitize(renderedHtml)
     │       └─ 即使 Markdown 渲染有漏洞，DOMPurify 兜底
     │
     ├─ 3. Akismet 反垃圾检查（如果配置了 API Key）
     │       ├─ 检查内容、作者名、邮箱、IP、User-Agent
     │       ├─ 根据 userRole（administrator/user/guest）调整评分权重
     │       └─ 标记为 spam 则直接拒绝
     │
     ├─ 4. minDelay 频率控制（用户级别）
     │       └─ 所有 Guest 共享 id=2，因此全站点 Guest 共享此限制
     │
     └─ 5. 写入 DB：content（原始）+ render（HTML）同时存储
```

### 2.10.4 安全层次总结

评论内容经过**五道安全防线**：

1. **输入过滤**：`html: false` 禁止原始 HTML
2. **代码转义**：代码块内容用 `_.escape()` 转义
3. **XSS 清洗**：DOMPurify 对最终 HTML 做白名单过滤
4. **反垃圾**：Akismet 基于内容和用户特征的垃圾检测
5. **频率控制**：IP 级（15s/次）+ 用户级（minDelay 配置）双重限流

---

## 3. 权限校验链路

评论功能的权限校验分布在**三个不同层次**，各层职责不同：

### 3.1 第一层：GraphQL @auth 指令（全局权限门控）

| 操作 | 所需权限（满足任一即可） | 定义位置 |
|------|--------------------------|----------|
| 查看评论列表 | `read:comments` 或 `manage:system` | `comment.graphql:23` |
| 查看单条评论 | `read:comments` 或 `manage:system` | `comment.graphql:27` |
| 创建评论 | `write:comments` 或 `manage:system` | `comment.graphql:45` |
| 更新评论 | `write:comments` 或 `manage:comments` 或 `manage:system` | `comment.graphql:50` |
| 删除评论 | `manage:comments` 或 `manage:system` | `comment.graphql:54` |
| 管理提供者 | `manage:system` | `comment.graphql:18,37` |
| 查看评论原始内容 | `write:comments` 或 `manage:comments` 或 `manage:system` | `comment.graphql:80` |
| 查看评论者邮箱 | `manage:system` | `comment.graphql:84` |
| 查看评论者 IP | `manage:system` | `comment.graphql:85` |

实现机制：`server/graph/directives/auth.js` 中的 `AuthDirective` 类，包装每个字段的 `resolve`，在调用前检查 `context.req.user.permissions` 是否与 `requiredScopes` 有交集。

### 3.2 第二层：WIKI.auth.checkAccess()（页面级权限）

即使通过了全局权限门控，在 Resolver/Model 中还会进行**基于页面路径、语言、标签**的精细化校验：

- `resolver/comment.js:49`：查询评论列表时，先查出页面的 tags，再 `checkAccess(user, ['read:comments'], { tags, locale, path })`
- `models/comments.js:100`：创建评论时，`checkAccess(user, ['write:comments'], { path, locale, tags })`
- `models/comments.js:138`：更新评论时，`checkAccess(user, ['manage:comments'], { path, locale, tags })`
- `models/comments.js:172`：删除评论时，`checkAccess(user, ['manage:comments'], { path, locale, tags })`

`checkAccess()` 定义在 `server/core/auth.js:221`，逻辑：
1. **超级管理员直达**：用户有 `manage:system` → 直接返回 true
2. **全局权限匹配**：用户 permissions 与所需 permissions 求交集，无交集 → false
3. **页面规则匹配**：遍历用户所在组的 `pageRules`，按规则匹配路径（START/END/REGEX/EXACT/TAG），结合优先级和 deny 标记判定

### 3.3 第三层：getEffectivePermissions()（前端展示门控）

`server/core/auth.js:496` 的 `getEffectivePermissions()` 在 SSR 阶段一次性计算出当前用户对当前页面的所有有效权限：

```js
comments: {
  read:  WIKI.config.features.featurePageComments
    ? WIKI.auth.checkAccess(req.user, ['read:comments'], page) : false,
  write: WIKI.config.features.featurePageComments
    ? WIKI.auth.checkAccess(req.user, ['write:comments'], page) : false,
  manage: WIKI.config.features.featurePageComments
    ? WIKI.auth.checkAccess(req.user, ['manage:comments'], page) : false,
}
```

关键点：**评论权限受全局开关 `featurePageComments` 约束**，关闭时所有评论权限返回 false。

这些权限通过 SSR 模板传入前端 → Vuex Store → Vue 组件，控制 UI 元素的显示/隐藏。

---

## 2.11 评论通知、Activity log、点赞收藏、@mention：功能状态与未实现分析

通过全代码库搜索，这四处特性均处于**未实现或部分规划**状态。以下是逐点的代码证据分析。

### 2.11.1 评论通知与 Activity log 的协同触发：不存在

**代码库中不存在评论事件的 event emit 逻辑**。

**事件系统架构**（`server/core/kernel.js:39-41`）：
```js
WIKI.events = {
  inbound: new EventEmitter(),
  outbound: new EventEmitter()
}
```

`db.js:248-256` 的事件总线仅用于跨实例的 HA 缓存失效：
```js
WIKI.events.outbound.onAny(this.notifyViaDB)  // 所有 outbound 事件都转发到 PG NOTIFY
```

**唯一被 emit 的 outbound 事件是 `deletePageFromCache`**（`models/pages.js` 中多处），用于页面变更时的缓存失效。

**全代码库搜索结果**：
- `WIKI.events.outbound.emit` 仅在 `pages.js` 中出现，参数固定为 `'deletePageFromCache'`
- 无任何 emit `commentCreated`、`commentUpdated`、`commentDeleted` 等评论相关事件
- 无 `activityLog`、`activity_log` 或 `commentNotification` 相关表或 Model
- `client/components/profile/profile.vue:344` 中 `commentsPosted` 硬编码为 `0`（注释功能从未对接）
- `client/components/profile/comments.vue` 是空白模板（无具体实现）

**Webhooks 系统**（`admin-webhooks.vue`）：
- 页面仅为框架代码，`save()` 按钮被 `disabled` 硬编码禁用
- Apollo query/mutation 引用的是 `mail-query-config` 和 `mail-mutation-save-config`（邮件配置）
- 实际上是**误用了邮件配置的 GraphQL 操作**，评论 webhook 从未实现

**结论**：
- 评论创建/更新/删除后，**不会**触发任何通知、Activity log 记录或 webhook
- 事件系统架构为 HA 缓存同步而设计，未用于评论通知
- Webhooks 管理页是半成品，不能实际使用

### 2.11.2 评论点赞 / 收藏：数据库和代码均未实现

**数据库层面**：
- `comments` 表（`migrations/2.0.0.js:63-69`）仅有字段：`id, content, createdAt, updatedAt`
- `migrations/2.4.36.js` 新增：`render, name, email, ip`
- `migrations/2.4.61.js` 新增：`replyTo`
- **没有** `likesCount`、`favoritesCount`、`likedBy` 等字段
- **没有** `commentLikes`、`commentFavorites`、`commentReactions` 等关联表

**GraphQL Schema 层面**（`comment.graphql`）：
- `CommentPost` type 仅有：`id, content, render, authorId, authorName, authorEmail, authorIP, createdAt, updatedAt`
- **没有** `like`、`unlike`、`favorite`、`unfavorite` 等 mutation
- **没有** `likes`、`likedByCurrentUser` 等字段

**前端 UI 层面**：
- `comments.vue` 无点赞按钮、收藏按钮、数字计数器
- 全代码库搜索 `mdi-heart`、`mdi-thumb-up`、`mdi-star`、`mdi-like` 等图标，仅在 `admin-contribute.vue` 和 `admin.vue` 中出现，与评论无关
- `page.vue:134-142` 中 `commentsCount` 被注释掉（半成品代码）

**全代码库搜索结果**：
- 无 `comment.*like`、`like.*comment`、`comment.*vote`、`comment.*favorite` 相关代码
- 无 `commentLikes`、`commentReactions` 等表定义

**结论**：
- 评论点赞/收藏功能**完全未实现**
- 数据库没有预留字段，GraphQL 没有定义，前端没有 UI

### 2.11.3 评论 @mention：仅在 CKEditor 中有 TODO 注释，无实际实现

**最接近的代码证据**在 `client/components/editor/editor-ckeditor.vue:72-82`：
```js
// TODO: Mention autocomplete
//
// mention: {
//   feeds: [
//     {
//       marker: '@',
//       feed: [ '@Barney', '@Lily', '@Marshall', '@Robin', '@Ted' ],
//       minimumCharacters: 1
//     }
//   ]
// },
```

这段是**页面编辑器**的 CKEditor 配置中的 TODO 注释，与评论功能无关。而且 feed 数据是硬编码的示例数组（《老爸老妈的浪漫史》角色名），从未对接实际用户搜索。

**评论组件层面**（`comments.vue`）：
- 输入框是 `<v-textarea>`（纯文本），**不支持 CKEditor**
- 没有 mention 自动补全
- 没有 `@` 字符的事件监听
- 没有用户搜索弹窗（`user-search.vue` 组件存在但从未在评论模块中被引用）

**后端层面**：
- 无 `parseMentions()`、`extractMentions()`、`notifyMentionedUsers()` 等函数
- 无 `mention` 相关的权限校验
- 无评论内容中 `@username` 模式的正则匹配
- 无通知发送逻辑（站内信、邮件等）

**GraphQL Schema 层面**：
- 无 `sendMentionNotification`、`resolveMention` 等 mutation
- 无 `mentionedUsers` 字段

**结论**：
- 评论 @mention 功能**完全未实现**
- 仅在页面编辑器的 CKEditor 配置中有一个占位 TODO，且是示例代码
- 评论输入是纯文本 textarea，没有富文本编辑能力，无法实现 mention 自动补全
- 后端无 mention 解析、权限校验、通知触发的任何代码

### 2.11.4 三项功能的总体现状总结

| 功能 | 数据库 | GraphQL Schema | 后端逻辑 | 前端 UI | 状态 |
|------|--------|----------------|----------|---------|------|
| 评论通知 / Activity log | 无相关表 | 无相关字段 | 无 event emit | 仅空白模板 `profile/comments.vue` | ❌ 未实现 |
| 评论点赞 / 收藏 | 无字段无表 | 无 mutation/字段 | 无逻辑 | 无按钮无计数 | ❌ 未实现 |
| 评论 @mention | 无相关表 | 无相关字段 | 无解析无通知 | 仅 textarea，无补全 | ❌ 未实现 |
| Webhooks 评论事件 | 无相关表 | 无相关 mutation | 无逻辑 | 管理页半成品，按钮禁用 | ❌ 未实现 |

**唯一的半成品残留**：
1. `editor-ckeditor.vue:72-82` 中页面编辑器的 mention TODO 注释（与评论无关）
2. `page.vue:134-142` 中 `commentsCount` 显示代码被注释掉
3. `profile.vue:344` 中 `commentsPosted` 硬编码为 `0`
4. `profile/comments.vue` 空白组件骨架
5. `admin-webhooks.vue` 误用邮件配置的半成品页面

---

## 4. 页面呈现链路

### 4.1 SSR 阶段：页面请求 → 服务端渲染

**控制器**：`server/controllers/common.js:417`（`GET /*` 路由）

1. 解析 URL → `pageHelper.parsePath()`
2. 从缓存/DB 获取页面 → `WIKI.models.pages.getPage()`
3. **计算有效权限** → `WIKI.auth.getEffectivePermissions(req, pageArgs)`（:441）
4. 检查 `effectivePermissions.pages.read`，无权限返回 403
5. **构建评论模板变量**（:530-545）：
   ```js
   const commentTmpl = {
     codeTemplate: WIKI.data.commentProvider.codeTemplate,
     head: WIKI.data.commentProvider.head,
     body: WIKI.data.commentProvider.body,
     main: WIKI.data.commentProvider.main
   }
   ```
   - 如果 `featurePageComments` 启用且 provider 有 `codeTemplate`，替换 `{{pageUrl}}` 和 `{{pageId}}` 占位符
   - 如果没有 `codeTemplate`（默认内置提供者），`main` 为 `<comments></comments>`（Vue 组件占位符）

6. **渲染模板** → `res.render('page', { page, sidebar, injectCode, comments: commentTmpl, effectivePermissions, pageFilename })`

### 4.2 Pug 模板：`server/views/page.pug`

```pug
page(
  :comments-enabled=config.features.featurePageComments
  :effective-permissions=Buffer.from(JSON.stringify(effectivePermissions)).toString('base64')
  :comments-external=comments.codeTemplate
)
  template(slot='contents')
    div!= page.render
  template(slot='comments')
    div!= comments.main     ← 内置提供者时为 <comments></comments>
```

- `comments-enabled`：布尔值，来自全局特性开关
- `effective-permissions`：Base64 编码的 JSON 字符串
- `comments-external`：是否有外部代码模板（如 Disqus）
- `comments.main`：内置提供者为 `<comments></comments>`；外部提供者为第三方嵌入代码

### 4.3 主题层组件：`client/themes/default/components/page.vue`

1. **接收 props**：`commentsEnabled`、`effectivePermissions`、`commentsExternal`
2. **写入 Vuex Store**（:599-604）：
   ```js
   this.$store.set('page/effectivePermissions',
     JSON.parse(Buffer.from(this.effectivePermissions, 'base64').toString()))
   ```
3. **条件渲染评论区**（:331）：
   ```pug
   .comments-container#discussion(v-if='commentsEnabled && commentsPerms.read && !printView')
     .comments-main
       slot(name='comments')    ← 这里渲染 comments.vue 组件
   ```
4. **侧边栏评论卡片**（:129）：
   ```pug
   v-card.page-comments-card(v-if='commentsEnabled && commentsPerms.read')
   ```
5. **权限来源**（:532）：
   ```js
   commentsPerms: get('page/effectivePermissions@comments')
   ```
   即从 Vuex 的 `page/effectivePermissions.comments` 取值

### 4.4 Vuex Store：`client/store/page.js`

- `effectivePermissions.comments` 的默认值：`{ read: false, write: false, manage: false }`
- 在 `page.vue` 的 `created()` 钩子中被 SSR 传入的值覆盖
- `comments.vue` 通过 `get('page/effectivePermissions@comments')` 读取

### 4.5 评论组件：`client/components/comments.vue`

- **挂载触发**：`v-intersect.once='onIntersect'`，当评论区进入视口时才加载评论列表（懒加载）
- **读取权限**（:165）：
  ```js
  permissions: get('page/effectivePermissions@comments')
  ```
  得到 `{ read, write, manage }`

- **UI 权限控制**：

  | UI 元素 | 显示条件 | 权限 |
  |---------|----------|------|
  | 新评论输入框 | `permissions.write` | write:comments |
  | 访客姓名/邮箱 | `!isAuthenticated && permissions.write` | write:comments（Guest） |
  | 发布按钮 | `permissions.write` | write:comments |
  | 编辑/删除图标 | `permissions.manage` | manage:comments |
  | "Be the first" 提示 | `permissions.write`（无评论时） | write:comments |
  | "No comments" 提示 | 无 write 权限时 | - |

- **加载评论列表**（`fetch()` :175）：Apollo Query `comments.list(locale, path)` → Resolver 查 DB
- **创建评论**（`postComment()` :223）：Apollo Mutation `comments.create(...)` → 成功后 `fetch()` 刷新
- **编辑评论**（`editComment()` :330）：先 `comments.single(id)` 获取原始 content → 显示编辑框
- **更新评论**（`updateComment()` :375）：Apollo Mutation `comments.update(id, content)`
- **删除评论**（`deleteComment()` :446）：Apollo Mutation `comments.delete(id)`

---

## 5. CommentProvider 机制

评论功能支持可插拔的 Provider 架构：

### 5.1 Provider 注册与初始化

1. **启动时扫描磁盘**（`kernel.js:74`）：
   ```js
   await WIKI.models.commentProviders.refreshProvidersFromDisk()
   ```
   扫描 `server/modules/comments/` 目录下的 `definition.yml`，写入 `commentProviders` 表

2. **初始化启用的 Provider**（`kernel.js:84`）：
   ```js
   await WIKI.models.commentProviders.initProvider()
   ```

3. `initProvider()`（`models/commentProviders.js:101`）：
   - 查询 `commentProviders` 表中 `isEnabled=true` 的记录
   - 如果有 `codeTemplate`（如 Disqus），加载 `code.yml` 模板，替换配置占位符
   - 如果没有 `codeTemplate`（默认内置），`require()` 对应目录的 `comment.js` 模块并调用 `init()`
   - 结果存入 `WIKI.data.commentProvider`

### 5.2 内置 vs 外部 Provider

| 特征 | 内置（Default） | 外部（如 Disqus） |
|------|----------------|-------------------|
| codeTemplate | 无 | 有 |
| comments.main | `<comments></comments>` | 第三方嵌入 HTML |
| 评论存储 | 本地 DB `comments` 表 | 第三方服务 |
| 反垃圾 | Akismet | 第三方服务自带 |
| 管理界面 | admin-comments.vue 配置 | admin-comments.vue 配置 |

### 5.3 管理界面：`client/components/admin/admin-comments.vue`

- Apollo Query `comments.providers` 获取所有 Provider 列表
- 选择并配置 Provider → Apollo Mutation `comments.updateProviders`
- 仅 `manage:system` 权限可访问

---

## 6. 关键文件索引

| 文件 | 职责 |
|------|------|
| `server/graph/schemas/comment.graphql` | GraphQL 类型与权限定义 |
| `server/graph/resolvers/comment.js` | GraphQL 解析器，串联权限与 Model |
| `server/graph/directives/auth.js` | @auth 指令，全局权限拦截 |
| `server/graph/directives/rate-limit.js` | @rateLimit 指令，频率限制 |
| `server/models/comments.js` | Comment Model，核心业务逻辑 + 页面级权限校验 |
| `server/models/users.js` | Guest 用户定义（id=2）、getGuestUser() |
| `server/models/commentProviders.js` | Provider 注册、初始化、磁盘扫描 |
| `server/modules/comments/default/comment.js` | 内置 Provider：Markdown渲染、Akismet、入库 |
| `server/modules/comments/default/definition.yml` | 内置 Provider 配置（akismet, minDelay 参数） |
| `server/db/migrations/2.0.0.js` | comments 表初始建表（id, content, createdAt, updatedAt） |
| `server/db/migrations/2.4.36.js` | comments 表新增 render, name, email, ip 字段 |
| `server/db/migrations/2.4.61.js` | comments 表新增 replyTo 字段 |
| `server/core/kernel.js` | 事件系统初始化（inbound/outbound EventEmitter） |
| `server/core/db.js` | High-Availability 事件总线（仅用于缓存失效） |
| `server/core/auth.js` | `checkAccess()` 页面级权限引擎、`getEffectivePermissions()` |
| `server/controllers/common.js` | SSR 控制器，计算 effectivePermissions + 注入评论模板 |
| `server/views/page.pug` | SSR 模板，传递 comments 变量到 Vue 组件 |
| `client/themes/default/components/page.vue` | 主题页面组件，条件渲染评论区 |
| `client/store/page.js` | Vuex Store，存储 effectivePermissions |
| `client/components/comments.vue` | 评论交互组件，所有 CRUD 操作 |
| `client/components/common/user-search.vue` | 用户搜索组件（未在评论模块使用） |
| `client/components/profile/profile.vue` | 个人主页 Activity 卡片（commentsPosted 硬编码为 0） |
| `client/components/profile/comments.vue` | 我的评论列表（空白模板，未实现） |
| `client/components/admin/admin-comments.vue` | 管理后台评论 Provider 配置 |
| `client/components/admin/admin-webhooks.vue` | Webhooks 管理（半成品，按钮禁用，误用邮件配置） |
| `client/components/editor/editor-ckeditor.vue` | 页面编辑器（含 mention TODO 注释，与评论无关） |

---

## 7. 完整旁路分析总结

### 7.1 匿名 vs 已登录用户差异总览

```
┌─────────────────────────────────────────────────────────────────┐
│              Guest (id=2)               │   已登录用户 (id!=2)   │
├─────────────────────────────────────────┼────────────────────────┤
│  前端显示 guestName/guestEmail 输入框     │  显示"Posting as 用户名"│
│  前端额外校验 name/email                │  仅校验 content       │
│  后端额外校验 guestEmail/guestName       │  无额外校验           │
│  name/email 被 guestName/guestEmail 覆盖 │  使用用户本身的 name   │
│  Akismet role=guest（更严格）            │  role=user            │
│  所有 Guest 共享 minDelay 计数器         │  独立 minDelay 计数器  │
│  存储 authorId=2                        │  存储 authorId=用户ID  │
└─────────────────────────────────────────┴────────────────────────┘
```

### 7.2 编辑/删除机制的设计限制

- **无版本历史**：更新直接覆盖 `content` 和 `render` 字段，不保存旧版本
- **硬删除**：`DELETE` 语句物理删除，不可恢复
- **无状态机**：没有 `status` 字段，评论只有"存在/不存在"两种状态
- **无作者权限**：作者本人不能编辑/删除自己的评论，需要 `manage:comments` 权限

### 7.3 渲染与安全管道关键参数

| 组件 | 参数 | 值 | 目的 |
|------|------|----|------|
| markdown-it | `html: false` | 禁止原始 HTML | 防止 XSS 第一层 |
| markdown-it | `breaks: true` | 换行转 `<br>` | 用户体验 |
| markdown-it | `linkify: true` | 自动识别链接 | 用户体验 |
| DOMPurify | 默认配置 | 白名单过滤 | 防止 XSS 第二层 |
| Akismet | `role=guest/user/admin` | 差异化评分 | 反垃圾 |
| @rateLimit | `1/15s per IP` | IP 级限流 | 防刷屏 |
| minDelay | 默认 30s | 用户级限流 | 防刷屏 |

---

## 8. 权限决策流程总结（含旁路）

```
请求到达
  │
  ├─ GraphQL @auth 指令
  │    检查用户全局 permissions 是否包含所需权限
  │    ↓ 通过
  │
  ├─ @rateLimit（创建评论时）
  │    同一 IP 15 秒内只允许 1 次
  │    ↓ 通过
  │
  ├─ Resolver / Model 层
  │    WIKI.auth.checkAccess(user, permissions, { path, locale, tags })
  │    ├─ manage:system → 直接放行
  │    ├─ 全局权限交集 → 无交集则拒绝
  │    └─ 页面规则匹配 → 按 START/END/REGEX/EXACT/TAG + 优先级 + deny 判定
  │    ↓ 通过
  │
  ├─ Model 层差异化校验（仅创建时）
  │    ├─ user.id === 2 (Guest) → 额外校验 guestName + guestEmail
  │    └─ user.id !== 2 → 直接通过
  │    ↓ 通过
  │
  ├─ Provider 层处理（内置 Default）
  │    ├─ 1. Markdown 渲染：html:false + breaks:true + linkify:true + emoji
  │    ├─ 2. XSS 防护：DOMPurify.sanitize()
  │    ├─ 3. Akismet 反垃圾（配置了 API Key 时）
  │    │    ├─ Guest → role=guest（更严格评分）
  │    │    └─ 已登录 → role=user
  │    ├─ 4. minDelay 频率控制
  │    │    ├─ Guest → 全站点共享 id=2 的计数器
  │    │    └─ 已登录 → 独立计数器
  │    └─ 5. 写入 DB：content(原始) + render(HTML)
  │
  │    ⚠️  以下功能均未实现：
  │    ├─ ❌ 评论通知 / Activity log（无 event emit）
  │    ├─ ❌ 点赞 / 收藏（无数据表、无 mutation）
  │    └─ ❌ @mention 解析与通知（仅页面编辑器 TODO 注释）
  │
  └─ 前端 UI 门控
       effectivePermissions.comments.{read,write,manage}
       ├─ read → 是否显示评论区
       ├─ write → 是否显示输入框和发布按钮
       │    └─ !isAuthenticated → 额外显示 guestName/guestEmail 字段
       └─ manage → 是否显示编辑/删除操作
            └─ 注意：作者本人也需要 manage:comments 权限

额外约束：
  - featurePageComments 全局开关关闭 → 所有评论权限为 false
  - 编辑/删除无状态机：更新直接覆盖，删除物理删除，无版本历史
  - 无 WYSIWYG：仅纯 Markdown 文本输入，使用独立的轻量级渲染管道
  - 无通知系统：评论事件无 event emit，Webhooks 管理页是半成品
  - 无社交互动：点赞、收藏、@mention 均未实现
```
