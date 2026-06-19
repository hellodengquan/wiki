# Wiki.js 权限与可见性矩阵笔记

基于代码 `server/core/auth.js`、`server/models/groups.js`、`server/models/users.js`、`server/controllers/common.js` 等核心文件梳理。

---

## 1. 三层模型总览

```
用户(User) ──多对多──> 组(Group) ──包含──> 全局权限(permissions) + 页面规则(pageRules)
                                                     │
                                                     ▼
                                              页面可见性 / 操作许可
```

一次完整的权限判断流程：

1. 从用户所属的**所有组**中，聚合出**全局权限**（并集）
2. 若涉及具体页面，再根据所有组的**页面规则**做细粒度匹配
3. 两者**联合判定**：全局权限是前提，页面规则在全局权限之上做二次裁剪

---

## 2. Group（组）

### 2.1 数据模型

**数据库表** `groups`（`server/db/beta/migrations/2.0.0-beta.1.js:59`）：

| 字段 | 类型 | 说明 |
|---|---|---|
| id | integer PK | 组 ID |
| name | string | 组名 |
| permissions | JSON | 该组的全局权限字符串数组 |
| pageRules | JSON | 该组的页面规则数组 |
| isSystem | boolean | 是否系统内置组 |
| redirectOnLogin | string | 登录后重定向路径 |
| createdAt / updatedAt | string | 时间戳 |

**系统内置组**（id=1 和 id=2）：
- **id=1 — Administrators**：拥有 `manage:system`，等同于超级管理员
- **id=2 — Guests**：未登录用户所属组，由 `WIKI.auth.guest` 缓存

### 2.2 用户-组关系

通过中间表 `userGroups` 实现多对多：

```
users.id <── userGroups.userId ── userGroups.groupId ──> groups.id
```

用户可以同时属于多个组。

### 2.3 组在内存中的缓存

`server/core/auth.js:393-396`：

```js
async reloadGroups () {
  const groupsArray = await WIKI.models.groups.query()
  this.groups = _.keyBy(groupsArray, 'id')   // { 1: {id, name, permissions, pageRules, ...}, 2: {...}, ... }
}
```

`WIKI.auth.groups` 是以 `id` 为 key 的内存对象，每次组变更后调用 `reloadGroups()` 刷新。

---

## 3. 全局权限（Permissions）

### 3.1 权限字符串格式

格式为 `action:resource`，完整列表（来自前端 `admin-groups-edit-rules.vue` 及 GraphQL schema）：

| 权限 | 说明 |
|---|---|
| `read:pages` | 读取页面 |
| `write:pages` | 创建/编辑页面 |
| `manage:pages` | 重命名/移动页面 |
| `delete:pages` | 删除页面 |
| `read:source` | 查看页面源码 |
| `read:history` | 查看页面历史 |
| `read:assets` | 读取/使用资源 |
| `write:assets` | 上传资源 |
| `manage:assets` | 编辑/删除资源 |
| `write:scripts` | 编辑页面脚本 |
| `write:styles` | 编辑页面样式 |
| `read:comments` | 读取评论 |
| `write:comments` | 创建评论 |
| `manage:comments` | 编辑/删除评论 |
| `write:users` | 写用户 |
| `manage:users` | 管理用户 |
| `write:groups` | 写组 |
| `manage:groups` | 管理组 |
| `manage:navigation` | 管理导航 |
| `manage:theme` | 管理主题 |
| `manage:api` | 管理 API |
| `manage:system` | 系统管理员（超级权限） |

### 3.2 全局权限聚合

`server/models/users.js:153-155`：

```js
getGlobalPermissions() {
  return _.uniq(_.flatten(_.map(this.groups, 'permissions')))
}
```

**取所有组的 permissions 并集、去重**。即：用户只要有任何一个组授予了某权限，就拥有该权限。

### 3.3 新建组的默认权限

`server/app/data.yml:143-147`：

```yaml
defaultPermissions:
  - 'read:pages'
  - 'read:assets'
  - 'read:comments'
  - 'write:comments'
```

### 3.4 `manage:system` 的特殊地位

在 `checkAccess` 中，拥有 `manage:system` 的用户直接返回 `true`，跳过一切后续检查：

```js
if (_.includes(userPermissions, 'manage:system')) {
  return true
}
```

---

## 4. 页面规则（Page Rules）

### 4.1 规则结构

来自 GraphQL schema（`group.graphql:90-106`）：

```graphql
type PageRule {
  id: String!        # 前端生成的 nanoid
  deny: Boolean!     # true=拒绝, false=允许
  match: PageRuleMatch!  # 匹配方式
  roles: [String]!   # 作用的权限（如 ['read:pages', 'write:pages']）
  path: String!      # 匹配路径/正则/标签
  locales: [String]! # 作用的语言，空数组=所有语言
}
```

### 4.2 匹配方式（PageRuleMatch）

| match 值 | 含义 | path 示例 |
|---|---|---|
| `START` | 路径以此开头 | `geography` → 匹配 `/geography/anything` |
| `END` | 路径以此结尾 | `index` → 匹配 `/folder/index` |
| `EXACT` | 路径完全相等 | `geography/countries` → 只匹配 `/geography/countries` |
| `REGEX` | 正则匹配 | `^/geo.*ries$` |
| `TAG` | 标签名匹配 | `internal` → 匹配带有此 tag 的页面 |

### 4.3 新建组的默认规则

`server/app/data.yml:148-158`：

```yaml
defaultPageRules:
  - id: default
    deny: false
    match: START
    roles: ['read:pages', 'read:assets', 'read:comments', 'write:comments']
    path: ''
    locales: []
```

默认规则：允许所有路径（`path=''`，`START` 匹配一切）上的基本读权限。

---

## 5. 核心判定逻辑：`checkAccess`

`server/core/auth.js:221-295` 是整个权限系统的核心入口。

### 5.1 完整流程

```
checkAccess(user, permissions, page)
  │
  ├─ 1. 获取用户的全局权限列表 userPermissions
  │     （来自 user.permissions 或 user.getGlobalPermissions()）
  │
  ├─ 2. manage:system 短路 → return true
  │
  ├─ 3. 检查全局权限交集
  │     _.intersection(userPermissions, permissions).length < 1 → return false
  │     （要求的权限中，用户至少要拥有一个）
  │
  ├─ 4. 无页面上下文时 → return true
  │     （仅检查全局权限就够了）
  │
  └─ 5. 有页面上下文时，检查页面规则
        │
        ├─ 遍历用户所属的所有组
        ├─ 对每个组，从 WIKI.auth.groups[groupId].pageRules 取规则
        ├─ 对每条规则：
        │   ├─ 检查语言过滤（rule.locales）
        │   ├─ 检查权限交集（rule.roles 与 请求的 permissions 必须有交集）
        │   └─ 按 match 类型做路径匹配
        │       ├─ START:  _.startsWith(`/${page.path}`, `/${rule.path}`)
        │       ├─ END:    _.endsWith(page.path, rule.path)
        │       ├─ REGEX:  new RegExp(rule.path).test(page.path)
        │       ├─ TAG:    page.tags 中是否有 tag.tag === rule.path
        │       └─ EXACT:  `/${page.path}` === `/${rule.path}`
        │
        └─ 用 _applyPageRuleSpecificity 决定最终结果
```

### 5.2 规则优先级 / 特异性系统

`_applyPageRuleSpecificity`（`server/core/auth.js:368-388`）实现了规则的优先级仲裁。

**两个维度决定优先级：**

#### 维度一：路径特异性（path 越长越优先）

```js
if (rule.path.length < checkState.specificity.length) {
  return checkState  // 更短的路径不覆盖更长的
}
```

例如：规则 path=`/geography/countries` 会覆盖 path=`/geography`。

#### 维度二：匹配类型优先级（从低到高）

```
START (最低) < END < REGEX < TAG < EXACT (最高)
```

当两条规则的 path 长度相同时，匹配类型优先级高的不被优先级低的覆盖：

```js
if (rule.path.length === checkState.specificity.length) {
  if (_.includes(higherPriority, checkState.match)) {
    return checkState  // 不覆盖更高优先级的匹配类型
  }
}
```

各 match 类型的 `higherPriority` 列表：

| 当前 match | 不能覆盖的更高优先级类型 |
|---|---|
| START | END, REGEX, EXACT, TAG |
| END | REGEX, EXACT, TAG |
| REGEX | EXACT, TAG |
| TAG | EXACT |
| EXACT | （无，最高优先级） |

#### 维度三：DENY 优先原则

当路径特异性和匹配类型都相同时，DENY 规则优先于 ALLOW：

```js
if (rule.match === checkState.match && checkState.deny && !rule.deny) {
  return checkState  // 已有 DENY，不被 ALLOW 覆盖
}
```

### 5.3 最终判定

```js
return (checkState.match && !checkState.deny)
```

- `match=false`：没有匹配到任何规则 → **拒绝**（默认行为是最严格）
- `match=true, deny=false`：匹配到允许规则 → **通过**
- `match=true, deny=true`：匹配到拒绝规则 → **拒绝**

**关键理解**：如果用户所属的所有组都没有产生任何匹配的页面规则，`checkState.match` 保持 `false`，则返回拒绝。这意味着页面规则本质上是一种"白名单"——必须至少有一条规则显式允许。

---

## 6. 全局权限 vs 页面规则的合并关系

### 6.1 两阶段"与"逻辑

```
最终结果 = 全局权限检查 AND 页面规则检查
```

| 全局权限 | 页面规则 | 最终结果 |
|---|---|---|
| ✅ 有 | ✅ 允许 | ✅ 通过 |
| ✅ 有 | ❌ 拒绝 | ❌ 拒绝 |
| ✅ 有 | ⚪ 无规则匹配 | ❌ 拒绝 |
| ❌ 无 | — | ❌ 拒绝（不会走到规则检查） |

**核心要点**：
1. 全局权限是**入口门槛**——没有全局权限直接拒绝
2. 页面规则是**二次过滤**——即使有全局权限，仍可被页面规则拒绝
3. 页面规则的 `roles` 字段必须与请求的 `permissions` 有交集才生效，这意味着页面规则是在特定权限维度上做覆盖

### 6.2 多组规则合并

用户属于多个组时，所有组的所有 pageRules **全部参与竞争**，通过特异性系统选出最终胜出者。

**不是"并集"**——不是说任何一个组允许就允许。而是所有规则竞争，最长路径/最高优先级的规则胜出。

**但注意**：全局权限是"并集"（`_.flatten` + `_.uniq`），页面规则不是。

---

## 7. GraphQL 层的权限检查

### 7.1 `@auth` 指令

`server/graph/directives/auth.js`：

```graphql
directive @auth(requires: [String]) on QUERY | FIELD_DEFINITION | ARGUMENT_DEFINITION
```

在 GraphQL schema 中标注所需权限：

```graphql
pages: PageQuery @auth(requires: ["manage:system", "read:pages"])
```

实现逻辑（`auth.js:46`）：

```js
if (!_.some(context.req.user.permissions, pm => _.includes(requiredScopes, pm))) {
  throw new Error('Forbidden')
}
```

**注意**：`@auth` 指令只做全局权限的交集检查（`_.some` + `_.includes`），**不做页面规则检查**。页面规则检查在 resolver 内部手动调用 `WIKI.auth.checkAccess` 完成。

### 7.2 resolver 内的细粒度检查

如 `server/graph/resolvers/page.js:19-21`：

