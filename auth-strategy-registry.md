# Wiki.js 多登录策略注册表与挑选逻辑详解

Wiki.js 支持多达 21 种登录方式（Local、Google、GitHub、LDAP、SAML、OIDC 等），它们并非硬编码在登录流程中，而是以 **策略（Strategy）模式** 统一注册、按场景挑选执行。整个机制涉及三层注册表 + 两层挑选逻辑，下文逐一拆解。

---

## 一、三层注册表

| 层次 | 存储位置 | 数据形态 | 作用 |
|------|---------|---------|------|
| **磁盘定义层** | `server/modules/authentication/<策略名>/definition.yml` | YAML 静态文件 | 声明策略的元信息（key、title、useForm、props 等） |
| **数据库持久层** | `authentication` 表 | 关系型行记录 | 存储管理员在后台配置的启用状态、排序、配置值、自注册开关等 |
| **内存运行层** | `WIKI.auth.strategies` 对象 | JS 运行时字典 | 合并磁盘定义 + 数据库配置后的运行时策略实例，供请求时直接查找 |

### 1.1 磁盘定义层 — definition.yml

每个策略目录下必须存在 `definition.yml`，它定义策略的"能力声明"。以 `local` 和 `google` 为例：

**Local**（`server/modules/authentication/local/definition.yml`）：
```yaml
key: local
title: Local Database
useForm: true          # 使用表单登录（用户名+密码）
usernameType: email    # 表单字段类型
props: {}              # 无额外配置项
```

**Google**（`server/modules/authentication/google/definition.yml`）：
```yaml
key: google
title: Google
useForm: false         # 社交登录，跳转 OAuth
scopes:
  - profile
  - email
  - openid
props:
  clientId:
    type: String
    title: Client ID
    order: 1
  clientSecret:
    type: String
    title: Client Secret
    order: 2
```

关键字段含义：
- **`useForm`**：`true` = 表单类策略（Local/LDAP），`false` = 社交跳转类策略（Google/GitHub 等）
- **`props`**：管理员在后台可配置的参数定义，含类型、默认值、顺序
- **`scopes`**：OAuth 类策略需要的授权范围

### 1.2 数据库持久层 — authentication 表

对应模型：`server/models/authentication.js`。

```js
// server/models/authentication.js:13-16
module.exports = class Authentication extends Model {
  static get tableName() { return 'authentication' }
  static get idColumn() { return 'key' }
}
```

表结构核心列：

| 列名 | 说明 |
|------|------|
| `key` | 策略实例唯一标识（主键），如 `local`、`google` |
| `strategyKey` | 策略类型标识，指向磁盘定义的 `key`，如 `local`、`google` |
| `isEnabled` | 是否启用 |
| `order` | 排序权重，决定在登录页上的显示顺序 |
| `displayName` | 管理员自定义的显示名称 |
| `config` | JSON，管理员填写的 props 配置值 |
| `selfRegistration` | 是否允许自注册 |
| `domainWhitelist` | 邮箱域名白名单 |
| `autoEnrollGroups` | 自动加入的分组 |

**关键：`key` 与 `strategyKey` 的区别**
- `strategyKey` 指向磁盘上的策略类型（如 `google`）
- `key` 是实例标识，理论上同一类型可以创建多个实例（如 `google-company-a`、`google-company-b`）

#### 初始数据写入

在系统初始化时（`server/setup.js:259-269`），默认插入 Local 策略：

```js
await WIKI.models.authentication.query().insert({
  key: 'local',
  strategyKey: 'local',
  displayName: 'Local',
  isEnabled: true,
  selfRegistration: false,
  order: 0,
  config: {},
  domainWhitelist: {v: []},
  autoEnrollGroups: {v: []}
})
```

#### 从磁盘刷新定义

`refreshStrategiesFromDisk()` 方法（`server/models/authentication.js:78-130`）负责：

1. 扫描 `server/modules/authentication/` 下所有子目录
2. 读取每个目录下的 `definition.yml`，解析后存入 `WIKI.data.authentication` 数组
3. 对比数据库已有策略：
   - 如果磁盘定义已被删除 → 从数据库中删除该策略
   - 如果磁盘定义新增了 props → 用默认值补齐 `config` 中缺失的键并写回数据库
   - 修复旧版本缺少 `displayName` 的记录

```js
// server/models/authentication.js:83-92
const authDirs = await fs.readdir(path.join(WIKI.SERVERPATH, 'modules/authentication'))
WIKI.data.authentication = []
for (let dir of authDirs) {
  const defRaw = await fs.readFile(path.join(WIKI.SERVERPATH, 'modules/authentication', dir, 'definition.yml'), 'utf8')
  const def = yaml.safeLoad(defRaw)
  WIKI.data.authentication.push({
    ...def,
    props: commonHelper.parseModuleProps(def.props)
  })
}
```

### 1.3 内存运行层 — WIKI.auth.strategies

对应核心代码：`server/core/auth.js:59-104` 的 `activateStrategies()` 方法。

这是三层中最终被请求消费的注册表，存储在 `WIKI.auth.strategies` 对象中（键为策略 `key`）。

激活流程：

```
activateStrategies()
    │
    ├─ 1. 清空旧策略：WIKI.auth.strategies = {}，passport.unuse() 卸载所有已注册策略
    │
    ├─ 2. 注册 JWT 策略（固定，用于后续请求鉴权）
    │
    └─ 3. 遍历数据库中已启用的策略：
         │
         ├─ a. 按 strategyKey 动态 require 对应的 authentication.js
         │     require(`../modules/authentication/${stg.strategyKey}/authentication.js`)
         │
         ├─ b. 注入 callbackURL 和 key 到配置
         │     stg.config.callbackURL = `${WIKI.config.host}/login/${stg.key}/callback`
         │
         ├─ c. 调用 strategy.init(passport, stg.config)
         │     → 每个策略自行调用 passport.use() 注册到 Passport
         │
         └─ d. 合并写入内存注册表
               WIKI.auth.strategies[stg.key] = { ...strategy, ...stg }
               包含：init、config、key、strategyKey、displayName、isEnabled、
                     selfRegistration、domainWhitelist、autoEnrollGroups 等
```

关键代码：

```js
// server/core/auth.js:79-93
const enabledStrategies = await WIKI.models.authentication.getStrategies()
for (let idx in enabledStrategies) {
  const stg = enabledStrategies[idx]
  const strategy = require(`../modules/authentication/${stg.strategyKey}/authentication.js`)

  stg.config.callbackURL = `${WIKI.config.host}/login/${stg.key}/callback`
  stg.config.key = stg.key
  strategy.init(passport, stg.config)
  strategy.config = stg.config

  WIKI.auth.strategies[stg.key] = {
    ...strategy,     // { init, config, logout? }
    ...stg           // { key, strategyKey, displayName, isEnabled, ... }
  }
}
```

