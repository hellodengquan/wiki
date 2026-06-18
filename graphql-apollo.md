# Wiki.js GraphQL Schema 与 Apollo 架构梳理

## 一、GraphQL Schema 模块化划分

### 1.1 整体架构

服务端的 GraphQL Schema 采用**文件级模块化**方案，位于 `server/graph/` 目录下：

```
server/graph/
├── schemas/          # 20 个按领域划分的 .graphql schema 文件
├── resolvers/        # 对应的 resolver 实现文件
├── directives/       # 自定义指令（auth、rate-limit）
├── scalars/          # 自定义标量（Date）
└── index.js          # Schema 组装入口
```

### 1.2 Schema 组装机制

在 `server/graph/index.js:22-26` 中，通过读取 `schemas/` 目录下所有 `.graphql` 文件并拼接成 typeDefs 数组：

```javascript
let typeDefs = [createRateLimitTypeDef()]
let schemas = fs.readdirSync(path.join(WIKI.SERVERPATH, 'graph/schemas'))
schemas.forEach(schema => {
  typeDefs.push(fs.readFileSync(path.join(WIKI.SERVERPATH, `graph/schemas/${schema}`), 'utf8'))
})
```

Resolvers 通过 `auto-load` 模块自动加载 `resolvers/` 目录下的所有文件，然后用 `_.merge` 深度合并：

```javascript
const resolversObj = _.values(autoload(path.join(WIKI.SERVERPATH, 'graph/resolvers')))
resolversObj.forEach(resolver => {
  _.merge(resolvers, resolver)
})
```

### 1.3 各模块 Schema 详解

#### （1）公共基础 — `common.graphql`

定义了根类型和全局共享的类型与指令：

- **根类型占位**：`type Query`、`type Mutation`、`type Subscription`（空定义，供其他模块 extend）
- **自定义指令**：`@auth(requires: [String])` — 用于 QUERY、FIELD_DEFINITION、ARGUMENT_DEFINITION 的权限控制
- **通用类型**：
  - `KeyValuePair` / `KeyValuePairInput` — 键值对
  - `DefaultResponse` — 通用变更响应（包含 `ResponseStatus`）
  - `ResponseStatus` — 响应状态：`succeeded`、`errorCode`、`slug`、`message`

#### （2）标量 — `scalars.graphql`

仅定义了 `scalar Date`，实现在 `server/graph/scalars/date.js`。

#### （3）页面模块 — `page.graphql`

Wiki.js 的核心模块。通过命名空间模式组织：

```graphql
extend type Query { pages: PageQuery }
extend type Mutation { pages: PageMutation }
```

**查询（PageQuery）**：
- `history` — 页面历史版本
- `version` — 指定版本内容
- `search` — 搜索页面
- `list` — 页面列表（支持按标签、语言、创建者等过滤）
- `single` / `singleByPath` — 获取单页
- `tags` / `searchTags` — 标签相关
- `tree` — 页面目录树
- `links` — 页面链接
- `checkConflicts` / `conflictLatest` — 编辑冲突检测

**变更（PageMutation）**：
- `create` / `update` / `convert` — 页面创建、更新、格式转换
- `move` / `delete` — 移动、删除
- `deleteTag` / `updateTag` — 标签管理
- `flushCache` / `rebuildTree` / `render` — 维护操作
- `restore` / `purgeHistory` — 历史版本恢复与清理

**核心类型**：`Page`、`PageTag`、`PageHistory`、`PageVersion`、`PageSearchResponse`、`PageTreeItem` 等。

#### （4）认证模块 — `authentication.graphql`

```graphql
extend type Query { authentication: AuthenticationQuery }
extend type Mutation { authentication: AuthenticationMutation }
```

**查询**：`apiKeys`、`apiState`、`strategies`、`activeStrategies`

**变更**：
- `login` / `loginTFA` / `loginChangePassword` — 登录流程（带 `@rateLimit` 限流）
- `forgotPassword` / `register` — 密码找回与注册
- `createApiKey` / `revokeApiKey` / `setApiState` — API Key 管理
- `updateStrategies` / `regenerateCertificates` / `resetGuestUser` — 系统级配置

登录响应类型 `AuthenticationLoginResponse` 包含：`jwt`、`mustChangePwd`、`mustProvideTFA`、`mustSetupTFA`、`continuationToken`、`tfaQRImage` 等字段，支持多阶段登录流程。

#### （5）用户模块 — `user.graphql`

```graphql
extend type Query { users: UserQuery }
extend type Mutation { users: UserMutation }
```

**查询**：`list`、`search`、`single`、`profile`（当前用户）、`lastLogins`

**变更**：`create`、`update`、`delete`、`verify`、`activate`、`deactivate`、`enableTFA`、`disableTFA`、`resetPassword`、`updateProfile`、`changePassword`

注意 `profile` 查询和 `updateProfile`/`changePassword`/`resetPassword` 没有 `@auth` 指令限制，它们在 resolver 内部做更细粒度的鉴权。

#### （6）用户组模块 — `group.graphql`

```graphql
extend type Query { groups: GroupQuery }
extend type Mutation { groups: GroupMutation }
```

**查询**：`list`、`single`

**变更**：`create`、`update`、`delete`、`assignUser`、`unassignUser`

核心类型 `Group` 包含 `permissions`（权限字符串数组）和 `pageRules`（页面访问规则）。页面规则支持 `START`/`EXACT`/`END`/`REGEX`/`TAG` 五种匹配模式。

#### （7）站点配置 — `site.graphql`