```js
if (WIKI.auth.checkAccess(context.req.user, ['read:history'], {
  path: page.path,
  locale: page.localeCode
})) { ... }
```

---

## 8. 页面可见性的实际应用场景

### 8.1 页面浏览（查看页面）

`server/controllers/common.js:417-582`，路由 `/*`：

```
1. getPage() 获取页面
2. getEffectivePermissions(req, pageArgs) 计算所有权限
3. 检查 effectivePermissions.pages.read
4. 检查发布状态（publishStartDate / publishEndDate）
   → 未发布且无 write 权限 → 403
```

### 8.2 页面树可见性

`server/graph/resolvers/page.js:285-290`：

```js
results.filter(r => {
  return WIKI.auth.checkAccess(context.req.user, ['read:pages'], {
    path: r.path,
    locale: r.localeCode
  })
})
```

用户只能看到自己有 `read:pages` 权限的页面树节点。

### 8.3 搜索结果过滤

`server/graph/resolvers/page.js:57-62`：

```js
results: _.filter(resp.results, r => {
  return WIKI.auth.checkAccess(context.req.user, ['read:pages'], {
    path: r.path,
    locale: r.locale,
    tags: r.tags
  })
})
```

搜索结果也按权限过滤，且会传入 `tags`（因为 TAG 类型的规则需要）。

### 8.4 导航可见性

`server/models/navigation.js:62-65`：

```js
static getAuthorizedItems(tree = [], groups = []) {
  return _.filter(tree, leaf => {
    return leaf.visibilityMode === 'all' || _.intersection(leaf.visibilityGroups, groups).length > 0
  })
}
```

导航项有两种可见性模式：
- `visibilityMode === 'all'`：所有人可见
- 否则：只有用户所属组在 `visibilityGroups` 中的才可见

这是**独立于页面规则**的第三层可见性控制，直接基于组 ID 做交集判断。

### 8.5 `getEffectivePermissions`

`server/core/auth.js:496-521`：

将 `checkAccess` 封装为页面级权限矩阵，前端用此来决定显示哪些操作按钮：

```js
getEffectivePermissions(req, page) {
  return {
    comments: { read, write, manage },
    history: { read },
    source: { read },
    pages: { read, write, manage, delete, script, style },
    system: { manage }
  }
}
```

---

## 9. JWT 中的权限信息

`server/models/users.js:440-452`：

```js
jwt.sign({
  id: user.id,
  permissions: user.getGlobalPermissions(),  // 全局权限数组
  groups: user.getGroups()                   // 组 ID 数组
}, ...)
```

JWT 中只包含全局权限和组 ID 列表，**不包含页面规则**。页面规则在服务端通过 `WIKI.auth.groups[groupId].pageRules` 实时读取。

---

## 10. 完整判定流程图

```
请求到达
  │
  ▼
JWT 解析 → 得到 user.permissions + user.groups
  │
  ▼
@auth 指令检查（仅全局权限）
  │ 失败 → 403 Forbidden
  ▼ 通过
Resolver / Controller 执行
  │
  ▼
WIKI.auth.checkAccess(user, requiredPermissions, page?)
  │
  ├─ manage:system? → ✅ 直接通过
  │
  ├─ 全局权限交集检查 → ❌ 失败直接拒绝
  │
  └─ 页面规则检查（如有 page 上下文）
       │
       ├─ 遍历所有组的 pageRules
       ├─ 语言过滤
       ├─ 权限维度过滤（rule.roles ∩ requiredPermissions）
       ├─ 路径匹配（START/END/EXACT/REGEX/TAG）
       ├─ 特异性仲裁（路径长度 > 匹配类型 > DENY 优先）
       │
       └─ checkState.match && !checkState.deny → ✅/❌
```

---

## 11. 易混淆点总结

| 易混淆点 | 正确理解 |
|---|---|
| 多组权限合并方式 | 全局权限=**并集**；页面规则=**竞争**（非并集） |
| 页面规则无匹配时的行为 | **拒绝**（默认最严，需显式允许） |
| `@auth` 指令是否检查页面规则 | **否**，只检查全局权限 |
| TAG 匹配的 path 字段含义 | 不是路径，是**标签名** |
| 页面规则的 roles 字段 | 不是"角色"，是该规则作用的**权限维度** |
| Administrators 组(id=1)的特殊性 | 前端显示"此组可访问所有内容"，后端通过 `manage:system` 短路实现 |
| 导航可见性 vs 页面可见性 | 导航=基于组ID的简单交集；页面=全局权限+页面规则两阶段 |
| Guest 用户(id=2) | 未登录用户自动映射为 Guest，其权限来自 id=2 的组，结果缓存1分钟 |

---

## 12. 多级 Page Tree 下的级联路径权限继承

### 12.1 Page Tree 的数据结构

`server/jobs/rebuild-tree.js` 中的 `pageTree` 表存储的是**扁平的树节点**，每个页面路径的每一层都有独立的节点：

```
页面路径：geography/countries/china

在 pageTree 中生成的节点：
1. geography        (depth=1, isFolder=true,  pageId=null, parent=null)
2. countries        (depth=2, isFolder=true,  pageId=null, parent=1)
3. china            (depth=3, isFolder=false, pageId=实际页面ID, parent=2)
```

关键字段（`rebuild-tree.js:33-45`）：
```js
tree.push({
  id: pik,                      // 节点自增ID
  localeCode: page.localeCode,  // 语言
  path: currentPath,            // 该层路径（如 'geography/countries'）
  depth: depth,                 // 层级深度
  title: isFolder ? part : page.title,
  isFolder: isFolder,           // 是否为文件夹节点
  parent: parentId,             // 父节点ID
  pageId: isFolder ? null : page.id,  // 实际页面ID（仅叶子节点）
  ancestors: JSON.stringify(ancestors) // 祖先节点ID数组
})
```

### 12.2 Tree 查询的级联过滤

`server/graph/resolvers/page.js:249-294` 的 `tree` 查询：

```
1. 先从 pageTree 表按 parent 条件批量拉取节点
2. 对结果数组调用 filter()，每个节点独立做 checkAccess 检查
3. 只有通过 checkAccess 的节点才会返回给前端
```

**过滤是"节点级"的，不是"树级"的**——即每个节点独立检查权限。但因为 `parent` 查询条件的存在，如果父节点被过滤掉，子节点虽然在数据库中存在，但不会被这一次查询拉取到。

### 12.3 START 规则实现"级联继承"

权限系统**没有**专门的"继承"机制，级联效果是通过 `START` 匹配类型实现的：

```
规则：{ match: 'START', path: 'geography', roles: ['read:pages'], deny: false }

匹配路径：
  ✅ /geography
  ✅ /geography/countries
  ✅ /geography/countries/china
  ✅ /geography/cities
  ❌ /history
```

`server/core/auth.js:254-257`：
```js
case 'START':
  if (_.startsWith(`/${page.path}`, `/${rule.path}`)) {
    checkState = this._applyPageRuleSpecificity({...})
  }
  break
```

**重要理解**：Wiki.js 的"级联权限"本质上是**前缀匹配**，不是显式的继承关系。如果在 `/geography/countries` 上设置了更具体的规则（EXACT 或更长路径），会因为特异性更高而覆盖 START 规则。

### 12.4 多层级树的权限判定过程

```
请求 GET /geography/countries/china
  │
  ▼
path = 'geography/countries/china', locale = 'en'
  │
  ▼
checkAccess(user, ['read:pages'], { path, locale })
  │
  ├─ 全局权限检查：用户是否有 read:pages？
  │
  └─ 页面规则检查：
      │
      ├─ 规则1：START 'geography' → 匹配 → specificity='geography'(9)
      │
      ├─ 规则2：START 'geography/countries' → 匹配 → specificity='geography/countries'(19)
      │   更长，覆盖规则1
      │
      └─ 规则3：EXACT 'geography/countries/china' → 匹配 → specificity相同, EXACT优先级更高
          覆盖规则2
```

**特异性系统天然支持多层级覆盖**——路径越长、匹配类型越精确的规则优先级越高。

### 12.5 文件夹节点的特殊处理

`pageTree` 中的文件夹节点（`isFolder=true`）没有实际的 `pageId`，但它的 `path` 是完整的中间路径（如 `geography/countries`）。权限检查时：

1. 文件夹节点用它自己的 `path` 做 checkAccess
2. 如果用户无权访问 `geography/countries` 文件夹，即使有权访问 `geography/countries/china` 页面，在树浏览时也看不到父文件夹
3. 但直接通过完整 URL 访问页面不受影响（因为直接用完整路径做权限检查）

---

## 13. 匿名用户的可见性裁剪代码挂载点

### 13.1 匿名用户身份注入

`server/core/auth.js:169-177` 是匿名用户的**核心挂载点**：

```js
// JWT is NOT valid, set as guest
if (!user) {
  if (WIKI.auth.guest.cacheExpiration <= DateTime.utc()) {
    WIKI.auth.guest = await WIKI.models.users.getGuestUser()
    WIKI.auth.guest.cacheExpiration = DateTime.utc().plus({ minutes: 1 })
  }
  req.user = WIKI.auth.guest
  return next()
}
```

**触发条件**：
- JWT 无效（缺失、过期、签名错误）
- 或 JWT 中的用户/组已被撤销

注入后，后续所有中间件和控制器看到的 `req.user` 就是 Guest 用户（id=2），权限检查完全走正常流程。

### 13.2 Guest 用户初始化

首次创建（`server/setup.js:246-253`）：
```js
const guestGroup = await WIKI.models.groups.query().insert({
  name: 'Guests',
  permissions: JSON.stringify(['read:pages', 'read:assets', 'read:comments']),
  pageRules: JSON.stringify([
    { id: 'guest', roles: ['read:pages', 'read:assets', 'read:comments'], 
      match: 'START', deny: false, path: '', locales: [] }
  ]),
  isSystem: true
})
```

默认权限：全局 `read:pages` + `read:assets` + `read:comments`，页面规则允许所有路径。

### 13.3 可见性裁剪的完整挂载点列表

**1. 路由入口：`server/core/auth.js:169-177`**
- 位置：Express 中间件 `authenticate` 内
- 作用：将无 JWT 请求映射为 Guest 用户

**2. 页面浏览：`server/controllers/common.js:417-457`（`/*` 路由）**
```js
const effectivePermissions = WIKI.auth.getEffectivePermissions(req, pageArgs)
if (!effectivePermissions.pages.read) {
  if (req.user.id === 2) {  // 是 Guest？
    res.cookie('loginRedirect', req.path, { maxAge: 15 * 60 * 1000 })
  }
  if (pageArgs.path === 'home' && req.user.id === 2) {
    return res.redirect('/login')
  }
  return res.status(403).render('unauthorized', { action: 'view' })
}
```
- Guest 访问被拒绝时，记录登录后跳转目标，首页直接跳转登录

**3. 页面编辑：`server/controllers/common.js:104-235`（`/e/*` 路由）**
```js
const effectivePermissions = WIKI.auth.getEffectivePermissions(req, pageArgs)
if (!(effectivePermissions.pages.write || effectivePermissions.pages.manage)) {
  return res.status(403).render('unauthorized', { action: 'edit' })
}
```

**4. 搜索结果：`server/graph/resolvers/page.js:52-64`**
```js
results: _.filter(resp.results, r => {
  return WIKI.auth.checkAccess(context.req.user, ['read:pages'], {
    path: r.path, locale: r.locale, tags: r.tags
  })
})
```
- 搜索结果按权限过滤，Guest 只能看到允许的页面