**每个策略模块的标准接口**（`authentication.js`）：

```js
module.exports = {
  init(passport, conf) {
    // 调用 passport.use() 注册 Passport 策略
  },
  logout(conf) {
    // 可选，策略特定的登出逻辑
  }
}
```

---

## 二、两层挑选逻辑

### 2.1 前端策略挑选 — 登录页面

**入口组件**：`client/components/login.vue`

#### 加载策略列表

通过 Apollo GraphQL 查询获取已启用的策略列表：

```graphql
# client/components/login.vue:670-688
{
  authentication {
    activeStrategies(enabledOnly: true) {
      key
      strategy {
        key
        logo
        color
        icon
        useForm
        usernameType
      }
      displayName
      order
      selfRegistration
    }
  }
}
```

返回结果按 `order` 排序：`update: (data) => _.sortBy(data.authentication.activeStrategies, ['order'])`

#### 挑选决策流程

```
用户访问 /login
    │
    ├─ 场景1：autoLogin=true 且 URL 无 ?all 参数
    │   → 查询数据库中 order 最小的策略
    │   → 若该策略 useForm=false（社交登录）→ 直接重定向到 /login/{stg.key}
    │   → 若 useForm=true → 显示登录表单
    │
    ├─ 场景2：hideLocal=true 且 URL 无 ?all 参数
    │   → 从策略列表中过滤掉 key='local' 的策略
    │
    └─ 场景3：正常多策略选择
        │
        ├─ 若策略数 > 1 → 显示策略选择列表
        │   用户点击某策略 → selectedStrategyKey 变化
        │
        └─ 根据 selectedStrategy.strategy.useForm 分流：
            ├─ useForm=true → 显示用户名+密码表单
            │   提交时 GraphQL mutation 携带 strategy 参数
            │
            └─ useForm=false → 直接 window.location.assign('/login/' + newValue)
                跳转到后端 OAuth 回调流程
```

关键代码：

```js
// client/components/login.vue:328-341
selectedStrategyKey (newValue, oldValue) {
  this.selectedStrategy = _.find(this.strategies, ['key', newValue])
  this.screen = 'login'
  if (!this.selectedStrategy.strategy.useForm) {
    this.isLoading = true
    window.location.assign('/login/' + newValue)  // 社交登录跳转
  } else {
    this.$nextTick(() => {
      this.$refs.iptEmail.focus()  // 表单登录聚焦
    })
  }
}
```

#### 表单登录的 GraphQL 请求

```graphql
mutation($username: String!, $password: String!, $strategy: String!) {
  authentication {
    login(username: $username, password: $password, strategy: $strategy) {
      responseResult { succeeded }
      jwt
      mustChangePwd
      mustProvideTFA
      mustSetupTFA
      continuationToken
      redirect
      tfaQRImage
    }
  }
}
```

注意 `strategy` 参数传递的是策略 `key`（如 `local`），而非 `strategyKey`。

### 2.2 后端策略挑选 — 请求路由与执行

#### 路由层挑选

**Express 路由**（`server/controllers/auth.js`）根据 URL 模式分发：

| 路由 | 场景 |
|------|------|
| `GET /login` | 渲染登录页面 |
| `GET /login/:strategy` | 社交登录：发起 OAuth 跳转 |
| `ALL /login/:strategy/callback` | 社交登录：OAuth 回调处理 |
| `POST /login` | 旧版表单登录（Legacy IE） |

社交登录的路由参数 `:strategy` 就是策略的 `key`，通过 `WIKI.models.users.login()` 转交内部挑选逻辑。

#### 内部挑选 — User.login()

核心方法在 `server/models/users.js:293-332`：

```js
static async login (opts, context) {
  if (_.has(WIKI.auth.strategies, opts.strategy)) {
    const selStrategy = _.get(WIKI.auth.strategies, opts.strategy)
    if (!selStrategy.isEnabled) {
      throw new WIKI.Error.AuthProviderInvalid()
    }

    const strInfo = _.find(WIKI.data.authentication, ['key', selStrategy.strategyKey])

    // useForm 策略：注入用户名/密码到请求体
    if (strInfo.useForm) {
      _.set(context.req, 'body.email', opts.username)
      _.set(context.req, 'body.password', opts.password)
      _.set(context.req.params, 'strategy', opts.strategy)
    }

    // 调用 Passport 认证
    return new Promise((resolve, reject) => {
      WIKI.auth.passport.authenticate(selStrategy.key, {
        session: !strInfo.useForm,
        scope: strInfo.scopes ? strInfo.scopes : null
      }, async (err, user, info) => {
        // 认证成功后执行 afterLoginChecks
        const resp = await WIKI.models.users.afterLoginChecks(user, context, {
          skipTFA: !strInfo.useForm,
          skipChangePwd: !strInfo.useForm
        })
        resolve(resp)
      })(context.req, context.res, () => {})
    })
  } else {
    throw new WIKI.Error.AuthProviderInvalid()
  }
}
```

**挑选决策树**：

```
login(opts, context)
    │
    ├─ 1. 检查策略是否存在于内存注册表
    │   └─ WIKI.auth.strategies[opts.strategy] 不存在 → 抛 AuthProviderInvalid
    │
    ├─ 2. 检查策略是否启用
    │   └─ selStrategy.isEnabled === false → 抛 AuthProviderInvalid
    │
    ├─ 3. 查找磁盘定义获取策略元信息
    │   └─ _.find(WIKI.data.authentication, ['key', selStrategy.strategyKey])
    │
    ├─ 4. 根据 useForm 分流
    │   ├─ useForm=true  → 注入表单数据到 req.body，session=false
    │   └─ useForm=false → 不注入表单数据，session=true，附带 OAuth scopes
    │
    └─ 5. 调用 passport.authenticate(selStrategy.key, ...)
        → Passport 查找之前 init() 注册的策略实例执行认证
        → 认证回调中执行 afterLoginChecks
            ├─ 2FA 检查
            ├─ 强制改密检查
            └─ 签发 JWT
```

---

## 三、策略生命周期完整时序

