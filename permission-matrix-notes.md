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
