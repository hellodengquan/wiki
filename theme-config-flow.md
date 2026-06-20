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

## 五、站点级配置与页面级配置的覆盖顺序

Wiki.js 的配置覆盖并非"站点 → 空间(locale)"的层级覆盖模型，而是**站点级全局配置 + 页面级注入叠加**模型。系统没有"空间级"独立配置层，locale 只影响语言和导航，不影响主题和外观。

### 5.1 三级配置叠加：默认值 → 磁盘 → 数据库

```
优先级：低 ──────────────────────────────────────────────── 高

  data.yml (defaults)     config.yml (磁盘)     settings 表 (DB)
       │                      │                      │
       └── defaultsDeep ──────┘                      │
                    │                                │
              WIKI.config (init 后)                   │
                    │                                │
                    └──── defaultsDeep ──────────────┘
                              │
                     WIKI.config (最终)
```

**关键代码**: `server/core/config.js`

- 第 53 行: `appconfig = _.defaultsDeep(appconfig, appdata.defaults.config)` — 磁盘配置覆盖默认值
- 第 86 行: `WIKI.config = _.defaultsDeep(conf, WIKI.config)` — DB 配置覆盖磁盘+默认值

**`_.defaultsDeep` 的语义**: 源对象（第一个参数）中已有的属性不会被目标对象覆盖；只补全源对象中缺失的属性。因此 DB 中已存储的值会"赢"，DB 中没有的 key 回退到磁盘配置或默认值。

### 5.2 页面级注入叠加在站点级之上

站点级配置（`WIKI.config.theming.injectCSS/Head/Body`）提供全局主题注入。页面自身还存储了 `page.extra.css` 和 `page.extra.js`，在渲染时**追加到**站点级注入之后。

**代码路径**: `server/controllers/common.js:492-507`

```js
const injectCode = {
  css: WIKI.config.theming.injectCSS,   // 第1层：站点全局 CSS
  head: WIKI.config.theming.injectHead, // 第1层：站点全局 head
  body: WIKI.config.theming.injectBody  // 第1层：站点全局 body
}

page.extra = page.extra || { css: '', js: '' }

if (!_.isEmpty(page.extra.css)) {
  injectCode.css = `${injectCode.css}\n${page.extra.css}`  // 第2层：页面 CSS 追加
}
if (!_.isEmpty(page.extra.js)) {
  injectCode.body = `${injectCode.body}\n${page.extra.js}` // 第2层：页面 JS 追加
}
```

**叠加规则**:
- CSS: 站点全局 → 页面自定义，**后者追加在后面**，CSS 层叠规则下页面级优先级更高
- Body JS: 站点全局 → 页面自定义，**后者追加在后面**，页面 JS 后执行
- 页面级**没有** `injectHead` 的独立注入点（仅站点级有）

**页面 extra 字段的存储**: `server/models/pages.js:57-58` — `jsonAttributes: ['extra']`，Objection.js 自动序列化为 JSON 存入 `pages.extra` 列。

**页面 extra 的写入**:
- 创建页面: `server/models/pages.js:315-318` — 写入 `{ js: scriptJs, css: scriptCss }`
- 更新页面: `server/models/pages.js:434-438` — 合并 `ogPage.extra` 后覆盖
- 权限守卫: `write:styles` 权限控制 CSS 写入，`write:scripts` 权限控制 JS 写入（line 278-296）

### 5.3 locale 不构成独立配置层

虽然 `WIKI.config.lang` 支持多语言 namespacing，但 locale 仅影响：
- 请求路由中的语言前缀 (`/en/path`)
- `req.i18n.changeLanguage()` 切换 i18n 语言
- `siteConfig.lang` 和 `siteConfig.rtl` 的值

locale **不**影响主题、darkMode、tocPosition、injectCode 等外观配置。所有 locale 共享同一套 `WIKI.config.theming`。

### 5.4 用户级偏好的覆盖范围

用户偏好**仅覆盖暗色模式**，不影响其他主题配置：

```
站点级 darkMode (WIKI.config.theming.darkMode)
       │
       └── 被用户 appearance (JWT) 条件性覆盖
            client/client-app.js:199-202
```

覆盖逻辑（非叠加，而是替换）：
```js
let darkModeEnabled = siteConfig.darkMode          // 默认取全局
if ((store.get('user/appearance') || '').length > 0) {
  darkModeEnabled = (store.get('user/appearance') === 'dark')  // 用户设置了就替换
}
```

### 5.5 完整覆盖层级总结

```
┌──────────────────────────────────────────────────────┐
│  优先级从低到高                                        │
│                                                      │
│  1. data.yml 默认值                                   │
│       ↓ defaultsDeep                                 │
│  2. config.yml 磁盘用户配置                            │
│       ↓ defaultsDeep                                 │
│  3. settings 数据库配置                                │
│       ↓                                              │
│  4. page.extra 页面级 CSS/JS 注入 (追加叠加)           │
│       ↓                                              │
│  5. user.appearance 用户暗色模式偏好 (条件替换)         │
│                                                      │
│  注意：不存在 locale 级配置层                           │
└──────────────────────────────────────────────────────┘
```

---

## 六、主题资源的预加载与懒加载策略

Wiki.js 的主题资源加载是**构建时确定、运行时懒加载**的模式。当前系统仅有一个 `default` 主题，主题切换需要重新构建或重启服务，**不存在运行时动态切换主题**的能力。

### 6.1 构建时：主题名通过 DefinePlugin 注入

**文件**: `dev/webpack/webpack.prod.js:244-247`

```js
new webpack.DefinePlugin({
  'process.env.NODE_ENV': JSON.stringify('production'),
  'process.env.CURRENT_THEME': JSON.stringify(_.defaultTo(yargs.theme, 'default'))
})
```

- 构建时可通过 `--theme=xxx` 参数指定主题名
- 默认 `'default'`
- `CURRENT_THEME` 可在代码中引用，但实际上客户端代码使用的是 `siteConfig.theme`（运行时变量），而非此构建常量

### 6.2 运行时：siteConfig.theme 决定资源路径

所有主题资源加载都基于 `siteConfig.theme` 变量，该值由服务端在每个请求的中间件中从 `WIKI.config.theming.theme` 读取，通过 `master.pug` 注入为 `window.siteConfig.theme`。

### 6.3 懒加载策略详解

#### 类型 A — webpack 懒加载 chunk（无预取）

**文件**: `client/index-app.js:20,26`

```js
import(/* webpackChunkName: "theme" */ './themes/' + siteConfig.theme + '/scss/app.scss')
import(/* webpackChunkName: "theme" */ './themes/' + siteConfig.theme + '/js/app.js')
```

- `webpackChunkName: "theme"` 将两个 import 打入**同一个** chunk
- **没有** `webpackPrefetch` 或 `webpackPreload` 指令
- 加载时机：`index-app.js` 执行时立即触发 import，但由于是异步 chunk，不会阻塞主 bundle 执行
- 实际效果：主题 SCSS/JS 作为独立 chunk 在首屏并行加载，但**不预取**

#### 类型 B — Vue 异步组件（无预取）

**文件**: `client/client-app.js:177-178`

```js
Vue.component('NavFooter', () => import(/* webpackChunkName: "theme" */ './themes/' + siteConfig.theme + '/components/nav-footer.vue'))
Vue.component('Page', () => import(/* webpackChunkName: "theme" */ './themes/' + siteConfig.theme + '/components/page.vue'))
```

- 同样 `webpackChunkName: "theme"`，与类型 A 合并到同一 chunk
- **没有** `webpackPrefetch`
- 加载时机：Vue 首次渲染 `<page>` / `<nav-footer>` 组件时触发
- 由于 `page.vue` 是首屏必需组件，实际触发时间约等于 DOM Ready

#### 类型 C — 带预取的组件（对比）

```js
Vue.component('Loader', () => import(/* webpackPrefetch: true, webpackChunkName: "ui-extra" */ './components/common/loader.vue'))
Vue.component('Login', () => import(/* webpackPrefetch: true, webpackChunkName: "login" */ './components/login.vue'))
```

- 这些**非主题**组件使用了 `webpackPrefetch: true`
- 浏览器空闲时会预取这些 chunk
- 主题组件**没有**使用 `webpackPrefetch`，因为它们是首屏同步需要的，不需要预取——它们在 chunk 加载后立即使用

### 6.4 CSS 分层加载顺序

```
1. <link> webpack 主 CSS bundle (含 Vuetify 基础样式 + app.scss 全局样式)
2. <link> 图标字体 CSS (根据 iconset，服务端 Pug 条件加载 CDN 或 webpack chunk)
3. <style> injectCode.css (站点全局 + 页面自定义 CSS，内联在 HTML 中)
4. <link> theme chunk CSS (主题 SCSS 编译产物，异步 chunk)
```

- 层 1 和 2 由 `master.pug` 的 `<head>` 直接输出（阻塞渲染）
- 层 3 由 `page.pug` 的 `block head` 内联输出（阻塞渲染）
- 层 4 由 webpack 异步 chunk 加载（非阻塞，但可能造成 FOUC）

### 6.5 JS 加载顺序

```
1. <script async> webpack runtime.js (同步加载，ScriptExtHtmlWebpackPlugin 配置)
2. <script async> vendor.js (公共依赖)
3. <script async> app.js (主 bundle，含 client-app.js)
4. <script async> theme.js chunk (主题 SCSS + JS + Vue 组件)
```

**文件**: `dev/webpack/webpack.prod.js:232-235`

```js
new ScriptExtHtmlWebpackPlugin({
  sync: 'runtime.js',
  defaultAttribute: 'async'
})
```

所有脚本默认 `async` 加载，仅 `runtime.js` 同步。

### 6.6 主题切换的局限性

当前架构下，主题名在**构建时**通过 webpack 动态 `import()` 硬编码路径：

```js
import('./themes/' + siteConfig.theme + '/scss/app.scss')
```

webpack 在构建时只能分析出**静态路径**下的模块。`siteConfig.theme` 是运行时变量，webpack 的处理方式是：
- 将 `./themes/` 下所有子目录中匹配 `scss/app.scss` 的模块都打入 chunk
- 实际运行时只加载 `siteConfig.theme` 对应的那一个

