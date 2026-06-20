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
  │     ❌ 无审核状态机 → Akismet 要么拒绝要么直接入库（无待审）
  │     ❌ 无 IP 黑名单 → 仅有账号级 isActive 封禁
  │     ❌ 无附件上传 → 仅能引用外部图片 URL（无病毒扫描）
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
| 评论审核 / 待审 | 无 status 字段 | 无 approve/reject mutation | 无状态机 | 无审核 UI | ❌ 未实现 |
| 评论 IP 黑名单 | 无 IP 封禁表 | 无相关 mutation | 无封禁逻辑 | 无管理 UI | ❌ 未实现 |
| 评论附件上传 | 无 commentAttachments 表 | 无 upload mutation | 无上传逻辑 | 无上传控件 | ❌ 未实现 |
| 评论病毒扫描 | — | — | 无 ClamAV/扫描器 | — | ❌ 未实现 |

**唯一的半成品残留**：
1. `editor-ckeditor.vue:72-82` 中页面编辑器的 mention TODO 注释（与评论无关）
2. `page.vue:134-142` 中 `commentsCount` 显示代码被注释掉
3. `profile.vue:344` 中 `commentsPosted` 硬编码为 `0`
4. `profile/comments.vue` 空白组件骨架
5. `admin-webhooks.vue` 误用邮件配置的半成品页面

---

## 2.12 评论审核 / 待审状态机：完全未实现

### 2.12.1 数据库结构分析

`comments` 表的字段演进（三次迁移）：

| 迁移版本 | 新增字段 |
|----------|---------|
| `2.0.0.js:63-69` | `id, content, createdAt, updatedAt` |
| `2.4.36.js` | `render, name, email, ip` |
| `2.4.61.js` | `replyTo` |

**没有任何状态字段**：
- 无 `status` / `state` 字段
- 无 `isApproved` / `approvedAt` / `approvedBy`
- 无 `isPending` / `pendingReason`
- 无 `isSpam` / `spamScore`
- 无 `moderatedAt` / `moderatedBy`
- 无 `commentReview` / `commentModerations` 关联表

### 2.12.2 GraphQL Schema 分析（`comment.graphql`）

```graphql
type CommentPost {
  id: Int!
  content: String! @auth(requires: [...])
  render: String!
  authorId: Int!
  authorName: String!
  authorEmail: String! @auth(requires: ["manage:system"])
  authorIP: String! @auth(requires: ["manage:system"])
  createdAt: Date!
  updatedAt: Date!
}
```

**没有** `status` / `isApproved` / `isPending` 等状态字段。

Mutation 中**没有**：
- `approveComment` / `rejectComment`
- `setCommentStatus`
- `submitForReview`
- `bulkModerateComments`

Query 中**没有**按状态筛选的参数（如 `list(status: PENDING)`）。

### 2.12.3 后端逻辑分析

`Resolver.list()`（`resolver/comment.js:50`）：
```js
const comments = await WIKI.models.comments.query()
  .where('pageId', page.id)
  .orderBy('createdAt')
```

**没有** `where('status', 'APPROVED')` 这样的过滤条件，**所有评论一创建就直接显示**。

`Provider.create()`（`default/comment.js:118`）：
```js
const newComment = await WIKI.models.comments.query().insert(...)
return newComment.id
```

创建后直接 `insert` 入库，**没有待审队列，没有审核流程**。

### 2.12.4 Akismet 反垃圾的唯一处理

**Akismet 标记为 spam 的评论是直接拒绝，不是进入待审队列**：

```js
// default/comment.js:104-106
if (isSpam) {
  throw new Error('Comment was rejected because it is marked as spam.')
}
```

- `isSpam = true` → 直接 `throw`，评论**不入库**，前端显示错误信息
- `isSpam = false` → 直接 `insert`，立即显示
- **没有中间状态**（如 `PENDING` / `REVIEW`）

### 2.12.5 前端 UI 分析

`comments.vue` 中：
- 没有"待审核"状态徽章或提示
- 没有审核队列管理界面
- 没有"批准/拒绝"操作按钮
- `admin-comments.vue` 仅配置 Provider 参数（Akismet Key、minDelay），**不管理评论内容**

### 2.12.6 审核状态机总结

| 设计要素 | 状态 | 代码证据 |
|----------|------|---------|
| 状态字段 | ❌ 未实现 | 数据库无 `status`/`isApproved` 等字段 |
| 审核队列表 | ❌ 未实现 | 无 `commentModerations` 等关联表 |
| 按状态过滤查询 | ❌ 未实现 | Resolver.list() 无 status filter |
| 审核 mutation | ❌ 未实现 | Schema 无 approve/reject mutation |
| 待审 UI 提示 | ❌ 未实现 | 前端无 PENDING 状态显示 |
| 审核管理界面 | ❌ 未实现 | admin-comments 仅配置参数 |
| Akismet spam 处理 | ⚠️ 直接拒绝 | 不进入待审，直接 throw 错误 |

**结论**：评论审核功能完全未实现。Akismet 只做"二进制判定"——要么直接入库，要么直接拒绝，没有中间审核流程。

---

## 2.13 评论 IP 限流、反垃圾和黑名单：已有反垃圾与限流的完整分析

评论的反垃圾和限流策略分为**四层防线**，全部在 `default/comment.js` 和 GraphQL 指令中实现。但**IP 黑名单功能不存在**，只有用户级（isActive）封禁。

### 2.13.1 第一层：IP 级频率限制（GraphQL 指令）

`server/graph/schemas/comment.graphql:45`：
```graphql
create(...) @rateLimit(limit: 1, duration: 15)
```

