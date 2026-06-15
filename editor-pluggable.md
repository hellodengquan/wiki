# Wiki.js 可插拔编辑器：注册与挂载衔接机制

## 全局概览

Wiki.js 的编辑器系统采用**"服务端 definition 注册 + 前端动态组件挂载"**的二级架构：

1. **服务端**：每个编辑器是一个 `server/modules/editor/<key>/` 目录，通过 `definition.yml` 声明身份；启动时统一扫描入库，供管理后台启用/禁用。
2. **前端**：`editor.vue` 是编辑器宿主页面，用 Vue 的 `<component :is="currentEditor">` 动态渲染当前编辑器组件；各编辑器组件在 `mounted()` 时把自身 `editorKey` 写入 Vuex store。

二者之间的衔接纽带是 **`editorKey`** 字符串：服务端页面渲染时把 `page.editorKey` 传入前端 prop，前端据此拼出组件名完成首次挂载；用户手动切换编辑器时，弹窗重新设定 `editorKey`，触发动态组件切换。

---

## 一、服务端注册流程

### 1.1 definition.yml — 编辑器身份声明

每个编辑器在 `server/modules/editor/<key>/definition.yml` 声明元信息：

```yaml
# 以 markdown 为例（server/modules/editor/markdown/definition.yml）
key: markdown              # 唯一标识，贯穿前后端
title: Markdown            # 显示名称
description: Basic Markdown editor
contentType: markdown      # 该编辑器产出的内容 MIME 类型
author: requarks.io
props: {}                  # 可配置属性（当前均为空）
```

当前磁盘上的 7 个编辑器定义：

| 目录 | key | contentType |
|------|-----|-------------|
| `editor/api/` | `api` | `yml` |
| `editor/asciidoc/` | `asciidoc` | `asciidoc` |
| `editor/ckeditor/` | `ckeditor` | `html` |
| `editor/code/` | `code` | `html` |
| `editor/markdown/` | `markdown` | `markdown` |
| `editor/redirect/` | `redirect` | `redirect` |
| `editor/wysiwyg/` | `wysiwyg` | `html` |

### 1.2 refreshEditorsFromDisk() — 启动时扫描入库

**调用时机**：`server/core/kernel.js:75` 在 `postBootMaster()` 中调用：

```js
await WIKI.models.editors.refreshEditorsFromDisk()
```

**核心逻辑**（`server/models/editors.js:37-95`）：

```
1. 读取数据库中已有的 editors 记录 → dbEditors
2. 扫描 server/modules/editor/ 下所有子目录
3. 逐个读取 definition.yml → diskEditors[]
4. 将 diskEditors 缓存到 WIKI.data.editors（含解析后的 props）
5. 对比 dbEditors 与 diskEditors：
   - 新增的 → INSERT（key, isEnabled=false, config=默认值）
   - 已有的 → PATCH（补全新增的 props 默认值，不动已有配置）
6. 新增的批量 INSERT 使用事务
```

入库后数据库 `editors` 表结构：

| 列 | 类型 | 说明 |
|----|------|------|
| key | string (PK) | 编辑器唯一标识 |
| isEnabled | boolean | 是否启用 |
| config | json | 编辑器配置项 |

### 1.3 页面保存时绑定 editorKey

当页面被创建或更新时，GraphQL mutation 的 `editor` 字段取值来自前端 Vuex store 中的 `editor/editorKey`，随内容一起持久化到 `pages` 表的 `editorKey` 列。这样下次打开编辑时，服务端就知道该页面对应哪种编辑器。

---

## 二、前端挂载流程

### 2.1 编辑器宿主组件 editor.vue

文件：`client/components/editor.vue`

#### 2.1.1 组件注册（静态映射）