**5. 页面列表：`server/graph/resolvers/page.js:76-147`**
```js
results = _.filter(results, r => {
  return WIKI.auth.checkAccess(context.req.user, ['read:pages'], {
    path: r.path, locale: r.locale
  })
})
```

**6. 页面树：`server/graph/resolvers/page.js:249-294`**
```js
return results.filter(r => {
  return WIKI.auth.checkAccess(context.req.user, ['read:pages'], {
    path: r.path, locale: r.localeCode
  })
})
```

**7. 标签列表：`server/graph/resolvers/page.js:200-213`**
```js
const allTags = _.filter(pages, r => {
  return WIKI.auth.checkAccess(context.req.user, ['read:pages'], {
    path: r.path, locale: r.locale
  })
}).flatMap(r => r.tags)
```

**8. 页面链接：`server/graph/resolvers/page.js:299-350`**
```js
if (!WIKI.auth.checkAccess(..., ['read:pages'], { path: val.path, ... }) ||
    !WIKI.auth.checkAccess(..., ['read:pages'], { path: val.link, ... })) {
  return result  // 链接两端都要有权限
}
```

**9. 导航菜单：`server/models/navigation.js:62-65`**
```js
static getAuthorizedItems(tree = [], groups = []) {
  return _.filter(tree, leaf => {
    return leaf.visibilityMode === 'all' || 
           _.intersection(leaf.visibilityGroups, groups).length > 0
  })
}
```
- 导航项可见性直接基于组 ID 交集，Guest 的组是 [2]

**10. 资源访问：`server/controllers/common.js:575-581`**
```js
if (!WIKI.auth.checkAccess(req.user, ['read:assets'], pageArgs)) {
  return res.sendStatus(403)
}
```

### 13.4 Guest 可见性裁剪的两层防御

```
HTTP 请求到达
  │
  ▼
┌──────────────────────────────────────┐
│  第一层：身份映射                      │
│  auth.js:169-177                      │
│  无 JWT → req.user = Guest(id=2)     │
└──────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────┐
│  第二层：各挂载点的权限检查            │
│  所有后续代码走完全相同的 checkAccess │
│  - 页面浏览 check read:pages         │
│  - 搜索结果过滤                       │
│  - 页面树过滤                         │
│  - 导航过滤                           │
│  - 资源访问检查                       │
└──────────────────────────────────────┘
  │
  ▼
返回 403 / 404 / 过滤后的结果
```

**关键设计**：Guest 用户没有"特殊对待"，完全复用普通用户的权限检查逻辑。区别仅在于：
1. Guest 属于固定的组 id=2
2. 首页访问被拒绝时自动跳转登录页
3. 被拒绝时记录登录跳转 cookie

---

## 14. 权限缓存与变更失效路径

### 14.1 多层级缓存架构

Wiki.js 的权限缓存分布在四个层级：

| 缓存层级 | 存储位置 | 缓存内容 | 过期时间 |
|---|---|---|---|
| **L1：JWT 令牌** | 客户端浏览器 Cookie | 用户ID、全局权限数组、组ID数组 | 30分钟（默认） |
| **L2：组信息内存缓存** | 服务端 `WIKI.auth.groups` | 所有组的 permissions + pageRules | 手动失效 |
| **L3：Guest 用户缓存** | 服务端 `WIKI.auth.guest` | Guest 用户的权限信息 | 1分钟 |
| **L4：撤销列表** | 服务端 `WIKI.auth.revocationList` | 已撤销的用户/组 ID | 30分钟（与 token 同） |
| **L5：导航树缓存** | 服务端 `WIKI.cache` | 各语言的导航树 | 300秒 |
| **L6：页面渲染缓存** | 本地文件系统 | 页面渲染后的 HTML（`.bin`） | 手动失效 |

### 14.2 L1：JWT 中的权限缓存

`server/models/users.js:440-452`，登录时签发：

```js
jwt.sign({
  id: user.id,
  permissions: user.getGlobalPermissions(),  // 全局权限数组快照
  groups: user.getGroups(),                   // 组ID数组快照
  iat: issuedAtTimestamp,
  exp: issuedAtTimestamp + 30min
}, privateKey, { algorithm: 'RS256' })
```

**优点**：服务端无状态，性能高
**缺点**：权限变更后不能立即生效，需等待 token 过期或触发撤销

### 14.3 L2：组信息内存缓存

`server/core/auth.js:393-396`，启动时加载 + 手动刷新：

```js
async reloadGroups () {
  const groupsArray = await WIKI.models.groups.query()
  this.groups = _.keyBy(groupsArray, 'id')  // { id: { permissions, pageRules, ... } }
  WIKI.auth.guest.cacheExpiration = DateTime.utc().minus({ days: 1 })  // 同时失效 Guest 缓存
}
```

调用时机（`server/graph/resolvers/group.js`）：
- 创建组：`create()` → `reloadGroups()`（`group.js:103`）
- 更新组：`update()` → `reloadGroups()`（`group.js:205`）
- 删除组：`delete()` → `reloadGroups()`（`group.js:123`）

### 14.4 L3：Guest 用户缓存

`server/core/auth.js:171-173`：
```js
if (WIKI.auth.guest.cacheExpiration <= DateTime.utc()) {
  WIKI.auth.guest = await WIKI.models.users.getGuestUser()
  WIKI.auth.guest.cacheExpiration = DateTime.utc().plus({ minutes: 1 })
}
```

Guest 用户的权限来自 id=2 的组，因此组变更时会通过 `reloadGroups()` 间接失效 Guest 缓存。

### 14.5 L4：撤销列表（Revocation List）

`server/core/auth.js:23`：
```js
revocationList: require('./cache').init()  // NodeCache 实例
```

**添加撤销记录**（`server/core/auth.js:526-527`）：
```js
revokeUserTokens ({ id, kind = 'u' }) {
  WIKI.auth.revocationList.set(
    `${kind}${_.toString(id)}`,        // key: 'u123' 或 'g456'
    Math.round(DateTime.utc().minus({ seconds: 5 }).toSeconds()),  // 值：撤销时间戳
    Math.ceil(ms(WIKI.config.auth.tokenExpiration) / 1000)         // TTL：token 有效期
  )
}
```

**撤销检查**（`server/core/auth.js:126-141`）：
```js
if (user && !user.api && !mustRevalidate) {
  // 检查用户级撤销
  const uRevalidate = WIKI.auth.revocationList.get(`u${user.id}`)
  if (uRevalidate && user.iat < uRevalidate) {
    mustRevalidate = true  // token 签发时间早于撤销时间 → 需要重新验证
  }
  // 检查服务重启（所有旧 token 失效）
  else if (DateTime.fromSeconds(user.iat) <= WIKI.startedAt) {
    mustRevalidate = true
  }
  // 检查组级撤销
  else {
    for (const gid of user.groups) {
      const gRevalidate = WIKI.auth.revocationList.get(`g${gid}`)
      if (gRevalidate && user.iat < gRevalidate) {
        mustRevalidate = true
        break
      }
    }
  }
}
```

### 14.6 完整的变更失效流程

**场景 1：组权限/规则变更**

```
管理员修改 Group(id=3) 的 pageRules
  │
  ▼
graph/resolvers/group.js update() 被调用
  │
  ├─ ① 更新 DB：groups 表的 permissions / pageRules
  │
  ├─ ② 撤销组 token：
  │    WIKI.auth.revokeUserTokens({ id: 3, kind: 'g' })
  │    → 写入 revocationList['g3'] = 当前时间戳
  │
  ├─ ③ 事件传播：
  │    WIKI.events.outbound.emit('addAuthRevoke', { id: 3, kind: 'g' })
  │    → 跨 HA 实例同步撤销
  │
  └─ ④ 刷新组缓存：
       WIKI.auth.reloadGroups()
       → 重新从 DB 读取所有组到 WIKI.auth.groups

下一次请求到来（用户 token 包含 groups: [1,3]）：
  │
  ▼
auth.js:134-140 检查 revocationList['g3']
  │
  ├─ user.iat < revocationList['g3'] → mustRevalidate = true
  │
  ▼
auth.js:145-162 重新从 DB 拉取用户+组信息，签发新 token
  │
  ▼
用户获得最新权限
```

**场景 2：用户所属组变更**

```
用户 User(id=123) 被加入/移出组
  │
  ▼
graph/resolvers/group.js assignUser() / unassignUser()
  │
  ├─ ① 更新 DB：userGroups 关联表
  │
  ├─ ② 撤销用户 token：
       WIKI.auth.revokeUserTokens({ id: 123, kind: 'u' })
       WIKI.events.outbound.emit('addAuthRevoke', { id: 123, kind: 'u' })

下一次请求：
  auth.js:128-130 检查 revocationList['u123']
  → mustRevalidate = true
  → 重新签发 token（包含最新 groups 数组）
```

**场景 3：服务重启**

```
服务启动时设置 WIKI.startedAt = 当前时间
  │
  ▼
请求携带旧 token（iat < startedAt）
  │
  ▼
auth.js:131-132
  else if (DateTime.fromSeconds(user.iat) <= WIKI.startedAt) {
    mustRevalidate = true
  }
```

所有重启前签发的 token 都会被强制重新验证。

### 14.7 L5：导航树缓存

`server/models/navigation.js:26-49`：
```js
const navTreeCached = await WIKI.cache.get(`nav:sidebar:${locale}`)
if (navTreeCached) {
  return bypassAuth ? navTreeCached : 
    WIKI.models.navigation.getAuthorizedItems(navTreeCached, groups)
}
// ... 未命中则从 DB 读取，写入缓存，TTL=300秒
await WIKI.cache.set(`nav:sidebar:${tree.locale}`, tree.items, 300)
```

**重要**：缓存的是**原始导航树**，权限过滤（`getAuthorizedItems`）在每次请求时实时进行。因此导航树缓存不影响权限判断，只是避免重复读取 DB。

### 14.8 L6：页面渲染缓存

`server/models/pages.js:1052-1078`，基于页面 hash 存储 `.bin` 文件：
```js
static async savePageToCache(page) {
  const cachePath = path.resolve(..., `cache/${page.hash}.bin`)
  await fs.outputFile(cachePath, WIKI.models.pages.cacheSchema.encode({
    id: page.id, render: page.render, ...
  }))
}
```

**注意**：页面缓存只缓存渲染结果，**不缓存权限判定结果**。每次访问页面时，权限检查先于缓存读取进行：
- 无权限 → 直接 403，不会读取缓存
- 有权限 → 才尝试读取缓存，未命中则从 DB 加载

### 14.9 失效链路全景图

```
权限变更（组/用户）
  │
  ├─ DB 写入
  ├─ revocationList 添加（key='uX'/'gX'，value=时间戳）
  ├─ outbound 事件 → 跨 HA 节点同步
  └─ reloadGroups() → 刷新 WIKI.auth.groups 内存缓存
        │
        └─ guest.cacheExpiration 置为过期 → Guest 缓存下次请求时刷新

下一次用户请求
  │
  ├─ 读取 JWT → 获得 user.iat, user.permissions, user.groups
  │
  ├─ 检查 revocationList
  │   ├─ 用户 ID 是否在撤销列表中？
  │   ├─ 用户的任一组是否在撤销列表中？
  │   └─ token.iat <= 服务启动时间？
  │
  ├─ ✅ 无需重验证 → 使用 JWT 内的权限快照
  │
  └─ ❌ 需要重验证 → 从 DB 重新拉取用户+组信息，签发新 JWT
        │
        └─ 新 JWT 包含最新的 permissions 和 groups 数组

权限检查（每次操作都执行）
  │
  ├─ checkAccess(user, permissions, page?)
  │   ├─ 全局权限检查（使用 user.permissions）
  │   └─ 页面规则检查（使用 WIKI.auth.groups[gid].pageRules）
  │
  └─ getEffectivePermissions() → 生成页面级权限矩阵，传给前端模板
```