`server/graph/directives/rate-limit.js:1-5` 实现：
```js
const { createRateLimitDirective } = require('graphql-rate-limit-directive')

module.exports = createRateLimitDirective({
  keyGenerator: (directiveArgs, source, args, context, info) =>
    `${context.req.ip}:${info.parentType}.${info.fieldName}`
})
```

**机制**：
- 以 `IP:父类型.字段名` 为 key（如 `192.168.1.1:CommentMutation.create`）
- 同一 IP 对 `comments.create` mutation 每 15 秒只能调用 1 次
- 使用 `graphql-rate-limit-directive` 库（基于内存的滑动窗口）
- **注意**：多实例部署时是**每实例独立计数**，不是全局共享

### 2.13.2 第二层：用户级最小发言间隔（Provider 配置）

`definition.yml:17-23` 配置 `minDelay`（默认 30 秒）：
```js
// default/comment.js:110-120
if (WIKI.data.commentProvider.config.minDelay > 0) {
  const lastComment = await WIKI.models.comments.query()
    .select('updatedAt')
    .findOne('authorId', user.id)   // ← 按 authorId 过滤
    .orderBy('updatedAt', 'desc')
  if (lastComment) {
    const timeDiff = Date.now() - new Date(lastComment.updatedAt).getTime()
    if (timeDiff < (WIKI.data.commentProvider.config.minDelay * 1000)) {
      throw new Error('Minimum delay between comments not elapsed.')
    }
  }
}
```

**关键细节**：
- 按 `authorId` 查询，**所有 Guest 用户共享 id=2**，即全站点匿名用户共用一个计数器
- 用 `updatedAt`（不是 `createdAt`），意味着编辑评论也会重置计时器
- 配置值 `0` 表示禁用此限制
- 与 IP 限流的区别：IP 限流是 15s（不可配置），用户级是 minDelay 秒（可配置，默认 30s）

### 2.13.3 第三层：Akismet 反垃圾检测

`default/comment.js:75-107` 完整实现：

```js
if (!_.isEmpty(WIKI.data.commentProvider.config.akismet)) {
  const akismetClient = new AkismetClient({
    apiKey: WIKI.data.commentProvider.config.akismet,
    blog: WIKI.config.host,
    debug: _.get(WIKI.data.commentProvider.config, 'debugAkismet', false)
  })

  let userRole = 'user'
  if (user.groups.indexOf(1) >= 0) {
    userRole = 'administrator'
  } else if (user.groups.indexOf(2) >= 0) {
    userRole = 'guest'          // ← Guest 组用户标记为 guest，评分更严格
  }

  let isSpam = false
  try {
    isSpam = await akismetClient.checkSpam({
      ip: user.ip,
      useragent: user.agentagent,   // ⚠️ BUG: 应为 user.userAgent（多了一个 agent）
      content,
      name: user.name,
      email: user.email,
      role: userRole
    })
  } catch (err) {
    WIKI.logger.warn('Akismet Comment Validation: [ FAILED ]')
    WIKI.logger.warn(err)
  }

  if (isSpam) {
    throw new Error('Comment was rejected because it is marked as spam.')
  }
}
```

**已知 Bug**（`default/comment.js:90`）：
```js
useragent: user.agentagent,   // 应该是 user.userAgent
```
参数名拼写错误，`user` 对象中没有 `agentagent` 字段，Akismet 实际上**永远收不到 user-agent**，可能降低 spam 检测准确率。

### 2.13.4 第四层：用户账号级封禁

系统有 `users.isActive` 字段（`models/users.js:36`），但这是**全局账号封禁**，不是评论专属黑名单：

- 登录时检查（`users.js:230-232`）：
  ```js
  if (!user.isActive) {
    throw new WIKI.Error.AuthAccountBanned()
  }
  ```
- Token 刷新时检查（`users.js:427-430`）
- API Key 验证时检查（`users.js:502-504`）

**封禁效果**：用户完全无法登录，自然也无法发表评论。这是最严格的封禁级别，但**不支持"仅禁止评论"的细粒度封禁**。

### 2.13.5 不存在的功能：IP 黑名单

全代码库搜索结果：
- 无 `blocklist` / `blacklist` / `denylist` 相关配置
- 无 IP 封禁表（如 `ipBans` / `blockedIps`）
- 无 Akismet 命中后自动封禁 IP 的逻辑
- 无手动封禁某个 IP 的管理功能

`admin-security.vue`（安全管理页面）中也没有 IP 黑名单相关 UI。

### 2.13.6 限流与反垃圾总览

| 防线 | 实现位置 | 类型 | 作用范围 | 可配置？ |
|------|---------|------|---------|---------|
| 第一 | `@rateLimit(1, 15s)` | IP 级限流 | 创建评论 | ❌ 硬编码 |
| 第二 | `minDelay`（默认 30s） | 用户级限流 | 创建评论 | ✅ Provider 配置 |
| 第三 | Akismet `checkSpam()` | 内容级反垃圾 | 文本内容 + 用户角色 | ✅ API Key 配置 |
| 第四 | `users.isActive` | 账号级封禁 | 全站登录 | ✅ 用户管理 |
| — | IP 黑名单 | IP 级封禁 | — | ❌ 未实现 |
| — | 待审状态机 | 人工审核流程 | — | ❌ 未实现 |

---

## 2.14 评论附件上传与安全过滤、病毒扫描：完全未实现

### 2.14.1 评论 Schema 中无上传相关定义

`comment.graphql` 中：
- 无 `commentAttachments` / `uploadCommentAttachment` mutation
- 无 `CommentAttachment` type
- `CommentPost` 中无 `attachments` 字段