```js
components: {
  editorApi:       () => import('./editor/editor-api.vue'),
  editorCode:      () => import('./editor/editor-code.vue'),
  editorCkeditor:  () => import('./editor/editor-ckeditor.vue'),
  editorAsciidoc:  () => import('./editor/editor-asciidoc.vue'),
  editorMarkdown:  () => import('./editor/editor-markdown.vue'),
  editorRedirect:  () => import('./editor/editor-redirect.vue'),
  // ... 以及各 modal 组件
}
```

关键点：
- 编辑器组件以 **lazy import**（`webpackMode: "lazy"`）注册，只在需要时加载。
- 组件名的命名规则：**`editor` + 首字母大写的 key**，如 `editorMarkdown`、`editorCkeditor`。

#### 2.1.2 Vuex store 注册

```js
import editorStore from '../store/editor'
WIKI.$store.registerModule('editor', editorStore)
```

在组件定义之前，先把 `editor` module 注册到 Vuex。store 结构：

```js
// client/store/editor.js
state: {
  editor: '',          // 当前编辑器组件名，如 'editorMarkdown'
  editorKey: '',       // 当前编辑器 key，如 'markdown'
  content: '',         // 编辑器内容
  mode: 'create',     // 'create' | 'update'
  activeModal: '',     // 当前激活的弹窗组件名
  activeModalData: null,
  // ...
}
```

- `editor` 与 `editorKey` 是两个不同字段：
  - `editor`：Vue 组件名（驱动 `<component :is>`）
  - `editorKey`：编辑器逻辑标识（驱动保存时上报服务端）

#### 2.1.3 动态组件渲染

```pug
component(:is='currentEditor', :save='save')
```

`currentEditor` 通过 `sync('editor/editor')` 双向绑定到 Vuex 的 `editor/editor` 字段。当该值变化时，Vue 动态组件机制自动卸载旧组件、挂载新组件。

#### 2.1.4 初始挂载（mounted 生命周期）

```js
mounted() {
  this.$store.set('editor/mode', this.initMode || 'create')
  this.initContentParsed = this.initContent ? Base64.decode(this.initContent) : ''
  this.$store.set('editor/content', this.initContentParsed)

  if (this.mode === 'create' && !this.initEditor) {
    // 新建页面且未指定编辑器 → 弹出编辑器选择弹窗
    _.delay(() => { this.dialogEditorSelector = true }, 500)
  } else {
    // 编辑页面 / 新建时已有指定编辑器 → 直接挂载
    this.currentEditor = `editor${_.startCase(this.initEditor || 'markdown')}`
  }
}
```

`initEditor` prop 来自服务端 `editor.pug` 传入的 `page.editorKey`：
- **编辑已有页面**：`page.editorKey` 有值 → 直接挂载对应编辑器
- **新建页面**：`page.editorKey` 为 null → 弹出选择弹窗
- **从模板新建**：`page.editorKey` 取模板页面的 editorKey → 直接挂载

### 2.2 编辑器选择弹窗

文件：`client/components/editor/editor-modal-editorselect.vue`

用户点击编辑器卡片时：

```js
selectEditor(name) {
  this.currentEditor = `editor${_.startCase(name)}`
  this.isShown = false
}
```

- `name` 是编辑器 key（如 `'markdown'`、`'ckeditor'`）
- `_.startCase('markdown')` → `'Markdown'`
- 拼出组件名 `'editorMarkdown'`，赋值给 `currentEditor`
- 赋值触发 Vuex 状态变更 → `<component :is>` 重新渲染 → 新编辑器组件挂载

### 2.3 各编辑器组件的 mounted() — 自注册 editorKey

每个编辑器组件在被 Vue 挂载后，第一步就是把自己的 key 写入 store：

| 组件 | mounted() 中 | 设置的 editorKey |
|------|-------------|-----------------|
| `editor-markdown.vue:728` | `this.$store.set('editor/editorKey', 'markdown')` | `'markdown'` |
| `editor-code.vue:176` | `this.$store.set('editor/editorKey', 'code')` | `'code'` |
| `editor-ckeditor.vue:66` | `this.$store.set('editor/editorKey', 'ckeditor')` | `'ckeditor'` |
| `editor-asciidoc.vue:386` | `this.$store.set('editor/editorKey', 'asciidoc')` | `'asciidoc'` |
| `editor-api.vue:380` | `this.$store.set('editor/editorKey', 'api')` | `'api'` |
| `editor-redirect.vue:155` | `this.$store.set('editor/editorKey', 'redirect')` | `'redirect'` |