```graphql
extend type Query { site: SiteQuery }
extend type Mutation { site: SiteMutation }
```

管理全局站点配置（host、title、安全设置、上传限制、认证配置等），所有操作均需 `manage:system` 权限。

#### （8）日志模块 — `logging.graphql`（含 Subscription）

```graphql
extend type Query { logging: LoggingQuery }
extend type Mutation { logging: LoggingMutation }
extend type Subscription { loggingLiveTrail: LoggerTrailLine }
```

这是**唯一使用了 Subscription** 的模块。`loggingLiveTrail` 订阅实时推送管理员控制台的日志行（`level`、`output`、`timestamp`）。

#### （9）资源/媒体模块 — `asset.graphql`

```graphql
extend type Query { assets: AssetQuery }
extend type Mutation { assets: AssetMutation }
```

管理上传资源（图片、二进制文件），支持文件夹操作。

#### （10）其他模块

| Schema 文件 | 功能说明 |
|------------|---------|
| `analytics.graphql` | 统计分析服务配置 |
| `comment.graphql` | 评论系统配置 |
| `contribute.graphql` | 贡献者信息 |
| `localization.graphql` | 多语言本地化 |
| `mail.graphql` | 邮件服务配置与测试 |
| `navigation.graphql` | 导航菜单配置 |
| `rendering.graphql` | 渲染引擎配置 |
| `search.graphql` | 搜索引擎配置与索引重建 |
| `storage.graphql` | 存储后端配置与同步 |
| `system.graphql` | 系统信息、升级、标志位、遥测、导出导入 |
| `theming.graphql` | 主题配置 |

### 1.4 命名空间模式

所有业务模块都采用了**两级命名空间**的设计模式，以避免根级别字段爆炸：

```graphql
# 根级只暴露各模块入口
extend type Query {
  pages: PageQuery
  users: UserQuery
  authentication: AuthenticationQuery
  # ...
}

# 具体操作在二级命名空间下
type PageQuery {
  list(...): [PageListItem!]!
  single(id: Int!): Page
  # ...
}
```

客户端调用示例：
```graphql
query {
  pages {
    list(limit: 10) {
      id title path
    }
  }
}
```

Resolver 层面对应实现（以 pages 为例，`server/graph/resolvers/page.js:7-12`）：

```javascript
Query: {
  async pages() { return {} }
},
Mutation: {
  async pages() { return {} }
},
PageQuery: {
  async history(obj, args, context, info) { /* ... */ }
  // ...
}
```

根级 resolver 仅返回空对象作为"命名空间载体"，实际逻辑在二级类型的 resolver 中。

---

## 二、Apollo 客户端缓存与订阅配置

### 2.1 客户端初始化

客户端 Apollo 配置位于 `client/client-app.js:49-132`。

```javascript
import { ApolloClient } from 'apollo-client'
import { BatchHttpLink } from 'apollo-link-batch-http'
import { ApolloLink, split } from 'apollo-link'
import { WebSocketLink } from 'apollo-link-ws'
import { ErrorLink } from 'apollo-link-error'
import { InMemoryCache } from 'apollo-cache-inmemory'
import { getMainDefinition } from 'apollo-utilities'
```

### 2.2 端点配置

```javascript
const graphQLEndpoint = window.location.protocol + '//' + window.location.host + '/graphql'
const graphQLWSEndpoint = ((window.location.protocol === 'https:') ? 'wss:' : 'ws:') + '//' + window.location.host + '/graphql-subscriptions'
```

- **HTTP**：`/graphql`（用于 query / mutation）
- **WebSocket**：`/graphql-subscriptions`（用于 subscription）
- 根据页面协议自动切换 `http/https` 和 `ws/wss`

### 2.3 Link 管道

HTTP 请求链路由 `ApolloLink.from` 组合：

```
ErrorLink → BatchHttpLink（自定义 fetch）
```

#### （1）ErrorLink — 全局错误处理

`client/client-app.js:56-79`：

- **GraphQL 错误**：区分 `Forbidden`（权限错误）和其他错误，通过 Vuex store 触发不同提示消息
- **网络错误**：显示 `Network Error: {message}` 通知

#### （2）BatchHttpLink — 批量 HTTP 请求

`client/client-app.js:80-110` 使用 `apollo-link-batch-http` 将多个 GraphQL 操作合并为一次 HTTP 请求。

关键配置：
- `includeExtensions: true` — 包含 extensions 字段
- `credentials: 'include'` — 请求携带 Cookie
- **自定义 fetch** 实现了三件事：
  1. **剥离 `__typename`**：发送前递归移除 variables 中的 `__typename` 字段（避免服务端校验错误）
  2. **注入 JWT**：从 Cookie 读取 `jwt`，设置 `Authorization: Bearer <token>` 请求头
  3. **处理 Token 续期**：检查响应头 `new-jwt`，若存在则更新本地 Cookie

### 2.4 WebSocketLink — 订阅连接

`client/client-app.js:113-123`：

```javascript
const graphQLWSLink = new WebSocketLink({
  uri: graphQLWSEndpoint,
  options: {
    reconnect: true,          // 自动重连
    lazy: true,               // 延迟连接（首次订阅时才建立）
    connectionParams: () => { // 每次连接时传递参数
      const token = Cookies.get('jwt')
      return token ? { token } : {}
    }
  }
})
```

### 2.5 split 路由分发

`client/client-app.js:126-129` 使用 `split` 根据操作类型自动选择链路：

