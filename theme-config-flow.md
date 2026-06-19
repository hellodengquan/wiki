# Theme & Site Config 流程梳理

本文档梳理 Wiki.js 中 site config 读取、主题选择、页面渲染三者之间的传递关系。

---

## 一、配置读取与加载流程

配置采用**三级叠加**策略：磁盘默认值 → 磁盘用户配置 → 数据库配置，后者优先级更高。

### 1.1 启动入口

**文件**: `server/index.js:15-40`

```
WIKI 全局对象初始化
  └─> WIKI.configSvc.init()         // 第1步：从磁盘加载配置
  └─> WIKI.kernel.init()
        └─> WIKI.configSvc.loadFromDb()  // 第2步：从数据库加载配置（DB就绪后）
        └─> WIKI.configSvc.applyFlags()
```

### 1.2 磁盘配置加载

**文件**: `server/core/config.js:14-78`

| 步骤 | 说明 |
|------|------|
| 读取 `config.yml` | 路径优先顺序：`CONFIG_FILE` 环境变量 → dockerdev 特殊路径 → 项目根 `config.yml` |
| 读取 `data.yml` | `server/app/data.yml`，包含**所有默认配置值** |
| 环境变量替换 | `server/helpers/config.js:18-24` — 将 `$(ENV_VAR)` 或 `$(ENV_VAR:default)` 替换为实际环境变量值 |
| YAML 解析 | 使用 `js-yaml` 解析 |
| 合并默认值 | `_.defaultsDeep(appconfig, appdata.defaults.config)` — 用户配置覆盖默认值 |
| Docker Secret | 若设置 `DB_PASS_FILE`，从文件读取数据库密码 |
| 结果存入 | `WIKI.config`、`WIKI.data`、`WIKI.version` 等全局变量 |

**默认值定义**: `server/app/data.yml:6-115` — 包含 theming、auth、features、security 等所有配置项的默认值。

### 1.3 数据库配置加载

**文件**: `server/core/config.js:83-91`

| 步骤 | 说明 |
|------|------|
| 调用 | `WIKI.models.settings.getConfig()` |
| 查询表 | `settings` 表（key-value 结构，value 为 JSON） |
| 结构还原 | `server/models/settings.js:37-47` — 将扁平 key（如 `theming.theme`）还原为嵌套对象；处理 `{v: xxx}` 包装格式 |
| 合并策略 | `_.defaultsDeep(conf, WIKI.config)` — **DB 配置优先于磁盘配置** |
| 空库处理 | 若 DB 无配置，标记 `WIKI.config.setup = true`，进入安装向导模式 |

### 1.4 配置持久化

**文件**: `server/core/config.js:98-119`

- 调用 `WIKI.configSvc.saveToDb(keys, propagate=true)`
- 按 key 逐条 upsert 到 `settings` 表
- 对象类型值原样存储，非对象值包装为 `{v: value}`
- 保存成功后通过 `WIKI.events.outbound.emit('reloadConfig')` 广播 HA 集群同步

---

## 二、主题选择机制

### 2.1 主题配置结构

存储路径: `WIKI.config.theming`

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `theme` | String | `'default'` | 主题名称，对应 `client/themes/{name}/` 目录 |
| `iconset` | String | `'md'` | 图标集：`md` (Material Design Icons) / `fa` (Font Awesome 5) / `fa4` (Font Awesome 4) |
| `darkMode` | Boolean | `false` | 全局暗色模式开关（可被用户偏好覆盖） |
| `tocPosition` | String | `'left'` | 目录位置：`left` / `right` / `hidden` |
| `injectCSS` | String | `''` | 全局自定义 CSS（经 CleanCSS 压缩存储） |
| `injectHead` | String | `''` | 注入 `<head>` 的自定义 HTML |
| `injectBody` | String | `''` | 注入 `<body>` 末尾的自定义 HTML |

### 2.2 主题的文件结构

```
client/themes/{theme}/
  ├── theme.yml          # 主题元信息定义（名称、作者、可配置 props）
  ├── thumbnail.png      # 主题预览图
  ├── scss/
  │   └── app.scss       # 主题样式入口
  ├── js/
  │   └── app.js         # 主题 JS 入口
  └── components/
      ├── page.vue       # 页面布局主组件
      ├── nav-sidebar.vue# 侧边栏导航
      ├── nav-footer.vue # 页脚
      └── tabset.vue     # 选项卡组件

server/themes/{theme}/   # 服务端主题资源（目前仅预览图和 theme.yml）
  ├── theme.yml
  └── thumbnail.png
```

**theme.yml 定义示例**: `client/themes/default/theme.yml:1-50`
- 声明主题名称、作者、版本、兼容的 Wiki.js 版本范围
- `props` 字段定义主题级可配置项（如 sdPosition、showTOC、showTags 等），供管理员 UI 表单使用

