# 评论与注释功能——完整代码链路分析

本文档追踪 Wiki.js 中评论（Comment）功能从创建、权限校验到页面呈现的完整代码链路。
代码分散在 GraphQL 层、Model 层、Provider 模块、SSR 控制器、Vuex Store、Vue 组件共 6 个位置，逻辑不直白，下面逐层拆解。

---

## 1. 整体架构一图

```
用户浏览器                    服务端
──────────                  ──────────────────────────────────────────────
comments.vue                GraphQL Schema (comment.graphql)
  │  Apollo mutation/create ──▶ @auth 指令（全局权限门控）
  │                           │
  │                           ▼
  │                         Resolver (comment.js)
  │                           │  WIKI.models.comments.postNewComment()
  │                           ▼
  │                         Comment Model (comments.js)
  │                           │  WIKI.auth.checkAccess()（页面级权限）
  │                           ▼
  │                         CommentProvider (default/comment.js)
  │                           │  Markdown渲染 + DOMPurify + Akismet反垃圾
  │                           ▼
  │                         DB INSERT (Objection.js → comments 表)
  │
  ├─ Apollo query/list ──────▶ Resolver.list()
  │                           │  查 pages 表 → checkAccess → 查 comments 表
  │                           ▼
  │                         返回 [CommentPost]

page.vue (主题层)           SSR Controller (common.js)
  │  读取 effectivePermissions ◀── WIKI.auth.getEffectivePermissions()
  │  决定是否渲染评论区         │  → page.pug → 传入 commentsEnabled / effectivePermissions
  │                           ▼
page.pug (SSR 模板)         → 传入 <page :comments-enabled :effective-permissions>
  │                           → <template slot='comments'> 嵌入 comments.main
  ▼
comments.vue 挂载
  │  v-intersect → fetch()
  │  根据 permissions.write / permissions.manage 控制UI
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
| `server/models/commentProviders.js` | Provider 注册、初始化、磁盘扫描 |
| `server/modules/comments/default/comment.js` | 内置 Provider：Markdown渲染、Akismet、入库 |
| `server/core/auth.js` | `checkAccess()` 页面级权限引擎、`getEffectivePermissions()` |
| `server/controllers/common.js` | SSR 控制器，计算 effectivePermissions + 注入评论模板 |
| `server/views/page.pug` | SSR 模板，传递 comments 变量到 Vue 组件 |
| `client/themes/default/components/page.vue` | 主题页面组件，条件渲染评论区 |
| `client/store/page.js` | Vuex Store，存储 effectivePermissions |
| `client/components/comments.vue` | 评论交互组件，所有 CRUD 操作 |
| `client/components/admin/admin-comments.vue` | 管理后台评论 Provider 配置 |

---

## 7. 权限决策流程总结

```
请求到达
  │
  ├─ GraphQL @auth 指令
  │    检查用户全局 permissions 是否包含所需权限
  │    ↓ 通过
  │
  ├─ Resolver / Model 层
  │    WIKI.auth.checkAccess(user, permissions, { path, locale, tags })
  │    ├─ manage:system → 直接放行
  │    ├─ 全局权限交集 → 无交集则拒绝
  │    └─ 页面规则匹配 → 按 START/END/REGEX/EXACT/TAG + 优先级 + deny 判定
  │    ↓ 通过
  │
  └─ 前端 UI 门控
       effectivePermissions.comments.{read,write,manage}
       ├─ read → 是否显示评论区
       ├─ write → 是否显示输入框和发布按钮
       └─ manage → 是否显示编辑/删除操作

额外约束：
  - featurePageComments 全局开关关闭 → 所有评论权限为 false
  - @rateLimit → 创建评论时同一 IP 15 秒限 1 次
  - Guest 用户 → 必须提供 guestName + guestEmail
  - 内置 Provider → Akismet 反垃圾 + minDelay 最小发言间隔
```