### 2.14.2 评论组件无上传 UI

`client/components/comments.vue` 完整模板分析：
- 输入框是 `<v-textarea>` 纯文本（:3-16）
- **无文件选择按钮**（`<input type="file">` 或类似 UI）
- **无图片拖拽上传区域**
- **无附件列表显示**
- 工具栏只有 Post/Cancel 按钮，无媒体选择器

### 2.14.3 资产（Asset）系统的独立存在

Wiki.js 有独立的资产上传系统，但**与评论功能完全隔离**：

**Asset Schema**（`asset.graphql`）：
```graphql
type AssetItem {
  id: Int!
  filename: String!
  ext: String!
  kind: AssetKind!         // 'binary' | 'image'
  mime: String!
  fileSize: Int!
  metadata: String
  createdAt: Date!
  updatedAt: Date!
}

type AssetMutation {
  createFolder(...)
  renameAsset(...)
  flushTempUploads
}
```

**Asset 表结构**（`migrations/2.0.0.js:23-32`）：
```js
.createTable('assets', table => {
  table.string('filename').notNullable()
  table.string('hash').notNullable()
  table.string('ext').notNullable()
  table.enum('kind', ['binary', 'image']).notNullable()
  table.string('mime').notNullable().defaultTo('application/octet-stream')
  table.integer('fileSize').unsigned().comment('In kilobytes')
  table.json('metadata')
  table.string('createdAt').notNullable()
  table.string('updatedAt').notNullable()
  table.integer('folderId').unsigned().references('id').inTable('assetFolders')
  table.integer('authorId').unsigned().references('id').inTable('users')
})
```

Asset 系统仅用于**页面编辑器**中的图片和附件（通过 `editor-modal-media.vue` 管理），评论模块完全不使用它。

### 2.14.4 全代码库无病毒扫描实现

全代码库搜索：
- 无 `clamav` / `clamd` / `clamscan`
- 无 `antivirus` / `malware`
- 无 `virus.*scan` / `infected` / `cleanFile`
- 无 MIME 类型白名单校验（在评论上传上下文中）
- 无文件扩展名黑名单（在评论上传上下文中）

`StorageResolver`（`resolver/storage.js`）只处理存储目标（S3/Local FS 等）的配置与同步，**不做任何文件安全扫描**。

`editor-modal-media.vue`（页面媒体上传对话框）中即使有上传逻辑，也没有病毒扫描相关代码——Asset 系统本身也没有病毒扫描。

### 2.14.5 评论内容中的图片

虽然评论不能上传文件，但**Markdown 中可引用外部图片 URL**：

Markdown 渲染（`default/comment.js:14-19`）配置：
```js
html: false,   // 禁止 <img> 标签（XSS 防护）
// ...
```

由于 `html: false`，用户无法用 `<img src="...">` 嵌入图片。但 Markdown 的 `![alt](url)` 语法在 `markdown-it` 默认支持，**外部图片链接可以渲染**。

DOMPurify 会对最终生成的 `<img>` 标签做白名单检查，但这是**对已渲染 HTML 的清洗**，不是对图片内容的病毒扫描。

### 2.14.6 附件上传总结

| 功能 | 数据库 | GraphQL Schema | 后端逻辑 | 前端 UI | 状态 |
|------|--------|----------------|----------|---------|------|
| 评论附件上传表 | ❌ 无 `commentAttachments` 表 | ❌ 无 mutation | ❌ 无处理逻辑 | ❌ 无上传控件 | ❌ 未实现 |
| 评论文件类型校验 | — | — | ❌ 无 MIME/ext 校验 | — | ❌ 未实现 |
| 评论病毒扫描 | — | — | ❌ 无 ClamAV/扫描器 | — | ❌ 未实现 |
| 资产系统对接评论 | ❌ 无 comment-fk | ❌ 无关联字段 | ❌ 无关联逻辑 | ❌ 无评论媒体按钮 | ❌ 未实现 |
| 页面 Asset 系统 | ✅ `assets` 表 | ✅ `asset.graphql` | ✅ 上传/文件夹逻辑 | ✅ `editor-modal-media.vue` | ✅ 独立存在（仅供页面编辑器） |
| **批量删除 / 批量改状态** | ❌ 无状态字段 | ❌ 无 bulk mutation | ❌ 无 whereIn().delete() | ❌ 无全选/批量菜单 | ❌ 未实现 |
| **批量导入** | — | ❌ 无 import mutation | ❌ 无导入逻辑 | ❌ 无 UI | ❌ 未实现 |
| **全量 JSON 导出** | — | ✅ `system.export` | ✅ 流式 50 条/批 + Gzip | ✅ System 模块 | ✅ 已实现（全量） |
| **API Key 按组鉴权** | ✅ `apiKeys` 表 | ✅ `authentication.createApiKey` | ✅ JWT RS256 + grp 映射 | ✅ `admin-api.vue` | ✅ 完整实现 |
| **Schema 自动迁移** | ✅ `migrations` 表 | — | ✅ Knex migrate.latest() + semver 排序 | — | ✅ 完整实现 |
| **V1→V2 评论迁移** | — | — | ❌ 仅迁移用户，评论被丢弃 | — | ❌ 未实现 |
| **跨实例增量同步** | — | — | ❌ 无队列/消息机制 | — | ❌ 未实现 |

**结论**：评论不能上传任何文件或图片附件。用户只能通过 Markdown 的 `![alt](url)` 引用**外部**图片 URL，但图片内容不会经过病毒扫描。批量操作、批量导入、跨实例增量同步均未实现，仅 System Export 支持全量 Gzip JSON 导出。

---

## 2.15 评论批量操作：仅单个删除，无批量操作