### 14.10 缓存一致性保障

| 变更类型 | 失效范围 | 生效延迟 |
|---|---|---|
| 组权限/规则变更 | 组所有成员的 token | 最多到 token 过期（30分钟），或下一次请求时立即刷新 |
| 用户组关系变更 | 单个用户的 token | 同上 |
| 组删除 | 组所有成员 | 同上 |
| 服务重启 | 所有用户 token | 立即（下一次请求强制刷新） |
| 页面内容更新 | 单个页面缓存 | 立即（保存时删除缓存文件） |

**关键设计原则**：
1. **最终一致性**：不追求强一致，利用 token 过期自然轮转
2. **撤销列表**：作为紧急通道，允许强制失效（不等待 token 过期）
3. **分层缓存**：越上层（JWT）缓存时间越长，越下层（页面缓存）失效越快
4. **HA 同步**：通过事件总线跨节点同步撤销事件

---

## 15. 搜索索引（Search Index）上的权限隔离路径

### 15.1 搜索架构的两层模型

Wiki.js 的搜索采用**"索引全量 + 查询时过滤"**的架构：

```
┌──────────────────────┐      ┌──────────────────────┐
│   搜索引擎索引层      │      │   GraphQL Resolver  │
│   (DB/ES/Algolia)    │      │       过滤层         │
│                      │      │                      │
│  索引所有页面数据     │ ───► │  checkAccess 逐页过滤 │
│  不做权限隔离        │      │  返回可见的结果      │
└──────────────────────┘      └──────────────────────┘
```

**核心原则**：搜索引擎本身不感知权限，索引所有页面；权限隔离在查询结果返回前的应用层完成。

### 15.2 索引写入：全量索引，无权限标记

以 Elasticsearch 为例（`server/modules/search/elasticsearch/engine.js:228-244`）：

```js
async created(page) {
  await this.client.index({
    index: this.config.indexName,
    id: page.hash,
    body: {
      suggest: this.buildSuggest(page),
      locale: page.localeCode,
      path: page.path,
      title: page.title,
      description: page.description,
      content: page.safeContent,
      tags: await this.buildTags(page.id)
    },
    refresh: true
  })
}
```

索引文档中**不包含任何组ID、权限标记或可见性信息**，只存储：
- 元数据：`locale`、`path`、`title`、`description`
- 内容：`content`（安全渲染后的内容）
- 标签：`tags`
- 建议词：`suggest`

**所有搜索引擎实现（DB / PostgreSQL / Elasticsearch / Algolia / Azure / AWS / Sphinx / Manticore）均遵循这一模式**。

### 15.3 查询过滤：Resolver 层的事后过滤

`server/graph/resolvers/page.js:52-64` 是搜索权限隔离的核心挂载点：

```js
async search (obj, args, context) {
  if (WIKI.data.searchEngine) {
    const resp = await WIKI.data.searchEngine.query(args.query, args)
    return {
      ...resp,
      results: _.filter(resp.results, r => {
        return WIKI.auth.checkAccess(context.req.user, ['read:pages'], {
          path: r.path,
          locale: r.locale,
          tags: r.tags  // 标签用于 TAG 类型规则的匹配
        })
      })
    }
  }
}
```

**过滤过程**：
1. 搜索引擎返回所有匹配结果（可能包含用户无权访问的页面）
2. 遍历每一条结果，调用 `checkAccess` 做完整权限检查
3. 只保留通过检查的结果
4. `totalHits` 不会被修正（返回的是搜索引擎原始命中数，不是过滤后的数量）

### 15.4 DB 搜索引擎的特殊情况

`server/modules/search/db/engine.js:22-55`，即默认的数据库搜索引擎：

```js
async query(q, opts) {
  const results = await WIKI.models.pages.query()
    .column('pages.id', 'title', 'description', 'path', 'localeCode as locale')
    .withGraphJoined('tags')  // 特意关联 tags，用于后续权限检查
    .modifyGraph('tags', builder => {
      builder.select('tag')
    })
    .where(builder => {
      builder.where('isPublished', true)
      // ... LIKE 查询
    })
    .limit(WIKI.config.search.maxHits)
  return {
    results,
    suggestions: [],
    totalHits: results.length
  }
}
```

注意：DB 引擎**不在 SQL 层做权限过滤**，而是和其他引擎一样，返回结果后在 resolver 层过滤。这保持了各搜索引擎实现的一致性。

### 15.5 搜索权限隔离的完整调用链

```
用户搜索请求
  │
  ▼
GraphQL: pages.search(query, path, locale)
  │
  ▼
@auth 指令检查：用户是否有 read:pages 全局权限
  │  无 → 直接拒绝
  ▼  有
WIKI.data.searchEngine.query(q, opts)
  │
  ├─ DB 引擎：LIKE 查询 pages 表
  ├─ ES 引擎：查询 Elasticsearch 索引
  └─ 其他引擎：各自的查询方式
  │
  ▼
获得原始搜索结果（可能包含不可见页面）
  │
  ▼
_.filter(results, r => checkAccess(user, ['read:pages'], r))
  │
  ├─ 全局权限检查（通常已通过，因为 @auth 已检查）
  ├─ 页面规则检查
  │   ├─ START/END/EXACT/REGEX/TAG 匹配
  │   └─ 特异性仲裁
  │
  └─ 通过 → 保留；不通过 → 过滤掉
  │
  ▼
返回过滤后的结果给前端
```

### 15.6 权限隔离的其他搜索相关挂载点

**1. 标签搜索（searchTags）**：`server/graph/resolvers/page.js:56-59`

```
所有标签来自用户有权限的页面 → 过滤掉无权限页面的标签
```

**2. 页面列表（list）**：`server/graph/resolvers/page.js:76-147`

```js
results = _.filter(results, r => {
  return WIKI.auth.checkAccess(context.req.user, ['read:pages'], {
    path: r.path,
    locale: r.locale
  })
})
```

**3. 页面树（tree）**：`server/graph/resolvers/page.js:249-294`

```js
return results.filter(r => {
  return WIKI.auth.checkAccess(context.req.user, ['read:pages'], {
    path: r.path,
    locale: r.localeCode
  })
})
```

**4. 页面链接（links）**：`server/graph/resolvers/page.js:299-350`

```js
if (!WIKI.auth.checkAccess(..., ['read:pages'], { path: val.path, ... }) ||
    !WIKI.auth.checkAccess(..., ['read:pages'], { path: val.link, ... })) {
  return result  // 链接两端都要有权限才返回
}
```

### 15.7 设计权衡分析

| 设计选择 | 优点 | 缺点 |
|---|---|---|
| **索引全量 + 查询过滤** | 实现简单，索引逻辑与权限解耦；搜索引擎变更不影响权限逻辑 | 搜索性能有损耗（大结果集下过滤慢）；`totalHits` 不准确；可能泄露"存在不可见内容"的信息 |
| **索引时标记权限** | 查询性能好；总数准确 | 索引复杂度高；组变更时需重建索引；权限变更延迟 |

Wiki.js 选择了**前者**，优先保证实现简单和权限逻辑的一致性。

---

## 16. 跨空间（Locale / 组）的权限合并路径

> **注意**：Wiki.js 2.x 没有名为 "Space" 的一等概念。权限"空间"主要体现在两个维度：
> 1. **Locale（语言空间）**：不同语言的内容是独立的，页面规则可按语言限定
> 2. **Groups（组空间）**：不同组有不同的权限集合，用户可跨组
> 3. **Path 命名空间**：通过 START 匹配实现的路径级权限分区
>
> 以下从代码层面梳理这三个维度的权限合并逻辑。

### 16.1 跨 Locale（语言空间）的权限合并

#### 16.1.1 多语言内容的隔离基础

Wiki.js 中每个页面都有 `localeCode`，不同语言的同名页面是独立的记录：

```
pages 表：
  id    path              localeCode
  1     home               en
  2     home               zh
  3     geography/countries  en
  4     geography/countries  zh
```

`pageTree` 也是按语言分开构建的（`rebuild-tree.js:13` 中 `orderBy(['localeCode', 'path'])`）。

#### 16.1.2 页面规则的 Locale 过滤

`server/core/auth.js:249-251`，在规则匹配前先做语言过滤：

```js
if (rule.locales && rule.locales.length > 0) {
  if (!rule.locales.includes(page.locale)) { return }
}
```

**规则的 `locales` 字段语义**：
- 空数组 `[]`：规则作用于**所有语言**
- 非空数组 `['en', 'zh']`：规则**只作用于**指定的语言

#### 16.1.3 跨语言的权限合并过程

```
用户访问 /geography/countries（locale=en）
  │
  ▼
checkAccess(user, ['read:pages'], { path: 'geography/countries', locale: 'en' })
  │
  ├─ 遍历用户所有组的所有 pageRules
  │
  ├─ 规则 A：locales=[], match=START, path='geography' → 跳过语言过滤 → 匹配
  │
  ├─ 规则 B：locales=['zh'], match=EXACT, path='geography/countries' → 语言不匹配 → 跳过
  │
  ├─ 规则 C：locales=['en'], match=EXACT, path='geography/countries' → 语言匹配 → 匹配
  │
  ▼
在匹配的规则中按特异性竞争决出最终结果
```

**关键理解**：语言过滤是第一道关卡——语言不匹配的规则直接被忽略，不参与后续的特异性竞争。

#### 16.1.4 多语言下的权限继承

每个语言空间的权限是**独立计算**的，不会跨语言继承：

```
组有一条规则：{ locales: ['en'], match: 'START', path: 'geography', roles: ['read:pages'] }

用户访问：
  ✓ /en/geography/countries → 通过（en 语言有 START 规则）
  ✗ /zh/geography/countries → 拒绝（zh 语言没有匹配的规则）
```

如果要让所有语言都生效，规则的 `locales` 必须设为空数组（或者包含所有语言）。

### 16.2 跨 Groups（组空间）的权限合并

#### 16.2.1 全局权限：并集合并

`server/models/users.js:153-155`：

```js
getGlobalPermissions() {
  return _.uniq(_.flatten(_.map(this.groups, 'permissions')))
}
```

**合并策略**：所有组的权限数组扁平化 → 去重 → 并集

```
组 A: [read:pages, write:pages]
组 B: [read:pages, manage:assets]
─────────────────────────────────
用户: [read:pages, write:pages, manage:assets]
```

#### 16.2.2 页面规则：竞争合并

与全局权限的"并集"不同，页面规则是**竞争**关系，不是并集。

`server/core/auth.js:246-289`：

```js
user.groups.forEach(grp => {
  const grpId = _.isObject(grp) ? _.get(grp, 'id', 0) : grp
  _.get(WIKI.auth.groups, `${grpId}.pageRules`, []).forEach(rule => {
    // ... 语言过滤 + 权限维度过滤 + 路径匹配
    // 匹配的规则通过 _applyPageRuleSpecificity 竞争
  })
})
```