```javascript
link: split(({ query }) => {
  const { kind, operation } = getMainDefinition(query)
  return kind === 'OperationDefinition' && operation === 'subscription'
}, graphQLWSLink, graphQLLink)
```

- **subscription** → WebSocketLink
- **query / mutation** → BatchHttpLink

### 2.6 缓存配置

`client/client-app.js:130` 使用默认的 `InMemoryCache`，未定制 `dataIdFromObject` 或 type policies。

```javascript
cache: new InMemoryCache()
```

开发环境下启用 DevTools：

```javascript
connectToDevTools: (process.env.node_env === 'development')
```

### 2.7 服务端 Apollo Server 配置

位于 `server/core/servers.js:123-163`：

```javascript
this.servers.graph = new ApolloServer({
  ...graphqlSchema,   // { typeDefs, resolvers, schemaDirectives }
  context: ({ req, res }) => ({ req, res }),
  subscriptions: {
    onConnect: (connectionParams, webSocket) => { /* 鉴权逻辑 */ },
    path: '/graphql-subscriptions'
  }
})
this.servers.graph.applyMiddleware({ app: WIKI.app, cors: false })
```

通过 `installSubscriptionHandlers` 将订阅处理器挂载到 HTTP/HTTPS 服务器上（`server/core/servers.js:26`、`:88`）。

### 2.8 订阅实现机制

以日志实时流为例：

**服务端发布**（`server/graph/index.js:46-66`）：
自定义 Winston Transport `LiveTrailLogger`，在日志输出时通过 PubSub 发布事件：

```javascript
log (info, callback = () => {}) {
  WIKI.GQLEmitter.publish('livetrail', {
    loggingLiveTrail: {
      timestamp: new Date(),
      level: info[LEVEL],
      output: info[MESSAGE]
    }
  })
  callback(null, true)
}
```

**服务端订阅 resolver**（`server/graph/resolvers/logging.js:13-17`）：

```javascript
Subscription: {
  loggingLiveTrail: {
    subscribe: () => WIKI.GQLEmitter.asyncIterator('livetrail')
  }
}
```

**客户端订阅查询**（`client/graph/admin/logging/logging-subscription-livetrail.gql`）：

```graphql
subscription {
  loggingLiveTrail {
    level
    output
    timestamp
  }
}
```

---

## 三、登录态权限信息在请求中的携带方式

### 3.1 Token 存储与状态管理

客户端使用 **Cookie + Vuex** 双重存储：

**Cookie**：`jwt` — 存储原始 JWT 字符串（`client/client-app.js:96-98`、`106`）

**Vuex Store**（`client/store/user.js`）：在应用启动时通过 `REFRESH_AUTH` mutation 从 Cookie 解码 JWT 并提取用户信息：

```javascript
REFRESH_AUTH(st) {
  const jwtCookie = Cookies.get('jwt')
  if (jwtCookie) {
    try {
      const jwtData = jwt.decode(jwtCookie)
      st.id = jwtData.id
      st.email = jwtData.email
      st.name = jwtData.name
      st.permissions = jwtData.permissions
      st.authenticated = true
      // ... 其他字段
    } catch (err) {
      console.debug('Invalid JWT. Silent authentication skipped.')
    }
  }
}
```

JWT Payload 中包含的字段：`id`、`email`、`name`、`av`（头像）、`lc`（语言）、`tz`（时区）、`df`（日期格式）、`ap`（外观）、`permissions`（权限字符串数组）、`iat`、`exp`。

### 3.2 HTTP 请求中的 Token 传递

在 `BatchHttpLink` 的自定义 fetch 中实现（`client/client-app.js:95-99`）：

```javascript
const jwtToken = Cookies.get('jwt')
if (jwtToken) {
  options.headers.Authorization = `Bearer ${jwtToken}`
}
```

同时设置了 `credentials: 'include'`，确保 Cookie 也会随请求发送（作为兜底方案）。

### 3.3 WebSocket 订阅中的 Token 传递

在 WebSocketLink 的 `connectionParams` 回调中传递（`client/client-app.js:118-121`）：

```javascript
connectionParams: () => {
  const token = Cookies.get('jwt')
  return token ? { token } : {}
}
```

服务端在 `subscriptions.onConnect` 中接收并验证（`server/core/servers.js:129-158`），优先级为：
1. `connectionParams.token`（连接参数）
2. WebSocket 升级请求头中的 Cookie（`cookies.jwt`）

### 3.4 服务端 Token 提取

`server/helpers/security.js:27-40` 使用 `passport-jwt` 的多提取器策略：

```javascript
extractJWT: passportJWT.ExtractJwt.fromExtractors([
  passportJWT.ExtractJwt.fromAuthHeaderAsBearerToken(),  // Authorization: Bearer <token>
  (req) => {
    let token = null
    if (req && req.cookies) {
      token = req.cookies['jwt']                          // Cookie: jwt=<token>
    }
    if (req.path.toLowerCase() === '/u') {
      return null  // 上传接口强制使用 Header（避免 Cookie 跨域问题）
    }
    return token
  }
])
```

### 3.5 Token 续签机制

当 Token 过期但仍在续期窗口内时，服务端自动签发新 Token：

**服务端判断**（`server/core/auth.js:118-124`）：
```javascript
if (info instanceof Error && info.name === 'TokenExpiredError') {
  const expiredDate = (info.expiredAt instanceof Date) ? info.expiredAt.toISOString() : info.expiredAt
  if (DateTime.utc().minus(ms(WIKI.config.auth.tokenRenewal)) < DateTime.fromISO(expiredDate)) {
    mustRevalidate = true
  }
}
```