### 2.15.1 代码库中无批量操作 API

`comment.graphql` Mutation 中仅定义了**单条操作**：

```graphql
type CommentMutation {
  updateProviders(...)           # 管理后台：配置 Provider
  create(pageId, replyTo, content, guestName, guestEmail)  # 单条创建
  update(id, content)            # 单条更新
  delete(id)                     # 单条删除（按 ID）
}
```

**没有**以下批量操作 API：
- ❌ `bulkDelete(ids: [Int!]!)` — 批量删除
- ❌ `bulkUpdate(ids: [Int!]!, status/action)` — 批量改状态
- ❌ `bulkExport` / `exportCommentsByPage` — 导出特定页面评论
- ❌ `bulkImport` — 从 JSON 批量导入

### 2.15.2 Resolver 仅实现单条操作

`server/graph/resolvers/comment.js` 的四个方法全是单条粒度：
```js
Mutation: {
  async comments() { return {} }
},
CommentMutation: {
  async updateProviders(...) { ... },
  async create(obj, args, context) {
    return WIKI.models.comments.postNewComment({ ...args, ... })  // 单条
  },
  async update(obj, args, context) {
    return WIKI.models.comments.updateComment({ ...args, ... })    // 单条
  },
  async delete(obj, args, context) {
    await WIKI.data.commentProvider.remove({ id: args.id, ... })   // 按 ID 单条删除
    return { responseResult: graphHelper.generateSuccess('...') }
  }
}
```

Provider 层的 `remove()` 也是单条：
```js
// default/comment.js:137-139
async remove ({ id, user }) {
  return WIKI.models.comments.query().findById(id).delete()  // DELETE WHERE id = ?
}
```

**没有** `deleteMany().whereIn('id', ids)` 或 `patch().whereIn(...)` 的批量 SQL。

### 2.15.3 前端 UI 仅单条操作按钮

`client/components/comments.vue:68-87` 中每条评论下方仅显示**单条**操作按钮：
```pug
span.ml-2(v-if='permissions.manage')
  v-tooltip(top, color='grey')
    template(v-slot:activator='{ on }')
      v-btn.animated.fadeInLeft(icon, x-small, @click='edit(comment)')
        v-icon(color='primary', small) mdi-pencil
  v-tooltip(top, color='grey')
    template(v-slot:activator='{ on }')
      v-btn.animated.fadeInLeft.wait-p1s(icon, x-small, @click='remove(comment)')
        v-icon(color='error', small) mdi-delete
```

- 无"全选"复选框
- 无"批量操作"下拉菜单
- 管理后台 `admin-comments.vue` 仅配置 Provider 参数，**不管理评论内容**

### 2.15.4 唯一的导出路径：System Export（全量）

评论数据的唯一"批量"操作是系统导出功能，但这是**全量导出所有评论**，不能按条件筛选：

GraphQL 入口（`system.graphql:52-55`）：
```graphql
export(entities: [String]!, path: String!): DefaultResponse @auth(requires: ["manage:system"])
```

Resolver（`resolver/system.js:280-300`）：
```js
async export (obj, args, context) {
  // ... 校验目录为空、不重复运行
  WIKI.system.export({
    entities: args.entities,   // 如 ['comments', 'pages', ...]
    path: desiredPath
  })
  // 异步启动，立即返回
  return { responseResult: graphHelper.generateSuccess('Export started...') }
}
```

导出执行细节（`core/system.js:141-198`）见 **2.17 节**。

### 2.15.5 批量操作总结

| 批量功能 | GraphQL API | 后端实现 | 前端 UI | 状态 |
|----------|-------------|----------|---------|------|
| 批量删除 | ❌ 无 `bulkDelete` | ❌ 无 `whereIn().delete()` | ❌ 无全选/批量菜单 | ❌ 未实现 |
| 批量改状态 | ❌ 无 mutation | ❌ 无 status 字段 | ❌ 无此功能 | ❌ 未实现 |
| 按条件导出 | ❌ 无参数 | ❌ 仅全量导出 | ❌ 仅 System Export 页 | ❌ 未实现 |
| 批量导入 | ❌ 无 import mutation | ❌ 无导入逻辑 | ❌ 无 UI | ❌ 未实现 |
| 全量导出 | ✅ `system.export` | ✅ 流式 + Gzip | ✅ 管理后台（System 模块） | ✅ 已实现（全量） |
| 单条 CRUD | ✅ 齐全 | ✅ 齐全 | ✅ 齐全 | ✅ 已实现 |

---

## 2.16 评论与 API 鉴权策略、第三方插件读评论的权限边界

评论功能与 REST/GraphQL API 的鉴权使用**同一套 JWT 体系**，API Key 支持"按组分配权限"的粒度，评论权限边界完全由现有三层权限系统控制。

### 2.16.1 API Key 创建机制

`server/models/apiKeys.js:43-70` 创建 API Key：

```js
static async createNewKey ({ name, expiration, fullAccess, group }) {
  const entry = await WIKI.models.apiKeys.query().insert({
    name,
    key: 'pending',
    expiration: moment.utc().add(ms(expiration), 'ms').toISO(),
    isRevoked: true   // 先标记为 revoked
  })

  const key = jwt.sign({
    api: entry.id,          // ← 标识这是 API Key 请求
    grp: fullAccess ? 1 : group  // ← fullAccess=1(Admin组)，否则指定组
  }, {
    key: WIKI.config.certs.private,
    passphrase: WIKI.config.sessionSecret
  }, {
    algorithm: 'RS256',
    expiresIn: expiration,
    audience: WIKI.config.auth.audience,
    issuer: 'urn:wiki.js'
  })

  await WIKI.models.apiKeys.query().findById(entry.id).patch({
    key,
    isRevoked: false        // JWT 签发后启用
  })
  return key
}
```