### 2.3 主题配置的读写

**GraphQL Schema**: `server/graph/schemas/theming.graphql`

**Resolver**: `server/graph/resolvers/theming.js`

| 操作 | 说明 |
|------|------|
| `Query.theming.config` | 返回当前主题配置；CSS 经 CleanCSS 美化输出以便编辑 |
| `Query.theming.themes` | 返回可用主题列表（目前硬编码仅 default） |
| `Mutation.theming.setConfig` | 接收 theme/ iconset/ darkMode/ tocPosition/ inject* 参数；CSS 入库前压缩；保存到 DB |

### 2.4 用户级暗色模式覆盖

用户个人偏好可覆盖全局暗色模式设置：

**JWT Token 存储**: 用户登录时将 `appearance` 字段写入 JWT
- 取值: `'dark'` / `'light'` / `''`（空表示跟随全局）

**客户端判定逻辑**: `client/client-app.js:199-202`
```js
let darkModeEnabled = siteConfig.darkMode           // 先取全局设置
if ((store.get('user/appearance') || '').length > 0) {
  darkModeEnabled = (store.get('user/appearance') === 'dark')  // 用户设置优先
}
```

**用户状态来源**: `client/store/user.js:26-48` — 从 JWT Cookie 解码并提交 `REFRESH_AUTH` mutation

---

## 三、页面渲染传递链路

整个渲染链路分为 **4 个阶段**：服务端 Express → Pug 模板 → 客户端启动 → Vue 组件渲染。

### 3.1 阶段一：Express 中间件层 → res.locals

**文件**: `server/master.js:122-163`

#### 应用级 locals（所有请求共享）

```js
app.locals.siteConfig = {}              // 初始化占位
app.locals.config = WIKI.config         // 完整配置对象（模板内可用 config.xxx）
app.locals.analyticsCode = {}
app.locals.pageMeta = { title, description, ... }
app.locals.devMode = WIKI.devMode
app.locals.basedir = WIKI.ROOTPATH
```

#### 请求级中间件（每个请求设置）

```js
app.use(async (req, res, next) => {
  res.locals.siteConfig = {
    title: WIKI.config.title,
    theme: WIKI.config.theming.theme,          // 主题名
    darkMode: WIKI.config.theming.darkMode,    // 全局暗色模式
    tocPosition: WIKI.config.theming.tocPosition, // TOC 位置
    lang: WIKI.config.lang.code,
    rtl: WIKI.config.lang.rtl,
    company: WIKI.config.company,
    contentLicense: WIKI.config.contentLicense,
    footerOverride: WIKI.config.footerOverride,
    logoUrl: WIKI.config.logoUrl
  }
  // ... langs, analyticsCode
})
```

> ⚠️ 注意 `config` 是**完整配置**（所有字段），而 `siteConfig` 是**裁剪后暴露给前端的子集**。

### 3.2 阶段二：Controller → Pug 模板参数

**文件**: `server/controllers/common.js:417-582`

以页面视图路由 `router.get('/*', ...)` 为例：

#### 构建 injectCode（主题代码注入）

```js
// 全局注入代码
const injectCode = {
  css: WIKI.config.theming.injectCSS,
  head: WIKI.config.theming.injectHead,
  body: WIKI.config.theming.injectBody
}

// 合并页面级自定义 CSS/JS（存储在 page.extra）
page.extra = page.extra || { css: '', js: '' }
if (!_.isEmpty(page.extra.css))  injectCode.css  += `\n${page.extra.css}`
if (!_.isEmpty(page.extra.js))   injectCode.body += `\n${page.extra.js}`
```

#### 渲染 page.pug 模板

```js
res.render('page', {
  page,                // 页面数据对象（title, content, render, toc, tags, ...）
  sidebar,             // 侧边栏导航树
  injectCode,          // 主题代码注入 { css, head, body }
  comments: commentTmpl, // 评论系统模板
  effectivePermissions, // 用户权限
  pageFilename         // 页面文件名（用于"在外部仓库编辑"按钮）
})
```

编辑器页面 (`/e/*`)、历史页面 (`/h/*`) 等流程类似，都传递 `injectCode`。

### 3.3 阶段三：Pug 模板渲染链

#### master.pug — 基础布局

**文件**: `dev/templates/master.pug`

**核心动作① — 暴露 siteConfig 为 JS 全局变量**
```pug
script.
  var siteConfig = !{JSON.stringify(siteConfig)}
  var siteLangs = !{JSON.stringify(langs)}
```
这是客户端获取配置的**唯一桥梁**，后续所有 JS 都从此读取。