**服务端返回新 Token**（`server/core/auth.js:145-150`）：
重新签发新 Token 并通过响应头返回。

**客户端接收并保存**（`client/client-app.js:103-107`）：
```javascript
const newJWT = resp.headers.get('new-jwt')
if (newJWT) {
  Cookies.set('jwt', newJWT, { expires: 365, secure: window.location.protocol === 'https:' })
}
```

### 3.6 权限校验：@auth 指令

定义在 `server/graph/schemas/common.graphql:8`：
```graphql
directive @auth(requires: [String]) on QUERY | FIELD_DEFINITION | ARGUMENT_DEFINITION
```

实现在 `server/graph/directives/auth.js`，通过 `SchemaDirectiveVisitor` 包装字段 resolver：

**校验逻辑**：
1. 检查字段或其对象类型上是否有 `_requiredAuthScopes`
2. 若无，直接执行原 resolver
3. 若有，检查 `context.req.user` 是否存在（未登录 → `Unauthorized`）
4. 检查用户的 `permissions` 数组是否包含任意一个所需权限（不满足 → `Forbidden`）
5. 权限通过后才执行原 resolver

```javascript
const requiredScopes = field._requiredAuthScopes || objectType._requiredAuthScopes
if (!requiredScopes) return resolve.apply(this, args)

const context = args[2]
if (!context.req.user) throw new Error('Unauthorized')
if (!_.some(context.req.user.permissions, pm => _.includes(requiredScopes, pm))) {
  throw new Error('Forbidden')
}
return resolve.apply(this, args)
```

**权限字符串示例**（来自 schema）：
- `read:pages` / `write:pages` / `delete:pages` / `manage:pages`
- `read:history` / `read:source`
- `manage:users` / `write:users`
- `manage:groups` / `write:groups`
- `manage:api`
- `manage:system`（超级权限，几乎所有操作都包含此项作为备选）
- `read:assets` / `write:assets` / `manage:assets`

### 3.7 限流：@rateLimit 指令

用于登录等敏感操作（`server/graph/directives/rate-limit.js`）：

```javascript
module.exports = createRateLimitDirective({
  keyGenerator: (directiveArgs, source, args, context, info) => 
    `${context.req.ip}:${info.parentType}.${info.fieldName}`
})
```

以 `客户端IP + 类型名.字段名` 作为限流 key。

**当前使用 @rateLimit 的字段**（全在 mutation 上）：

| 字段 | 限流规则 | 所在 schema |
|------|---------|------------|
| `authentication.login` | 每分钟 5 次 | `authentication.graphql` |
| `authentication.loginTFA` | 每分钟 5 次 | `authentication.graphql` |
| `authentication.loginChangePassword` | 每分钟 5 次 | `authentication.graphql` |
| `authentication.forgotPassword` | 每分钟 3 次 | `authentication.graphql` |
| `comments.create` | 每 15 秒 1 次 | `comment.graphql` |

#### 3.7.1 @rateLimit 在 Subscription 路径上是否生效？

**结论：当前代码中 subscription 路径上 @rateLimit 完全不生效**，原因有两层：

**第一层：现有 subscription 字段上没有加 @rateLimit**

全局唯一的 subscription 字段是 `loggingLiveTrail`（`server/graph/schemas/logging.graphql:14`）：

```graphql
extend type Subscription {
  loggingLiveTrail: LoggerTrailLine
}
```

该字段上**没有** `@rateLimit` 指令，也没有 `@auth` 指令（但实际在 `subscriptions.onConnect` 中做了更严格的鉴权）。

**第二层：即便加上，graphql-rate-limit-directive 对 subscription 的行为也与 query/mutation 不同**

`graphql-rate-limit-directive` 1.2.1 版本是通过包装字段的 `resolve` 函数实现限流的。而 subscription 有两个函数：
- `subscribe` — 返回 AsyncIterator，在订阅建立时执行一次
- `resolve` — 每条推送消息经过时执行（可选，用于转换 payload）

在 Wiki.js 的实现中（`server/graph/resolvers/logging.js:14-16`）：

```javascript
Subscription: {
  loggingLiveTrail: {
    subscribe: () => WIKI.GQLEmitter.asyncIterator('livetrail')
  }
}
```

只有 `subscribe` 函数，没有 `resolve` 函数。如果给该字段加 `@rateLimit`，指令会尝试包装 `resolve` 函数，但由于不存在自定义 resolve，限流逻辑**只可能作用在订阅建立的瞬间**（而不是每条推送消息）。

但实际测试中，`graphql-rate-limit-directive` 对 Subscription 类型的支持并不完整，因为：
1. `info.parentType` 对于 Subscription 是 `Subscription`
2. 但限流指令通常绑定的是字段 resolver，而 subscription 的核心是 `subscribe` 而非 `resolve`
3. WebSocket 连接建立时走的是 `subscriptions.onConnect`，根本不经过 schema directive

#### 3.7.2 超出限流时：断开连接 vs 拒收消息？

**HTTP 路径（query/mutation）**：
超出限流时，`graphql-rate-limit-directive` 会返回 GraphQL error，错误信息类似 `"You are trying to access 'login' too often"`，HTTP 状态码仍然是 200（GraphQL 错误作为 errors 数组返回）。连接本身**不会断开**，只是请求被拒绝。

**WebSocket 路径（subscription）**：
由于现有代码中 subscription 没有 @rateLimit，不存在"超出限流"的场景。但如果从连接维度看：