**关键点**：
- API Key 本质是**带特殊载荷的 RS256 JWT**，`payload.api` 是 apiKey ID，`payload.grp` 是组 ID
- `fullAccess=true` 映射到组 `1`（Administrators，拥有 `manage:system` 权限）
- 否则绑定到一个指定的用户组（group），**API 请求会完全使用该组的 permissions + pageRules**

### 2.16.2 API Key 的请求生命周期

`server/core/auth.js:113-212` 的 `authenticate()` 中间件处理流程：

```
请求到达
  │
  ▼
passport-jwt 策略验证（RS256 + audience/issuer 校验）
  │
  ├─ JWT 含 payload.api → 进入 "Process API tokens" 分支（:179-204）
  │    │
  │    ├─ 检查全局开关 WIKI.config.api.isEnabled → 关闭则报错
  │    ├─ 检查 api.id 是否在 WIKI.auth.validApiKeys 中
  │    │    └─ validApiKeys 由 reloadApiKeys() 从 DB 加载（:402-404）：
  │    │       「WHERE isRevoked=false AND expiration > NOW()」
  │    │
  │    ▼
  │    构造一个假的 user 对象（:184-199）：
  │    req.user = {
  │      id: 1,                              // ← 硬编码为用户 ID 1
  │      email: 'api@localhost',             // ← 虚拟邮箱
  │      name: 'API',
  │      permissions: WIKI.auth.groups[user.grp].permissions,  // ← 从绑定组继承
  │      groups: [user.grp],                 // ← 绑定组
  │      getGlobalPermissions() { ... },
  │      getGroups() { ... }
  │    }
  │    ↓ 注入到请求上下文，继续后续 GraphQL 处理
  │
  └─ JWT 含 payload.id → 正常用户登录（:206-210）
```

### 2.16.3 评论权限边界的具体推导

当第三方插件用 API Key 请求评论时，权限链完全复用现有三层机制：

| 请求场景 | API Key 绑定组 A（仅 `read:comments`） | API Key 绑定组 B（`read:comments` + `write:comments`） | API Key `fullAccess=true`（组 1） |
|----------|--------------------------------------|-----------------------------------------------------|-----------------------------------|
| **GraphQL 第一层：@auth 指令** | | | |
| `comments.list(locale, path)` | ✅ 通过（需要 `read:comments`） | ✅ 通过 | ✅ 通过 |
| `comments.single(id)` | ✅ 通过 | ✅ 通过 | ✅ 通过 |
| `comments.create(...)` | ❌ 拒绝（需要 `write:comments`） | ✅ 通过 | ✅ 通过 |
| `comments.update(id, content)` | ❌ 拒绝（需要 `manage:comments`/`manage:system`） | ❌ 拒绝（缺 manage） | ✅ 通过（`manage:system`） |
| `comments.delete(id)` | ❌ 拒绝 | ❌ 拒绝 | ✅ 通过 |
| | | | |
| **Model 第二层：checkAccess 页面级** | | | |
| 目标页面路径在组 A pageRules 中 deny | ❌ 拒绝（即使全局有 read） | ❌ 拒绝 | ✅ 跳过（manage:system 豁免） |
| 目标页面路径匹配 START 且 allow | ✅ 通过 | ✅ 通过 | ✅ 跳过 |
| Guest 组 (id=2) 无 write 规则 | — | ❌ 拒绝（即使全局有 write，页面级不匹配） | ✅ 跳过 |
| | | | |
| **第三层：字段级 @auth（CommentPost）** | | | |
| `authorEmail` 字段 | ❌ 拒绝（需要 `manage:system`） | ❌ 拒绝 | ✅ 通过 |
| `authorIP` 字段 | ❌ 拒绝（需要 `manage:system`） | ❌ 拒绝 | ✅ 通过 |
| `content` 字段（含 @auth） | ❌ 拒绝（需要 write/manage 权限） | ✅ 通过 | ✅ 通过 |
| `render` 字段 | ✅ 无条件返回 | ✅ 无条件返回 | ✅ 无条件返回 |

**关键洞察**：
1. 评论 Schema 中 `authorEmail` 和 `authorIP` 字段必须有 `manage:system` 才能访问，**普通 API Key 即使绑定了有 read 权限的组也看不到邮箱和 IP**
2. `content` 原始 Markdown 字段被 `@auth(requires: ["write:comments", "manage:comments", "manage:system"])` 保护，仅读权限的 API 只能拿到 `render`（渲染后的 HTML），拿不到原始 Markdown
3. `@rateLimit(limit: 1, duration: 15)` 以 IP 为 key，API Key 请求也受此限制
4. API 全局开关 `api.isEnabled` 关闭时，**所有 API Key 请求被拒绝**（即使 JWT 本身合法）

### 2.16.4 API Key 的性能与生命周期

**validApiKeys 列表的刷新时机**（`auth.js:402-404`）：
```js
async reloadApiKeys () {
  const keys = await WIKI.models.apiKeys.query().select('id')
    .where('isRevoked', false)
    .andWhere('expiration', '>', DateTime.utc().toISO())
  this.validApiKeys = _.map(keys, 'id')
}
```

这个 `reloadApiKeys()` 在以下场景被调用：
- 内核启动时（`init()` 流程中）
- 创建 / 撤销 API Key 时
- 跨实例通过 PG NOTIFY 通知（`core/db.js` 的事件总线）同步

### 2.16.5 API 鉴权边界总览图