**所有组的所有规则放在同一个竞争池中**，按特异性优先级决出最终结果：
- 路径更长的规则覆盖更短的
- 匹配类型优先级高的覆盖低的
- 同优先级下 DENY 覆盖 ALLOW

#### 16.2.3 跨组合并的示例

```
组 A（编辑组）的规则：
  - 规则1: START 'geography', roles=[read:pages, write:pages], deny=false

组 B（审核组）的规则：
  - 规则2: START 'geography/countries', roles=[read:pages], deny=false
  - 规则3: EXACT 'geography/countries/china', roles=[write:pages], deny=true

用户同时属于 A 和 B 两个组

权限判定：
  /geography/rivers → 匹配规则1（START） → 允许读+写
  /geography/countries → 匹配规则1 + 规则2 → 规则2路径更长胜出 → 允许读
  /geography/countries/china → 匹配规则1 + 规则2 + 规则3 → 规则3 EXACT 优先级最高 → 拒绝写
```

**注意**：即使组 A 允许写 `geography/countries/china`，只要组 B 有一条更高特异性的 DENY 规则，最终结果就是 DENY。因为所有规则共同竞争，不是"任一组允许就允许"。

### 16.3 Path 命名空间的权限分区

#### 16.3.1 START 规则的命名空间效果

虽然 Wiki.js 没有名为 "namespace" 或 "space" 的正式概念，但通过 `START` 匹配可以实现路径级的权限分区：

```
规则 A：START 'engineering' → 作用于 engineering/ 下所有页面
规则 B：START 'marketing'   → 作用于 marketing/ 下所有页面
规则 C：START ''            → 作用于所有页面（空路径匹配一切）
```

这在效果上等同于把 wiki 分成多个"命名空间"，每个命名空间有独立的权限控制。

#### 16.3.2 命名空间的嵌套覆盖

由于特异性系统的存在，子命名空间可以覆盖父命名空间的规则：

```
规则 1（公司级）：START ''               → 全员 read:pages
规则 2（部门级）：START 'engineering'    → eng 组 write:pages
规则 3（项目级）：START 'engineering/secret' → 只有 management 组 read:pages
```

权限从粗到细逐层收敛，最具体的规则胜出。

### 16.4 三维度合并的总流程

```
权限判定总公式：
  最终结果 = 全局权限（组并集） ∩ 页面规则（跨组竞争 + 语言过滤 + 路径特异性）
```

完整流程：

```
用户发起操作（带 locale + path）
  │
  ├─ 第一步：全局权限检查
  │   └─ 合并所有组的 permissions（并集）
  │   └─ 与所需权限做交集
  │   └─ 无交集 → 直接拒绝
  │
  ├─ 第二步：页面规则匹配（无页面上下文时跳过）
  │   │
  │   ├─ 遍历所有组的所有 pageRules
  │   │
  │   ├─ 语言过滤：rule.locales 包含 page.locale？
  │   │   └─ 不包含 → 跳过该规则
  │   │
  │   ├─ 权限维度过滤：rule.roles 与所需权限有交集？
  │   │   └─ 无交集 → 跳过该规则
  │   │
  │   ├─ 路径匹配：按 match 类型（START/END/EXACT/REGEX/TAG）判断
  │   │   └─ 不匹配 → 跳过该规则
  │   │
  │   └─ 特异性仲裁：
  │       ├─ 路径更长 → 覆盖
  │       ├─ 路径相同但匹配类型优先级更高 → 覆盖
  │       ├─ 都相同但 DENY → 覆盖 ALLOW
  │       └─ 否则 → 保持当前胜出规则
  │
  └─ 第三步：最终判定
      ├─ 没有任何规则匹配 → 拒绝（默认最严）
      ├─ 匹配且 DENY → 拒绝
      └─ 匹配且 ALLOW → 通过
```

### 16.5 Private NS（私有命名空间）：未完成的空间机制

代码中存在 `isPrivate` 和 `privateNS` 字段（`server/models/pages.js:45`），但从实现来看**功能未完成**，多处使用 `'TODO'` 占位：

```js
// server/models/pages.js:306
hash: pageHelper.generateHash({ path: opts.path, locale: opts.locale, privateNS: opts.isPrivate ? 'TODO' : '' }),
```

`pageHelper.generateHash` 的设计（`server/helpers/page.js:72`）：
```js
generateHash(opts) {
  return crypto.createHash('sha1').update(`${opts.locale}|${opts.path}|${opts.privateNS}`).digest('hex')
}
```

**设计意图推测**：privateNS 是为了实现真正的"私有命名空间"，同一 locale + path 在不同命名空间下是不同的页面（hash 不同）。但目前这一功能尚未实现。

`checkAccess` 函数中也**没有使用 `page.private` 或 `page.privateNS` 做特殊处理**——私有页面的权限控制目前完全依赖页面规则实现，没有独立的私有命名空间权限机制。

### 16.6 跨空间权限合并的关键特性总结

| 维度 | 合并策略 | 关键代码位置 |
|---|---|---|
| **跨组全局权限** | 并集（任一组成员拥有即拥有） | `users.js:153-155` |
| **跨组页面规则** | 竞争（特异性最高者胜出） | `auth.js:246-291` |
| **跨语言规则** | 过滤（不匹配语言的规则忽略） | `auth.js:249-251` |
| **跨路径（命名空间）** | 前缀匹配 + 特异性覆盖 | `auth.js:254-257` + `_applyPageRuleSpecificity` |
| **私有命名空间** | 未实现（TODO 占位） | `pages.js:306` 等 |

---

## 17. 权限批量修改时缓存失效顺序的代码路径

### 17.1 概述

Wiki.js 的权限变更分为三类操作，每类的缓存失效步骤和顺序不同。本节从 resolver 代码逐行追踪各操作下**内存缓存 → 撤销列表 → 事件传播**的精确执行顺序。

### 17.2 组更新（update）— 最完整的失效链路

`server/graph/resolvers/group.js:160-211`：

```js
async update (obj, args, { req }) {
  // ① 安全检查（unsafe regex、权限提升检查）
  //    - checkExclusiveAccess 校验操作者权限边界

  // ② 写入数据库
  await WIKI.models.groups.query().patch({
    name: args.name,
    redirectOnLogin: args.redirectOnLogin,
    permissions: JSON.stringify(args.permissions),
    pageRules: JSON.stringify(args.pageRules)
  }).where('id', args.id)

  // ③ 撤销组 token（本地内存）
  WIKI.auth.revokeUserTokens({ id: args.id, kind: 'g' })

  // ④ 跨 HA 节点同步撤销
  WIKI.events.outbound.emit('addAuthRevoke', { id: args.id, kind: 'g' })

  // ⑤ 刷新组内存缓存（本地）
  await WIKI.auth.reloadGroups()

  // ⑥ 跨 HA 节点同步组缓存
  WIKI.events.outbound.emit('reloadGroups')
}
```

**执行顺序图**：

```
① 安全校验
     │
     ▼
② DB 写入（事务提交）
     │
     ▼
③ 本地 revocationList 写入  ←─ key='g3', value=时间戳
     │
     ▼
④ outbound 事件广播          ←─ 其他 HA 节点收到后执行 revocationList 写入
     │
     ▼
⑤ 本地 WIKI.auth.groups 刷新 ←─ 从 DB 重新读取所有组
     │                         ←─ 同时：guest.cacheExpiration 置为过期
     ▼
⑥ outbound 事件广播          ←─ 其他 HA 节点收到后执行 reloadGroups()
```

**顺序关键点**：

1. **② 先于 ③**：DB 写入成功后才开始缓存失效，避免读到旧数据
2. **③ 先于 ⑤**：撤销列表先写入，确保在刷新组缓存的过程中，如果有请求进来，token 也会被标记为需要重验证
3. **⑤ 是 await**：`reloadGroups()` 必须等待完成才返回响应，保证后续请求能读到最新组信息
4. **④ 和 ⑥ 是 fire-and-forget**：事件广播不等待其他节点完成

### 17.3 组创建（create）— 无撤销，只有缓存刷新

`server/graph/resolvers/group.js:96-109`：

```js
async create (obj, args, { req }) {
  const group = await WIKI.models.groups.query().insertAndFetch({
    name: args.name,
    permissions: JSON.stringify(WIKI.data.groups.defaultPermissions),
    pageRules: JSON.stringify(WIKI.data.groups.defaultPageRules),
    isSystem: false
  })
  await WIKI.auth.reloadGroups()
  WIKI.events.outbound.emit('reloadGroups')
  return { ... }
}
```

**为什么不需要撤销 token？**

新建组时，没有任何用户属于这个组，所以不需要撤销任何用户的 token。只需刷新组缓存，让系统能识别这个新组。

```
① DB 插入
     │
     ▼
② 本地 WIKI.auth.groups 刷新
     │
     ▼
③ outbound 事件广播
```

### 17.4 组删除（delete）— 撤销 + 刷新

`server/graph/resolvers/group.js:113-129`：

```js
async delete (obj, args) {
  await WIKI.models.groups.query().deleteById(args.id)

  WIKI.auth.revokeUserTokens({ id: args.id, kind: 'g' })
  WIKI.events.outbound.emit('addAuthRevoke', { id: args.id, kind: 'g' })

  await WIKI.auth.reloadGroups()
  WIKI.events.outbound.emit('reloadGroups')
  return { ... }
}
```

**顺序**：DB 删除 → 撤销组 token → 刷新组缓存。

**与 update 的区别**：删除操作不需要安全检查（因为组 ID 1/2 已在入口处拦截），其余步骤一致。

### 17.5 用户分配到组（assignUser）— 只撤销用户 token

`server/graph/resolvers/group.js:36-92`：

```js
async assignUser (obj, args, { req }) {
  // ① 安全校验（Guest 用户、组有效性、权限边界）
  // ② 检查是否已存在关系
  // ③ 写入 userGroups 关联
  await grp.$relatedQuery('users').relate(usr.id)

  // ④ 撤销该用户 token
  WIKI.auth.revokeUserTokens({ id: usr.id, kind: 'u' })
  WIKI.events.outbound.emit('addAuthRevoke', { id: usr.id, kind: 'u' })
}
```

**关键**：
- 只撤销**单个用户**的 token（kind='u'），不是组级撤销
- **不需要 `reloadGroups()`**：组定义没变，只是用户-组关系变了
- 用户下次请求时，重验证流程会从 DB 重新读取用户的组关系

### 17.6 用户移出组（unassignUser）— 同 assignUser

`server/graph/resolvers/group.js:133-156`：

```js
async unassignUser (obj, args) {
  // ① 安全校验
  // ② 删除 userGroups 关联
  await grp.$relatedQuery('users').unrelate().where('userId', usr.id)

  // ③ 撤销该用户 token
  WIKI.auth.revokeUserTokens({ id: usr.id, kind: 'u' })
  WIKI.events.outbound.emit('addAuthRevoke', { id: usr.id, kind: 'u' })
}
```

### 17.7 用户删除 / 停用 — 撤销用户 token

`server/graph/resolvers/user.js:80-99`（delete）：
```js
WIKI.auth.revokeUserTokens({ id: args.id, kind: 'u' })
WIKI.events.outbound.emit('addAuthRevoke', { id: args.id, kind: 'u' })
```

`server/graph/resolvers/user.js:138-153`（deactivate）：
```js
WIKI.auth.revokeUserTokens({ id: args.id, kind: 'u' })
WIKI.events.outbound.emit('addAuthRevoke', { id: args.id, kind: 'u' })
```