- **连接建立阶段**（`subscriptions.onConnect`）：鉴权失败直接 `throw new Error('Unauthorized'/'Forbidden')`，Apollo Server 会拒绝 WebSocket 连接，连接**不会建立**
- **连接已建立后**：没有消息级别的限流，服务端通过 PubSub 主动推送，客户端无法"频繁请求"消息，因此也不需要消息级限流

### 3.8 Resolver 内部二次鉴权

除了 `@auth` 指令的全局权限校验外，部分 resolver 内部还会做更细粒度的页面级权限检查。以 `PageQuery.history` 为例（`server/graph/resolvers/page.js:17-31`）：

```javascript
async history(obj, args, context, info) {
  const page = await WIKI.models.pages.query().select('path', 'localeCode').findById(args.id)
  if (WIKI.auth.checkAccess(context.req.user, ['read:history'], {
    path: page.path,
    locale: page.localeCode
  })) {
    return WIKI.models.pageHistory.getHistory(...)
  } else {
    throw new WIKI.Error.PageHistoryForbidden()
  }
}
```

此处调用 `WIKI.auth.checkAccess`，结合用户组的 `pageRules`（路径、语言、标签匹配规则）进行更精细的页面级访问控制。

### 3.9 登录态信息在 Apollo Client Cache 中的写入位置

#### 3.9.1 用户登录态的存储分层

Wiki.js 客户端的登录态信息**不直接写入 Apollo cache**，而是采用三层存储架构：

| 存储层 | 存储内容 | 位置 | 生命周期 |
|--------|---------|------|---------|
| 第一层 | 原始 JWT Token | Cookie（`jwt`） | 持久化，365 天过期 |
| 第二层 | 解码后的用户信息（id/name/email/permissions 等） | Vuex Store（`user` module） | 内存中，页面刷新后重建 |
| 第三层 | 查询结果缓存（如 profile 数据） | Apollo InMemoryCache | 内存中，页面刷新后清空 |

#### 3.9.2 登录态写入流程

**登录时**（`client/components/login.vue:644`）：
```javascript
Cookies.set('jwt', respObj.jwt, { expires: 365, secure: window.location.protocol === 'https:' })
// 随后 window.location.replace('/') —— 整页跳转
```

登录成功后只做两件事：
1. 把 JWT 写入 Cookie
2. **整页跳转**到首页（或登录前页面）

**应用启动时**（`client/client-app.js:46`）：
```javascript
store.commit('user/REFRESH_AUTH')
```

在 `client/store/user.js:26-48` 的 `REFRESH_AUTH` mutation 中：
- 从 Cookie 读取 JWT
- 前端 `jwt.decode` 解码 payload（不验证签名）
- 提取 `id`、`email`、`name`、`permissions` 等字段写入 Vuex state
- 设置 `authenticated = true`

**注意**：这里的解码只是前端展示用，**实际权限校验永远在服务端**。

#### 3.9.3 Apollo Cache 中是否有登录态信息？

**结论：没有主动写入的登录态信息，但查询结果会被自动缓存。**

代码中**不存在**任何 `cache.writeQuery`、`cache.writeData`、`cache.writeFragment` 等手动写入用户信息的调用。

但 Apollo cache 会自动缓存查询结果，例如：
- 个人资料页的 `users.profile` 查询（`client/components/profile/profile.vue:887-918`）
- 导航栏等组件发起的各类查询

这些缓存数据中可能包含用户相关信息，但它们是**查询副作用**，不是登录态的"官方存储位置"。

特别地，`profile` 查询使用了 `fetchPolicy: 'network-only'`（`client/components/profile/profile.vue:913`），意味着该查询每次都走网络、不读缓存，进一步降低了缓存中残留敏感用户信息的可能性。

### 3.10 退出登录时的缓存清理范围与残留问题

#### 3.10.1 退出登录的代码路径

退出登录入口在 `client/components/common/nav-header.vue:475-477`：
```javascript
logout () {
  window.location.assign('/logout')
}
```

**关键特点：整页跳转，不是 SPA 内的状态切换。**

服务端处理（`server/controllers/auth.js:129-134`）：
```javascript
router.get('/logout', async (req, res) => {
  const redirURL = await WIKI.models.users.logout({ req, res })
  req.logout()          // Passport session 登出
  res.clearCookie('jwt') // 清除 JWT Cookie
  res.redirect(redirURL) // 重定向到首页
})
```

#### 3.10.2 缓存清理的实际范围

由于退出登录是**整页跳转 + 服务端重定向**，浏览器会加载一个全新的页面：

| 存储层 | 清理方式 | 是否完全清理 |
|--------|---------|------------|
| Cookie 中的 JWT | 服务端 `res.clearCookie('jwt')` | ✅ 是 |
| Vuex Store | 页面刷新，JS 内存完全重建 | ✅ 是 |
| Apollo InMemoryCache | 页面刷新，JS 内存完全重建 | ✅ 是 |
| localStorage / sessionStorage | 不涉及 | N/A |

**不存在手动调用 `client.resetStore()` 或 `client.clearStore()` 的代码**，因为根本不需要——整页刷新后一切归零。

#### 3.10.3 残留缓存是否会被后续匿名查询误用？

**结论：不会。** 原因有三层保障：

**第一层：架构层面 — 整页跳转自然清空内存**

退出登录通过 `window.location.assign('/logout')` 触发整页跳转，服务端重定向回首页。新页面是一个完全独立的 JavaScript 执行上下文：
- Apollo Client 重新初始化
- InMemoryCache 从零开始构建
- 不存在"缓存数据残留到下一个会话"的可能