```
第三方插件 / 外部系统
         │
         │ HTTP POST /graphql
         │ Authorization: Bearer <API Key JWT>
         ▼
 ┌─ passport-jwt（RS256 验签 + audience/issuer 校验）
 │
 ├─ payload 含 api → API Key 分支
 │    ├─ 检查 WIKI.config.api.isEnabled（全局开关）
 │    ├─ 检查 api.id ∈ validApiKeys（DB + 过期时间）
 │    ├─ 根据 payload.grp 从 WIKI.auth.groups[] 获取 permissions + pageRules
 │    └─ 构造假 user 对象（id=1, name='API'），继承组的权限
 │
 └─ 后续处理（完全等同于正常登录用户）
      ├─ @auth 指令（全局 permissions 校验）
      ├─ checkAccess()（页面级规则匹配）
      ├─ @rateLimit（IP 级限流，API Key 也受限制）
      └─ 字段级 @auth（authorEmail/authorIP/content 受限）
```

---

## 2.17 评论数据的备份、跨实例迁移和数据库迁移脚本

评论数据的备份 / 迁移能力分为**三层**：DB Schema 自动迁移、全量 JSON 导出、以及 V1→V2 用户导入（不含评论）。评论**没有专用的跨实例增量同步机制**。

### 2.17.1 数据库 Schema 迁移系统

Wiki.js 使用 **Knex.js 的 `migrate.latest()`** 自动管理 Schema 版本：

启动入口（`server/core/db.js:197-202`）：
```js
async syncSchemas () {
  return self.knex.migrate.latest({
    tableName: 'migrations',
    migrationSource   // 自定义源，按 semver 排序
  })
}
```

自定义迁移源（`core/db.js:177-195`）：
```js
const baseMigrationPath = path.join(WIKI.SERVERPATH,
  (WIKI.config.db.type !== 'sqlite') ? 'db/migrations' : 'db/migrations-sqlite')
const migrationSource = {
  async getMigrations() {
    const migrationFiles = await fs.readdir(baseMigrationPath)
    return migrationFiles.sort(semver.compare).map(m => ({ ... }))  // 按语义版本排序
  },
  getMigrationName(migration) { return migration.file },
  async getMigration(migration) { return require(path.join(baseMigrationPath, migration.file)) }
}
```

**机制**：
- `migrations` 表记录已执行的文件名（如 `"2.4.36.js"`）
- 启动时按 `semver.compare` 排序，找到未执行的迁移
- 评论相关的迁移（3次）按版本号顺序执行

### 2.17.2 评论相关的 3 次 Schema 迁移

| 迁移文件 | 版本 | 操作 | 关键代码 |
|----------|------|------|---------|
| `2.0.0.js:62-69` | 2.0.0 | **CREATE TABLE comments** | `table.increments('id').primary(); table.text('content')...` |
| `2.0.0.js:285-288` | 2.0.0 | **添加外键** | `table.integer('pageId').unsigned().references('id').inTable('pages');` <br> `table.integer('authorId').unsigned().references('id').inTable('users');` |
| `2.4.14.js:8-19` | 2.4.14 | **CREATE TABLE commentProviders** | Provider 配置表（`key`, `isEnabled`, `config`） |
| `2.4.36.js:3-10` | 2.4.36 | **ALTER TABLE comments 新增列** | `render`, `name`, `email`, `ip` |
| `2.4.61.js:3-5` | 2.4.61 | **ALTER TABLE comments 新增列** | `replyTo`（回复父评论 ID，默认 0） |

**外键级联行为**：
- `pageId` → `pages.id`，`authorId` → `users.id`
- 迁移脚本中使用 `references().inTable()`，但**没有指定 `onDelete('CASCADE')`**
- 意味着：删除页面时，若该页面有评论，DB 级外键会**阻止删除**（依赖于数据库默认 FK 行为）

### 2.17.3 评论 Model 的关联映射（Objection.js relationMappings）

`server/models/comments.js:31-50` 定义了两个 BelongsToOne 关系：

```js
static get relationMappings() {
  return {
    author: {
      relation: Model.BelongsToOneRelation,
      modelClass: require('./users'),
      join: { from: 'comments.authorId', to: 'users.id' }
    },
    page: {
      relation: Model.BelongsToOneRelation,
      modelClass: require('./pages'),
      join: { from: 'comments.pageId', to: 'pages.id' }
    }
  }
}
```

**在导出时使用**（见下节），让评论 JSON 中内嵌 author 和 page 的部分信息。

### 2.17.4 全量 JSON 导出（System Export）

评论是可选的导出实体之一。GraphQL 入口调用 `WIKI.system.export({ entities: ['comments', 'pages', ...] })`。

评论导出的完整实现（`core/system.js:141-198`）：