**核心动作② — 根据 iconset 加载图标字体 CSS**
```pug
if config.theming.iconset === 'fa'
  link(rel='stylesheet', href='https://use.fontawesome.com/releases/v5.10.0/css/all.css')
else if config.theming.iconset === 'fa4'
  link(rel='stylesheet', href='https://cdn.jsdelivr.net/npm/font-awesome@4.7.0/css/font-awesome.min.css')
// md (Material Design Icons) 由客户端 JS 动态加载
```

**核心动作③ — 注入 webpack 打包产物**
- 遍历 `htmlWebpackPlugin.files.css` 和 `htmlWebpackPlugin.files.js`
- 若 `config.security.securitySRI` 开启，则添加 `integrity` 和 `crossorigin` 属性

#### page.pug — 页面具体模板

**文件**: `server/views/page.pug:1-42`

```pug
extends master.pug

block head
  if injectCode.css
    style(type='text/css')!= injectCode.css        // 全局+页面自定义 CSS
  if injectCode.head
    != injectCode.head                              // 全局自定义 head HTML
  if config.features.featurePageComments
    != comments.head                                // 评论系统 head 代码

block body
  #root
    page(
      locale=page.localeCode
      path=page.path
      title=page.title
      description=page.description
      :tags=page.tags
      created-at=page.createdAt
      updated-at=page.updatedAt
      author-name=page.authorName
      :author-id=page.authorId
      editor=page.editorKey
      :is-published=page.isPublished.toString()
      toc=Buffer.from(page.toc).toString('base64')          // base64 编码传值
      :page-id=page.id
      sidebar=Buffer.from(JSON.stringify(sidebar)).toString('base64')
      nav-mode=config.nav.mode
      comments-enabled=config.features.featurePageComments
      effective-permissions=Buffer.from(JSON.stringify(effectivePermissions)).toString('base64')
      comments-external=comments.codeTemplate
      edit-shortcuts=Buffer.from(JSON.stringify(config.editShortcuts)).toString('base64')
      filename=pageFilename
    )
      template(slot='contents')
        div!= page.render                // 已渲染的 HTML 内容插槽
      template(slot='comments')
        div!= comments.main              // 评论主体 HTML 插槽

  if injectCode.body
    != injectCode.body                   // 全局+页面自定义 body 尾部 HTML
  if config.features.featurePageComments
    != comments.body
```

### 3.4 阶段四：客户端启动

#### index-app.js — 资源动态加载

**文件**: `client/index-app.js:1-26`

```
根据 siteConfig.lang 加载对应字体 (arabic.scss / default.scss)
  └─> require('./scss/app.scss')                          // 全局基础样式
  └─> import('./themes/' + siteConfig.theme + '/scss/app.scss')  // ★ 主题样式（动态）
  └─> import('@mdi/font/css/materialdesignicons.css')    // MDI 图标
  └─> require('./client-app.js')                          // Vue 应用初始化
  └─> import('./themes/' + siteConfig.theme + '/js/app.js')      // ★ 主题 JS（动态）
```

> 使用 webpack 动态 `import()`，主题资源单独打成 `theme` chunk。

#### client-app.js — Vue 应用初始化

**文件**: `client/client-app.js:1-239`

**核心动作① — 读取全局 siteConfig**
```js
moment.locale(siteConfig.lang)
store.commit('user/REFRESH_AUTH')   // 从 JWT Cookie 恢复用户信息（含 appearance）
```

**核心动作② — 动态注册主题组件**
```js
Vue.component('NavFooter', () => import('./themes/' + siteConfig.theme + '/components/nav-footer.vue'))
Vue.component('Page', () => import('./themes/' + siteConfig.theme + '/components/page.vue'))
```

**核心动作③ — Vuetify 暗色模式初始化**
```js
vuetify: new Vuetify({
  rtl: siteConfig.rtl,
  theme: {
    dark: darkModeEnabled   // 用户 appearance > 全局 darkMode
  }
})
```

### 3.5 阶段五：Vuex Store & 组件渲染

#### Vuex site 模块初始化

**文件**: `client/store/site.js:5-20`

```js
const state = {
  company: siteConfig.company,
  contentLicense: siteConfig.contentLicense,
  footerOverride: siteConfig.footerOverride,
  dark: siteConfig.darkMode,        // 供组件响应式读取
  tocPosition: siteConfig.tocPosition,  // 供组件响应式读取
  mascot: true,
  title: siteConfig.title,
  logoUrl: siteConfig.logoUrl,
  // ... 其他 UI 状态
}
```

#### 主题页面组件消费配置

**文件**: `client/themes/default/components/page.vue`