### 17.8 各操作的缓存失效步骤对比

| 操作 | DB 写入 | revocationList（本地） | outbound 同步 | reloadGroups（本地） | outbound 同步 |
|---|---|---|---|---|---|
| **组更新** | ✅ patch | ✅ kind='g' | ✅ addAuthRevoke | ✅ await | ✅ reloadGroups |
| **组创建** | ✅ insert | — | — | ✅ await | ✅ reloadGroups |
| **组删除** | ✅ delete | ✅ kind='g' | ✅ addAuthRevoke | ✅ await | ✅ reloadGroups |
| **用户分配到组** | ✅ relate | ✅ kind='u' | ✅ addAuthRevoke | — | — |
| **用户移出组** | ✅ unrelate | ✅ kind='u' | ✅ addAuthRevoke | — | — |
| **用户删除** | ✅ delete | ✅ kind='u' | ✅ addAuthRevoke | — | — |
| **用户停用** | ✅ patch | ✅ kind='u' | ✅ addAuthRevoke | — | — |

### 17.9 HA 事件订阅：跨节点缓存同步

`server/core/auth.js:478-490`：

```js
subscribeToEvents() {
  WIKI.events.inbound.on('reloadGroups', () => {
    WIKI.auth.reloadGroups()
  })
  WIKI.events.inbound.on('addAuthRevoke', (args) => {
    WIKI.auth.revokeUserTokens(args)
  })
}
```

**事件方向**：

```
节点 A 执行组更新
  │
  ├─ 本地：revokeUserTokens + reloadGroups
  │
  └─ outbound.emit('addAuthRevoke') + outbound.emit('reloadGroups')
       │
       ▼
  消息队列 / 进程间通信
       │
       ▼
  节点 B 的 inbound 收到事件
       │
       ├─ inbound.on('addAuthRevoke') → revokeUserTokens()
       └─ inbound.on('reloadGroups')  → reloadGroups()
```

**注意**：HA 同步是异步的，不保证两个节点同时生效。存在短暂的不一致窗口。

### 17.10 `revokeUserTokens` 的精确语义

`server/core/auth.js:526-528`：

```js
revokeUserTokens ({ id, kind = 'u' }) {
  WIKI.auth.revocationList.set(
    `${kind}${_.toString(id)}`,          // key: 'u123' 或 'g3'
    Math.round(                          // value: 撤销时间戳（5秒前）
      DateTime.utc().minus({ seconds: 5 }).toSeconds()
    ),
    Math.ceil(ms(WIKI.config.auth.tokenExpiration) / 1000)  // TTL: 与 token 有效期相同
  )
}
```

**设计细节**：
- 时间戳减去 5 秒：留出时钟偏移的容错，确保撤销前 5 秒签发的 token 也能被强制重验证
- TTL 设为 token 有效期：撤销记录不需要比 token 存活更久，过期自动清理
- kind='g'（组级撤销）：影响该组所有用户的 token
- kind='u'（用户级撤销）：只影响单个用户的 token

### 17.11 请求时的失效检查时序

`server/core/auth.js:113-177`，每次请求经过 `authenticate` 中间件时：

```
请求到达
  │
  ▼
JWT 解码 → 得到 user.id, user.iat, user.groups
  │
  ▼
检查1：用户级撤销
  revocationList.get(`u${user.id}`)
  如果存在且 user.iat < 撤销时间 → mustRevalidate = true
  │
  ▼
检查2：服务重启
  DateTime.fromSeconds(user.iat) <= WIKI.startedAt → mustRevalidate = true
  │
  ▼
检查3：组级撤销（遍历 user.groups）
  for (gid of user.groups):
    revocationList.get(`g${gid}`)
    如果存在且 user.iat < 撤销时间 → mustRevalidate = true
  │
  ▼
mustRevalidate?
  ├─ true: 从 DB 重新读取用户信息 → 签发新 JWT → 写入响应
  └─ false: 使用当前 JWT 继续
  │
  ▼
JWT 无效?
  └─ true: req.user = Guest（缓存1分钟）
```

**检查顺序的设计理由**：
1. 先检查用户级（最常见、最快判断），再检查组级（需遍历）
2. 服务重启检查放在中间，因为只在启动后短时间内有意义
3. 一旦任一检查触发 mustRevalidate，就跳过后续检查

---

## 18. 超级管理员绕过权限链的代码挂载点

### 18.1 概述

超级管理员通过 `manage:system` 权限实现全面绕过。这不是一个统一的"superadmin 模式"，而是在权限链的**不同层级**各有独立的短路点。本节逐一列出所有挂载点。

### 18.2 挂载点 1：`checkAccess` — 运行时权限检查短路

`server/core/auth.js:224-227`：

```js
checkAccess(user, permissions = [], page = false) {
  const userPermissions = user.permissions ? user.permissions : user.getGlobalPermissions()

  // System Admin
  if (_.includes(userPermissions, 'manage:system')) {
    return true
  }
  // ... 后续全局权限检查和页面规则检查全部跳过
}
```

**影响范围**：所有调用 `checkAccess` 的地方，包括：
- 页面浏览（`common.js:298`）
- 页面树过滤（`page.js:286`）
- 搜索结果过滤（`page.js:58`）
- `getEffectivePermissions`（`auth.js:496-520`）
- 标签、链接、资源访问等所有可见性判断

**效果**：超级管理员**完全跳过页面规则检查**，无论什么路径、什么语言、什么标签，一律通过。

### 18.3 挂载点 2：`@auth` GraphQL 指令 — 查询级权限检查

`server/graph/directives/auth.js:46`：

```js
if (!_.some(context.req.user.permissions, pm => _.includes(requiredScopes, pm))) {
  throw new Error('Forbidden')
}
```

**这里没有 `manage:system` 短路！** `@auth` 指令用的是**精确匹配**：只要用户的任一权限在 `requiredScopes` 列表中就通过。

但**实际上 `manage:system` 能通过所有 `@auth` 检查**，因为 GraphQL schema 中几乎所有 `@auth` 指令都把 `manage:system` 列为允许的权限之一：

```graphql
# group.graphql
list: ... @auth(requires: ["write:users", "manage:users", "write:groups", "manage:groups", "manage:system"])
create: ... @auth(requires: ["write:groups", "manage:groups", "manage:system"])
update: ... @auth(requires: ["write:groups", "manage:groups", "manage:system"])
delete: ... @auth(requires: ["write:groups", "manage:groups", "manage:system"])

# page.graphql
search: ... @auth(requires: ["manage:system", "read:pages"])
list: ... @auth(requires: ["manage:system", "read:pages"])
single: ... @auth(requires: ["read:pages", "manage:system"])
```

**关键理解**：`@auth` 层的 `manage:system` 绕过是**声明式的**（在 schema 中显式列出），不是代码中的硬编码短路。如果有人添加了一个新的 GraphQL 字段但忘记在 `@auth` 的 `requires` 中加入 `manage:system`，超级管理员也会被拒绝。

### 18.4 挂载点 3：`checkExclusiveAccess` — 权限提升检查短路

`server/core/auth.js:304-318`：

```js
checkExclusiveAccess(user, includePermissions = [], excludePermissions = []) {
  const userPermissions = user.permissions ? user.permissions : user.getGlobalPermissions()

  if (_.intersection(userPermissions, includePermissions).length < 1) {
    return false
  }
  if (_.intersection(userPermissions, excludePermissions).length > 0) {
    return false
  }
  return true
}
```

这个函数本身**没有** `manage:system` 短路。但调用它的代码通过**将 `manage:system` 放入 excludePermissions 参数**来实现等效效果：

```js
// group.js:50 — assignUser 中的权限提升检查
WIKI.auth.checkExclusiveAccess(req.user, ['manage:users', 'write:groups'], ['manage:groups', 'manage:system'])
```

**逻辑**：`checkExclusiveAccess(用户, ['manage:users'], ['manage:system'])` 的含义是——"用户是否**有** `manage:users` 但**没有** `manage:system`？"

- 超级管理员有 `manage:system` → 在 excludePermissions 中命中 → 返回 `false` → 跳过限制
- 非 `manage:system` 的 `manage:users` 用户 → 不在 excludePermissions 中 → 返回 `true` → 受到限制

**这是一个反向绕过机制**：不是"有 `manage:system` 就放行"，而是"有 `manage:system` 就不受限制"。

### 18.5 挂载点 4：`checkAssignUserToGroupAccess` — 组分配权限短路

`server/core/auth.js:327-361`：

```js
async checkAssignUserToGroupAccess(requester, groupIds = []) {
  const requesterPermissions = requester.permissions ? requester.permissions : requester.getGlobalPermissions()

  // System Admin
  if (requesterPermissions.includes('manage:system')) {
    return true   // ← 超级管理员可以直接分配用户到任何组
  }

  // 非 manage:system 的后续检查...
  // - 基本权限检查
  // - 组内 manage:system 权限检查（非超管不能分配用户到有 manage:system 的组）
  // - 组内管理权限检查（非 manage:groups 不能分配用户到有管理权限的组）
}
```

**调用位置**：
- `user.js:67` — 创建用户时
- `user.js:103` — 更新用户时

**效果**：只有超级管理员能分配用户到拥有 `manage:system` 权限的组，其他管理员（`manage:users` / `manage:groups`）都不能。

### 18.6 挂载点 5：`getRootUser` — 内部操作的硬编码超管

`server/models/users.js:891-898`：

```js
static async getRootUser () {
  let user = await WIKI.models.users.query().findById(1)
  user.permissions = ['manage:system']  // ← 硬编码，不读组
  return user
}
```

**用途**：系统内部操作（如定时任务、搜索索引重建等需要绕过权限的场景）使用 `getRootUser()` 获取一个不受任何限制的身份。

**特点**：
- 不通过 `getGlobalPermissions()` 计算权限，直接硬编码 `['manage:system']`
- 只读 id=1 的用户，不关心该用户实际属于什么组
- 不是 HTTP 请求路径，是服务端内部调用

### 18.7 挂载点 6：API Token 的权限来源

`server/core/auth.js:179-204`：

```js
if (_.has(user, 'api')) {
  // ...
  req.user = {
    id: 1,
    permissions: _.get(WIKI.auth.groups, `${user.grp}.permissions`, []),
    groups: [user.grp],
    // ...
  }
}
```

API Token 的权限来自创建时指定的组（`user.grp`）。如果 API Token 绑定的组拥有 `manage:system`，则 API 请求也拥有超级管理员权限。

### 18.8 超级管理员绕过链路全景图

```
请求到达
  │
  ▼
┌─────────────────────────────────────────────────────┐
│  层1：GraphQL @auth 指令                              │
│  auth.js:46                                          │
│  manage:system 在 schema 的 requires 列表中 → 通过    │
│  （声明式，不是代码短路）                               │
└─────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────┐
│  层2：Resolver 内的权限提升检查                        │
│                                                      │
│  2a. checkExclusiveAccess                            │
│      manage:system 在 excludePermissions 中          │
│      → 返回 false → 跳过限制                         │
│      （group.js:50, 61, 175, 186）                   │
│                                                      │
│  2b. checkAssignUserToGroupAccess                    │
│      manage:system → 直接 return true                │
│      （auth.js:335）                                  │
└─────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────┐
│  层3：checkAccess 运行时权限检查                       │
│  auth.js:225-227                                     │
│  manage:system → 直接 return true                    │
│  （跳过全局权限检查 + 页面规则检查）                    │
└─────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────┐
│  层4：getEffectivePermissions                        │
│  auth.js:496-520                                     │
│  内部调用 checkAccess → 同样被 manage:system 短路     │
│  → 所有权限维度（read/write/manage/delete）全为 true  │
└─────────────────────────────────────────────────────┘
```