但由于只有一个 `default` 主题目录存在，实际效果等价于静态引入。**如果新增主题目录，需要重新构建**前端资源，新的 SCSS/JS/Vue 才会进入 theme chunk。

**管理后台切换主题后** (`admin-theme.vue:206-238`):
- 仅修改了 `WIKI.config.theming.theme` 并存入 DB
- **不会**触发热更新或前端重载
- 用户需要**刷新页面**才能加载新主题的资源
- 如果新主题目录不存在于构建产物中，刷新后会报 chunk 加载失败

---

## 七、配置校验失败时的回退路径与默认值机制

Wiki.js **没有统一的配置校验层**。校验分散在三个层面，回退策略各不相同。

### 7.1 第一层：启动时磁盘配置加载 — 硬性校验，失败则终止

**文件**: `server/core/config.js:34-49`

```js
try {
  appconfig = yaml.safeLoad(cfgHelper.parseConfigValue(fs.readFileSync(confPaths.config, 'utf8')))
  appdata = yaml.safeLoad(fs.readFileSync(confPaths.data, 'utf8'))
  console.info(chalk.green.bold(`OK`))
} catch (err) {
  console.error(chalk.red.bold(`FAILED`))
  console.error(err.message)
  console.error(chalk.red.bold(`>>> Unable to read configuration file! Did you create the config.yml file?`))
  process.exit(1)   // ← 直接退出进程
}
```

**回退策略**: **无回退**。config.yml 读取或解析失败直接 `process.exit(1)` 终止进程。不尝试降级运行。

同样，`DB_PASS_FILE` 读取失败也是 `process.exit(1)` (`server/core/config.js:62-71`)。

### 7.2 第二层：数据库配置加载 — 缺失则进入 Setup 模式

**文件**: `server/core/config.js:83-91`

```js
async loadFromDb() {
  let conf = await WIKI.models.settings.getConfig()
  if (conf) {
    WIKI.config = _.defaultsDeep(conf, WIKI.config)  // DB 有配置则合并
  } else {
    WIKI.logger.warn('DB Configuration is empty or incomplete. Switching to Setup mode...')
    WIKI.config.setup = true   // ← 标记进入安装向导
  }
}
```

**回退策略**: DB 无配置时不崩溃，而是设置 `setup = true` 进入安装向导。安装向导是一个独立的 Express 应用 (`server/setup.js`)，只提供 setup 页面，不加载正常功能。

**安装向导完成后的配置初始化** (`server/setup.js:76-193`):
- 硬编码一套初始默认值（如 `theming: { theme: 'default', darkMode: false, iconset: 'mdi' }`）
- 调用 `saveToDb([...keys], false)` 写入 DB（`propagate=false` 因为此时只有单实例）
- 安装完成后 `WIKI.config.setup = false`，重启进入正常模式 (`server/setup.js:360`)

**安装失败回退** (`server/setup.js:369-375`):
```js
catch (err) {
  try {
    await WIKI.models.knex('settings').truncate()  // ← 清空 settings 表
  } catch (err) {}
  WIKI.telemetry.sendError(err)
  res.json({ ok: false, error: err.message })     // ← 返回错误，用户可重试
}
```

安装失败时清空 settings 表，下次重启会再次进入 Setup 模式。

### 7.3 第三层：运行时字段级软回退 — `||` 和 `_.get` 模式

运行时配置读取时，个别字段可能缺失（如旧版本 DB 中没有新字段），系统使用两种模式软回退：

#### 模式 A — `||` 运算符回退

**文件**: `server/graph/resolvers/theming.js:27,48-50`

```js
// 读取时
tocPosition: WIKI.config.theming.tocPosition || 'left',   // 缺失则回退 'left'

// 写入时
tocPosition: args.tocPosition || 'left',     // 参数缺失则回退 'left'
injectCSS: args.injectCSS || '',              // 参数缺失则回退空字符串
injectBody: args.injectBody || '',
```

**文件**: `server/master.js:152`

```js
tocPosition: WIKI.config.theming.tocPosition || 'left',   // 同样的回退
```

#### 模式 B — `_.get(args, key, WIKI.config.xxx)` 保留旧值回退

**文件**: `server/graph/resolvers/site.js:77-125`

```js
WIKI.config.seo = {
  description: _.get(args, 'description', WIKI.config.seo.description),
  robots: _.get(args, 'robots', WIKI.config.seo.robots),
  // ...
}
```

当 mutation 请求中某个字段未提供时，`_.get` 回退到当前 `WIKI.config` 中的值，**保留旧值不变**。这是部分更新（partial update）的标准模式。

### 7.4 GraphQL 层校验：仅权限校验，无值校验

**文件**: `server/graph/schemas/theming.graphql:27-36`

```graphql
type ThemingMutation {
  setConfig(
    theme: String!       # 非空约束
    iconset: String!     # 非空约束
    darkMode: Boolean!   # 非空约束
    tocPosition: String  # 可选
    injectCSS: String    # 可选
    injectHead: String   # 可选
    injectBody: String   # 可选
  ): DefaultResponse @auth(requires: ["manage:theme", "manage:system"])
}
```

- **类型校验**: GraphQL schema 保证 `theme`/`iconset`/`darkMode` 必须提供且类型正确
- **权限校验**: `@auth` directive (`server/graph/directives/auth.js:5-54`) 检查用户是否有 `manage:theme` 或 `manage:system` 权限
- **无值域校验**: 没有校验 `theme` 值是否对应一个存在的主题目录，没有校验 `tocPosition` 是否为 `'left'/'right'/'off'`，没有校验 `iconset` 是否为合法值

如果管理员提交 `theme: 'nonexistent'`，系统会照常存入 DB，下次刷新页面时客户端动态 import 会失败（chunk 404）。

### 7.5 CSS 校验：CleanCSS 压缩容错

**文件**: `server/graph/resolvers/theming.js:37-41`

```js
if (!_.isEmpty(args.injectCSS)) {
  args.injectCSS = new CleanCSS({ inline: false }).minify(args.injectCSS).styles
}
```

- CleanCSS 压缩时对无效 CSS **不会抛出错误**，而是尽量输出压缩结果
- 无效的 CSS 规则会被静默丢弃
- 写入时压缩，读取时美化（`new CleanCSS({ format: 'beautify' })`），便于编辑

### 7.6 客户端回退：darkMode 的两段式判定

**文件**: `client/client-app.js:199-202`

```js
let darkModeEnabled = siteConfig.darkMode
if ((store.get('user/appearance') || '').length > 0) {
  darkModeEnabled = (store.get('user/appearance') === 'dark')
}
```

- `siteConfig.darkMode` 如果为 `undefined`（理论上不应发生，因有默认值 `false`），JS 中 `undefined` 为 falsy，等价于 `false`
- `user.appearance` 如果 JWT 中不存在，`jwt.decode` 返回 `undefined`，`|| ''` 保护后长度为 0，跳过用户覆盖

### 7.7 管理后台暗色模式预览的回退

**文件**: `client/components/admin/admin-theme.vue:194-204`

```js
watch: {
  'darkMode' (newValue, oldValue) {
    this.$vuetify.theme.dark = newValue       // 实时预览暗色模式
  }
},
beforeDestroy() {
  this.darkMode = this.darkModeInitial         // ← 退出页面时恢复原始值
  this.$vuetify.theme.dark = this.darkModeInitial
}
```

- 管理员在主题设置页面切换暗色模式开关时**实时预览**效果
- 如果管理员**未保存就离开页面**，`beforeDestroy` 钩子将 darkMode 恢复到进入页面时的初始值
- 如果管理员**保存了**，`save()` 方法中 `this.darkModeInitial = this.darkMode` 更新初始值，离开时不会回退

### 7.8 配置回退机制总结

| 场景 | 校验位置 | 失败行为 | 回退策略 |
|------|----------|----------|----------|
| config.yml 读取/解析失败 | `config.js:34-49` | `process.exit(1)` | **无回退，终止进程** |
| DB_PASS_FILE 读取失败 | `config.js:62-71` | `process.exit(1)` | **无回退，终止进程** |
| settings 表为空 | `config.js:88-89` | 设置 `setup=true` | **进入安装向导** |
| 安装向导失败 | `setup.js:369-374` | 清空 settings 表 | **下次重启重新安装** |
| 单字段缺失（读取时） | `||` 运算符 | 使用硬编码默认值 | `tocPosition \|\| 'left'` |
| 单字段缺失（更新时） | `_.get(args, key, 旧值)` | 保留当前配置值 | 部分更新不覆盖 |
| GraphQL 类型不符 | Schema 类型系统 | 请求被拒绝 | 前端收到错误提示 |
| GraphQL 权限不足 | `@auth` directive | 抛出 Forbidden | 前端收到 403 |
| theme 值不对应已有目录 | **无校验** | 存入 DB | 刷新后 chunk 404 |
| injectCSS 内容无效 | CleanCSS 压缩 | 静默丢弃无效规则 | 输出压缩后的有效部分 |

---

## 八、主题上传与磁盘扫描机制

### 8.1 当前状态：主题上传与扫描功能未实现

Wiki.js 目前**没有实现**自定义主题的上传、安装、磁盘扫描机制。以下是代码中的证据：

#### 证据一：管理后台 UI 被注释掉

**文件**: `client/components/admin/admin-theme.vue:69-93`

```pug
//- v-card.animated.fadeInUp.wait-p2s
//-   v-toolbar(color='teal', dark, dense, flat)
//-     v-toolbar-title.subtitle-1 {{$t('admin:theme.downloadThemes')}}
//-     v-spacer
//-     v-chip(label, color='white', small).teal--text coming soon
//-   v-data-table(
//-     :headers='headers',
//-     :items='themes',
//-     hide-default-footer,
//-     item-key='value',
//-     :items-per-page='1000'
//-   )
```

整个"下载/安装主题"卡片被 `//-` Pug 注释，标记为 **coming soon**。前端 UI 预留了 `isDownloading` / `isInstalled` / `installDate` / `updatedAt` 等字段，但无实际逻辑。

#### 证据二：前端主题列表硬编码

**文件**: `client/components/admin/admin-theme.vue:142-144`

```js
themes: [
  { text: 'Default', author: 'requarks.io', value: 'default', isInstalled: true, installDate: '', updatedAt: '' }
]
```