**第二层：服务端层面 — 权限校验在服务端**

即便假设（理论上）缓存中有数据，匿名用户发起新查询时：
- Apollo cache 基于 `__typename + id` 做归一化，没有对应 id 的数据不会命中
- 更重要的是：**实际数据永远由服务端返回**，缓存只是加速读取
- 匿名请求到达服务端后，`@auth` 指令会拒绝需要权限的查询，返回错误而不是数据

**第三层：fetchPolicy 层面 — 敏感查询不走缓存**

涉及用户信息的关键查询（如 `profile`）使用 `fetchPolicy: 'network-only'`，每次都强制走网络，不依赖缓存。

#### 3.10.4 理论上的边界情况

如果在**同一个页面内**（不刷新）实现登录/登出切换，就会出现缓存残留问题。但 Wiki.js 当前的架构不支持这种模式：
- 登录成功 → `window.location.replace('/')` 整页跳转（`client/components/login.vue:661`）
- 退出登录 → `window.location.assign('/logout')` 整页跳转（`client/components/common/nav-header.vue:476`）
- Token 续签 → 只更新 Cookie，不影响当前页面的 Apollo cache 数据

所有身份状态变化都伴随着页面刷新，从根本上避免了单页应用中常见的"登出后缓存残留"问题。

### 3.11 多 Tab 协同：跨 Tab 同步与登出联动

#### 3.11.1 跨 Tab 同步机制：代码中是否存在？

**结论：不存在任何跨 Tab 同步机制。**

全局搜索结果：

| 搜索项 | 结果 |
|--------|------|
| `BroadcastChannel` | 整个项目零匹配 |
| `addEventListener('storage', ...)` / `onstorage` | 客户端零匹配 |
| `SharedWorker` | 零匹配 |
| `postMessage` (跨窗口) | 零匹配 |

客户端对 `localStorage` 的使用仅限于两个无关场景：
- `nav-sidebar.vue:115` — 存储导航偏好 (`navPref`)
- `admin-utilities-cache.vue:90-95` — 管理员清除 i18n 缓存

**没有任何代码监听 `storage` 事件或使用 `BroadcastChannel` 来实现跨 Tab 状态同步。**

#### 3.11.2 一个 Tab 退出登录，其他 Tab 会怎样？

**场景**：用户在 Tab A 点击退出登录，Tab B（同源）仍在运行。

**Tab A 的行为**（`client/components/common/nav-header.vue:475-477`）：
```javascript
logout () {
  window.location.assign('/logout')
}
```

服务端 `server/controllers/auth.js:129-134`：
```javascript
router.get('/logout', async (req, res) => {
  const redirURL = await WIKI.models.users.logout({ req, res })
  req.logout()
  res.clearCookie('jwt')    // 服务端清除 Cookie
  res.redirect(redirURL)    // 重定向到首页
})
```

**Tab B 的状态**：

| 维度 | 状态 | 说明 |
|------|------|------|
| Apollo Cache | 仍然存在 | 内存中的 JS 对象，Tab B 的 JS 上下文不受 Tab A 影响 |
| Vuex Store | 仍然存在 | 同上，内存独立 |
| Cookie 中的 JWT | **已被清除** | `res.clearCookie('jwt')` 是服务端操作，Cookie 是浏览器共享的，Tab A 的请求导致同源 Cookie 被删除 |
| WebSocket 连接 | 仍然存活 | 见 3.12 节详述 |

**Tab B 继续使用会发生什么**：

1. **下一次 GraphQL 请求**：BatchHttpLink 自定义 fetch 从 Cookie 读 JWT（`client/client-app.js:96-99`），此时 Cookie 已空，不会注入 `Authorization` 头。请求到达服务端后，Passport JWT 认证失败，服务端设置 `req.user` 为 guest 用户（`server/core/auth.js:170-177`）。
2. **需要权限的查询**：`@auth` 指令检查 `req.user.permissions`，guest 用户的权限不满足 → 返回 `Forbidden` 错误。
3. **ErrorLink 处理**：`client/client-app.js:60-68` 捕获 `Forbidden` 错误，弹出通知 "You are not authorized to access this resource."
4. **Vuex 中的用户信息**：Tab B 的 Vuex 仍然认为用户已认证（`authenticated: true`），这是**过时的状态**。

**关键问题**：Tab B 不会自动感知到登出事件，Vuex 中的 `authenticated` 和 `permissions` 会保持过期状态，直到：
- 用户手动刷新 Tab B（触发 `REFRESH_AUTH`，Cookie 已空 → `authenticated = false`）
- 用户在 Tab B 发起任何需要权限的操作（被服务端拒绝，但不会自动跳转登录页）

**当前没有代码处理这种"静默登出"场景**——不会自动跳转登录页，也不会主动清除 Vuex 状态。

#### 3.11.3 `res.clearCookie('jwt')` 的 Cookie 属性匹配问题

Cookie 的 `set` 和 `clear` 必须使用相同的 `domain`、`path`、`secure` 属性才能正确删除。

**写入时**（`server/helpers/common.js:45-49`）：
```javascript
getCookieOpts () {
  return {
    expires: DateTime.utc().plus({ days: 365 }).toJSDate(),
    ...(WIKI.config.host.startsWith('https://') ? { secure: true } : {})
  }
}
```

**清除时**（`server/controllers/auth.js:132`）：
```javascript
res.clearCookie('jwt')
```