```
┌──────────────────────────────────────────────────────────────────────┐
│  系统启动                                                             │
│                                                                      │
│  1. refreshStrategiesFromDisk()                                      │
│     扫描磁盘 → WIKI.data.authentication[]                            │
│     对比数据库 → 补齐新 props、删除已移除策略                            │
│                                                                      │
│  2. activateStrategies()                                             │
│     查询数据库 → getStrategies() 按 order 排序                        │
│     遍历 → require 模块 → init(passport, config)                     │
│     合并 → WIKI.auth.strategies[key] = {...strategy, ...stg}         │
│                                                                      │
│  3. Passport 内部注册表                                               │
│     passport._strategies[key] = 策略实例                              │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  管理员更新策略（GraphQL mutation updateStrategies）                    │
│                                                                      │
│  1. 写入数据库：patch 已有策略 或 insert 新策略                         │
│  2. 删除被移除的策略（检查是否有用户关联）                               │
│  3. 调用 WIKI.auth.activateStrategies() 重新加载                       │
│  4. 发送 HA 事件 WIKI.events.outbound.emit('reloadAuthStrategies')    │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  用户登录请求                                                         │
│                                                                      │
│  前端：Apollo 查询 activeStrategies(enabledOnly:true) → 按 order 排序  │
│       选择策略 → useForm? 表单提交 : 跳转 /login/:key                  │
│                                                                      │
│  后端：WIKI.auth.strategies[key] → 查找 → passport.authenticate()     │
│       → 策略实例执行认证 → 回调 → afterLoginChecks → 签发 JWT          │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 四、21 种内置策略分类

| 分类 | 策略 | useForm | 特点 |
|------|------|---------|------|
| **表单类** | `local` | ✅ | 本地数据库认证，支持自注册、2FA、密码重置 |
| **表单类** | `ldap` | ✅ | LDAP 目录认证 |
| **OAuth 跳转类** | `google`, `github`, `gitlab`, `facebook`, `microsoft`, `azure`, `discord`, `slack`, `twitch`, `dropbox`, `rocketchat` | ❌ | 标准 OAuth2/OAuth1 跳转流程 |
| **企业协议类** | `saml` | ❌ | SAML 2.0 断言，字段映射丰富 |
| **企业协议类** | `oidc` | ❌ | OpenID Connect |
| **企业协议类** | `cas` | ❌ | CAS 协议 |
| **企业平台类** | `auth0`, `okta`, `keycloak`, `firebase` | ❌ | 特定平台的 OAuth/OIDC 封装 |
| **通用类** | `oauth2` | ❌ | 通用 OAuth2 适配器 |

---

## 六、策略加载顺序对选中结果的影响

`order` 字段是贯穿整个策略体系的核心排序依据，它在 **注册**、**展示**、**自动登录** 三个阶段持续发挥作用。

### 6.1 order 在各阶段的流转

| 阶段 | 排序执行点 | 代码位置 | 影响 |
|------|-----------|---------|------|
| 数据库查询 | `getStrategies()` 以 `orderBy('order')` 返回 | `server/models/authentication.js:38` | 决定 `activateStrategies()` 遍历顺序 |
| 内存注册 | `activateStrategies()` 按查询结果顺序遍历 | `server/core/auth.js:79-99` | 遍历顺序不影响字典结构，但影响日志输出顺序 |
| 前端展示 | Apollo `update` 回调以 `_.sortBy(strategies, ['order'])` 排序 | `client/components/login.vue:690` | 决定登录页策略列表从上到下的排列 |
| 自动登录 | 服务端 `orderBy('order').first()` 取排序最前的策略 | `server/controllers/auth.js:38` | 决定 autoLogin 场景下使用哪个策略 |
| 管理后台 | 管理页 `_.sortBy(activeStrategies, ['order'])` 排序 | `client/components/admin/admin-auth.vue:404` | 决定管理页策略列表从上到下的排列 |
| 管理后台保存 | `order: idx` 用数组下标重算 | `client/components/admin/admin-auth.vue:317` | 拖拽排序后新 order 写入数据库 |

### 6.2 自动登录场景：order 的决定性作用

当 `WIKI.config.auth.autoLogin = true` 时，用户访问 `/login` 会被自动跳转，此时 **order 值最小的策略被选中**：

```js
// server/controllers/auth.js:37-43
if (WIKI.config.auth.autoLogin && !req.query.all) {
  const stg = await WIKI.models.authentication.query().orderBy('order').first()
  const stgInfo = _.find(WIKI.data.authentication, ['key', stg.strategyKey])
  if (!stgInfo.useForm) {
    return res.redirect(`/login/${stg.key}`)
  }
}
```

关键细节：
- **不检查 `isEnabled`**：`orderBy('order').first()` 未加 `where('isEnabled', true)` 过滤，如果 order 最小的策略被禁用，仍会被选中。但后续 `User.login()` 的 `isEnabled` 检查会拦截，导致自动登录失败
- **仅跳转非表单策略**：如果 order 最小的策略 `useForm=true`（如 Local），不跳转，而是照常渲染登录页让用户手动填写
- **`?all` 参数绕过**：用户可在 URL 加 `?all` 强制显示策略选择列表，无视 autoLogin

### 6.3 前端默认选中：filteredStrategies 的首项

前端登录组件通过 `watch: filteredStrategies` 自动选中排序后第一个表单策略：

```js
// client/components/login.vue:323-327
filteredStrategies (newValue, oldValue) {
  if (_.head(newValue).strategy.useForm) {
    this.selectedStrategyKey = _.head(newValue).key
  }
}
```

这意味着：
- 如果 `hideLocal=true`，Local 被过滤后，排序最靠前的非 Local 策略若为表单类则自动选中
- 如果排序最靠前的策略为社交类（`useForm=false`），不自动选中，用户需手动点击，点击后立即触发跳转

### 6.4 管理后台拖拽排序 → order 重算

管理后台使用 `vuedraggable` 组件实现拖拽排序，保存时以数组下标作为新 order 值：

```js
// client/components/admin/admin-auth.vue:313-323
strategies: this.activeStrategies.map((str, idx) => ({
  key: str.key,
  strategyKey: str.strategy.key,
  order: idx,  // 数组下标即为新 order
  isEnabled: str.isEnabled,
  ...
}))
```

因此管理员在后台拖拽调整策略顺序后点击 Apply，即重写了所有策略的 order 值，直接影响登录页展示和自动登录行为。

---

## 七、多策略并存时的优先级与冲突解决

Wiki.js 的认证体系是 **用户显式选择策略**，而非系统自动尝试多个策略，因此不存在传统意义上的"回退链"或"优先级链"。但多策略并存时仍有几类冲突需要关注。

### 7.1 用户-策略绑定模型

每个用户记录通过 `providerKey` 字段绑定到创建该用户的策略实例：

```js
// server/models/users.js:170-173
let user = await WIKI.models.users.query().findOne({
  providerId: _.toString(profile.id),
  providerKey
})
```

这意味着：
- 用户由策略 A 创建后，其 `providerKey = A.key` 是固定的
- 用户登录时必须通过同一个策略 A 认证，**不能跨策略登录**
- Local 策略的用户只能用邮箱+密码登录，不能通过 Google 登录同一账号（即使邮箱相同）

### 7.2 同邮箱跨策略的冲突处理

`processProfile()` 中有针对邮箱冲突的特殊处理逻辑：

```js
// server/models/users.js:194-205
if (!user) {
  user = await WIKI.models.users.query().findOne({
    email: primaryEmail,
    providerId: null,
    providerKey
  })
  if (user) {
    user = await user.$query().patchAndFetch({
      providerId: _.toString(profile.id)
    })
  }
}
```

这段逻辑处理的是 **同一策略下** 的"待关联用户"：如果存在同邮箱且 `providerId` 为空（即尚未完成社交登录关联）的用户，自动补上 `providerId`。但 **不会跨策略关联**——`providerKey` 是查询条件之一。

### 7.3 hideLocal 场景：策略可见性冲突

配置 `WIKI.config.auth.hideLocal = true` 时，Local 策略从前端列表中隐藏：

```js
// client/components/login.vue:310-316
filteredStrategies () {
  const qParams = new URLSearchParams(window.location.search)
  if (this.hideLocal && !qParams.has('all')) {
    return _.reject(this.strategies, ['key', 'local'])
  } else {
    return this.strategies
  }
}
```

隐藏只是前端展示层面的——Local 策略仍在内存注册表和 Passport 中活跃。如果用户直接构造 GraphQL 请求 `strategy: 'local'`，仍然可以登录。这不是安全机制，而是 UX 简化。

### 7.4 删除策略时的用户关联冲突

管理员尝试删除策略时，系统检查是否有用户仍在使用：

```js
// server/graph/resolvers/authentication.js:232-238
for (const str of _.differenceBy(previousStrategies, args.strategies, 'key')) {
  const hasUsers = await WIKI.models.users.query().count('* as total').where({ providerKey: str.key }).first()
  if (_.toSafeInteger(hasUsers.total) > 0) {
    throw new Error(`Cannot delete ${str.displayName} as 1 or more users are still using it.`)
  } else {
    await WIKI.models.authentication.query().delete().where('key', str.key)
  }
}
```

这是硬性约束：有用户的策略不可删除，只能禁用。禁用后已绑定该策略的用户将无法登录。

### 7.5 策略初始化失败时的容错

`activateStrategies()` 中单个策略初始化失败不影响其他策略：

```js
// server/core/auth.js:82-98
for (let idx in enabledStrategies) {
  const stg = enabledStrategies[idx]
  try {
    const strategy = require(`../modules/authentication/${stg.strategyKey}/authentication.js`)
    strategy.init(passport, stg.config)
    WIKI.auth.strategies[stg.key] = { ...strategy, ...stg }
    WIKI.logger.info(`Authentication Strategy ${stg.displayName}: [ OK ]`)
  } catch (err) {
    WIKI.logger.error(`Authentication Strategy ${stg.displayName} (${stg.key}): [ FAILED ]`)
    WIKI.logger.error(err)
    // 不抛出，继续加载下一个策略
  }
}
```

失败的策略不会进入 `WIKI.auth.strategies`，因此用户在登录页看不到也无法使用它。但数据库中它仍是 `isEnabled=true`，只是运行时未成功注册。

### 7.6 同类型多实例共存

由于 `key`（实例标识）和 `strategyKey`（类型标识）的分离设计，管理员可以创建同一策略类型的多个实例。例如添加两个 Google 策略实例，key 分别为 `google-team-a` 和 `google-team-b`，各自配置不同的 clientId/clientSecret 和 hostedDomain。

这在登录页上会显示为两个独立条目，用户选择其中一个进行认证。两个实例的 Passport 策略通过 `conf.key` 区分（见 `server/modules/authentication/google/authentication.js:59`：`passport.use(conf.key, strategy)`），互不干扰。

---

## 八、策略动态启用/禁用与运行时切换链路

### 8.1 完整切换链路

管理员在后台修改策略配置并点击 Apply 后，触发以下链路：

```
┌─ 前端 admin-auth.vue ─────────────────────────────────────────────┐
│  save() → GraphQL mutation updateStrategies                        │
│  变量：activeStrategies.map((str, idx) => ({                       │
│    key: str.key,                                                   │
│    strategyKey: str.strategy.key,                                  │
│    order: idx,                                                     │
│    isEnabled: str.isEnabled,    ← 禁用/启用开关                    │
│    config: [...],               ← 配置变更                         │
│    ...                                                             │
│  }))                                                               │
└────────────────────────┬──────────────────────────────────────────┘
                         │
                         ▼