这意味着 **editorKey 的写入不是集中式的，而是分散在每个编辑器组件内部**。组件被挂载 → 自身负责声明"我是谁"。

### 2.4 editorKey 的消费

保存页面时（`editor.vue` 的 `save()` 方法），GraphQL mutation 的 `editor` 变量取自：

```js
editor: this.$store.get('editor/editorKey')
```

这个值随内容一起提交到服务端，持久化到 `pages` 表。

---

## 三、完整衔接链路图

```
┌─────────────── 服务端启动 ───────────────┐
│                                          │
│  kernel.postBootMaster()                 │
│    └─→ editors.refreshEditorsFromDisk()  │
│          ├─ 扫描 modules/editor/*/       │
│          ├─ 读取 definition.yml          │
│          └─ 新增的 INSERT 到 DB          │
│                                          │
└──────────────────────────────────────────┘
                    │
                    ▼
┌─────────────── 用户访问 /e/... ──────────┐
│                                          │
│  controllers/common.js (GET /e/*)        │
│    ├─ 查 DB 获取 page                    │
│    ├─ page.editorKey → 已有页面有值      │
│    │            null → 新建页面           │
│    └─ res.render('editor', { page })     │
│                                          │
└──────────────────────────────────────────┘
                    │
                    ▼
┌─────────────── 服务端模板 ───────────────┐
│                                          │
│  editor.pug                              │
│    <editor :init-editor="page.editorKey" │
│            :init-content="page.content"  │
│            :init-mode="page.mode" ... /> │
│                                          │
└──────────────────────────────────────────┘
                    │
                    ▼
┌─────────────── 前端 editor.vue ──────────┐
│                                          │
│  mounted():                              │
│    if (create && !initEditor)            │
│      → 弹出 editor-modal-editorselect    │
│    else                                  │
│      currentEditor =                     │
│        `editor${_.startCase(initEditor)}`│
│                                          │
│  template:                               │
│    <component :is="currentEditor"        │
│               :save="save" />            │
│                                          │
└──────────────────────────────────────────┘
                    │
                    ▼
┌─────────────── 编辑器组件挂载 ───────────┐
│                                          │
│  editor-markdown.vue mounted():          │
│    this.$store.set('editor/editorKey',   │
│                    'markdown')           │
│    // 初始化 CodeMirror 等...            │
│                                          │
└──────────────────────────────────────────┘
                    │
                    ▼
┌─────────────── 保存页面 ─────────────────┐
│                                          │
│  editor.vue save():                      │
│    editor: this.$store.get(              │
│      'editor/editorKey')                 │
│    → GraphQL mutation → DB pages.editor  │
│                                          │
└──────────────────────────────────────────┘
```

---

## 四、关键设计特点与"绕"的地方

### 4.1 组件名与 key 的双重映射

- **editorKey**（如 `'markdown'`）是业务标识，存在 DB 里，在 API 间传递。
- **组件名**（如 `'editorMarkdown'`）是 Vue 组件标识，驱动 `<component :is>`。
- 二者之间的转换靠 `_.startCase()`：`'markdown'` → `'Markdown'` → `'editorMarkdown'`。

这种隐式约定不靠注册表，而是靠**命名规则**保证一致。新增编辑器时必须同时满足：
1. `server/modules/editor/<key>/definition.yml` 的 `key` 字段
2. `client/components/editor/editor-<key>.vue` 文件名
3. `editor.vue` 的 `components` 中注册为 `editor<Key首字母大写>`
4. 组件内 `mounted()` 写入对应的 `editorKey`

四处不一致就会断裂，且无编译期检查。