`clearCookie` 默认使用 `{ path: '/' }`，而 `cookie()` 也默认 `path: '/'`。`secure` 属性在 HTTPS 环境下：`set` 时带 `secure: true`，`clearCookie` 默认不带。但 Express 的 `res.clearCookie` 在 v4.x 中会根据当前请求是否为 HTTPS 自动处理，所以实际不构成问题。

但如果配置了自定义 `domain`（通过反向代理），`clearCookie` 不带 `domain` 可能导致无法删除 Cookie。当前代码中没有显式设置 `domain`，所以默认行为是安全的。

### 3.12 WebSocket / SSE 长连接在 Token 失效时的断开行为

#### 3.12.1 SSE：不存在

**代码中不存在任何 SSE（Server-Sent Events / EventSource）实现。** 全局搜索 `EventSource`、`text/event-stream`、`SSE` 均零匹配（服务端的少量匹配来自存储模块的无关代码）。

Wiki.js 的实时通信完全依赖 GraphQL Subscription over WebSocket。

#### 3.12.2 WebSocket 连接的建立与生命周期

**唯一使用 WebSocket Subscription 的功能**：管理员日志实时控制台（`client/components/admin/admin-logging-console.vue:75-91`）。

**连接建立**（`client/client-app.js:113-123`）：
```javascript
const graphQLWSLink = new WebSocketLink({
  uri: graphQLWSEndpoint,
  options: {
    reconnect: true,   // 断开后自动重连
    lazy: true,        // 延迟连接（首次订阅时才建立）
    connectionParams: () => {
      const token = Cookies.get('jwt')
      return token ? { token } : {}
    }
  }
})
```

**服务端鉴权**（`server/core/servers.js:128-162`）：
```javascript
subscriptions: {
  onConnect: (connectionParams, webSocket) => {
    let token = _.get(connectionParams, 'token', null)
    if (!token) {
      const cookieHeader = _.get(webSocket, 'upgradeReq.headers.cookie', '')
      if (cookieHeader) {
        const cookies = cookie.parse(cookieHeader)
        token = cookies.jwt || null
      }
    }
    if (!token) throw new Error('Unauthorized')
    // JWT 验证...
    if (!_.includes(user.permissions, 'manage:system')) throw new Error('Forbidden')
    return { user }
  },
  // 注意：没有 onDisconnect 回调
  path: '/graphql-subscriptions'
}
```

#### 3.12.3 Token 失效后 WebSocket 是否自然断开？

**结论：不会自然断开。** 具体分析：

**WebSocket 连接一旦建立，就不再校验 Token。**

`subscriptions.onConnect` 只在连接建立时执行一次。连接成功后，服务端**没有 `onDisconnect` 回调**（全局搜索确认），也没有任何定时校验 Token 有效性的机制。

这意味着：
- **Token 过期** → WebSocket 连接仍然存活，继续推送消息
- **Cookie 被清除**（另一 Tab 退出登录）→ 不影响已建立的 WebSocket 连接
- **服务端 Token 撤销**（`revokeUserTokens`）→ 不影响已建立的 WebSocket 连接

**但实际影响有限**，原因：

1. **只有 `manage:system` 权限的管理员能建立 WebSocket**：`onConnect` 中硬性检查（`server/core/servers.js:151`），非管理员根本无法建立订阅连接
2. **唯一的使用场景是日志实时流**（`admin-logging-console.vue`），仅在管理员主动打开日志控制台时才建立连接
3. **管理员退出登录时的整页跳转会销毁 WebSocket**：Tab A 执行 `window.location.assign('/logout')` → 页面卸载 → JS 上下文销毁 → WebSocket 连接关闭

#### 3.12.4 WebSocket 重连时的 Token 校验

`apollo-link-ws` 配置了 `reconnect: true`。当连接意外断开后尝试重连时：

```javascript
connectionParams: () => {
  const token = Cookies.get('jwt')
  return token ? { token } : {}
}
```

`connectionParams` 是一个**函数**，每次重连时重新调用。此时：

| 场景 | Cookie 状态 | 重连结果 |
|------|-----------|---------|
| Token 仍有效 | `jwt` 存在且有效 | 正常重连 |
| 另一 Tab 已退出登录 | `jwt` 已被清除 | `connectionParams` 返回 `{}`，服务端 `onConnect` 中 `token` 为 `null` → `throw new Error('Unauthorized')` → **重连被拒绝** |
| Token 过期 | `jwt` 仍存在但过期 | 服务端 JWT 验证失败 → `throw new Error('Unauthorized')` → **重连被拒绝** |

重连失败后，`apollo-link-ws` 会按指数退避策略持续尝试重连，但每次都会被 `onConnect` 拒绝。**不会出现用过期 Token 成功重连的情况。**

但客户端**没有对重连失败做用户可感知的处理**。`admin-logging-console.vue:84-90` 的 `error` 回调：
```javascript
error(error) {
  self.$store.commit('showNotification', {
    style: 'red',
    message: error.message,
    icon: 'warning'
  })
}
```

只在首次错误时弹出通知，后续的持续重连失败不会重复通知。日志控制台的 UI 仍显示 "Streaming..." 状态，用户可能误以为连接正常。

#### 3.12.5 服务端 logout 时的 Token 撤销

**关键发现：服务端 `/logout` 路由没有调用 `revokeUserTokens`。**

`revokeUserTokens` 只在以下场景被调用（`server/graph/resolvers/user.js`、`server/graph/resolvers/group.js`）：
- 管理员删除用户
- 管理员停用用户
- 管理员分配/取消分配用户组
- 管理员删除/更新用户组