### 18.9 各层绕过方式的差异

| 层级 | 绕过方式 | 是否硬编码 | 漏洞风险 |
|---|---|---|---|
| **@auth 指令** | 声明式（schema 中列出 `manage:system`） | 否 | 新增字段时可能遗漏 |
| **checkExclusiveAccess** | 反向排除（`manage:system` 在 exclude 中使检查返回 false） | 半硬编码 | 调用方需正确传参 |
| **checkAssignUserToGroupAccess** | 硬编码短路（`includes('manage:system') → return true`） | 是 | 无 |
| **checkAccess** | 硬编码短路（`includes('manage:system') → return true`） | 是 | 无 |
| **getRootUser** | 硬编码覆盖（`permissions = ['manage:system']`） | 是 | 无 |

### 18.10 `manage:system` 的权限提升保护

系统防止非 `manage:system` 用户自行获取 `manage:system`，有三道防线：

**防线 1：@auth 指令**（`group.graphql:43`）
```graphql
update: ... @auth(requires: ["write:groups", "manage:groups", "manage:system"])
```
只有 `write:groups` / `manage:groups` / `manage:system` 才能调用组更新接口。

**防线 2：checkExclusiveAccess**（`group.js:184-190`）
```js
if (
  WIKI.auth.checkExclusiveAccess(req.user, ['manage:groups'], ['manage:system']) &&
  args.permissions.some(p => _.last(p.split(':')) === 'system')
) {
  throw new gql.GraphQLError('...')
}
```
有 `manage:groups` 但没有 `manage:system` 的用户，不能给组添加 `manage:system` 权限。

**防线 3：checkAssignUserToGroupAccess**（`auth.js:347-348`）
```js
if (grp.permissions.includes('manage:system')) {
  return false
}
```
非 `manage:system` 用户不能把其他用户分配到拥有 `manage:system` 的组。

**三层防线的关系**：
- 防线 1 阻止无权限者调用接口
- 防线 2 阻止有组管理权限者给自己添加 `manage:system`
- 防线 3 阻止有用户管理权限者把他人放入 `manage:system` 组

**只有已经是 `manage:system` 的用户才能分配 `manage:system` 权限**——形成闭环保护。

---

## 19. 权限审计记录的导出路径

### 19.1 概述：没有专门的"权限审计"功能

Wiki.js 2.x **没有独立的权限审计日志系统**（没有 `auditLog` / `permissionLog` 表，也没有操作日志记录谁在什么时候改了什么权限）。

所谓"权限审计"只能通过以下三个间接路径获得：

1. **系统导出（System Export）**：把组、用户、设置等完整数据导出为 JSON 文件，用于离线审计
2. **组 / 用户 GraphQL 查询接口**：管理员通过 UI 或 API 实时读取权限快照
3. **页面历史 + 评论导出**：间接查看页面内容变更（但不含权限变更记录）

### 19.2 系统导出：GraphQL 入口

`server/graph/schemas/system.graphql:52-56`：

```graphql
export(
  entities: [String]!    # 要导出的实体类型
  path: String!          # 服务器本地目录（相对 WIKI.ROOTPATH）
): DefaultResponse @auth(requires: ["manage:system"])
```

**唯一权限**：`manage:system`。只有超级管理员能触发导出。

### 19.3 系统导出：Resolver 触发层

`server/graph/resolvers/system.js:277-307`：

```js
async export (obj, args, context) {
  // ① 安全检查
  //    - 已有导出在运行？防止并发
  //    - entities 数组非空？
  //    - 目标目录存在且为空？

  // ② 触发异步导出（不阻塞响应）
  WIKI.system.export({
    entities: args.entities,  // 如 ['groups', 'users', 'pages', 'settings', ...]
    path: desiredPath
  })

  // ③ 立即返回 "已启动" 响应
  return graphHelper.generateSuccess('Export started successfully.')
}
```

**关键特征**：`WIKI.system.export()` 是**异步 fire-and-forget**，GraphQL 响应返回时导出可能才刚开始。客户端需要轮询 `exportStatus` 查询进度。

### 19.4 导出进度查询

`server/graph/resolvers/system.js:46-53`：

```js
async exportStatus () {
  return {
    status: WIKI.system.exportStatus.status,      // 'notrunning' | 'running' | 'success' | 'error'
    progress: Math.ceil(WIKI.system.exportStatus.progress), // 0-100
    message: WIKI.system.exportStatus.message,
    startedAt: WIKI.system.exportStatus.startedAt
  }
}
```

同样需要 `manage:system` 权限（`system.graphql:21`）。

### 19.5 系统导出：核心实现层

`server/core/system.js:93-460`，按 `entities` 参数逐个处理。

以下是**与权限相关的实体导出**路径：

#### 19.5.1 Groups 导出

`server/core/system.js:204-212`：

```js
case 'groups': {
  WIKI.logger.info('Exporting groups...')
  const outputPath = path.join(opts.path, 'groups.json')
  const groups = await WIKI.models.groups.query()   // 读所有组
  await fs.outputJSON(outputPath, groups, { spaces: 2 })
  // groups 每条记录包含：
  //   - id, name, isSystem, redirectOnLogin
  //   - permissions (JSON string)   ← 全局权限数组
  //   - pageRules (JSON string)     ← 页面规则数组
  this.exportStatus.progress += progressMultiplier * 100
  break
}
```

输出 `groups.json`，包含完整的 `permissions` 和 `pageRules`，可用于离线审计"每个组有什么权限"。

#### 19.5.2 Users 导出

`server/core/system.js:391-450`：

```js
case 'users': {
  const rs = Readable({ objectMode: true })
  // ...
  const fetchUsersBatch = async (offset) => {
    const users = await WIKI.models.users.query()
      .offset(offset).limit(50)
      .withGraphJoined({
        groups: true,      // ← 关联 userGroups → 组的 id + name
        provider: true     // ← 关联 authentication → 登录方式
      })
    // 逐条流式推送
  }
  // Gzip 压缩输出 users.json.gz
}
```

输出 `users.json.gz`，每条用户记录包含：
- `id, name, email, providerId, providerKey, isActive, isSystem, ...`
- `groups: [{ id, name }]` ← 用户所属组（ID + 名称，不含组的权限）
- `provider: { key, strategyKey, displayName }` ← 认证策略信息

**注意**：导出的用户记录**不包含其实际权限数组**（只含所属组）。要审计用户实际权限，需要结合 `groups.json` 中的 `permissions` / `pageRules` 自行计算。

#### 19.5.3 Settings 导出（含认证配置）

`server/core/system.js:364-386`：

```js
case 'settings': {
  const config = {
    ...WIKI.config,           // 当前配置
    modules: {
      analytics: await WIKI.models.analytics.query(),
      authentication: (await WIKI.models.authentication.query()).map(a => ({
        ...a,
        domainWhitelist: _.get(a, 'domainWhitelist.v', []),
        autoEnrollGroups: _.get(a, 'autoEnrollGroups.v', [])  // ← 自动加入的组
      })),
      commentProviders: ...,
      renderers: ...,
      searchEngines: ...,
      storage: ...
    },
    apiKeys: await WIKI.models.apiKeys.query().where('isRevoked', false)
  }
}
```

输出 `settings.json`，其中与权限审计相关的字段：
- `authentication[*].autoEnrollGroups`：新用户自动加入的组（权限来源之一）
- `authentication[*].domainWhitelist`：域名白名单
- `apiKeys[*]`：未撤销的 API Key（含 `grp` 字段 → 对应的组 ID）

#### 19.5.4 Navigation 导出

`server/core/system.js:282-293`：

```js
case 'navigation': {
  const navigationRaw = await WIKI.models.navigation.query()
  const navigation = navigationRaw.reduce((obj, cur) => {
    obj[cur.key] = cur.config  // config 中包含 visibilityGroups 等可见性配置
    return obj
  }, {})
  await fs.outputJSON(outputPath, navigation, { spaces: 2 })
}
```

输出 `navigation.json`，其中每个导航项的 `visibilityMode` 和 `visibilityGroups` 是审计"导航可见性"的关键。

#### 19.5.5 其他实体（与权限间接相关）

| 实体 | 输出文件 | 权限相关内容 |
|---|---|---|
| **pages** | `pages.json.gz` | 不含权限，只有 path/locale/title/content |
| **history** | `pages-history.json.gz` | 页面变更历史（不含权限变更） |
| **assets** | `assets/` 目录 | 资源文件（不含权限） |
| **comments** | `comments.json.gz` | 评论（含作者信息） |

### 19.6 权限审计导出的完整文件清单

一次完整的审计导出（`entities = ['groups','users','settings','navigation']`）会生成：

```
目标目录/
├── groups.json          ← 所有组的 permissions + pageRules
├── users.json.gz        ← 所有用户 + 所属组关系
├── settings.json        ← 认证策略 + 自动入组 + API Keys
└── navigation.json      ← 导航可见性配置
```

**审计数据链**：
```
users.json.gz → 用户所属组 ID
                    │
                    ▼
          groups.json → 组的 permissions + pageRules
                    │
                    ▼
          可重建出每个用户的实际有效权限
```

### 19.7 导出的安全边界

| 安全控制 | 位置 | 说明 |
|---|---|---|
| **入口权限** | `system.graphql:55` | `@auth(requires: ["manage:system"])` |
| **并发控制** | `system.js:284` | 同时只能有一个导出在运行（内存标记 `exportStatus.status`） |
| **目录空检查** | `system.js:294` | 目标目录必须为空 |
| **敏感信息** | `system.js:367-383` | `settings.json` 中包含认证配置的敏感字段（密钥等） |
| **不含密码** | `users` 导出 | 通过 Objection.js model 的 `$hidden` 排除了 `password` 字段 |

### 19.8 权限审计的能力边界（缺失项）

Wiki.js 2.x 导出功能**不能**直接提供以下审计信息：

1. ❌ **谁在什么时候改了权限**：没有操作日志（audit trail）
2. ❌ **权限变更的 diff**：只能导出当前快照，无法导出变更历史
3. ❌ **用户实际权限的预计算**：需要手工结合 users + groups 推导
4. ❌ **页面级权限覆盖记录**：导出的 pages 不包含规则（规则在 groups 中）
5. ❌ **Guest 用户访问日志**：没有访问日志（需靠 Web 服务器 log 弥补）

这些缺失意味着：Wiki.js 的"导出"更像是**数据备份**，而非合规意义上的**权限审计**。

---

## 20. 跨实例（HA）同步时的权限继承与缓存失效挂载点

### 20.1 HA 架构总览

Wiki.js 2.x 通过 **PostgreSQL LISTEN/NOTIFY**（`pg-pubsub` 模块）实现多实例间的事件传播。不支持其他数据库的 HA 同步。

`server/core/db.js:231-264`：