主题列表不是动态从后端读取，而是写死一个 default 条目。

#### 证据三：后端 `theming.themes` resolver 硬编码

**文件**: `server/graph/resolvers/theming.js:30-33`

```js
themes() {
  return [
    { author: 'requarks.io', isCompatible: true, isInstalled: true, name: 'Default', title: 'Default', userInput: '' }
  ]
}
```

`Query.theming.themes` 直接返回写死的单元素数组，不扫描磁盘，不读 DB。

#### 证据四：没有 `refreshThemesFromDisk` 对应的模型文件

`server/core/kernel.js:71-87` 的 `postBootMaster()` 中，所有可插拔模块都有 `refreshXxxFromDisk()` 调用：

```
await WIKI.models.analytics.refreshProvidersFromDisk()
await WIKI.models.authentication.refreshStrategiesFromDisk()
await WIKI.models.commentProviders.refreshProvidersFromDisk()
await WIKI.models.editors.refreshEditorsFromDisk()        ← editors 模型有
await WIKI.models.loggers.refreshLoggersFromDisk()
await WIKI.models.renderers.refreshRenderersFromDisk()
await WIKI.models.searchEngines.refreshSearchEnginesFromDisk()
await WIKI.models.storage.refreshTargetsFromDisk()
                                                    ← 没有 WIKI.models.themes.refreshThemesFromDisk()
```

editors 模块的 `refreshEditorsFromDisk()`（`server/models/editors.js:37-95`）展示了标准扫描模式，但主题**没有对应的模型**和对应的调用。

#### 证据五：sideloader 也不处理主题

**文件**: `server/core/sideloader.js:1-78`

离线 sideload 机制只处理 `importLocales()`，不处理主题包导入。sideload 目录结构中没有 themes 相关的扫描逻辑。

### 8.2 标准模块的扫描模式（供主题参考）

以 editors 模块为参照，标准 `refreshFromDisk` 流程如下：

**文件**: `server/models/editors.js:37-95`

```
1. 从 DB 查询已注册的 editors
2. fs.readdir(server/modules/editor/) 读取磁盘目录
3. 逐个读 definition.yml，yaml.safeLoad 解析
4. 存入 WIKI.data.editors（内存缓存）
5. 对比 DB 与磁盘：
   - DB 中没有 → 加入 newEditors 列表
   - DB 中已有 → patch config 补充新增 prop 的默认值
6. 启动 Objection transaction
7. 批量 insert newEditors
8. commit / rollback
```

标准模块的目录结构（对照主题如果实现应类似）：
```
server/modules/editor/{editor-key}/definition.yml   ← 模块元数据+props定义
server/modules/editor/{editor-key}/editor.js        ← 模块实现代码
client/themes/{theme-name}/theme.yml                ← 目前只有 theme.yml，无 definition.yml
server/themes/{theme-name}/theme.yml                ← 服务端也只有 theme.yml，无 definition.yml
```

### 8.3 假设主题上传功能实现时应有的代码路径

根据 coming soon UI 和现有模式，推测应包含：

1. **服务端扫描模型**：新建 `server/models/themes.js`，实现 `refreshThemesFromDisk()`
2. **上传 API**：GraphQL Mutation 接收 tarball/zip，校验 theme.yml，解压到 `server/themes/` 和 `client/themes/`
3. **Resolver**：`Query.theming.themes` 改为调用模型动态返回
4. **前端**：解除 admin-theme.vue 注释，调用 download/install mutation
5. **构建触发**：上传成功后需触发 webpack 重建客户端（因为 client 端主题代码在构建时被打包），这是架构上的难题——当前主题名是构建时通过 `import('./themes/' + siteConfig.theme + '/...')` 固定路径打入 chunk 的

### 8.4 主题上传/扫描相关的结论

| 功能 | 状态 | 代码位置 |
|------|------|----------|
| 主题列表从磁盘扫描 | **未实现** | resolver 硬编码 `server/graph/resolvers/theming.js:30-33` |
| 主题上传/安装 | **未实现** | UI 注释 `client/components/admin/admin-theme.vue:69-93` |
| 服务端主题模型 | **不存在** | 没有 `server/models/themes.js` 文件 |
| Sideload 离线导入主题 | **未实现** | `server/core/sideloader.js` 只处理 locales |
| 构建时动态打包主题 | **部分支持** | webpack 的动态 import 会扫描 `client/themes/*/` 下所有匹配文件，只要目录存在就会打包 |

---

## 九、主题变量覆盖的优先级与生效顺序

Wiki.js 不采用 CSS 自定义属性（CSS Variables）体系，主题变量是三层"硬编码"机制。**不存在 `entity` 级的主题变量覆盖**（entity 在此语境中没有实现），实际的样式覆盖链如下：

### 9.1 第一层：SCSS 全局变量（构建时）

**文件**: `client/scss/global.scss:6-32`

```scss
$tablet: 769px !default;
$desktop: 980px !default;
$widescreen: 1180px !default;

$grid-breakpoints: (
  'xs': 0,
  'sm': 600px,
  'md': 960px,
  'lg': 1280px - 16px,
  'xl': 1920px - 16px
) !default;

$display-breakpoints: (...) !default;
```

- 使用 `!default` 声明，允许被更早加载的同名变量覆盖
- 通过 `sass-resources-loader`（`dev/webpack/webpack.prod.js:124-128`）注入到**每个** `.scss` 文件中，主题 SCSS 可以直接使用
- Vuetify 的 material color 函数 `mc('grey', '800')` 也来自此基础变量层（`base/material`、`base/mixins` import）

### 9.2 第二层：Vuetify 主题系统（运行时，JS 驱动）

**文件**: `client/client-app.js:211-216`

```js
vuetify: new Vuetify({
  rtl: siteConfig.rtl,
  theme: {
    dark: darkModeEnabled   // 仅此一项，不自定义 primary/secondary 等色板
  }
})
```

Wiki.js **没有自定义 Vuetify 的色板**（primary、secondary、accent、error、info、success、warning），完全使用 Vuetify 默认配色。主题开发者只能通过覆写 CSS 类名来改变颜色。

运行时变更方式：
- 组件内通过 `this.$vuetify.theme.dark = newValue` 切换（Vue 响应式）
- 例如 `admin-theme.vue:195` 在切换开关时实时预览
- 例如 `profile/profile.vue:690-692` 用户保存 appearance 后即时切换

组件内消费方式：
- **模板内直接取值**: `v-app(:dark='$vuetify.theme.dark')`（page.vue:2）
- **三元表达式驱动 class**: `:class='$vuetify.theme.dark ? `grey darken-4-d4` : `primary`'`（page.vue:6）
- **三元表达式驱动 icon**: `{{ $vuetify.rtl ? `mdi-chevron-left` : `mdi-chevron-right` }}`（page.vue:100）
- **Script 中取**: `if (this.$vuetify.theme.dark) { ... }`（page.vue:609）

### 9.3 第三层：主题 SCSS（运行时，异步 chunk 加载）

**文件**: `client/themes/default/scss/app.scss:1-1300`

主题 SCSS 处理三类覆盖：
- **Vuetify 通用组件样式覆写**：直接写 CSS 选择器覆盖 Vuetify class
- **`.contents` 内容区域样式**：Markdown 渲染后的标题、链接、blockquote、表格等
- **暗色模式适配**：大量使用 `@at-root .theme--dark &` 选择器，当 `<body>` 上存在 `.theme--dark` class 时生效

典型的暗色模式 SCSS 写法：
```scss
blockquote.is-info {
  background-color: mc('blue', '50');
  @at-root .theme--dark & {
    background-color: mc('blue', '900');  // 暗色模式下覆盖
  }
}
```

典型的 RTL 适配：
```scss
@at-root .is-rtl & {
  margin-left: 0;
  margin-right: ($n / 12 * 100) * 1%;
}
```

### 9.4 第四层：全局 injectCSS + 页面级 CSS（运行时，HTML 内联）

**文件**: `server/views/page.pug:4-5`

```pug
block head
  if injectCode.css
    style(type='text/css')!= injectCode.css
```

`injectCode.css = 站点 injectCSS + 页面 page.extra.css`，内联在 HTML 的 `<head>` 中，以 `<style>` 块输出。

**优先级分析**（结合 CSS 源顺序与特异性）：

```
特异性相同的情况下，后加载的优先级更高：

1. Vuetify 基础样式 + global.scss（主 CSS bundle，head 中 <link>）
       ↓ 后加载
2. 主题 SCSS 样式（theme chunk CSS，异步加载）
       ↓ 后加载 + 位置在 head 末尾（最高优先级的 <link>）
3. injectCode.css（HTML 内联 <style>，在最后）
```

injectCode.css 由于内联在 HTML，且在 theme chunk 之后，特异性相同时**最高优先级**。

页面级 `page.extra.css` 追加在站点 `injectCSS` 之后，所以**页面级 > 站点级**（CSS 层叠顺序规则）。

### 9.5 完整优先级链总结

```
构建时（不可动态改变）                    运行时（可动态改变）
─────────────────────────                 ─────────────────────────

SCSS !default 变量
    ↓ 低
Vuetify 暗色模式（$vuetify.theme.dark）
    ↓
主题 SCSS 覆写（theme chunk CSS）         ← 可通过切换主题替换
    ↓
站点级 injectCSS（settings.theming）       ← 管理后台随时改
    ↓
页面级 page.extra.css（pages.extra）      ← 编辑页面时改
    ↓ 高
```

### 9.6 关于 "entity 级覆盖" 的说明

代码中**不存在**"entity（实体）级主题变量覆盖"这一层级，原因：
- Wiki.js 不使用 CSS custom properties（`--theme-primary: xxx`）运行时变量体系
- Vuetify 的色板（primary/secondary 等）没有被自定义或暴露给管理员
- theme.yml 中的 `props`（sdPosition/showTOC/showTags 等）只是元数据声明，**从未被代码读取和使用**（搜索整个代码库找不到消费 `theme.yml props` 的逻辑）

theme.yml `props` 的现状：定义了 sdPosition、showTOC 等 7 个可配置项，但它们只影响管理后台表单的生成，实际渲染逻辑没有从这些 props 取值——布局用的是硬编码的 Vuex getter `site/tocPosition` 和 template 中的 `v-if`。如果要真正实现主题级 props，需要在渲染组件时读取 `theming.themeProps[siteConfig.theme]` 并作为参数传入。