### 4.2 editorKey 的分散式写入

`editorKey` 不在切换点（editorselect 弹窗或 editor.vue mounted）统一设置，而是**延迟到各编辑器组件的 mounted() 各自设置**。这样做的"好处"是编辑器组件自包含，但代价是：
- 切换逻辑和 key 声明分离，读代码时需要跳转到各组件才能确认 key 值
- 如果组件未正确设置 editorKey，保存时会上报错误的编辑器类型

### 4.3 选择弹窗硬编码列表

`editor-modal-editorselect.vue` 中的编辑器选项是**硬编码**的 HTML，没有从 Vuex 或 API 动态读取。意味着：
- 新增编辑器后，需同时修改此弹窗模板才能在前端选择器中可见
- 后台 admin-editor.vue 的启用/禁用开关目前也未实际生效（`disabled` 状态）

### 4.4 服务端 definition.yml 与前端组件无直接关联

服务端只通过 `definition.yml` 的 `key` + `contentType` 做元数据管理和渲染匹配，前端组件是独立的 `.vue` 文件。两者之间没有 import/require 关系，完全靠命名约定和 `initEditor` prop 传递的 key 值衔接。

---

## 五、如果新增一个编辑器需要改哪些地方

以新增 key 为 `myeditor` 的编辑器为例：

| 步骤 | 文件 | 操作 |
|------|------|------|
| 1 | `server/modules/editor/myeditor/definition.yml` | 新建，声明 key/title/contentType 等 |
| 2 | `client/components/editor/editor-myeditor.vue` | 新建，实现编辑器组件，`mounted()` 中 `this.$store.set('editor/editorKey', 'myeditor')` |
| 3 | `client/components/editor.vue` | 在 `components` 中添加 `editorMyeditor: () => import('./editor/editor-myeditor.vue')` |
| 4 | `client/components/editor/editor-modal-editorselect.vue` | 在弹窗模板中添加对应的卡片 `@click='selectEditor("myeditor")'` |
| 5 | （可选）`client/components/admin/admin-editor.vue` | 在 editors 列表中添加条目 |

---

## 六、存储格式：editorKey / contentType / content 三字段对齐

### 6.1 pages 表中的核心字段

`pages` 表（`server/models/pages.js:33-54`）保存编辑器相关的三个关键列：

| 列 | 类型 | 含义 |
|----|------|------|
| `content` | string | 编辑器原始内容（Markdown/HTML/AsciiDoc 等字符串） |
| `contentType` | string | 内容格式标识，供渲染器/文件扩展名映射使用 |
| `editorKey` | string | 编辑器 key，与 `editors.key` 关联，决定下次用哪个编辑器打开 |

三者的关系：**editorKey → (通过 definition.yml) → contentType**，而 content 的实际格式由 contentType 决定。

### 6.2 editorKey 到 contentType 的映射

映射保存在每个编辑器的 `definition.yml` 中：

```yaml
# markdown:
key: markdown
contentType: markdown

# ckeditor / code / wysiwyg:
key: ckeditor
contentType: html

# asciidoc:
key: asciidoc
contentType: asciidoc

# api:
key: api
contentType: yml

# redirect:
key: redirect
contentType: redirect
```

写入数据库时，服务端通过 `WIKI.data.editors`（启动时从磁盘 definition.yml 解析得到的缓存）查找：

```js
// server/models/pages.js:303 — createPage()
contentType: _.get(_.find(WIKI.data.editors, ['key', opts.editor]), `contentType`, 'text'),
editorKey: opts.editor,
```

即前端 `editor/editorKey` → GraphQL `editor` 参数 → `opts.editor` → 查 `WIKI.data.editors` 得到 `contentType`。

### 6.3 contentType 到文件扩展名的映射

`server/helpers/page.js:11-16` 定义了映射表：

```js
const contentToExt = {
  markdown: 'md',
  asciidoc: 'adoc',
  html: 'html'
}
```

用于 `/d/` 下载端点（`controllers/common.js:89`）时给用户返回正确后缀的文件。