┌─ 后端 GraphQL Resolver ───────────────────────────────────────────┐
│  updateStrategies() — server/graph/resolvers/authentication.js:200│
│                                                                    │
│  1. 获取 previousStrategies（变更前快照）                            │
│                                                                    │
│  2. 遍历 args.strategies：                                         │
│     ├─ 已存在 → patch 更新（含 isEnabled、order、config 等）         │
│     └─ 新增 → insert 新行                                          │
│                                                                    │
│  3. 遍历被删除的策略：                                              │
│     ├─ 有关联用户 → 抛错，整个事务回滚                               │
│     └─ 无关联用户 → delete 删除                                     │
│                                                                    │
│  4. await WIKI.auth.activateStrategies()  ← 本进程立即重载          │
│                                                                    │
│  5. WIKI.events.outbound.emit('reloadAuthStrategies')              │
│     ← 通知集群其他进程重载                                          │
└────────────────────────┬──────────────────────────────────────────┘
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
┌─ 本进程 activateStrategies() ─┐  ┌─ 其他进程（HA 集群）──────────┐
│  1. WIKI.auth.strategies = {}  │  │                              │
│  2. passport.unuse() 卸载全部  │  │  PG LISTEN/NOTIFY 通道       │
│  3. passport.use('jwt', ...)   │  │  → wiki channel              │
│  4. 遍历数据库已启用策略：      │  │  → payload.event             │
│     require → init → 写入      │  │    = 'reloadAuthStrategies'  │
│     WIKI.auth.strategies[key]  │  │  → WIKI.events.inbound.emit  │
│                                │  │    ('reloadAuthStrategies')   │
│  效果：                        │  │  → WIKI.auth.activateStrategies()│
│  - 被禁用的策略从内存移除       │  │    在另一进程重新执行         │
│  - 新启用的策略注册到 Passport  │  │                              │
│  - 配置变更的策略重新初始化     │  │                              │
└────────────────────────────────┘  └──────────────────────────────┘
```

### 8.2 activateStrategies() 的全量重载策略

`activateStrategies()` 不是增量更新，而是 **全量清空重建**：

```js
// server/core/auth.js:61-65
WIKI.auth.strategies = {}
const currentStrategies = _.keys(passport._strategies)
_.pull(currentStrategies, 'session')
_.forEach(currentStrategies, stg => { passport.unuse(stg) })
```

步骤分解：
1. 清空内存注册表 `WIKI.auth.strategies = {}`
2. 从 Passport 内部注册表 `passport._strategies` 取出所有已注册策略名
3. 排除 `session`（Passport 内置策略，不可卸载）
4. 逐个 `passport.unuse()` 卸载
5. 重新注册 JWT 策略（固定的，用于请求鉴权）
6. 从数据库查询 `isEnabled` 的策略，逐个 `require → init → 写入`

这意味着每次策略变更都触发一次 **完整的卸载-重装周期**，不存在"只禁用某个策略"的增量操作。

### 8.3 HA 集群间的运行时同步

在多实例部署（HA 模式）下，策略变更需要同步到所有进程。同步机制基于 PostgreSQL 的 `LISTEN/NOTIFY`：

```js
// server/core/db.js:250-256
this.listener.addChannel('wiki', payload => {
  if (_.has(payload, 'event') && payload.source !== WIKI.INSTANCE_ID) {
    WIKI.events.inbound.emit(payload.event, payload.value)
  }
})
WIKI.events.outbound.onAny(this.notifyViaDB)
```

关键设计：
- **`source !== WIKI.INSTANCE_ID`**：过滤掉自己发出的通知，避免重复执行
- **inbound 事件映射**（`server/core/auth.js:478-491`）：

| 事件名 | 触发动作 |
|--------|---------|
| `reloadAuthStrategies` | `WIKI.auth.activateStrategies()` |
| `reloadGroups` | `WIKI.auth.reloadGroups()` |
| `reloadApiKeys` | `WIKI.auth.reloadApiKeys()` |
| `addAuthRevoke` | `WIKI.auth.revokeUserTokens(args)` |

### 8.4 运行时切换的瞬时行为

策略重载期间，系统的行为如下：

1. **`passport.unuse()` 瞬间生效**：被卸载的策略如果此时恰好有 OAuth 回调到达，Passport 找不到对应策略会返回 `authenticate` 失败
2. **重载是同步阻塞的**：`activateStrategies()` 内部虽然用了 `for...in` 遍历，但每个策略的 `init()` 是同步调用（策略模块的 `passport.use()` 本身同步），因此重载窗口极短
3. **JWT 策略优先恢复**：`passport.use('jwt', ...)` 在业务策略之前注册，确保正在使用 JWT 的用户请求不受影响
4. **已持有 JWT 的用户不受影响**：策略重载只影响登录流程，不影响 `authenticate()` 中间件对已有 JWT 的验证

### 8.5 Local 策略的特殊保护

Local 策略在管理后台有特殊约束：

```html
<!-- client/components/admin/admin-auth.vue:98 -->
:disabled='strategy.key === `local`'
```

- **不可禁用**：Local 策略的启用开关始终灰显，无法关闭
- **不可删除**：删除按钮在 Local 策略上同样灰显
- **初始 order=0**：系统初始化时 Local 策略 order 固定为 0（`server/setup.js:266`），在排序中默认最靠前

这是安全兜底设计：确保管理员不会把自己锁在系统外面（至少总有 Local 方式可以登录）。

### 8.6 策略禁用后的影响范围

一个策略被禁用（`isEnabled: false`）后的影响：

| 维度 | 行为 |
|------|------|
| 内存注册表 | 该策略不进入 `WIKI.auth.strategies`，无法被 `User.login()` 查找 |
| Passport | 该策略不被 `passport.use()` 注册，无法通过 `passport.authenticate()` 调用 |
| 登录页 | `activeStrategies(enabledOnly: true)` 不返回该策略，前端不显示 |
| 已绑定用户 | 这些用户的 `providerKey` 仍指向该策略，但他们无法登录 |
| 数据库记录 | 记录保留，`isEnabled` 标记为 `false` |
| OAuth 回调 | `/login/:strategy/callback` 路由仍存在，但 `User.login()` 查找不到策略会抛 `AuthProviderInvalid` |

### 8.7 证书重签触发策略重载

`regenerateCertificates()` 在重签 RSA 证书后也需要重载策略，因为 JWT 策略使用证书进行验证：

```js
// server/core/auth.js:410-441
async regenerateCertificates () {
  // 生成新证书...
  await WIKI.configSvc.saveToDb(['certs', 'sessionSecret'])
  await WIKI.auth.activateStrategies()     // 本进程重载
  WIKI.events.outbound.emit('reloadAuthStrategies')  // 集群同步
}
```

---

## 九、策略链失败时的 fallback 与短路逻辑

Wiki.js 的认证体系设计是 **用户显式选择策略**，而非系统自动遍历策略链，因此不存在传统意义上的"依次尝试多个策略"的 fallback 链。但在特定场景下仍存在短路（short-circuit）和回退（fallback）行为。

### 9.1 无策略链设计：一次请求对应一个策略

核心逻辑在 `User.login()` 中（`server/models/users.js:293-332`）：

```js
static async login (opts, context) {
  if (_.has(WIKI.auth.strategies, opts.strategy)) {
    const selStrategy = _.get(WIKI.auth.strategies, opts.strategy)
    if (!selStrategy.isEnabled) {
      throw new WIKI.Error.AuthProviderInvalid()   // 短路：策略禁用，直接失败
    }
    // ... Passport 认证
  } else {
    throw new WIKI.Error.AuthProviderInvalid()     // 短路：策略不存在，直接失败
  }
}
```

**关键点**：
- 请求携带的 `strategy` 参数直接定位唯一策略
- 不存在 `for...of` 遍历多个策略依次尝试的逻辑
- 任一前置检查失败（策略不存在、未启用）立即短路返回错误，不继续执行

### 9.2 Passport 认证失败的短路

`passport.authenticate()` 的回调函数同样不做 fallback：

```js
// server/models/users.js:314-316
async (err, user, info) => {
  if (err) { return reject(err) }                 // 策略内部错误，短路
  if (!user) { return reject(new WIKI.Error.AuthLoginFailed()) }  // 认证失败，短路
  // ... 执行 afterLoginChecks
}
```

例如 LDAP 认证失败、Google OAuth 返回错误、Local 密码错误等，均直接 `reject`，不会自动尝试下一个策略。

### 9.3 策略初始化失败的容错：非短路

与请求时的行为相反，**系统启动/重载时单个策略初始化失败不会导致整个系统瘫痪**：

```js
// server/core/auth.js:82-99
for (let idx in enabledStrategies) {
  const stg = enabledStrategies[idx]
  try {
    const strategy = require(`../modules/authentication/${stg.strategyKey}/authentication.js`)
    strategy.init(passport, stg.config)
    WIKI.auth.strategies[stg.key] = { ...strategy, ...stg }
    WIKI.logger.info(`Authentication Strategy ${stg.displayName}: [ OK ]`)
  } catch (err) {
    WIKI.logger.error(`Authentication Strategy ${stg.displayName} (${stg.key}): [ FAILED ]`)
    WIKI.logger.error(err)
    // 不抛出，不短路，继续加载下一个策略
  }
}
```

这是容错（fault-tolerant）设计而非 fallback 设计——失败的策略只是从注册表中缺席，用户看不到也用不了，但其他策略正常工作。

### 9.4 自动登录场景的 fallback

`autoLogin` 场景下存在两处隐性 fallback：

**Fallback 1：自动登录只跳转非表单策略**
```js
// server/controllers/auth.js:37-43
if (WIKI.config.auth.autoLogin && !req.query.all) {
  const stg = await WIKI.models.authentication.query().orderBy('order').first()
  const stgInfo = _.find(WIKI.data.authentication, ['key', stg.strategyKey])
  if (!stgInfo.useForm) {
    return res.redirect(`/login/${stg.key}`)  // 社交策略：跳转
  }
  // 否则：不跳转，fallback 到正常登录页
}
```
如果 order 最小的策略是 Local（`useForm=true`），不执行自动跳转，回退到让用户手动输入用户名密码。

**Fallback 2：`?all` 参数绕过自动登录**
用户可在 URL 加 `?all` 显式绕过 autoLogin 逻辑，强制显示完整策略列表。这是用户主动发起的 fallback 行为。

### 9.5 自动登录选中禁用策略的失效场景

注意 `autoLogin` 的 `orderBy('order').first()` **不检查 `isEnabled`**：
```js
const stg = await WIKI.models.authentication.query().orderBy('order').first()
// 没有 where('isEnabled', true)
```

如果管理员把 order=0 的策略禁用了，`autoLogin` 仍会选中它。虽然随后 `User.login()` 会检查 `isEnabled` 并抛错，但自动登录已经失效了。这是一个设计上的不完善——禁用的策略应从 `autoLogin` 候选中排除。

### 9.6 已禁用策略的 OAuth 回调短路

策略被禁用后，`/login/:strategy/callback` 路由仍然存在，但 `User.login()` 会拦截：

```
用户发起 OAuth 登录 → 策略被管理员禁用 → 用户完成第三方授权 →
回调到达 /login/google/callback → User.login() 检查 isEnabled=false →
抛 AuthProviderInvalid → 用户看到登录失败页面
```

整个链路不会执行 Passport 认证逻辑，在策略查找阶段就短路了。

### 9.7 Local 策略作为最终 fallback

Local 策略的特殊保护机制使其成为系统的最终 fallback：
- 不可禁用（管理后台开关灰显）
- 不可删除（管理后台删除按钮灰显）
- 初始 `order=0`，永远在策略列表中
- 即使所有其他策略都初始化失败，Local 只要配置正确就一定可用

这是关键的安全兜底设计——管理员永远不会把自己锁在系统外面。

---

## 十、策略组合的协同处理（LDAP 与 OAuth 互补场景）

Wiki.js 没有内置的"LDAP 失败自动 fallback 到 OAuth"或"多策略合并认证"逻辑。但通过策略设计和业务配置，可以实现若干种组合协同模式。

### 10.1 表单类策略的共用前端

Local 和 LDAP 都是 `useForm: true` 的表单类策略，它们**共享同一个登录表单**：

```js
// client/components/login.vue:41-43
template(v-if='screen === `login` && selectedStrategy.strategy.useForm')
  v-text-field(v-model='username' :placeholder='isUsernameEmail ? ... : ...')
  v-text-field(v-model='password' type='password')
  v-btn(@click='login')