```js
case 'comments': {
  WIKI.logger.info('Exporting comments...')
  const outputPath = path.join(opts.path, 'comments.json.gz')

  // 1. 统计总数
  const commentsCountRaw = await WIKI.models.comments.query().count('* as total').first()
  const commentsCount = parseInt(commentsCountRaw.total)
  if (commentsCount < 1) { break }  // 无评论则跳过

  // 2. 分批次流式拉取（50条一批，避免内存爆炸）
  const commentsProgressMultiplier = progressMultiplier / Math.ceil(commentsCount / 50)
  const rs = Readable({ objectMode: true })
  rs._read = () => {}

  const fetchCommentsBatch = async (offset) => {
    const comments = await WIKI.models.comments.query()
      .offset(offset).limit(50)
      .withGraphJoined({    // ← 同时 JOIN 作者和页面信息
        author: true,
        page: true
      })
      .modifyGraph('author', builder => {
        builder.select('users.id', 'users.name', 'users.email', 'users.providerKey')
      })
      .modifyGraph('page', builder => {
        builder.select('pages.id', 'pages.path', 'pages.localeCode', 'pages.title')
      })
    if (comments.length > 0) {
      for (const cmt of comments) { rs.push(cmt) }   // 推入 Readable 流
      fetchCommentsBatch(offset + 50)                 // 递归取下一批
    } else {
      rs.push(null)    // 流结束标记
    }
    this.exportStatus.progress += commentsProgressMultiplier * 100
  }
  fetchCommentsBatch(0)   // 启动递归

  // 3. 流式写入 JSON → Gzip → 文件
  let marker = 0
  await pipeline(
    rs,          // Objection.js 查询结果流（对象模式）
    new Transform({
      objectMode: true,
      transform(chunk, encoding, callback) {
        marker++
        let outputStr = marker === 1 ? '[\n' : ''
        outputStr += JSON.stringify(chunk, null, 2)
        if (marker < commentsCount) { outputStr += ',\n' }
        callback(null, outputStr)
      },
      flush(callback) {
        callback(null, '\n]\n')   // 补闭合括号
      }
    }),
    zlib.createGzip(),                    // 压缩
    fs.createWriteStream(outputPath)      // 写入 comments.json.gz
  )
}
```

**导出文件格式**（`comments.json.gz` 解压后）：
```json
[
  {
    "id": 1,
    "content": "原始 Markdown",
    "render": "<p>渲染后的 HTML</p>",
    "name": "",
    "email": "",
    "ip": "192.168.1.100",
    "createdAt": "2024-01-01T00:00:00.000Z",
    "updatedAt": "...",
    "replyTo": 0,
    "pageId": 42,
    "authorId": 5,
    "author": {           // ← 来自 withGraphJoined
      "id": 5,
      "name": "张三",
      "email": "zhangsan@example.com",
      "providerKey": "local"
    },
    "page": {             // ← 来自 withGraphJoined
      "id": 42,
      "path": "docs/intro",
      "localeCode": "zh-cn",
      "title": "介绍"
    }
  },
  ...
]
```

**注意**：导出文件包含 **IP 地址**、**用户邮箱**等敏感字段，没有脱敏处理。

### 2.17.5 不存在的功能：评论导入 / 跨实例增量同步

**导入能力分析**：
- `system.graphql` 中 **无** `import` mutation（除 `importUsersFromV1`，仅用于 V1 MongoDB → V2 用户迁移）
- `core/system.js` 中 **无** 评论导入的 case（仅 export）
- 全代码库搜索 `importComments` / `comments.*json.gz` / `restoreComment` → **零匹配**
- System Export UI（若存在）中也一定**无"Import 评论"按钮**

**V1→V2 用户迁移**（`resolver/system.js:112-242`）：
`importUsersFromV1` 实现了从 MongoDB 导入用户和构建组规则：
```js
// V1 用户权限映射到 V2 组（:167-170）
let roles = ['read:pages', 'read:assets', 'read:comments', 'write:comments']
if (r.role === `write`) {
  roles = _.concat(roles, ['write:pages', 'manage:pages', ...])
}
```
但这段代码**只导入用户和组，不导入任何评论**。V1 的评论数据（如果存在）在迁移中被完全丢弃。

### 2.17.6 外键约束对迁移的影响

由于 `pageId` 和 `authorId` 有外键：
- **导入评论前必须先导入 pages 和 users**（否则 FK 约束失败）
- 若用户 ID 映射关系变了（新实例重新分配了 ID），导入时必须**重映射 authorId**
- pageId 同理（页面路径相同但 ID 不同）
- `replyTo` 字段（引用评论自身 ID）也需要重映射（自引用）

**评论 JSON 导出**中的 author/page 信息仅作为**人类可读元数据**存在，导入时（若未来实现）需要通过 `page.path+locale`、`author.email+providerKey` 等业务字段查找新 ID，不能直接用旧 ID。

### 2.17.7 备份与迁移方案总结

| 能力 | 实现 | 说明 |
|------|------|------|
| Schema 自动迁移 | ✅ Knex migrate.latest() | 启动时按 semver 排序执行，评论 3 次迁移 |
| 评论全量导出 | ✅ System Entity Export | 流式 JSON + Gzip，含 author/page 元数据，含明文 IP/邮箱 |
| 评论增量备份 | ❌ 未实现 | 无 WAL 归档、无 binlog 导出封装 |
| 评论导入 | ❌ 未实现 | 无 import mutation，无导入逻辑 |
| 跨实例评论同步 | ❌ 未实现 | 无双向/单向同步，无队列/消息机制 |
| V1 MongoDB → V2 迁移 | ⚠️ 仅用户 | V1→V2 用户导入，评论未迁移（被丢弃） |
| 外键级联删除 | ⚠️ 无 onDelete | 删除 page/user 时可能因 FK 失败（依赖数据库配置） |

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
| `server/graph/schemas/system.graphql` | System Export 入口（`export` mutation） |
| `server/graph/resolvers/system.js` | System Export Resolver + V1 用户导入 |
| `server/graph/schemas/authentication.graphql` | API Key 创建/撤销/开关定义 |
| `server/graph/directives/auth.js` | @auth 指令，全局权限拦截 |
| `server/graph/directives/rate-limit.js` | @rateLimit 指令，IP 级频率限制（`IP:父类型.字段名` 为 key） |
| `server/models/comments.js` | Comment Model，核心业务逻辑 + 页面级权限校验 |
| `server/models/users.js` | Guest 用户定义（id=2）、getGuestUser()、refreshToken |
| `server/models/apiKeys.js` | API Key Model：RS256 JWT 签发、payload.grp 组映射 |
| `server/models/commentProviders.js` | Provider 注册、初始化、磁盘扫描 |
| `server/modules/comments/default/comment.js` | 内置 Provider：Markdown渲染、Akismet、入库 |
| `server/modules/comments/default/definition.yml` | 内置 Provider 配置（akismet, minDelay 参数） |
| `server/db/migrations/2.0.0.js` | comments 表初始建表 + FK（pageId, authorId） |
| `server/db/migrations/2.4.14.js` | commentProviders 配置表建表 |
| `server/db/migrations/2.4.36.js` | comments 表新增 render, name, email, ip 字段 |
| `server/db/migrations/2.4.61.js` | comments 表新增 replyTo 字段 |
| `server/graph/schemas/asset.graphql` | 资产上传 Schema（独立于评论系统） |
| `server/core/kernel.js` | 事件系统初始化（inbound/outbound EventEmitter） + DB init |
| `server/core/db.js` | High-Availability 事件总线 + Knex migrate.latest() + PG NOTIFY |
| `server/core/system.js` | System Export 全量实现（50 条/批 + stream + Gzip） |
| `server/core/auth.js` | `authenticate()` 中间件、API Key 处理、`checkAccess()` 引擎、`getEffectivePermissions()`、reloadGroups()、reloadApiKeys() |
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
| `client/components/admin/admin-api.vue` | API Key 管理（创建/撤销/开关） |
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