---

## 十、配置写回数据库的事务与并发处理

### 10.1 saveToDb 核心实现

**文件**: `server/core/config.js:98-119`

```js
async saveToDb(keys, propagate = true) {
  try {
    for (let key of keys) {
      let value = _.get(WIKI.config, key, null)
      if (!_.isPlainObject(value)) {
        value = { v: value }
      }
      let affectedRows = await WIKI.models.settings.query().patch({ value }).where('key', key)
      if (affectedRows === 0 && value) {
        await WIKI.models.settings.query().insert({ key, value })
      }
    }
    if (propagate) {
      WIKI.events.outbound.emit('reloadConfig')
    }
  } catch (err) {
    WIKI.logger.error(`Failed to save configuration to DB: ${err.message}`)
    return false
  }
  return true
}
```

### 10.2 事务分析：**没有事务保护**

关键发现：`saveToDb` **不使用数据库事务**。

与 editors 模块对比：

| 模块 | 是否有事务 | 代码位置 |
|------|-----------|----------|
| settings (saveToDb) | **❌ 无事务** | `server/core/config.js:98-119` |
| editors refresh | ✅ 有事务 | `server/models/editors.js:79` `trx = await Objection.transaction.start(...)` |
| storage refresh | ✅ 有事务 | `server/models/storage.js:88` |
| searchEngines refresh | ✅ 有事务 | `server/models/searchEngines.js:80` |
| renderers refresh | ✅ 有事务 | `server/models/renderers.js:84` |
| loggers refresh | ✅ 有事务 | `server/models/loggers.js:81` |

editors 事务模式示例：
```js
trx = await WIKI.models.Objection.transaction.start(WIKI.models.knex)
for (let editor of newEditors) {
  await WIKI.models.editors.query(trx).insert(editor)  // ← 传 trx
}
await trx.commit()
// catch 中 trx.rollback()
```

`saveToDb` 的 `patch` 和 `insert` 每个 key 都是独立语句，如果中途出错（如第 3 个 key 写入失败）：
- 前 2 个 key 已提交，不会回滚
- 第 3 个及之后的 key 没有写入
- 结果是 **部分成功、部分失败**，DB 处于不一致状态

### 10.3 Upsert 机制：patch + insert 两步（非原子）

每个配置 key 的写入是两步操作：
1. `patch({ value }).where('key', key)` —— 尝试更新
2. 若 `affectedRows === 0` 说明 key 不存在 → 再 `insert({ key, value })`

**问题**：这两步之间存在**竞态条件**。两个并发请求同时为同一个新 key 执行 upsert：
- 请求 A patch → 0 行受影响
- 请求 B patch → 0 行受影响
- 请求 A insert → 成功
- 请求 B insert → **主键冲突异常**（settings 表 `key` 是主键）

代码中没有捕获这个冲突并重试 patch，insert 失败会进入 catch，整个 saveToDb 返回 false。

**对比 Objection 提供的原子 upsert**：
Objection.js 支持 `.onConflict(key).merge()` 的原子 upsert，但此代码使用的是旧版 API，没有采用。标准做法应为：
```js
await WIKI.models.settings.query().insert({ key, value }).onConflict('key').merge()
```

### 10.4 串行化保障：JavaScript 单线程 + 异步队列

Node.js 是单线程 event loop，这提供了一定程度的隐式串行：
- 同一个 Node.js 实例中，多个 resolver 调用 saveToDb **不会并行执行**到数据库
- 但在 `await` 让出 CPU 后，后续语句可能与其他请求交错
- 所以两步 upsert 在单实例中**依然存在竞态窗口**（await patch 返回后，await insert 之前，另一个请求可以插入）

### 10.5 HA 多实例并发：reloadConfig 事件传播

在 Wiki.js HA（高可用）集群部署中，配置变更的一致性通过事件总线实现：

**文件**: `server/core/kernel.js:39-42`

```js
WIKI.events = {
  inbound: new EventEmitter(),
  outbound: new EventEmitter()
}
```

- **outbound**：本实例发出的事件，需通过外部消息中间件广播到其他实例
- **inbound**：接收其他实例发来的事件

**文件**: `server/core/config.js:110-111` 和 `130-135`

```js
// 写入方：发出事件
if (propagate) {
  WIKI.events.outbound.emit('reloadConfig')
}

// 接收方：处理事件
subscribeToEvents() {
  WIKI.events.inbound.on('reloadConfig', async () => {
    await WIKI.configSvc.loadFromDb()     // 从 DB 重新拉取配置
    await WIKI.configSvc.applyFlags()
  })
}
```

注意：EventEmitter2 是**进程内**事件，不是分布式消息。实际的跨实例广播依赖于 `server/servers/` 中的消息中间件（如 Redis Pub/Sub、MQTT 等），它们负责将 outbound 事件路由到所有实例的 inbound。

**HA 并发一致性问题**：
- 两个实例上的管理员**同时**修改不同配置（例如 A 改主题，B 改邮件），可能会覆盖彼此的设置
- 原因：`saveToDb` 写的是 `_.get(WIKI.config, key)`，而 `loadFromDb()` 使用的是 `defaultsDeep`，不会清除已被删除的 key
- 但由于 key 是独立的（`theming` 与 `mail` 是 settings 表不同行），**不同 key 的并发写入互相独立，不会冲突**
- 同一 key 的并发写入，由数据库的**行级锁**保证最终一致性——后提交者覆盖先提交者（last write wins）

### 10.6 设置模型的 Hook 时间戳

**文件**: `server/models/settings.js:30-35`

```js
$beforeUpdate() {
  this.updatedAt = new Date().toISOString()
}
$beforeInsert() {
  this.updatedAt = new Date().toISOString()
}
```

`updatedAt` 由 Objection hook 自动设置，但 `createdAt` **没有 hook**，也没有设置默认值。第一次插入时 `createdAt` 为 `NULL`。

### 10.7 并发与事务问题总结

| 问题 | 严重程度 | 现状 | 建议 |
|------|---------|------|------|
| saveToDb 无事务 | 中 | 多 key 写入部分成功后无法回滚 | 包一层 Objection.transaction，所有 patch/insert 在同一事务中 |
| Upsert 非原子 | 低（罕见场景） | patch→insert 两步竞态 | 用 `onConflict('key').merge()` 原子 upsert |
| 同一 key 并发写入 | 低 | 数据库行锁，Last-Write-Wins | 通常可接受；如需更强一致，加乐观锁 version 列 |
| HA 实例 reloadConfig 延迟 | 低 | 事件广播有毫秒级延迟 | 通常可接受 |
| createdAt 未设置 | 极低 | 新插入行 createdAt 为 NULL | 在 `$beforeInsert` 中设置 `this.createdAt = new Date().toISOString()` |

---

## 十一、页面布局编辑器与主题变量的双向绑定

Wiki.js **没有可视化拖拽的布局编辑器**。所谓"双向绑定"是通过三层机制实现的：Vuex pathify 自动 mutations + Vuetify `v-model` + watcher 实时预览。

### 11.1 双向绑定的基础：Vuex-pathify 自动 mutations

**文件**: `client/store/page.js:59`

```js
import { make } from 'vuex-pathify'

export default {
  namespaced: true,
  state,
  mutations: make.mutations(state)  // ← 自动为每个 state 字段生成 SET_xxx mutation
}
```

`vuex-pathify` 自动为 `title`、`description`、`path`、`scriptJs`、`scriptCss` 等所有 state 字段生成 mutations，使得 `sync('page/scriptCss')` 可以直接作为 Vue 计算属性使用，支持 `v-model` 双向绑定。

### 11.2 页面属性编辑器的双向绑定（页面级 CSS/JS）

**文件**: `client/components/editor/editor-modal-properties.vue`

#### 数据绑定层（v-model → Vuex）

```pug
// 基本信息 Tab
v-text-field(v-model='title' ...)
v-text-field(v-model='description' ...)
v-select(v-model='locale' ...)
v-text-field(v-model='path' ...)
v-combobox(v-model='newTag' ...)

// 脚本 Tab (权限：pages.script)
textarea(ref='codejs')   ← CodeMirror 接管的原始 textarea

// 样式 Tab (权限：pages.style)
textarea(ref='codecss')  ← CodeMirror 接管的原始 textarea
```

#### 计算属性层（sync 与 get）

```js
// 完全双向绑定
title: sync('page/title'),
description: sync('page/description'),
path: sync('page/path'),
tags: sync('page/tags'),
scriptJs: sync('page/scriptJs'),
scriptCss: sync('page/scriptCss'),

// 只读
mode: get('editor/mode'),
hasScriptPermission: get('page/effectivePermissions@pages.script'),
hasStylePermission: get('page/effectivePermissions@pages.style'),
```

#### CodeMirror 编辑器的双向绑定

由于 CodeMirror 不直接支持 `v-model`，通过手动事件监听桥接：

**文件**: `editor-modal-properties.vue:360-394`

```js
loadEditor(ref, mode) {
  this.cm = CodeMirror.fromTextArea(ref, {
    tabSize: 2,
    mode: `text/${mode}`,
    theme: 'wikijs-dark',  // ← 编辑器自身主题硬编码为 wikijs-dark
    lineNumbers: true,
    lineWrapping: true,
    // ...
  })
  switch (mode) {
    case 'html':
      this.cm.setValue(this.scriptJs)  // 从 Vuex 读
      this.cm.on('change', c => {
        this.scriptJs = c.getValue()   // 写入 Vuex（触发 sync 自动 mutation）
      })
      break
    case 'css':
      this.cm.setValue(this.scriptCss) // 从 Vuex 读
      this.cm.on('change', c => {
        this.scriptCss = c.getValue()  // 写入 Vuex
      })
      break
  }
}
```

**Tab 切换时的 CodeMirror 销毁与重建**：
```js
watch: {
  currentTab (newValue, oldValue) {
    if (this.cm) {
      this.cm.toTextArea()  // ← 切 Tab 时销毁 CodeMirror，将内容写回原始 textarea
    }
    if (newValue === 2) {   // 脚本 Tab
      this.$nextTick(() => setTimeout(() => this.loadEditor(this.$refs.codejs, 'html'), 100))
    } else if (newValue === 3) { // 样式 Tab
      this.$nextTick(() => setTimeout(() => this.loadEditor(this.$refs.codecss, 'css'), 100))
    }
  }
}
```