### 6.4 contentType 与元数据注入格式（YAML frontmatter）

当从存储同步/导出页面时，`helpers/page.js:77-99` 根据 contentType 注入不同格式的 frontmatter：

- **markdown**：YAML `---` 块（标准 Jekyll 格式）
- **html**：HTML 注释 `<!-- -->` 包裹
- **json**：合并到对象的 `_meta` 字段
- 其他：直接返回纯 content

解析时（`pages.js:194-233` 的 `parseMetadata()`）同样用正则分别匹配。

### 6.5 页面 content 从 DB 到前端的编码

服务端渲染 editor 页面时（`controllers/common.js:164`），content 会被 **base64 编码**：

```js
// 编辑模式
page.content = Buffer.from(page.content).toString('base64')
// 从模板创建时同理
page.content = Buffer.from(pageVersion.content).toString('base64')
```

前端 `editor.vue:241` 再解码：

```js
this.initContentParsed = this.initContent ? Base64.decode(this.initContent) : ''
this.$store.set('editor/content', this.initContentParsed)
```

编码原因是避免原始 HTML/Markdown 里的特殊字符破坏 Pug 模板的属性引号。

### 6.6 编辑器切换时的内容格式转换（convertPage）

如果用户在后台把 Markdown 页面转换成 CKEditor，或反过来，`pages.convertPage()`（`server/models/pages.js:497-657`）负责做 content 格式转换：

```
Markdown → HTML:  取已渲染好的 page.render（已去掉 toc-anchor、处理 tabset）
HTML → Markdown:  turndown + turndown-plugin-gfm，自定义 sub/sup/u/checkbox 规则
不支持的组合:     抛错 "Unsupported source / destination content types combination."
```

转换时同时更新：
```js
await WIKI.models.pages.query().patch({
  contentType: targetContentType,
  editorKey: opts.editor,
  ...(convertedContent ? { content: convertedContent } : {})
})
```

---

## 七、协作并发编辑冲突解决（Collab / Conflict Detection）

Wiki.js **没有真正的 OT/CRDT 实时协作合并算法**，而是采用**乐观锁 + 手动两版本对比选择**的方案。

### 7.1 冲突检测机制：checkoutDate 乐观锁

#### 7.1.1 checkoutDate 的来源

- 打开编辑页面时，服务端把 `page.updatedAt` 作为 `checkout-date` prop 传入 `editor.pug:24`
- 前端 `editor.vue:232` 将其存入 Vuex：`this.checkoutDateActive = this.checkoutDate`

#### 7.1.2 实时轮询检测（Apollo reactive query）

`editor.vue:556-577` 注册了一个 **Apollo smart query**，每 5 秒轮询一次：

```js
apollo: {
  isConflict: {
    query: gql`
      query ($id: Int!, $checkoutDate: Date!) {
        pages { checkConflicts(id: $id, checkoutDate: $checkoutDate) }
      }
    `,
    fetchPolicy: 'network-only',
    pollInterval: 5000,
    update: (data) => _.cloneDeep(data.pages.checkConflicts),
    skip () { return this.mode === 'create' || this.isSaving || !this.isDirty }
  }
}
```

跳过条件：创建模式 / 正在保存 / 内容未改动。

#### 7.1.3 服务端 checkConflicts 实现

`server/graph/resolvers/page.js:354-368`：

```js
async checkConflicts(obj, args, context, info) {
  let page = await WIKI.models.pages.query()
    .select('path', 'localeCode', 'updatedAt')
    .findById(args.id)
  // ...权限检查...
  return page.updatedAt > args.checkoutDate
}
```

只要 DB 里的 `updatedAt` 比前端持有的 `checkoutDate` **晚**，就判定冲突（返回 true），顶部导航栏出现琥珀色 Conflict 按钮。

#### 7.1.4 保存时二次校验

即使轮询没命中，`editor.vue` 的 `save()` 在真正 UPDATE 之前（`editor.vue:377-394`）也会再调一次 `checkConflicts`：