```

用户选择不同的表单类策略（Local / LDAP），表单字段的 placeholder 会变化（`isUsernameEmail` 计算属性），但表单 UI 是复用的。提交时 GraphQL mutation 携带 `strategy` 参数区分。

### 10.2 外部认证 + 本地用户创建的协同

**流程模式**：`外部身份源认证 → Wiki.js 自动创建/更新本地用户`

这是所有社交策略和企业协议策略的标准协同模式，以 LDAP 为例：

```js
// server/modules/authentication/ldap/authentication.js:34-49
async (req, profile, cb) => {
  const user = await WIKI.models.users.processProfile({
    providerKey: req.params.strategy,
    profile: {
      id: userId,
      email: _.get(profile, conf.mappingEmail, ''),
      displayName: _.get(profile, conf.mappingDisplayName, '???'),
      picture: _.get(profile, `_raw.${conf.mappingPicture}`, '')
    }
  })
  cb(null, user)
}
```

`processProfile()` 内部协同逻辑：

1. **查找已有用户**：用 `providerId + providerKey` 精确匹配
2. **自动关联**：同策略下同邮箱且 `providerId` 为空的用户自动补上 `providerId`
3. **自注册创建**：如果 `provider.selfRegistration=true` 且邮箱在白名单内，自动创建本地用户
4. **属性同步**：每次登录更新 email、displayName、picture

类似的协同也发生在 SAML、OIDC、Google、GitHub 等所有非 Local 策略中。

### 10.3 组映射的协同（LDAP/SAML + 本地分组）

LDAP 和 SAML 策略支持组映射，实现 `外部组 → Wiki.js 组` 的自动同步：

```js
// server/modules/authentication/ldap/authentication.js:51-63
if (conf.mapGroups) {
  const ldapGroups = _.get(profile, '_groups')
  const expectedGroups = Object.values(WIKI.auth.groups)
    .filter(g => ldapGroups.includes(g[conf.groupNameField]))
    .map(g => g.id)
  // 同步加入新组
  for (const groupId of _.difference(expectedGroups, currentGroups)) {
    await user.$relatedQuery('groups').relate(groupId)
  }
  // 同步移除不在 LDAP 中的组
  for (const groupId of _.difference(currentGroups, expectedGroups)) {
    await user.$relatedQuery('groups').unrelate().where('groupId', groupId)
  }
}
```

这是深度协同——不仅认证委托给外部，连用户的权限组也与外部源保持同步。

### 10.4 多 OAuth 策略共存的组合

多个社交策略可同时启用，它们之间是"并列选择"关系而非"顺序 fallback"关系：

```
登录页显示：
  [ Google 图标 ] Sign in with Google
  [ GitHub 图标 ] Sign in with GitHub
  [ Azure 图标  ] Sign in with Azure AD