### 11.3 主题设置页的双向绑定（站点级主题变量）

**文件**: `client/components/admin/admin-theme.vue`

#### 表单字段绑定

```pug
v-select(v-model='config.theme' ...)       // 主题下拉
v-select(v-model='config.iconset' ...)     // 图标集下拉
v-switch(v-model='darkMode' ...)           // 暗色模式开关
v-select(v-model='config.tocPosition' ...) // TOC 位置下拉
v-textarea(v-model='config.injectCSS' ...) // 自定义 CSS
v-textarea(v-model='config.injectHead' ...)// 自定义 Head
v-textarea(v-model='config.injectBody' ...)// 自定义 Body
```

#### 计算属性与 watcher 实时预览

```js
computed: {
  darkMode: sync('site/dark'),   // ← 与 Vuex site/dark 双向绑定
},
watch: {
  'darkMode' (newValue, oldValue) {
    this.$vuetify.theme.dark = newValue  // ← ★ watcher 实时推给 Vuetify
  }
},
mounted() {
  this.darkModeInitial = this.darkMode   // ← 保存初始值，用于离开时回退
},
beforeDestroy() {
  this.darkMode = this.darkModeInitial   // ← 未保存就离开时，恢复原始值
  this.$vuetify.theme.dark = this.darkModeInitial
}
```

**实时预览工作流**：
```
管理员拖动暗色模式开关
    ↓
v-model 触发 darkMode 计算属性 setter
    ↓
sync('site/dark') 自动调用 SET_DARK mutation
    ↓
Vuex state.site.dark 更新
    ↓
watch.darkMode 被触发
    ↓
this.$vuetify.theme.dark = newValue （即时生效，全站点变暗色）
    ↓
所有组件的 :class='$vuetify.theme.dark ? ... : ...' 响应式更新
```

**保存确认流**：
```
管理员点击 "Apply" 保存按钮
    ↓
调用 theming.setConfig GraphQL mutation
    ↓
参数包括 darkMode（当前 Vuex 值，可能≠DB 值）
    ↓
服务端更新 WIKI.config.theming.darkMode → saveToDb
    ↓
this.darkModeInitial = this.darkMode （更新初始值，防止离开时回退）
```

#### Apollo 数据绑定（config 整体）

```js
apollo: {
  config: {
    query: themeConfigQuery,
    fetchPolicy: 'network-only',
    update: (data) => data.theming.config,  // ← GraphQL 返回值 → this.config
  }
}
```

`this.config.theme/iconset/tocPosition/injectCSS/injectHead/injectBody` 由 Apollo 初始化，`v-model` 直接修改这些属性，保存时作为 mutation 变量提交。

### 11.4 页面内容编辑器与主题的绑定

**文件**: `client/components/editor/editor-markdown.vue:735-745`

```js
// 代码块主题随站点主题切换
codeBlockSettings: {
  theme: this.$vuetify.theme.dark ? `dark` : `default`
}
// inline code 主题固定为 wikijs-dark
inlineCodeSettings: {
  theme: 'wikijs-dark'
}
```

编辑器内部的代码高亮主题（Prism）会跟随 `$vuetify.theme.dark` 变化，但**没有反向绑定**——修改代码块主题不会影响全局主题。

### 11.5 双向绑定总结

| 场景 | 绑定方式 | 代码位置 |
|------|----------|----------|
| 页面标题/描述/路径 | `sync('page/xxx')` + `v-model` | `editor-modal-properties.vue:292-297` |
| 页面 JS/CSS | CodeMirror `on('change')` → `sync('page/scriptJs/scriptCss')` | `editor-modal-properties.vue:373-385` |
| 站点主题/Toc 位置 | Apollo `this.config` + `v-model` | `admin-theme.vue:25,40,63` |
| 站点暗色模式 | `sync('site/dark')` + watch 推送 `$vuetify.theme.dark` | `admin-theme.vue:163,194-196` |
| 代码块主题 | 读取 `$vuetify.theme.dark`（单向） | `editor-markdown.vue:737` |

---

## 十二、自定义 CSS 注入的安全过滤和转义路径

Wiki.js 的自定义 CSS/HTML 注入**没有 XSS 内容过滤**，只有三层防护机制：权限控制 + CleanCSS 语法压缩 + Pug 不转义输出。

### 12.1 四层防护链

```
┌──────────────────────────────────────────────────────────┐
│  1. 权限层（Permission Guard）                           │
│  - 站点级注入：manage:theme / manage:system 权限          │
│  - 页面级 CSS：pages.style 权限                          │
│  - 页面级 JS：pages.script 权限                          │
└──────────────────────────┬───────────────────────────────┘
                           ↓
┌──────────────────────────┴───────────────────────────────┐
│  2. 语法压缩层（CleanCSS 压缩）                          │
│  - 仅作用于 CSS，不作用于 HTML/JS                        │
│  - 语法错误的 CSS 规则被静默丢弃，不报错                 │
└──────────────────────────┬───────────────────────────────┘
                           ↓
┌──────────────────────────┴───────────────────────────────┐
│  3. 内容安全层（DOMPurify 过滤）                         │
│  - 仅作用于 Markdown 渲染后的页面内容（.contents）        │
│  - ❌ 不作用于 injectCode（CSS/Head/Body）               │
└──────────────────────────┬───────────────────────────────┘
                           ↓
┌──────────────────────────┴───────────────────────────────┐
│  4. 模板输出层（Pug 不转义）                             │
│  - 使用 != 而非 = 输出 injectCode                        │
│  - 内容原样输出到 HTML，不做任何 HTML 转义               │
└──────────────────────────────────────────────────────────┘
```

### 12.2 第一层：权限控制

**GraphQL 权限指令**: `server/graph/schemas/theming.graphql:36`

```graphql
setConfig(...): DefaultResponse
  @auth(requires: ["manage:theme", "manage:system"])
```

`@auth` directive 实现：`server/graph/directives/auth.js:5-54`
- 检查用户 JWT 中的 permissions 字段
- 任一权限满足即可通过
- 无权限时抛出 `Forbidden` 错误

**页面级权限检查**: `server/models/pages.js:278-296`

```js
if (_.has(input, 'extra') && input.extra !== null) {
  if (!WIKI.auth.checkAction(context.user, 'write:styles', pageArgs)) {
    input.extra.css = ogPage.extra ? ogPage.extra.css : ''
  }
  if (!WIKI.auth.checkAction(context.user, 'write:scripts', pageArgs)) {
    input.extra.js = ogPage.extra ? ogPage.extra.js : ''
  }
}
```

- 创建/更新页面时，若用户无 `write:styles` 权限，则 `input.extra.css` 被强制替换为原值
- 若无 `write:scripts` 权限，则 `input.extra.js` 被强制替换为原值
- 这是**静默的权限降级**，不报错，只丢弃未授权的修改

### 12.3 第二层：CleanCSS 语法压缩

**写入时压缩**: `server/graph/resolvers/theming.js:37-41`

```js
if (!_.isEmpty(args.injectCSS)) {
  args.injectCSS = new CleanCSS({
    inline: false   // ← 禁止 @import 内联，避免 SSRF 加载外部资源
  }).minify(args.injectCSS).styles
}
```

**读取时美化**: `server/graph/resolvers/theming.js:28`

```js
injectCSS: new CleanCSS({ format: 'beautify' }).minify(WIKI.config.theming.injectCSS).styles
```

**CleanCSS 的安全作用**（有限）：
- `inline: false` 防止 `@import url('http://evil.com/xss.css')` 被展开
- 语法无效的 CSS 选择器/规则会被**静默丢弃**，不会抛异常
- 但 `<style>` 标签内的 CSS 仍然可以包含 `expression()`（老IE）、`url(javascript:...)` 等历史攻击向量

**页面级 CSS 不经过 CleanCSS**：`page.extra.css` 直接存入 DB，没有压缩步骤。

### 12.4 第三层：DOMPurify（不作用于注入代码）

**文件**: `server/modules/rendering/html-security/renderer.js:1-42`

```js
if (config.safeHTML) {
  const window = new JSDOM('').window
  const DOMPurify = createDOMPurify(window)

  // 白名单扩展
  const allowedAttrs = ['v-pre', 'v-slot:tabs', 'v-slot:content', 'target']
  const allowedTags = ['tabset', 'template']

  input = DOMPurify.sanitize(input, {
    ADD_ATTR: allowedAttrs,
    ADD_TAGS: allowedTags,
    HTML_INTEGRATION_POINTS: { foreignobject: true }
  })
}
```

**关键**：这个渲染器只处理 `page.render`（Markdown → HTML 后的页面内容），不处理 `injectCSS/injectHead/injectBody`。所以注入代码可以包含任意 `<script>`、`<iframe>`、`<style>`。

### 12.5 第四层：Pug 不转义输出

**文件**: `server/views/page.pug:4-7, 39-40`

```pug
block head
  if injectCode.css
    style(type='text/css')!= injectCode.css  // ← != 不转义输出
  if injectCode.head
    != injectCode.head                       // ← != 不转义输出

block body
  ...

  if injectCode.body
    != injectCode.body                        // ← != 不转义输出
```

Pug 中 `=` 会转义 HTML 特殊字符（`<>&"` ），而 `!=` **原样输出**。Wiki.js 对所有注入代码都使用 `!=`。

### 12.6 完整注入路径一览

```
用户在 admin-theme.vue 输入 injectCSS
    ↓ v-model
this.config.injectCSS
    ↓ GraphQL mutation
args.injectCSS
    ↓ CleanCSS 压缩（server/graph/resolvers/theming.js:37-41）
WIKI.config.theming.injectCSS = args.injectCSS
    ↓ saveToDb('theming')
settings 表 key='theming' 的 value.json.injectCSS
    ↓ HTTP 请求到达
WIKI.config.theming.injectCSS (从 DB 读取到内存)
    ↓ server/controllers/common.js:493
injectCode.css = WIKI.config.theming.injectCSS
    ↓ 追加页面级
injectCode.css += '\n' + page.extra.css
    ↓ res.render('page', { injectCode })
    ↓ page.pug:5
style(type='text/css')!= injectCode.css
    ↓
用户浏览器看到 <style>/* 原始内容 */</style>
```