| 配置来源 | 读取方式 | 用途 |
|----------|----------|------|
| Props | `this.tocPosition / this.sidebar / this.toc` | 页面特定数据（从 Pug 传入的 base64 解码） |
| Vuex | `get('site/tocPosition')` (line 566) | 控制目录栏在左侧还是右侧，影响栅格 offset 和 order |
| Vuex | `sync('site/printView')` (line 577) | 打印视图开关 |
| Vuetify | `this.$vuetify.theme.dark` (template line 2 等) | 控制全站暗色样式 |
| Vuetify | `this.$vuetify.rtl` | 控制 RTL 布局（左右翻转） |
| Vuex | `get('page/editShortcuts')` | 控制编辑按钮显示 |

**布局影响示例**:
```pug
v-col.page-col-content.is-page-header(
  :offset-xl='tocPosition === `left` ? 2 : 0'
  :offset-lg='tocPosition === `left` ? 3 : 0'
  :xl='tocPosition === `right` ? 10 : false'
  :lg='tocPosition === `right` ? 9 : false'
)
```

---

## 四、完整流程示意图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SERVER SIDE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  server/app/data.yml (defaults)                                     │
│          ↓                                                          │
│  config.yml (user overrides)  ──► WIKI.configSvc.init()             │
│          ↓                                                          │
│  DB settings table            ──► WIKI.configSvc.loadFromDb()       │
│          ↓                                                          │
│  WIKI.config { theming, title, auth, features, ... }                │
│          ↓                                                          │
│  server/master.js                                                   │
│    ├─ app.locals.config = WIKI.config          (完整配置)           │
│    └─ res.locals.siteConfig = { ... }         (前端子集)            │
│          ↓                                                          │
│  server/controllers/common.js                                       │
│    └─ res.render('page', { page, injectCode, sidebar, ... })        │
│          ↓                                                          │
│  dev/templates/master.pug                                           │
│    ├─ <script> var siteConfig = JSON.stringify(...) </script>       │
│    └─ 根据 iconset 加载 CDN CSS                                     │
│          ↓                                                          │
│  server/views/page.pug                                              │
│    ├─ injectCode.css → <style>                                      │
│    ├─ injectCode.head → <head> 内                                   │
│    ├─ <page :tocPosition :sidebar ...>  (Vue 组件 props)            │
│    └─ injectCode.body → </body> 前                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ HTTP 响应
                              │
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT SIDE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  <script> var siteConfig = {...}; var siteLangs = [...]; </script>  │
│          ↓                                                          │
│  client/index-app.js                                                │
│    ├─ 动态 import themes/{siteConfig.theme}/scss/app.scss           │
│    └─ 动态 import themes/{siteConfig.theme}/js/app.js               │
│          ↓                                                          │
│  client/client-app.js                                               │
│    ├─ store.commit('user/REFRESH_AUTH')  (JWT → user.appearance)    │
│    ├─ Vue.component('Page', () => import(themes/{theme}/page.vue))  │
│    └─ new Vuetify({ theme: { dark: user.appearance || siteConfig    │
│                                          .darkMode } })             │
│          ↓                                                          │
│  client/store/site.js                                               │
│    └─ state = { dark: siteConfig.darkMode, tocPosition, ... }       │
│          ↓                                                          │
│  client/themes/default/components/page.vue                          │
│    ├─ Props: page 数据 (from Pug)                                   │
│    ├─ Vuex: tocPosition → 控制 TOC 左/右布局                        │
│    ├─ Vuetify: $vuetify.theme.dark → 暗色模式样式                   │
│    └─ Vuetify: $vuetify.rtl → RTL 布局                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键文件索引

| 层级 | 文件 | 职责 |
|------|------|------|
| 配置核心 | `server/core/config.js` | 磁盘/DB 配置加载与持久化 |
| 配置辅助 | `server/helpers/config.js` | 环境变量替换 |
| 配置数据模型 | `server/models/settings.js` | settings 表读写 |
| 默认配置 | `server/app/data.yml` | 所有配置项默认值 |
| Express 初始化 | `server/master.js` | locals 设置、中间件、路由 |
| 页面控制器 | `server/controllers/common.js` | 构建 injectCode、调用 res.render |
| 主题 GraphQL | `server/graph/resolvers/theming.js` | 主题配置读写 API |
| 基础模板 | `dev/templates/master.pug` | 注入 siteConfig JS 变量、图标 CSS |
| 页面模板 | `server/views/page.pug` | 注入主题代码、挂载 Vue 组件 |
| 客户端入口 | `client/index-app.js` | 动态加载主题 SCSS/JS |
| Vue 初始化 | `client/client-app.js` | 动态注册主题组件、Vuetify 暗色/RTL |
| Site Store | `client/store/site.js` | 前端配置状态（dark、tocPosition 等） |
| User Store | `client/store/user.js` | 用户偏好（appearance 暗色模式） |
| 主题页面组件 | `client/themes/default/components/page.vue` | 消费 tocPosition、dark、rtl 进行布局 |