```

每个策略独立进行 OAuth 跳转，独立回调，独立创建本地用户。同一个人可以同时存在 `google-123` 和 `github-456` 两个不同的本地账号（`providerKey` 不同）。

### 10.5 业务上的策略组合配置模式

虽然代码没有内置组合逻辑，但管理员可以通过配置实现常见的业务组合：

| 组合模式 | 配置方式 | 适用场景 |
|---------|---------|---------|
| **LDAP + Local 应急** | 同时启用 LDAP 和 Local；日常用 LDAP，LDAP 故障时管理员用 Local 登录 | 企业内部部署 |
| **SAML + Local 应急** | 同时启用 SAML 和 Local；日常走 SSO，SSO 故障时 Local 作为逃生门 | 企业内部部署 |
| **多 GitHub 组织** | 创建两个 `strategyKey=github` 实例，配置不同的 clientId，分别对应不同组织 | 多组织协作场景 |
| **OAuth + Local 注册** | OAuth 仅用于登录，Local 策略开启自注册用于首次创建账号 | 混合场景 |
| **OIDC + 组映射** | OIDC 登录时自动映射外部组到 Wiki.js 组，实现权限 SSO | 企业统一权限 |

### 10.6 Local 与 LDAP 的用户隔离

注意 Local 和 LDAP 是完全隔离的——即使邮箱相同，也是不同用户：

```
Local 用户 alice@company.com → providerKey='local'
LDAP 用户 alice@company.com → providerKey='ldap'
```

`processProfile()` 不会跨策略关联用户。如果企业要从 Local 迁移到 LDAP，需要手动更新现有用户的 `providerKey` 和 `providerId`，或者让用户通过 LDAP 重新创建账号（邮箱会冲突，需要先删除或修改原 Local 用户的邮箱）。

---

## 十一、策略实例化与配置热刷新完整链路

### 11.1 冷启动完整链路

Wiki.js 启动时的策略实例化遵循严格的顺序：

```
┌─ server/core/kernel.js:init() ────────────────────────────────────┐
│  1. WIKI.models = require('./db').init()                           │
│  2. await WIKI.configSvc.loadFromDb()     // 从 DB 加载配置        │
│  3. this.bootMaster()                                             │
└────────────────────────┬──────────────────────────────────────────┘
                         │
                         ▼