`/logout` 路由（`server/controllers/auth.js:129-134`）只做了：
1. 调用 `WIKI.models.users.logout()` — 只是获取策略级别的 logout 重定向 URL
2. `req.logout()` — Passport session 登出（但 Wiki.js 使用 `session: false`，这一步无实际效果）
3. `res.clearCookie('jwt')` — 清除浏览器 Cookie
4. `res.redirect(redirURL)` — 重定向

**这意味着**：如果用户在退出登录前，JWT 已被其他客户端截获，该 JWT 在过期前仍然有效（因为没有被加入撤销列表）。这是 Wiki.js 当前架构的一个安全特性缺失——logout 不导致 Token 失效。

不过，由于 JWT 有 `exp` 声明且服务端会验证过期时间，这个窗口期受 `authJwtExpiration` 配置限制（默认较短）。

### 3.13 多 Tab 场景完整时序图

```
Tab A (退出登录)                     服务端                       Tab B (仍在运行)
─────────────────                    ───────                      ──────────────
                                                                 │ Vuex: authenticated=true
                                                                 │ Cookie: jwt=<valid>
                                                                 │ WebSocket: 已建立(lazy模式可能未建立)
                                                                 │ Apollo Cache: 有数据
                                                                 │
window.location.assign('/logout')    │                           │
─────────────────────────────────>   │                           │
                                     │                           │
                         GET /logout  │                           │
                         ──────────>  │                           │
                                     │                           │
                      users.logout() │                           │
                      req.logout()   │                           │
                      clearCookie('jwt')  ← 浏览器全局生效!      │
                      redirect('/')  │                           │
                      <──────────     │                           │
                                     │                           │
  [整页刷新, JS上下文销毁]            │                      Cookie: jwt=空!
  Apollo Cache: 销毁                 │                      Vuex: authenticated=true (过期!)
  Vuex: 销毁                         │                      Apollo Cache: 仍在 (内存隔离)
  WebSocket: 断开                     │                      WebSocket: 仍在 (若已建立)
                                     │                           │
  [新页面加载]                        │                           │
  REFRESH_AUTH → Cookie为空           │                           │
  authenticated=false                 │                           │
                                     │                      用户操作Tab B:
                                     │                      ┌──────────────────┐
                                     │                      │ 发起GraphQL请求   │
                                     │                      │ → Cookie无jwt    │
                                     │                      │ → Header无Bearer │
                                     │                      │ → 服务端设guest   │
                                     │                      │ → @auth拒绝→Forbidden
                                     │                      │ → 弹出通知       │
                                     │                      │ Vuex仍为true!    │
                                     │                      └──────────────────┘
                                     │                           │
                                     │                      用户手动刷新Tab B:
                                     │                      ┌──────────────────┐
                                     │                      │ REFRESH_AUTH     │
                                     │                      │ → Cookie为空     │
                                     │                      │ → authenticated=false
                                     │                      │ → 回到正确状态    │
                                     │                      └──────────────────┘
```

---

## 四、完整请求流程总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        客户端 (Vue + Apollo)                        │
│                                                                     │
│  登录 mutation → 拿到 jwt → 存入 Cookie                             │
│       ↓                                                             │
│  Vuex REFRESH_AUTH → 解码 jwt → 填充用户信息/permissions            │
│       ↓                                                             │
│  ApolloClient发起请求:                                               │
│    ├─ query/mutation → BatchHttpLink                                │
│    │     ├─ 剥离 variables.__typename                               │
│    │     ├─ 从 Cookie 读 jwt → 注入 Authorization: Bearer <token>   │
│    │     ├─ credentials: 'include' (携带 Cookie)                    │
│    │     └─ 响应后检查 new-jwt 头 → 更新 Cookie                     │
│    └─ subscription → WebSocketLink                                  │
│          ├─ lazy: true, reconnect: true                             │
│          └─ connectionParams 传递 token                             │
└─────────────────────────────────────────────────────────────────────┘
                              ↓ HTTPS / WSS
┌─────────────────────────────────────────────────────────────────────┐
│                      服务端 (Express + Apollo Server)               │
│                                                                     │
│  1. Express 中间件:                                                  │
│     ├─ security.js → 各种安全头 (X-Frame-Options, HSTS 等)          │
│     └─ auth.js authenticate() → Passport JWT 认证                    │
│           ├─ 从 Header / Cookie 提取 jwt                             │
│           ├─ 验证签名、audience、issuer、过期时间                    │
│           ├─ 检查撤销列表 (用户/组级)                                │
│           ├─ 过期但在续期窗口内 → 重签 Token → new-jwt 响应头        │
│           └─ 将解码后的 user (含 permissions) 挂到 req.user          │
│                                                                     │
│  2. Apollo Server:                                                  │
│     ├─ context: ({ req, res }) => ({ req, res })                    │
│     ├─ subscriptions.onConnect → WebSocket 连接时验证 token         │
│     │     └─ 要求 manage:system 权限                                │
│     └─ Schema Directive 校验:                                       │
│           ├─ @rateLimit → IP + 字段 维度限流                         │
│           └─ @auth(requires: [...])                                 │
│                 ├─ 无 req.user → Unauthorized                       │
│                 ├─ permissions 不匹配 → Forbidden                   │
│                 └─ 通过 → 执行 resolver                              │
│                                                                     │
│  3. Resolver 内部 (可选):                                            │
│     └─ WIKI.auth.checkAccess() → 页面级规则校验 (path/locale/tags)  │
└─────────────────────────────────────────────────────────────────────┘
```