```js
const conflictResp = await this.$apollo.query({ query: checkConflicts, ... })
if (_.get(conflictResp, 'data.pages.checkConflicts', false)) {
  this.$root.$emit('saveConflict')   // 触发编辑器弹窗
  throw new Error(this.$t('editor:conflict.warning'))
}
```

两道关卡确保不会在别人已更新后还盲目覆盖。

### 7.2 冲突呈现：CodeMirror MergeView 对比

不同编辑器的冲突 UI 分两套：

#### 7.2.1 Markdown / Code / AsciiDoc：三栏差异对比

使用 `client/components/editor/editor-modal-conflict.vue`，核心是 CodeMirror 的 `MergeView` 插件（`addon/merge/merge.js`）：

```js
// editor-modal-conflict.vue:191-203
this.cm = CodeMirror.MergeView(this.$refs.cm, {
  value: this.$store.get('editor/content'),   // 左：本地编辑版本（可编辑）
  orig: resp.content,                          // 右：远端最新版本（只读）
  highlightDifferences: true,
  collapseIdentical: true,
  connect: null,                               // 不允许在两栏间直接推 chunk
  // ...
})
```

- **左栏**（L）：本地当前正在编辑的 content，可继续编辑 → 按钮"Use Local"
- **右栏**（R）：通过 `conflictLatest` GraphQL 查询拉到的最新 DB content，只读 → 按钮"Use Remote"

这是**纯 UI 层的 diff 展示**，没有自动合并（没有三路合并，没有 base 版本），全靠用户肉眼判断后整体选一边。

#### 7.2.2 CKEditor：简化弹窗

由于 CKEditor 本身不擅长做 diff，`client/components/editor/ckeditor/conflict.vue` 只显示警告信息和两个按钮，没有可视化 diff。

### 7.3 冲突解决后的 checkoutDate 更新

无论选 L 还是 R，解决后都会执行：

```js
// editor-modal-conflict.vue:130-135
overwriteAndClose() {
  this.checkoutDateActive = this.latest.updatedAt   // 把本地锁时间戳推进到最新
  this.$root.$emit('overwriteEditorContent')         // 通知编辑器实例同步内容
  this.$root.$emit('resetEditorConflict')            // 关掉顶部 Conflict 指示灯
  this.close()
}
```

关键是 **`checkoutDateActive = this.latest.updatedAt`**，把乐观锁的基线推进到最新版本时间，避免下一轮轮询立即再次触发冲突。

### 7.4 "没有合并算法"的本质

整个冲突模块里**没有 diff-match-patch 的实际合并调用**。`diff-match-patch.js`（`client/libs/codemirror-merge/diff-match-patch.js`）被 import 了，但只是 CodeMirror MergeView 用来计算差异高亮的依赖，最终决策权完全在用户手中：

- 选 Use Local → 把当前编辑器中的值写回 `editor/content`，推进 checkoutDate，下次保存就是以这个新时间戳为基线
- 选 Use Remote → 把最新 DB content 覆盖写回 `editor/content`，同时触发 `overwriteEditorContent` 事件让 CodeMirror/CKEditor 实例 `setValue()`

### 7.5 冲突链路全景

```
用户A 打开页面        用户B 打开页面
   │                    │
   ▼                    ▼
 checkoutDate = T1    checkoutDate = T1
   │                    │
   │                    ▼ 编辑并保存
   │                 DB updatedAt = T2 (T2 > T1)
   ▼
 Apollo pollInterval=5s 调用 checkConflicts(T1)
   │
   ├─→ page.updatedAt(T2) > T1 → 返回 true
   │
   ▼
 editor.vue isConflict = true
  → 顶部显示 Conflict 按钮
  → 用户点击 或 主动保存触发 $root.$emit('saveConflict')
   │
   ▼
 editor-modal-conflict 弹窗
  ├─ conflictLatest(id) 拉最新 DB content
  ├─ CodeMirror.MergeView(value=本地, orig=远端)
  └─ 用户二选一：
     ├─ Use Local   → content = cm.edit.getValue()，checkoutDate = T2
     └─ Use Remote  → content = latest.content，checkoutDate = T2
   │
   ▼
 继续保存（此时 checkConflicts(T2) 与 DB updatedAt 持平，不再冲突）
```