┌─ bootMaster() → postBootMaster() ─────────────────────────────────┐
│                                                                    │
│  第一步：从磁盘加载策略元数据                                       │
│  ────────────────────────────────────                             │
│  await WIKI.models.authentication.refreshStrategiesFromDisk()     │
│    │                                                              │
│    ├─ 扫描 server/modules/authentication/ 目录                     │
│    ├─ 读取每个子目录的 definition.yml                              │
│    ├─ 解析 props，存入 WIKI.data.authentication[]                 │
│    ├─ 对比数据库：删除已移除的策略，补齐新增的 props                │
│    └─ 写回数据库（如有变更）                                       │
│                                                                    │
│  第二步：策略初始化到运行时                                         │
│  ────────────────────────────────────                             │
│  await WIKI.auth.activateStrategies()                             │
│    │                                                              │
│    ├─ WIKI.auth.strategies = {}  // 清空内存                      │
│    ├─ passport.unuse(...) 批量卸载                                │
│    ├─ passport.use('jwt', ...) 注册 JWT 策略                      │
│    └─ 遍历数据库 isEnabled 策略：                                  │
│       ├─ require(`../modules/authentication/${stg.strategyKey}/...`)│
│       ├─ stg.config.callbackURL = `${host}/login/${stg.key}/callback`│
│       ├─ strategy.init(passport, stg.config)                      │
│       └─ WIKI.auth.strategies[stg.key] = { ...strategy, ...stg }  │
│                                                                    │
│  第三步：HA 事件订阅                                               │
│  ────────────────────────────────────                             │
│  await WIKI.models.subscribeToNotifications()                     │
│    └─ WIKI.auth.subscribeToEvents()                                │
│       ├─ inbound.on('reloadAuthStrategies') → activateStrategies()│
│       ├─ inbound.on('reloadGroups') → reloadGroups()              │
│       └─ inbound.on('reloadApiKeys') → reloadApiKeys()            │
└───────────────────────────────────────────────────────────────────┘
```

### 11.2 单个策略的实例化细节

每个策略模块 `authentication.js` 的 `init()` 方法完成 Passport 策略实例化：

以 Local 策略为例：
```js
// server/modules/authentication/local/authentication.js:11-43
module.exports = {
  init (passport, conf) {
    passport.use('local',
      new LocalStrategy({
        usernameField: 'email',
        passwordField: 'password'
      }, async (uEmail, uPassword, done) => {
        // ... 具体认证逻辑
      })
    )
  }
}
```

关键点：
- `passport.use(conf.key, new Strategy(...))` — 用策略实例的 `key` 作为 Passport 内部标识
- 配置 `conf` 包含管理员在后台填写的所有 `props` 值
- `callbackURL` 由 `activateStrategies()` 注入，确保与实例 `key` 对应
- 实例化后的策略函数保存在 `passport._strategies[conf.key]`

### 11.3 热更新触发源

策略配置热更新可由以下操作触发：

| 触发操作 | 触发点 | 直接调用 | 集群广播 |
|---------|-------|---------|---------|
| 管理员后台保存策略配置 | `updateStrategies` GraphQL | `activateStrategies()` | ✅ `reloadAuthStrategies` |
| 证书重签 | `regenerateCertificates()` | `activateStrategies()` | ✅ `reloadAuthStrategies` |
| 组变更 | 增删改组 | `reloadGroups()` | ✅ `reloadGroups` |
| API Key 变更 | 增删 API Key | `reloadApiKeys()` | ✅ `reloadApiKeys` |
| 系统启动 | `postBootMaster()` | `activateStrategies()` | ❌ |

### 11.4 热更新完整链路（含 HA 集群）

```
┌─ 管理后台 admin-auth.vue ─────────────────────────────────────────┐
│  save() → GraphQL mutation updateStrategies                        │
│  variables.strategies = activeStrategies.map(...)                   │
│  含：key, strategyKey, order, isEnabled, config, ...                │
└────────────────────────┬──────────────────────────────────────────┘
                         │
                         ▼
┌─ GraphQL Resolver updateStrategies() ──────────────────────────────┐
│  server/graph/resolvers/authentication.js:200-249                   │
│                                                                    │
│  1. previousStrategies = getStrategies()  // 变更前快照            │
│                                                                    │
│  2. for...of args.strategies：                                      │
│     ├─ 已存在 → patch（更新 isEnabled、order、config 等）           │
│     └─ 新增 → insert                                               │
│                                                                    │
│  3. for...of 被删除策略：                                           │
│     ├─ 有关联用户 → throw Error（阻止删除）                         │
│     └─ 无关联用户 → delete                                          │
│                                                                    │
│  4. await WIKI.auth.activateStrategies()  // 本进程立即重载        │
│                                                                    │
│  5. WIKI.events.outbound.emit('reloadAuthStrategies')               │
│     ↓ 被 db.js 的 onAny 捕获                                        │
│     ↓ 通过 PG NOTIFY 广播到 wiki channel                            │
└────────────────────────┬──────────────────────────────────────────┘
                         │
              ┌──────────┴──────────────────────┐
              ▼                                 ▼