```js
async subscribeToNotifications () {
  // 前置条件：
  //   ① WIKI.config.ha === true
  //   ② 数据库类型 === 'postgres'
  //   任一不满足则不启用 HA
  // ...

  const PGPubSub = require('pg-pubsub')
  this.listener = new PGPubSub(this.knex.client.connectionSettings)

  // 接收 DB NOTIFY → 分发到 inbound 事件总线
  this.listener.addChannel('wiki', payload => {
    if (payload.source !== WIKI.INSTANCE_ID) {   // 忽略自己发的事件
      WIKI.events.inbound.emit(payload.event, payload.value)
    }
  })

  // 把所有 outbound 事件广播到 DB NOTIFY
  WIKI.events.outbound.onAny(this.notifyViaDB)

  // 注册订阅者：三类权限相关
  WIKI.auth.subscribeToEvents()    // 组、API Key、认证策略、撤销
  WIKI.configSvc.subscribeToEvents() // 配置变更
  WIKI.models.pages.subscribeToEvents() // 页面缓存
}
```

**事件流向**：

```
节点 A 执行操作
  │
  ▼
outbound.emit('eventName', payload)
  │
  ▼
notifyViaDB() → PostgreSQL NOTIFY 'wiki' (source=A, event=..., value=...)
  │
  ▼
PostgreSQL 推送至所有 LISTEN 连接
  │
  ▼
节点 B 的 listener 收到
  │
  ├─ payload.source === B? → 忽略（自己发的）
  └─ payload.source !== B? → WIKI.events.inbound.emit(event, value)
       │
       ▼
     subscribeToEvents 中注册的处理器执行
```

### 20.2 三类事件总线与订阅者总览

```
WIKI.events
  │
  ├── outbound: 本实例主动发出的事件 → 经 DB NOTIFY 广播
  │
  └── inbound: 从其他实例接收的事件 → 触发本地缓存失效
       │
       ├── auth.subscribeToEvents()   ← 权限相关（核心）
       ├── configSvc.subscribeToEvents() ← 配置相关
       └── pages.subscribeToEvents()     ← 页面缓存相关
```

### 20.3 权限相关的 HA 同步挂载点（auth）

`server/core/auth.js:478-491`：

```js
subscribeToEvents() {
  // ── 挂载点 1：组缓存刷新 ──
  WIKI.events.inbound.on('reloadGroups', () => {
    WIKI.auth.reloadGroups()
    // → 重新从 DB 读取所有组
    // → 同时令 Guest 缓存过期（cacheExpiration = 一天前）
  })

  // ── 挂载点 2：API Key 缓存刷新 ──
  WIKI.events.inbound.on('reloadApiKeys', () => {
    WIKI.auth.reloadApiKeys()
    // → 重新从 DB 读取未撤销的 API Key
  })

  // ── 挂载点 3：认证策略激活 ──
  WIKI.events.inbound.on('reloadAuthStrategies', () => {
    WIKI.auth.activateStrategies()
    // → 重新初始化 Passport 策略
    // → 域名白名单、自动入组配置实时生效
  })

  // ── 挂载点 4：Token 撤销 ──
  WIKI.events.inbound.on('addAuthRevoke', (args) => {
    WIKI.auth.revokeUserTokens(args)
    // → args = { id, kind: 'u'|'g' }
    // → 写入本地 revocationList[key] = 时间戳
    // → 下一次请求时 mustRevalidate = true
  })
}
```

### 20.4 权限相关事件的完整触发链

#### 20.4.1 组更新（create / update / delete）

```
节点 A：graph/resolvers/group.js
  │
  ├─ ① DB 写入（insert/patch/delete）
  │
  ├─ ② WIKI.auth.revokeUserTokens({ id, kind: 'g' })
  │     ← 写入本地 revocationList
  │
  ├─ ③ WIKI.events.outbound.emit('addAuthRevoke', { id, kind: 'g' })
  │     ← 发往 PostgreSQL NOTIFY
  │
  ├─ ④ await WIKI.auth.reloadGroups()
  │     ← 本地组缓存刷新（同步等待）
  │
  └─ ⑤ WIKI.events.outbound.emit('reloadGroups')
        ← 发往 PostgreSQL NOTIFY
```

节点 B 接收：
```
┌─ inbound.on('addAuthRevoke', args) → revokeUserTokens(args)
└─ inbound.on('reloadGroups') → reloadGroups()
```

**顺序依赖**：`addAuthRevoke` 先于 `reloadGroups` 广播。这样即使节点 B 在 `reloadGroups` 前有请求进来，`revocationList` 也已写入，能强制 token 重验证。

#### 20.4.2 用户组关系变更（assignUser / unassignUser）

```
节点 A：graph/resolvers/group.js
  │
  ├─ ① userGroups 关联变更
  │
  ├─ ② WIKI.auth.revokeUserTokens({ id: userId, kind: 'u' })
  │
  └─ ③ WIKI.events.outbound.emit('addAuthRevoke', { id: userId, kind: 'u' })
```

节点 B 接收：
```
inbound.on('addAuthRevoke', args) → revokeUserTokens(args)
```

**不需要 `reloadGroups`**：组定义没变，只是单个用户的 token 需要失效。用户下一次请求时会重验证并从 DB 拉最新的组关系。

#### 20.4.3 用户删除 / 停用

```
节点 A：graph/resolvers/user.js
  │
  ├─ ① users 表 delete / patch isActive=false
  │
  ├─ ② WIKI.auth.revokeUserTokens({ id, kind: 'u' })
  │
  └─ ③ WIKI.events.outbound.emit('addAuthRevoke', { id, kind: 'u' })
```

与 20.4.2 完全一致的传播路径。

#### 20.4.4 V1 用户导入

`server/graph/resolvers/system.js:225-227`：

```js
if (args.groupMode !== `NONE`) {
  await WIKI.auth.reloadGroups()           // 本地刷新
  WIKI.events.outbound.emit('reloadGroups') // 广播
}
```

导入后只广播 `reloadGroups`（因为新建的组没有用户，不需要撤销 token）。

### 20.5 配置相关的 HA 同步（影响权限策略）

`server/core/config.js:130-134`：

```js
subscribeToEvents() {
  WIKI.events.inbound.on('reloadConfig', async () => {
    await WIKI.configSvc.loadFromDb()
    await WIKI.configSvc.applyFlags()
  })
}
```

`reloadConfig` 由设置变更时触发。与权限间接相关的配置项：
- 认证策略的域名白名单
- 认证策略的自动入组配置
- 功能开关（如评论功能 → 影响 `read:comments` / `write:comments` 的实际可用性）

### 20.6 页面缓存的 HA 同步

`server/models/pages.js:1166-1172`：

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

**与权限的间接关系**：页面渲染缓存不包含权限判定，但如果因为权限变更导致某个页面的渲染内容需要调整（如编辑器看到不同的按钮），缓存清除保证下次渲染时权限判定重新执行。

实际的权限检查在渲染前（`common.js:298`）执行，与缓存无关。

### 20.7 事件广播的底层实现

`server/core/db.js:282-288`：

```js
notifyViaDB (event, value) {
  WIKI.models.listener.publish('wiki', {
    source: WIKI.INSTANCE_ID,  // 实例唯一标识，用于过滤自己发的事件
    event,                     // 事件名：'reloadGroups' / 'addAuthRevoke' / ...
    value                      // 事件载荷
  })
}
```

通过 `pg-pubsub` 的 `publish('wiki', payload)` 发送 PostgreSQL NOTIFY。

### 20.8 HA 同步的能力边界

| 特性 | 支持情况 | 说明 |
|---|---|---|
| **组缓存同步** | ✅ 支持 | `reloadGroups` |
| **Token 撤销同步** | ✅ 支持 | `addAuthRevoke`（用户级 + 组级） |
| **API Key 同步** | ✅ 支持 | `reloadApiKeys` |
| **认证策略同步** | ✅ 支持 | `reloadAuthStrategies` |
| **导航缓存同步** | ❌ 缺失 | 导航有 300 秒 LRU 缓存，过期自动失效 |
| **Guest 缓存同步** | ⚠️ 间接支持 | 通过 `reloadGroups` 中间接置为过期，不单独广播 |
| **数据库直写旁路** | ❌ 不支持 | 直接改 DB 不触发同步，需重启实例或手动触发 |
| **非 PostgreSQL DB** | ❌ 不支持 | HA 同步依赖 PostgreSQL LISTEN/NOTIFY |
| **事务一致性** | ❌ 最终一致 | 事件异步传播，有短暂不一致窗口 |
| **事件丢失恢复** | ❌ 无重试 | 网络瞬断可能丢事件，需重启实例恢复 |

### 20.9 权限继承在跨实例间的时序一致性

```
T0: 管理员在节点 A 上更新 Group(id=3) 权限
     │
     ▼
T1: 节点 A
      ├─ DB commit（全局一致，所有节点从同一 DB 读取）
      ├─ revocationList['g3'] = T1
      ├─ 广播 addAuthRevoke(g3)
      ├─ reloadGroups()（从 DB 读最新组）
      └─ 广播 reloadGroups
     │
     ▼
T2: 节点 B 收到 addAuthRevoke(g3)
     → 写入本地 revocationList['g3'] = T1
     │
     ▼
T3: 节点 B 收到 reloadGroups
     → reloadGroups() → 从 DB 读最新组
     │
     ▼
T4: 用户请求到达节点 B（携带 JWT，iat = T0_5，groups: [3]）
     → auth.js:134 检查 revocationList['g3']
     → T0_5 < T1 → mustRevalidate = true
     → 从 DB 重新读取用户+组信息 → 签发新 token
     │
     ▼
T5: 用户在节点 B 获得最新权限
```

**一致性保证**：即使 `reloadGroups` 延迟到达（T3 > T4），`addAuthRevoke`（T2）写入的撤销记录也能确保请求被强制重验证，从 DB 直接读取最新数据。这就是 **撤销事件必须先于刷新事件广播** 的设计原因。

### 20.10 HA 故障场景分析

**场景 1：节点 B 网络瞬断，丢失 `addAuthRevoke` 事件**

```
节点 B revocationList 中没有 'g3' 记录
  → 用户请求到达，JWT iat 正常
  → mustRevalidate = false → 使用旧权限
  → 直到 JWT 自然过期（30 分钟）后才刷新
  窗口：最多 30 分钟权限不一致
```

**场景 2：节点 B 网络瞬断，丢失 `reloadGroups` 事件**

```
节点 B WIKI.auth.groups 仍是旧的 pageRules
  → 用户 token 被重验证（addAuthRevoke 收到了）
  → getGlobalPermissions() 从旧的 groups 缓存读
  → 权限仍是旧的
  → 直到下一次组变更触发 reloadGroups
  窗口：永久不一致（除非重启或手动改组）
```

**场景 3：直接在 DB 中修改 groups 表（不通过 GraphQL 接口）**

```
所有实例的 groups 缓存都不刷新
所有实例的 revocationList 都不添加
  → 旧 token 一直有效（30 分钟）
  → token 刷新时从 DB 读，但 groups 内存缓存仍是旧的
  → 即使 reloadGroups，也从 DB 读到了最新的，但需要手动触发
  窗口：永久不一致（除非重启实例）
```

**场景 4：服务重启**

```
WIKI.startedAt = 当前时间
  → 所有旧 token 的 iat <= startedAt
  → mustRevalidate = true（auth.js:131-132）
  → 所有用户下一次请求强制刷新权限
  → reloadGroups 在启动时自动执行（setup.js）
  窗口：重启后第一次请求立即生效
```

**关键结论**：
- **addAuthRevoke 是"软防线"**：丢了最多不一致 30 分钟
- **reloadGroups 是"硬防线"**：丢了会导致**永久不一致**，直到下一次组变更
- **重启是终极同步机制**：一次性清除所有不一致状态