---

## 八、保存与撤销链路（Save / Undo Stack）

Wiki.js 的撤销分为两个独立层次：**编辑器内部 undo（编辑会话内）** 和 **服务端版本历史（跨会话）**。两者互不相通。

### 8.1 编辑器内部 undo：依赖底层库自身实现

Wiki.js **没有自定义的统一 undo manager**，完全托管给 CodeMirror 或 CKEditor 自带的历史栈：

#### 8.1.1 CodeMirror 系（Markdown / Code / AsciiDoc）

CodeMirror 内置 `Doc` 历史栈，默认监听所有编辑操作自动压栈，快捷键：
- `Ctrl/Cmd+Z` → undo
- `Ctrl/Cmd+Shift+Z` / `Ctrl/Cmd+Y` → redo

Wiki.js 的额外 keybindings（`editor-markdown.vue:781-804`）只覆盖了 `Ctrl+S`（保存）、`Ctrl+B`（加粗）、`Ctrl+I`（斜体）、`Ctrl+Alt+Left/Right`（升降标题级），**没有拦截默认的 undo/redo**，所以原生栈直接可用。

但注意一个隐患：当 **冲突解决后调用 `cm.setValue(newContent)`**（`editor-modal-conflict.vue:137` / `$root.$on('overwriteEditorContent', ...)`），CodeMirror 的 `setValue` 会**清空整个 undo history**，用户之前的所有 Ctrl+Z 操作都丢失了。

#### 8.1.2 CKEditor 系

CKEditor 5 的 `DecoupledEditor` 自带 `History` 插件，工具栏有 undo/redo 按钮。Wiki.js 通过 `beautify()` debounce 300ms 把内容同步到 Vuex：

```js
// editor-ckeditor.vue:98-100
this.editor.model.document.on('change:data', _.debounce(evt => {
  this.$store.set('editor/content', beautify(this.editor.getData(), ...))
}, 300))
```

debounce 的存在意味着：
- 连续快速输入不会频繁触发 store 更新
- undo/redo 完全由 CKEditor 内部模型控制，和 Vuex 的 `editor/content` 没有双向绑定（store 只是单向镜像快照）

### 8.2 服务端版本历史：pageHistory 表

每次 UPDATE/MOVE/DELETE/RESTORE 都会把变更前快照写入 `pageHistory` 表。

#### 8.2.1 触发入口：updatePage() / movePage() / deletePage() / convertPage()

以 `updatePage()` 为例（`server/models/pages.js:390-396`）：

```js
// -> Create version snapshot
await WIKI.models.pageHistory.addVersion({
  ...ogPage,                          // 改前完整页面对象
  isPublished: ogPage.isPublished === true || ogPage.isPublished === 1,
  action: opts.action ? opts.action : 'updated',
  versionDate: ogPage.updatedAt       // 改前的 updatedAt 作为该版本的时间戳
})
```

`ogPage` 是刚从 DB 查出来的**原始版本**，patch 还没执行，所以快照内容准确。

#### 8.2.2 pageHistory 表结构（`server/models/pageHistory.js:14-32`）

| 列 | 说明 |
|----|------|
| id | 版本号（自增，前端叫 versionId） |
| pageId | 关联 pages.id |
| authorId | 造成此版本的用户（即本次编辑的作者） |
| content / contentType / editorKey | 改前快照 |
| action | `updated` / `moved` / `deleted` / `restored` |
| versionDate | 该版本对应的 updatedAt 时间戳 |
| createdAt | 本条历史记录写入时间 |

注意：历史里**保存了 editorKey**，所以恢复一个老版本时会同时恢复该版本当时使用的编辑器。

