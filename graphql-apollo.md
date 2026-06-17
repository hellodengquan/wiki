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

以 `客户端IP + 类型名.字段名` 作为限流 key。示例：

```graphql
login(...): AuthenticationLoginResponse @rateLimit(limit: 5, duration: 60)
```

登录接口限流为：同一 IP 每分钟最多 5 次。

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