### 12.7 安全风险总结

| 注入类型 | 权限要求 | CleanCSS | DOMPurify | Pug 输出 | 风险 |
|----------|---------|----------|-----------|----------|------|
| 站点 injectCSS | manage:theme | ✅ 压缩 | ❌ 不过滤 | != 原样 | 中（可注入任意 CSS） |
| 站点 injectHead | manage:theme | ❌ 不处理 | ❌ 不过滤 | != 原样 | 高（可注入 `<script>`） |
| 站点 injectBody | manage:theme | ❌ 不处理 | ❌ 不过滤 | != 原样 | 高（可注入 `<script>`） |
| 页面 page.extra.css | pages.style | ❌ 不处理 | ❌ 不过滤 | != 原样 | 中（可注入任意 CSS） |
| 页面 page.extra.js | pages.script | ❌ 不处理 | ❌ 不过滤 | != 原样 | 高（可注入 `<script>`） |

---

## 十三、主题切换的灰度发布与预览模式

Wiki.js **没有**灰度发布、canary 发布、按用户组/百分比逐步放量的主题切换机制。仅有的"预览"能力是管理后台暗色模式的实时开关预览。

### 13.1 管理后台暗色模式预览（唯一的预览机制）

**文件**: `client/components/admin/admin-theme.vue:193-204`

```js
watch: {
  'darkMode' (newValue, oldValue) {
    this.$vuetify.theme.dark = newValue   // ← 实时生效
  }
},
mounted() {
  this.darkModeInitial = this.darkMode    // ← 记录原始值
},
beforeDestroy() {
  this.darkMode = this.darkModeInitial    // ← 未保存则回退
  this.$vuetify.theme.dark = this.darkModeInitial
}
```

**预览工作流**：
1. 管理员进入主题设置页，当前 darkMode 值存入 `darkModeInitial`
2. 管理员切换暗色模式开关，watcher 立即推送至 `$vuetify.theme.dark`
3. 整个管理后台实时变为暗色/亮色，管理员可以预览效果
4. 场景 A：管理员点击 "Apply" 保存 → `darkModeInitial` 更新为当前值 → 离开时不回退
5. 场景 B：管理员不保存，直接跳转到其他页面 → `beforeDestroy` 钩子 → 恢复 `darkModeInitial` → 整个站点恢复之前的主题

### 13.2 不存在的灰度发布能力

经代码全量搜索，以下能力均**未实现**：

#### ❌ 按百分比放量（Canary Release）
没有 `canary: true` / `percentage: 10%` / `rollout` 等配置。搜索 `canary|percentage|rollout|variant` 等关键词仅命中：
- `server/core/system.js:14` `channel: 'BETA'` — 这是 Wiki.js 自身版本更新的通道（beta/stable），不是主题灰度
- `server/db/beta/` — 这是 beta 版本 DB 迁移脚本，不是功能开关

#### ❌ 按用户组灰度（Group-based Rollout）
没有 `groups: ['beta-testers']` 或 `roles: ['admin']` 限定主题可见范围的代码。

#### ❌ A/B 测试（A/B Testing）
没有 `variant` / `experiment` / `split` 相关的主题实验框架。

#### ❌ 按 locale / namespace 灰度
没有按语言或路径前缀应用不同主题的逻辑。所有 locale 共享同一套 `WIKI.config.theming`。

#### ❌ 主题预览专用 URL 参数
没有 `?theme=xxx&preview=true` 或 `?darkMode=toggle` 等临时预览参数。

### 13.3 功能标志系统（未用于主题）

**文件**: `server/app/data.yml:82-96`

Wiki.js 有 `features` 功能标志配置：
```yaml
features:
  featurePageComments: false
  featurePageRatings: false
  featurePageLikes: false
  featurePersonalWiki: false
```

这些是布尔型功能开关，**不用于主题灰度**。它们控制整个站点级功能的开启/关闭，不是渐进式放量。

### 13.4 主题切换的实际生效路径（全量立即生效）

```
管理员在 admin-theme.vue 点击 "Apply"
    ↓
GraphQL Mutation theming.setConfig
    ↓
服务端更新 WIKI.config.theming
    ↓
saveToDb(['theming'], propagate=true)
    ↓
WIKI.events.outbound.emit('reloadConfig')
    ↓
所有 HA 实例通过 inbound 收到事件 → loadFromDb() → applyFlags()
    ↓
⚠️  用户浏览器不自动刷新，需手动刷新页面才能加载新主题资源
    ↓
用户刷新 → client/index-app.js import(themeName) 加载新主题 chunk
```

**注意**：主题变更不会自动推送给在线用户。由于主题 SCSS/JS 被打包在独立 chunk 中，必须刷新页面才能重新加载。

### 13.5 用户级暗色模式的"灰度"效果（误打误撞）

虽然没有真正的灰度发布，但用户级 `appearance` 偏好（`client/store/user.js`）客观上形成了"部分用户先看到暗色模式"的效果：

```
管理员开启全局 darkMode = false（默认亮色）
    ↓
部分用户主动在 Profile 设置 appearance = 'dark'
    ↓
这部分用户看到暗色模式，其他用户看到亮色模式
    ↓
这相当于"自愿灰度"，但不是按百分比放量的受控灰度
```

### 13.6 主题切换预览/灰度能力总结

| 能力 | 状态 | 代码位置 |
|------|------|----------|
| 管理后台暗色模式实时预览 | ✅ 实现 | `admin-theme.vue:193-204` watcher + beforeDestroy |
| 管理后台注入 CSS 实时预览 | ❌ 未实现 | injectCSS 保存后需刷新页面才生效 |
| 切换主题名实时预览 | ❌ 未实现 | 切换 theme 下拉不触发 chunk 重新加载 |
| 按百分比灰度发布 | ❌ 未实现 | 无相关代码 |
| 按用户组灰度发布 | ❌ 未实现 | 无相关代码 |
| 按 locale 灰度发布 | ❌ 未实现 | 所有 locale 共享同一配置 |
| URL 参数临时预览主题 | ❌ 未实现 | 无 preview 查询参数处理 |
| 用户自愿切换暗色模式 | ✅ 实现 | `profile/profile.vue:690-692` appearance 设置 |

---

## 十四、移动端响应式主题适配的断点与样式切换

Wiki.js 采用**双层断点系统**：Vuetify 内置断点（JS 驱动响应式）+ 自定义 SCSS 断点 mixins（CSS 驱动响应式），两者独立但协同工作。

### 14.1 断点定义

#### SCSS 全局断点（全局变量层）

**文件**: `client/scss/global.scss:6-31`

```scss
$tablet: 769px !default;
$desktop: 980px !default;
$widescreen: 1180px !default;

$grid-breakpoints: (
  'xs': 0,
  'sm': 600px,
  'md': 960px,
  'lg': 1280px - 16px,
  'xl': 1920px - 16px
) !default;

$display-breakpoints: (
  'xs-only': 'only screen and (max-width: 599px)',
  'sm-only': 'only screen and (min-width: 600px) and (max-width: 959px)',
  'sm-and-down': 'only screen and (max-width: 959px)',
  'sm-and-up': 'only screen and (min-width: 600px)',
  'md-only': 'only screen and (min-width: 960px) and (max-width: 1263px)',
  'md-and-down': 'only screen and (max-width: 1263px)',
  'md-and-up': 'only screen and (min-width: 960px)',
  'lg-only': 'only screen and (min-width: 1264px) and (max-width: 1903px)',
  'lg-and-down': 'only screen and (max-width: 1903px)',
  'lg-and-up': 'only screen and (min-width: 1264px)',
  'xl-only': 'only screen and (min-width: 1904px)'
) !default;
```

**注意**：有两套定义，前者是 wiki 自定义的 `$tablet/desktop/widescreen`，后者是 Vuetify 风格的 `$grid-breakpoints/$display-breakpoints`。两者的值不同：

| 断点 | 自定义 ($tablet 系列) | Vuetify 风格 ($grid-breakpoints) |
|------|----------------------|---------------------------------|
| sm | —— | 600px |
| md/tabet | 769px | 960px |
| lg/desktop | 980px | 1264px |
| xl/widescreen | 1180px | 1904px |

#### SCSS Mixin 层（语义化断点）

**文件**: `client/scss/base/mixins.scss:77-129`

```scss
@mixin from($device) {
  @media screen and (min-width: $device) { @content; }
}
@mixin until($device) {
  @media screen and (max-width: $device - 1px) { @content; }
}
@mixin mobile {
  @media screen and (max-width: $tablet - 1px) { @content; }   // <769px
}
@mixin tablet {
  @media screen and (min-width: $tablet) { @content; }          // ≥769px
}
@mixin tablet-only {
  @media screen and (min-width: $tablet) and (max-width: $desktop - 1px) { @content; }  // 769-979px
}
@mixin touch {
  @media screen and (max-width: $desktop - 1px) { @content; }   // <980px
}
@mixin desktop {
  @media screen and (min-width: $desktop) { @content; }         // ≥980px
}
@mixin desktop-only {
  @media screen and (min-width: $desktop) and (max-width: $widescreen - 1px) { @content; }  // 980-1179px
}
@mixin widescreen {
  @media screen and (min-width: $widescreen) { @content; }      // ≥1180px
}
```

这些 mixin 使用 wiki 自定义的 `$tablet/desktop/widescreen` 值，而非 Vuetify 的值。

#### Vuetify JS 断点层（Vue 响应式）

Vuetify 内置的 `$vuetify.breakpoint` 对象使用自身的断点体系：
- `xs`: < 600px
- `sm`: 600-959px
- `md`: 960-1263px
- `lg`: 1264-1903px
- `xl`: ≥ 1904px

### 14.2 样式切换的两种触发方式

#### 方式 A — Vue 模板响应式（JS 驱动，基于 Vuetify 断点）

**文件**: `client/themes/default/components/page.vue`

在模板中直接使用 `$vuetify.breakpoint.xxx` 条件渲染：