#### 8.2.3 历史列表与版本详情

- `getHistory()`（`pageHistory.js:158-231`）分页返回时间线，只取轻量字段（id、作者、action、时间），不返回 content。
  - 通过比较相邻两条的 `path` 判断是 `edit`、`move` 还是 `initial`。
- `getVersion()`（`pageHistory.js:115-153`）按 versionId 取单条完整快照（含 content）。

#### 8.2.4 恢复版本：restore GraphQL mutation

`server/graph/resolvers/page.js:576-608`：

```js
const targetVersion = await WIKI.models.pageHistory.getVersion({ pageId, versionId })
await WIKI.models.pages.updatePage({
  ...targetVersion,
  id: targetVersion.pageId,
  user: context.req.user,
  action: 'restored'
})
```

本质就是把历史版本当作新内容再调一次 `updatePage()`，而 `updatePage()` 内部又会先把当前状态再拍一张历史快照，所以恢复操作本身也会在历史里留下一条 `restored` 记录，不会丢失中间任何版本。

### 8.3 保存链路完整调用栈

#### 8.3.1 前端 save()

`client/components/editor.vue:279-498`：

```
editor.vue save()
  ├─ showProgressDialog()
  ├─ if mode === 'create':
  │     pages.create() GraphQL mutation
  │       └─ 变量 editor = this.$store.get('editor/editorKey')
  └─ else (mode === 'update'):
        ├─ checkConflicts(id, checkoutDateActive)    // 冲突预检
        │    └─ 冲突 → $root.$emit('saveConflict') + throw
        └─ pages.update() GraphQL mutation
              └─ 变量 editor = this.$store.get('editor/editorKey')
```

保存成功后会：
- 把 `checkoutDateActive` 更新为新的 `page.updatedAt`（推进乐观锁基线）
- `initContentParsed = 最新 content`，重置 isDirty 判断基线
- 通知成功

#### 8.3.2 服务端 pages.create / update

```
graph/resolvers/page.js: PageMutation.create/update
  └─ models/pages.js: Page.createPage() / updatePage()
        ├─ 权限校验
        ├─ 空内容校验
        ├─ (update) addVersion() 写历史快照
        ├─ content + editorKey + contentType(查 WIKI.data.editors) 写入 pages 表
        ├─ tags 关联更新
        ├─ renderPage()  → 异步渲染 HTML 到 render 列
        ├─ 清理缓存 + 删除内存缓存
        ├─ searchEngine.created/updated()  → 重建索引
        ├─ storage.pageEvent()             → Git/Disk 等存储后端同步
        └─ reconnectLinks()                → 更新跨页面链接有效性
```

#### 8.3.3 保存与 undo 的边界

- **编辑器 Ctrl+Z**：只在当前浏览器会话内回退到之前的编辑状态，不涉及网络请求，不写历史。
- **保存（Ctrl+S / 按钮）**：把当前 `editor/content` 提交服务端，服务端写 `pageHistory` 快照并更新 `pages`。保存**不会**清空编辑器 undo 栈，但 `setValue()` 类的操作（冲突解决、切换编辑器、从模板加载）会清空。
- **恢复历史版本**：从 `pageHistory` 取老快照 → 走 `updatePage()` 正常流程 → 再写一条新历史。这是服务端层面的"撤销到过去版本"，和编辑器 undo 栈完全无关。

### 8.4 切换编辑器对 undo 的影响

用户在编辑中途切编辑器（例如从 Markdown 改选 CKEditor）：
1. `editor.vue` 的 `currentEditor` 变化 → Vue 卸载旧组件、挂载新组件
2. 旧 CodeMirror 实例销毁 → 其 undo 栈丢失
3. 新 CKEditor 实例从 Vuex 的 `editor/content` 读内容 → 开启自己全新的 undo 栈

切换编辑器是**断点**：之前在另一个编辑器里做的编辑，Ctrl+Z 追不回来。