┌─ 本进程（执行操作的进程） ─┐    ┌─ 其他进程（HA 集群节点） ───────┐
│ 已执行 activateStrategies() │    │ PG LISTEN wiki channel        │
│ 内存注册表已更新           │    │ 收到 payload.event = 'reload...'│
│ Passport 已重新注册        │    │ source !== INSTANCE_ID         │
│                            │    │ → WIKI.events.inbound.emit(...)│
│                            │    │ → activateStrategies() 执行     │
└────────────────────────────┘    └────────────────────────────────┘
```

### 11.5 `refreshStrategiesFromDisk()` 与 `activateStrategies()` 的区别

| 维度 | `refreshStrategiesFromDisk()` | `activateStrategies()` |
|------|-------------------------------|------------------------|
| 作用 | 同步磁盘定义 ↔ 数据库 | 同步数据库 ↔ 内存运行时 |
| 操作 | 扫描目录、读取 YAML、DB patch/delete | require 模块、Passport 注册、内存字典写入 |
| 输出 | `WIKI.data.authentication[]`（元数据） | `WIKI.auth.strategies{}`（运行实例） |
| 时机 | 系统启动 + 管理员手动刷新 | 启动 + 每次配置变更 |
| 集群同步 | 不广播（仅本地执行） | 广播 `reloadAuthStrategies` 事件 |
| 频率 | 低（启动一次，除非管理员刷新） | 高（每次策略配置变更） |

### 11.6 热刷新期间的 JWT 连续性

策略热刷新期间，**已登录用户的 JWT 不受影响**：

1. JWT 策略在 `activateStrategies()` 中第一个被重新注册
2. 请求鉴权中间件 `authenticate()` 使用 `'jwt'` 策略，与业务策略无关
3. 业务策略（Google/LDAP/SAML 等）只影响登录流程，不影响已持 JWT 的请求

因此管理员可以在白天正常业务时段调整策略配置，不会导致在线用户掉线。

### 11.7 配置变更的原子性

策略配置变更不是数据库事务级原子的：

```js
// server/graph/resolvers/authentication.js:203-229
for (const str of args.strategies) {
  if (_.some(previousStrategies, ['key', str.key])) {
    await WIKI.models.authentication.query().patch(...).where('key', str.key)
  } else {
    await WIKI.models.authentication.query().insert(...)
  }
}
// ... 然后遍历删除
// ... 然后 activateStrategies()
```

- 多个策略的 update/insert 是独立数据库操作，非原子事务
- 如果中途某个操作失败，已完成的操作不会回滚
- 但失败时 `activateStrategies()` 不会被调用，运行时状态保持不变
- 用户可见的影响仅限于未完成更新的策略在数据库层面不一致

### 11.8 动态新增策略实例

管理员可在管理后台"Add Strategy"动态创建新的策略实例：

```js
// client/components/admin/admin-auth.vue:268-289
addStrategy (str) {
  const newStr = {
    key: uuid(),          // 随机生成实例 key
    strategy: str,        // 关联到策略类型定义
    config: str.props.map(c => ({
      key: c.key,
      value: { ...c, value: c.default }
    })),
    order: this.activeStrategies.length,
    isEnabled: true,
    displayName: str.title,
    ...
  }
  this.activeStrategies = [...this.activeStrategies, newStr]
}
```

保存后经过完整热更新链路，新的策略实例就出现在登录页上。

---

## 十二、关键源码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 内存注册表定义 | `server/core/auth.js` | :17 |
| 策略激活 `activateStrategies()` | `server/core/auth.js` | :59-104 |
| JWT 中间件认证 `authenticate()` | `server/core/auth.js` | :113-212 |
| HA 事件订阅 `subscribeToEvents()` | `server/core/auth.js` | :478-491 |
| 证书重签 → 触发策略重载 | `server/core/auth.js` | :410-443 |
| 认证缓存（`revocationList`） | `server/core/auth.js` | :23 |
| 策略数据模型 | `server/models/authentication.js` | :13-131 |
| 从磁盘刷新策略定义 `refreshStrategiesFromDisk()` | `server/models/authentication.js` | :78-130 |
| 获取策略列表（按 order 排序） `getStrategies()` | `server/models/authentication.js` | :37-44 |
| 旧版客户端策略分类 | `server/models/authentication.js` | :46-76 |
| 用户登录挑选逻辑 `login()`（无策略链设计） | `server/models/users.js` | :293-332 |
| Passport 认证回调（失败短路） | `server/models/users.js` | :314-316 |
| 登录后检查 `afterLoginChecks()` | `server/models/users.js` | :337-413 |
| 社交登录 Profile 处理 `processProfile()`（外部认证+本地创建协同） | `server/models/users.js` | :165-288 |
| 路由分发 | `server/controllers/auth.js` | :25-96 |
| 自动登录（取 order 最前策略 / 不检查 isEnabled） | `server/controllers/auth.js` | :37-43 |
| GraphQL 策略更新 `updateStrategies()` | `server/graph/resolvers/authentication.js` | :200-249 |
| 删除策略时的用户关联检查 | `server/graph/resolvers/authentication.js` | :232-238 |
| `activeStrategies` 查询（enabledOnly 过滤） | `server/graph/resolvers/authentication.js` | :52-74 |
| HA 集群 PG LISTEN/NOTIFY | `server/core/db.js` | :240-265 |
| 冷启动时序 `postBootMaster()` | `server/core/kernel.js` | :71-90 |
| 配置服务（`loadFromDb`/`saveToDb`） | `server/core/config.js` | :83-135 |
| 通用缓存 `init()` | `server/core/cache.js` | :1-7 |
| 前端登录组件 | `client/components/login.vue` | :1-697 |
| 前端策略查询 Apollo（按 order 排序） | `client/components/login.vue` | :669-695 |
| 前端 `filteredStrategies`（hideLocal 逻辑） | `client/components/login.vue` | :310-316 |
| 前端 `selectedStrategyKey` watcher | `client/components/login.vue` | :328-342 |
| 前端 `filteredStrategies` watcher（默认选中首项） | `client/components/login.vue` | :323-327 |
| 前端表单模板（Local/LDAP 共用） | `client/components/login.vue` | :41-43 |
| 管理后台策略配置页 | `client/components/admin/admin-auth.vue` | :1-433 |
| 管理后台拖拽排序保存 | `client/components/admin/admin-auth.vue` | :294-339 |
| Local 策略不可禁用/删除约束 | `client/components/admin/admin-auth.vue` | :68, :98 |
| 动态新增策略实例 `addStrategy()` | `client/components/admin/admin-auth.vue` | :268-289 |
| Local 策略实现 | `server/modules/authentication/local/authentication.js` | :1-44 |
| Google 策略实现（含 `conf.key` 区分多实例） | `server/modules/authentication/google/authentication.js` | :11-64 |
| SAML 策略实现 | `server/modules/authentication/saml/authentication.js` | :1-86 |
| LDAP 策略实现（含组映射协同） | `server/modules/authentication/ldap/authentication.js` | :11-74 |
| LDAP 策略定义（`useForm=true`） | `server/modules/authentication/ldap/definition.yml` | :1-165 |
| Local 策略定义（`useForm=true`） | `server/modules/authentication/local/definition.yml` | :1-30 |
| 初始化插入 Local 策略（`order=0`） | `server/setup.js` | :259-269 |