```pug
// 导航抽屉：smAndDown (<960px) 时用临时抽屉（点击遮罩关闭），lgAndUp 时用永久抽屉
v-navigation-drawer(
  app,
  mobile-breakpoint='600',
  :temporary='$vuetify.breakpoint.smAndDown'  // ← 断点驱动 drawer 模式
  ...
)

// 子导航栏：仅 mdAndDown (<1264px) 显示
v-toolbar-items(v-if='$vuetify.breakpoint.mdAndDown')

// 顶部工具栏：仅 smAndUp (≥600px) 显示
v-toolbar(v-if='$vuetify.breakpoint.smAndUp')

// TOC 侧栏：仅 lgAndUp (≥1264px) 显示，且 tocPosition 不为 'off'
v-sheet(tile, :color='$vuetify.theme.dark ? `grey darken-4-d4` : `grey lighten-4`', v-if='tocPosition !== `off` && $vuetify.breakpoint.lgAndUp')
```

在 script 中监听断点切换：
```js
watch: {
  '$vuetify.breakpoint.lgAndUp' (newVal) {
    if (newVal) {
      // 从移动端切到桌面端，重新同步 TOC 位置
      this.toc = this.$refs.toc
    }
  }
}
```

#### 方式 B — SCSS Mixin 响应式（CSS 驱动，基于 wiki 自定义断点）

**文件**: `client/themes/default/scss/app.scss`

在主题 SCSS 中使用 `@include mobile`、`@include desktop` 等 mixin：

```scss
.v-application .text-h1 {
  @include mobile {
    font-size: 2.75rem !important;  // 移动端 h1 缩小
  }
  @include tablet {
    font-size: 3.5rem !important;   // 平板端 h1 适中
  }
  @include desktop {
    font-size: 4rem !important;     // 桌面端 h1 最大
  }
}

.v-toolbar__content {
  @include mobile {
    padding: 0 8px;  // 移动端工具栏 padding 缩小
  }
}
```

### 14.3 关键布局响应式细节

#### 导航抽屉的移动断点

**文件**: `client/themes/default/components/page.vue:10-11`

```pug
v-navigation-drawer(
  mobile-breakpoint='600',
  :temporary='$vuetify.breakpoint.smAndDown'
)
```

`mobile-breakpoint='600'` 是 Vuetify 组件自身的属性，指定宽度小于 600px 时进入"移动端模式"，抽屉会全屏宽度。`:temporary` 则指定 smAndDown (<960px) 时使用临时抽屉（点击外部关闭）。

#### TOC 的响应式

- `lgAndUp` (≥1264px): TOC 显示在右侧侧栏（固定宽度 260px）
- `mdAndDown` (<1264px): TOC 不显示侧栏，但可以在页面内容顶部显示为下拉菜单（如果配置 `tocPosition !== 'off'`）

#### 子导航栏的响应式

- `mdAndDown` (<1264px): 显示在顶部工具栏下方，包含页面标题、面包屑、操作按钮
- `lgAndUp` (≥1264px): 子导航栏隐藏，标题和操作按钮移到主工具栏

### 14.4 移动端与桌面端布局对比

| 屏幕尺寸 | 断点 | 导航抽屉 | 子导航栏 | TOC 侧栏 |
|----------|------|---------|---------|---------|
| 手机 < 600px | xs | 临时抽屉（全屏宽，点击关闭） | 显示 | 隐藏 |
| 平板 600-959px | sm | 临时抽屉（固定宽度，点击关闭） | 显示 | 隐藏 |
| 小桌面 960-1263px | md | 永久抽屉（常驻） | 显示 | 隐藏 |
| 桌面 ≥ 1264px | lg/xl | 永久抽屉（常驻） | 隐藏 | 显示 |

### 14.5 两套断点不一致的问题

`page.vue` 模板中使用 Vuetify 断点（lg=1264px），而 SCSS mixin 使用 wiki 自定义断点（desktop=980px），这意味着：

- 屏幕宽度 980px~1263px 时，Vue 模板认为是 `mdAndDown`（平板模式），但 SCSS 认为是 `desktop`（桌面模式）
- 可能出现"模板已切换为平板布局但 CSS 还在应用桌面样式"的不一致
- 实际影响有限，因为模板的 `v-if` 会直接移除 DOM 元素，CSS 样式即使应用也不生效

---

## 十五、主题资源 CDN / 本地化静态资源加载策略

Wiki.js 的静态资源加载是**本地化为主、CDN 为辅**的混合模式，主题资源（SCSS/JS/Vue 组件）完全本地化，仅图标字体和第三方库可选 CDN。

### 15.1 webpack publicPath 配置

**文件**: `dev/webpack/webpack.prod.js:38` / `webpack.dev.js:33`

```js
output: {
  publicPath: '/_assets/',
  // ...
}
```

所有 webpack 构建产物（JS/CSS chunk、图片、字体）的 URL 前缀都硬编码为 `/_assets/`，**不支持运行时配置 CDN 域名**。没有 `WIKI.config.cdnUrl` 或类似的配置项。

### 15.2 静态资源的三条加载路径

#### 路径 A — webpack 构建产物（本地化）

构建后的资源通过 `HtmlWebpackPlugin` 注入到 `master.pug`：

**文件**: `dev/templates/master.pug:59-63, 77-81`

```pug
link(
  v-for='(chunkCss, index) in htmlWebpackPlugin.files.css'
  rel='stylesheet'
  type='text/css'
  href='<%= chunkCss %>'  // ← 输出为 /_assets/app.abc123.css
  integrity=config.security.securitySRI ? '<%= htmlWebpackPlugin.files.cssIntegrity[index] %>' : false
  crossorigin='<%= webpackConfig.output.crossOriginLoading %>'
)

script(
  v-for='(chunkJs, index) in htmlWebpackPlugin.files.js'
  type='text/javascript'
  src='<%= chunkJs %>'  // ← 输出为 /_assets/runtime.abc123.js 等
  integrity=config.security.securitySRI ? '<%= htmlWebpackPlugin.files.jsIntegrity[index] %>' : false
  crossorigin='<%= webpackConfig.output.crossOriginLoading %>'
)
```

chunk 文件名带 hash（`app.abc123.js`），支持永久缓存。

#### 路径 B — 静态资源目录（本地化）

**文件**: `server/master.js:63-66`

```js
app.use('/_assets', express.static(path.join(WIKI.ROOTPATH, 'assets'), {
  index: false,
  maxAge: '7d'  // 缓存 7 天
}))
```

`/assets/` 目录下的文件通过 `/_assets/` 路径提供服务，包括：
- favicon 图标 (`assets/favicons/`)
- logo 图片 (`assets/svg/logo-wikijs.svg`)
- 上传的附件 (`assets/uploads/`)
- PWA manifest (`assets/manifest.json`)

#### 路径 C — 图标字体 CDN / 本地化

**文件**: `server/views/page.pug:2-7`

```pug
block head
  case iconset
    when 'mdi'
      link(rel='stylesheet', type='text/css', href='https://cdn.jsdelivr.net/npm/@mdi/font@4.9.95/css/materialdesignicons.min.css')
    when 'mdi-svg'
      //- SVG icon set bundled in the theme
    when 'fa'
      link(rel='stylesheet', type='text/css', href='https://cdn.jsdelivr.net/npm/font-awesome@4.7.0/css/font-awesome.min.css')
    when 'fa4'
      link(rel='stylesheet', type='text/css', href='https://cdn.jsdelivr.net/npm/font-awesome@4.7.0/css/font-awesome.min.css')
    when 'fa5'
      link(rel='stylesheet', type='text/css', href='https://cdn.jsdelivr.net/npm/@fortawesome/fontawesome-free@5.14.0/css/all.min.css')
    default
      link(rel='stylesheet', type='text/css', href='https://cdn.jsdelivr.net/npm/@mdi/font@4.9.95/css/materialdesignicons.min.css')
```

图标字体的 CDN 加载策略：
- **mdi-svg** 是本地化方式：图标通过 `client/themes/default/js/app.js` 中 import 的 `@mdi/js` 包嵌入到 theme chunk 中，不额外发请求
- **其他所有 iconset** (mdi, fa, fa4, fa5) 都通过 CDN 加载，URL 硬编码在 `page.pug` 中，无法配置镜像源或自定义 URL

#### 路径 D — Twemoji emoji 本地化

**文件**: `server/master.js:56-62`

```js
app.use('/_assets/svg/twemoji', async (req, res, next) => {
  try {
    WIKI.asar.serve('twemoji', req, res, next)  // ← 从 asar 打包的 twemoji 资源中读取
  } catch (err) {
    res.sendStatus(404)
  }
})
```

Twemoji 表情符号图片打包在 asar 文件中，完全本地化，不请求 CDN。

### 15.3 SRI 子资源完整性校验

**文件**: `dev/templates/master.pug:61,79`

```pug
integrity=config.security.securitySRI ? '<%= htmlWebpackPlugin.files.cssIntegrity[index] %>' : false
integrity=config.security.securitySRI ? '<%= htmlWebpackPlugin.files.jsIntegrity[index] %>' : false
```

当 `WIKI.config.security.securitySRI = true` 时，webpack 生成的 CSS/JS 文件会附带 SHA-384 哈希的 `integrity` 属性，浏览器会验证文件完整性。

但注意：
- SRI 校验只作用于 webpack 构建的静态资源（主 JS/CSS bundle）
- 图标字体 CDN 资源**没有** `integrity` 属性（硬编码 URL，未计算 hash）
- `crossOriginLoading` 默认为 `'anonymous'`，CDN 资源支持 CORS

### 15.4 主题资源的加载特性

| 资源类型 | 来源 | 配置项 | 缓存 | SRI |
|---------|------|-------|------|-----|
| 主题 SCSS | webpack chunk（`theme.js`） | `theming.theme` | hash 永久缓存 | ✅（如启用） |
| 主题 JS | webpack chunk（`theme.js`） | `theming.theme` | hash 永久缓存 | ✅（如启用） |
| 主题 Vue 组件 | webpack chunk（`theme.js`） | `theming.theme` | hash 永久缓存 | ✅（如启用） |
| 图标字体（CDN） | jsdelivr CDN | `theming.iconset` | 浏览器默认 | ❌ |
| 图标字体（mdi-svg） | webpack theme chunk | `theming.iconset=mdi-svg` | hash 永久缓存 | ✅ |
| Twemoji 表情 | 本地 asar 包 | —— | 7 天 | ❌ |
| 上传附件 | 本地 `assets/` 目录 | —— | 7 天 | ❌ |
| 站点 logo | 本地 `assets/` 目录 | `branding.logo` | 7 天 | ❌ |