### 7.3 反垃圾、限流、批量操作、API 鉴权关键参数

| 组件 | 参数 | 值 | 目的 |
|------|------|----|------|
| markdown-it | `html: false` | 禁止原始 HTML | 防止 XSS 第一层 |
| markdown-it | `breaks: true` | 换行转 `<br>` | 用户体验 |
| markdown-it | `linkify: true` | 自动识别链接 | 用户体验 |
| DOMPurify | 默认配置 | 白名单过滤 | 防止 XSS 第二层 |
| Akismet | `role=guest/user/admin` | 差异化评分 | 反垃圾 |
| Akismet | ⚠️ `useragent: user.agentagent` | **拼写错误 Bug** | user-agent 永远为空 |
| @rateLimit | `1/15s per IP` | IP 级限流 | 防刷屏 |
| minDelay | 默认 30s | 用户级限流（Guest 全局共享 id=2） | 防刷屏 |
| users.isActive | true/false | 全局账号封禁 | 防恶意用户 |
| API Key JWT | `RS256` + `payload.api + payload.grp` | API Key 按组授权 | 第三方集成 |
| API Key 校验 | `WIKI.auth.validApiKeys` | DB 白名单 + 过期时间 | 撤销/过期生效 |
| System Export | `50/batch + stream + Gzip` | 批量流式导出 | 备份迁移 |
| Schema 迁移 | `semver.sort` + Knex migrate.latest() | 启动时自动升级 | 版本兼容性 |

### 7.4 渲染与安全管道关键参数（已并入上一表，此节保留为空用于未来扩展）

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
  │    │    ├─ 已登录 → role=user
  │    │    └─ ⚠️ BUG: useragent: user.agentagent（拼写错误，UA 永远为空）
  │    ├─ 4. minDelay 频率控制
  │    │    ├─ Guest → 全站点共享 id=2 的计数器
  │    │    └─ 已登录 → 独立计数器
  │    └─ 5. 写入 DB：content(原始) + render(HTML)
  │    │
  │    ⚠️  以下功能均未实现：
  │    ├─ ❌ 评论通知 / Activity log（无 event emit）
  │    ├─ ❌ 点赞 / 收藏（无数据表、无 mutation）
  │    ├─ ❌ @mention 解析与通知（仅页面编辑器 TODO 注释）
  │    ├─ ❌ 审核状态机（Akismet 要么拒绝要么直接入库，无待审）
  │    ├─ ❌ IP 黑名单（仅有 users.isActive 账号级封禁）
  │    ├─ ❌ 附件上传与病毒扫描（仅可引用外部图片 URL）
  │    ├─ ❌ 批量删除 / 批量改状态（仅单条 CRUD，无 whereIn 批量 SQL）
  │    ├─ ❌ 评论导入（无 import mutation，无 JSON 还原逻辑）
  │    └─ ❌ 跨实例增量同步（无队列/消息机制）
  │
  │    ✅ 已实现的旁路：
  │    ├─ ✅ API Key 鉴权（payload.grp 绑定组，继承 permissions + pageRules）
  │    ├─ ✅ System Export（50 条/批，流式 JSON + Gzip）
  │    └─ ✅ Schema 迁移（Knex migrate.latest()，按 semver 排序）
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
  - 无审核流程：Akismet 要么直接拒绝，要么直接入库（无待审状态）
  - 无 IP 黑名单：仅账号级 `users.isActive` 全局封禁
  - 反垃圾 Bug：Akismet 参数名拼写错误（`user.agentagent`），user-agent 永远为空
  - 无附件上传：仅能引用外部图片 URL，无病毒扫描
  - 无批量操作：仅单条 CRUD，无 bulk mutation、无 whereIn 批量 SQL
  - 无评论导入：export 单向，comments.json.gz 无法反向还原
  - 外键安全：`pageId/authorId` FK 无 onDelete，删除页面/用户可能因 FK 约束失败
  - API 全局开关：api.isEnabled 关闭时所有 API Key 请求即使 JWT 合法也被拒绝
  - API 字段保护：authorEmail/authorIP 仅 manage:system 可见；content 原始 Markdown 仅 write/manage 可见
```
