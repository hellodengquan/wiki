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

## 五、关键源码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 内存注册表定义 | `server/core/auth.js` | :17 |
| 策略激活 `activateStrategies()` | `server/core/auth.js` | :59-104 |
| JWT 中间件认证 `authenticate()` | `server/core/auth.js` | :113-212 |
| 策略数据模型 | `server/models/authentication.js` | :13-131 |
| 从磁盘刷新策略定义 | `server/models/authentication.js` | :78-130 |
| 获取策略列表（排序） | `server/models/authentication.js` | :37-44 |
| 旧版客户端策略分类 | `server/models/authentication.js` | :46-76 |
| 用户登录挑选逻辑 `login()` | `server/models/users.js` | :293-332 |
| 登录后检查 `afterLoginChecks()` | `server/models/users.js` | :337-413 |
| 社交登录 Profile 处理 `processProfile()` | `server/models/users.js` | :165-288 |
| 路由分发 | `server/controllers/auth.js` | :25-96 |
| 自动登录逻辑 | `server/controllers/auth.js` | :37-43 |
| GraphQL 策略更新 | `server/graph/resolvers/authentication.js` | :200-249 |
| 前端登录组件 | `client/components/login.vue` | :1-697 |
| 前端策略查询 Apollo | `client/components/login.vue` | :669-695 |
| Local 策略实现 | `server/modules/authentication/local/authentication.js` | :1-44 |
| Google 策略实现 | `server/modules/authentication/google/authentication.js` | :1-64 |
| SAML 策略实现 | `server/modules/authentication/saml/authentication.js` | :1-86 |
| 初始化插入 Local 策略 | `server/setup.js` | :259-269 |