### 15.5 CDN 配置的局限性

目前的 CDN 支持**不完整**：
- ❌ 无法配置自定义 CDN 域名（`publicPath` 硬编码为 `/_assets/`）
- ❌ 图标字体 CDN URL 硬编码，无法换国内镜像
- ❌ 构建时才能改 publicPath，运行时无法切换
- ✅ SRI 完整性校验支持（webpack 构建资源）
- ✅ CORS `crossorigin='anonymous'` 支持

如果要支持自定义 CDN，需要修改：
1. `dev/webpack/webpack.prod.js:38` — 把 `publicPath` 改为可配置
2. `server/views/page.pug:2-7` — 把图标 CDN URL 改为从配置读取
3. `server/master.js:63` — 增加 CDN 域名重写中间件或反向代理支持

---

## 十六、主题导出与跨实例迁移

Wiki.js 的导出功能支持包含 `settings` entity，其中**包含主题配置**，但**不包含主题代码文件**。导入功能尚未实现。

### 16.1 导出入口

**GraphQL Mutation**: `system.export` (`server/graph/resolvers/system.js:280-308`)

参数：
```graphql
systemExport(
  path: String!     # 导出目标目录（必须为空）
  entities: [String]!  # 可选值：pages, assets, users, settings, groups, comments, pageshistory
): DefaultResponse
```

前端入口在管理后台的 **Utilities → Export** 页面（非主题设置页面）。

### 16.2 settings entity 导出内容

**文件**: `server/core/system.js:364-386`

```js
case 'settings': {
  WIKI.logger.info('Exporting settings...')
  const outputPath = path.join(opts.path, 'settings.json')
  const config = {
    ...WIKI.config,  // ← 包含 theming 配置！
    modules: {
      analytics: await WIKI.models.analytics.query(),
      authentication: (await WIKI.models.authentication.query()).map(a => ({
        ...a,
        domainWhitelist: _.get(a, 'domainWhitelist.v', []),
        autoEnrollGroups: _.get(a, 'autoEnrollGroups.v', [])
      })),
      commentProviders: await WIKI.models.commentProviders.query(),
      renderers: await WIKI.models.renderers.query(),
      searchEngines: await WIKI.models.searchEngines.query(),
      storage: await WIKI.models.storage.query()
    },
    apiKeys: await WIKI.models.apiKeys.query().where('isRevoked', false)
  }
  await fs.outputJSON(outputPath, config, { spaces: 2 })
  // ...
}
```

`...WIKI.config` 展开整个配置对象，`theming` 字段包含：
```json
{
  "theming": {
    "theme": "default",
    "iconset": "mdi",
    "darkMode": false,
    "tocPosition": "left",
    "injectCSS": "/* 自定义 CSS */",
    "injectHead": "<!-- 自定义 head -->",
    "injectBody": "<!-- 自定义 body -->"
  }
}
```

**注意**：还会导出 `branding`（logo、title、description）、`seo`、`security`、`features` 等其他配置。

### 16.3 导出的内容 vs 缺失的内容

| 主题相关内容 | 是否导出 | 说明 |
|-------------|---------|------|
| `theming.theme`（主题名） | ✅ 导出 | `settings.json` 中 |
| `theming.iconset`（图标集） | ✅ 导出 | `settings.json` 中 |
| `theming.darkMode`（暗色模式） | ✅ 导出 | `settings.json` 中 |
| `theming.tocPosition`（TOC 位置） | ✅ 导出 | `settings.json` 中 |
| `theming.injectCSS`（自定义 CSS） | ✅ 导出 | `settings.json` 中 |
| `theming.injectHead`（自定义 Head） | ✅ 导出 | `settings.json` 中 |
| `theming.injectBody`（自定义 Body） | ✅ 导出 | `settings.json` 中 |
| `branding.logo`（站点 logo） | ✅ 导出 | `settings.json` 中 |
| 主题代码文件（theme.yml、app.scss、app.js、*.vue） | ❌ 不导出 | 需要手动复制 `client/themes/{theme-name}/` 和 `server/themes/{theme-name}/` |
| 页面级 `page.extra.css/js` | ✅ 导出 | 在 `pages.json` 中（如果导出 pages entity） |
| 用户 `appearance` 偏好 | ✅ 导出 | 在 `users.json` 中（如果导出 users entity） |
| webpack 构建产物 | ❌ 不导出 | 需在目标实例重新构建 |

### 16.4 跨实例迁移的完整步骤（需手动操作）

由于缺少导入功能，完整的主题迁移需要手动操作：

```
源实例 A                          目标实例 B
───────────                       ───────────
1. 导出 settings.json
   (system.export entities: settings)

2. 导出 pages.json（含 page.extra.css/js）
   (system.export entities: pages)

3. 复制主题目录
   cp -r client/themes/mytheme/  →  B/client/themes/mytheme/
   cp -r server/themes/mytheme/  →  B/server/themes/mytheme/

4. 重新构建 B 的前端
   npm run build -- --theme=mytheme

5. 导入设置到 B
   方式 A：手动从 settings.json 复制 theming 字段
            到 B 的管理后台 Theme 设置页面
   方式 B：直接写入 B 的 settings 表
            key='theming', value=JSON({...})

6. 重启 B 实例
   loadFromDb() 读取新配置

7. 用户刷新浏览器
   重新加载 theme chunk
```

### 16.5 导入功能的缺失

**全代码库搜索确认**：没有 `system.import` / `restoreSettings` / `importSettings` 等 GraphQL mutation。也没有读取 `settings.json` 并导入的代码。

导出是**单向**的，只能备份，不能自动恢复。用户必须手动解析 JSON 并在目标实例的管理后台重新设置。

### 16.6 导出的并发控制

**文件**: `server/graph/resolvers/system.js:284-286`

```js
if (WIKI.system.exportStatus.status === 'running') {
  throw new Error('Another export is already running.')
}
```

同一时间只能有一个导出任务在运行。`WIKI.system.exportStatus` 是内存中的状态对象（`server/core/system.js:20-32`），HA 多实例部署时每个实例独立检查，可能出现多个实例同时导出。

### 16.7 主题相关 entity 导出总结

| 功能 | 状态 | 代码位置 |
|------|------|----------|
| 导出 settings（含 theming） | ✅ 实现 | `server/core/system.js:364-386` |
| 导出 pages（含 page.extra.css/js） | ✅ 实现 | `server/core/system.js:304-333` |
| 导出主题代码文件 | ❌ 未实现 | 需手动复制文件系统 |
| 导入 settings | ❌ 未实现 | 无相关代码 |
| 导入主题代码 | ❌ 未实现 | 需手动复制 + 重建 |
| 导出并发控制 | ✅ 单实例 | `server/graph/resolvers/system.js:284` |
| HA 跨实例导出并发 | ❌ 未实现 | 内存状态不共享 |

---

## 十七、关键文件索引

| 层级 | 文件 | 职责 |
|------|------|------|
| 配置核心 | `server/core/config.js` | 磁盘/DB 配置加载与持久化（saveToDb 无事务） |
| 配置辅助 | `server/helpers/config.js` | 环境变量替换 |
| 配置数据模型 | `server/models/settings.js` | settings 表读写（upsert、getConfig） |
| 默认配置 | `server/app/data.yml` | 所有配置项默认值（含 features 功能标志） |
| 模块扫描参考实现 | `server/models/editors.js` | refreshFromDisk 事务模式参考 |
| 事件总线 & 启动流程 | `server/core/kernel.js` | inbound/outbound EventEmitter、postBoot 扫描顺序 |
| HA 离线导入 | `server/core/sideloader.js` | 仅支持 locales，不处理主题 |
| HTML 安全过滤 | `server/modules/rendering/html-security/renderer.js` | DOMPurify 过滤页面内容（不作用于 injectCode） |
| GraphQL 权限指令 | `server/graph/directives/auth.js` | @auth 权限检查（write:styles/script 等） |
| Express 初始化 | `server/master.js` | locals 设置、中间件、路由 |
| 页面控制器 | `server/controllers/common.js` | 构建 injectCode、调用 res.render |
| 主题 GraphQL | `server/graph/resolvers/theming.js` | 主题配置读写（CleanCSS 压缩） |
| 主题后台 UI | `client/components/admin/admin-theme.vue` | 主题表单、watcher 实时预览、beforeDestroy 回退 |
| 页面属性编辑器 | `client/components/editor/editor-modal-properties.vue` | 页面级 JS/CSS 编辑、CodeMirror 双向绑定 |
| 页面 Store | `client/store/page.js` | vuex-pathify 自动 mutations、scriptJs/scriptCss 状态 |
| 基础模板 | `dev/templates/master.pug` | 注入 siteConfig JS 变量、图标 CSS |
| 页面模板 | `server/views/page.pug` | != 不转义输出 injectCode、挂载 Vue 组件 |
| 全局 SCSS 变量 | `client/scss/global.scss` | 断点、mixins、mc() 颜色函数 |
| Vuetify 主题初始化 | `client/client-app.js` | dark/rtl 设置，Vue 响应式切换 |
| 客户端入口 | `client/index-app.js` | 动态加载主题 SCSS/JS |
| Site Store | `client/store/site.js` | 前端配置状态（dark、tocPosition 等） |
| User Store | `client/store/user.js` | 用户偏好（appearance 暗色模式） |
| 主题 SCSS 覆写 | `client/themes/default/scss/app.scss` | 内容样式 + 暗色适配 + RTL 适配 |
| 主题页面组件 | `client/themes/default/components/page.vue` | 消费 tocPosition、dark、rtl 进行布局 |
| 用户 profile 暗色切换 | `client/components/profile/profile.vue` | 用户保存 appearance 后即时切换 $vuetify.theme.dark |
| Markdown 编辑器 | `client/components/editor/editor-markdown.vue` | 代码块主题随 $vuetify.theme.dark 变化 |
