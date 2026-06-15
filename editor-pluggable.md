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

---

## 九、编辑器主题切换

Wiki.js **没有独立的 ThemeManager 类**，编辑器主题切换完全依赖 Vuetify 的 `$vuetify.theme.dark` 响应式属性，各编辑器组件自行读取并适配。

### 9.1 全局暗色模式开关

暗色模式的源头是 `siteConfig.darkMode`（从服务端注入到前端全局变量），前端初始化时赋给 Vuetify：

```js
// 客户端初始化（各页面入口处）
WIKI.$vuetify.theme.dark = siteConfig.darkMode
```

用户在个人设置中切换时，修改 store 中的偏好，触发 `$vuetify.theme.dark` 的响应式更新。所有组件里 `$vuetify.theme.dark` 的引用自动重新求值。

### 9.2 CodeMirror 主题适配

CodeMirror 的主题在创建实例时通过 `theme` 配置项设定。Wiki.js 自定义了一个 `wikijs-dark` 主题（在 `editor-code.vue:371-446` 的 `<style>` 块中以 `.cm-s-wikijs-dark` 选择器定义）。

各 CodeMirror 编辑器的主题处理：

| 编辑器 | 代码位置 | 做法 |
|--------|----------|------|
| Markdown | `editor-markdown.vue:745` | `theme: 'wikijs-dark'` — 固定使用暗色主题 |
| Code | `editor-code.vue:187` | `theme: 'wikijs-dark'` — 固定使用暗色主题 |
| AsciiDoc | `editor-asciidoc.vue:394-403` | `theme: 'wikijs-dark'` — 固定使用暗色主题 |

**关键发现**：CodeMirror 的 `theme` 是在 `mounted()` 创建实例时硬编码的，**没有监听 `$vuetify.theme.dark` 的变化做动态切换**。也就是说，编辑器区域永远是暗色主题，不跟随系统明暗切换。

Mermaid 图表初始化（`editor-markdown.vue:735-738`）则读取了 dark 模式：

```js
mermaid.initialize({
  theme: this.$vuetify.theme.dark ? 'dark' : 'default'
})
```

但这也只在 `mounted()` 时读取一次，后续切换不生效。

### 9.3 CKEditor 主题适配

CKEditor 5 通过 DecoupledEditor 创建时没有传入主题配置。它的视觉样式完全由外部 CSS 控制。在 `editor-ckeditor.vue:143-156` 的样式中：

```scss
.editor-ckeditor {
  background-color: mc('grey', '200');
  @at-root .theme--dark & {
    background-color: mc('grey', '900');
  }
}
```

CKEditor 依赖 Vuetify 在根元素上切换 `theme--dark` 类名，配合 `@at-root .theme--dark &` 选择器实现样式跟随。这种方案**可以实时响应暗色切换**，因为 CSS 类名是响应式的。

### 9.4 编辑器外框（editor.vue）与弹窗的主题适配

`editor.vue` 的外层始终是暗色背景：

```scss
.editor {
  background-color: mc('grey', '900') !important;
}
```

各弹窗组件（media、properties、conflict 等）大量使用 `$vuetify.theme.dark` 三元表达式动态选择颜色，例如：

```pug
v-card(:light='!$vuetify.theme.dark', :dark='$vuetify.theme.dark')
v-toolbar(:color='$vuetify.theme.dark ? `teal` : `teal lighten-5`')
```

这些可以在运行时响应主题切换。

### 9.5 总结

| 层级 | 主题切换能力 | 机制 |
|------|-------------|------|
| 编辑器外框 | 固定暗色 | CSS 硬编码 |
| CodeMirror 编辑区 | 固定暗色 | `theme: 'wikijs-dark'` 在 mounted 时硬编码，不监听变化 |
| CKEditor 编辑区 | 可切换 | `@at-root .theme--dark &` CSS 选择器跟随 |
| 弹窗/工具栏 | 可切换 | `$vuetify.theme.dark` 三元表达式 |
| Markdown 预览区 | 可切换 | `@at-root .theme--dark &` CSS 选择器 |

---

## 十、快捷键绑定（Keymap）

Wiki.js **没有统一的 KeymapRegistry**，各编辑器组件在 `mounted()` 中独立注册自己的快捷键，机制取决于底层库的 API。

### 10.1 CodeMirror 系：extraKeys

CodeMirror 通过 `cm.setOption('extraKeys', keyBindings)` 注册自定义快捷键，**不影响内置的 undo/redo/search**。

#### 平台检测

```js
// editor-markdown.vue:235
const CtrlKey = /Mac/.test(navigator.platform) ? 'Cmd' : 'Ctrl'
```

所有 CodeMirror 编辑器共享此检测，确保 Mac 上用 `Cmd`、其他平台用 `Ctrl`。

#### 各编辑器的快捷键注册

| 编辑器 | 代码位置 | 注册的快捷键 |
|--------|----------|-------------|
| Markdown | `editor-markdown.vue:773-805` | `Ctrl+S` 保存、`Ctrl+B` 加粗、`Ctrl+I` 斜体、`Ctrl+Alt+Right` 降标题级、`Ctrl+Alt+Left` 升标题级 |
| Code | `editor-code.vue:211-219` | `F11` 全屏、`Esc` 退出全屏（仅此两个） |
| AsciiDoc | `editor-asciidoc.vue:426-443` | `Ctrl+B` 加粗(`*`）、`Ctrl+I` 斜体(`_`）、`F11` 全屏、`Esc` 退出全屏 |

注意 Code 编辑器**没有注册 Ctrl+S**——保存按钮在顶部导航栏，Code 编辑器里按 Ctrl+S 不会触发保存。Markdown 编辑器注册了 `Ctrl+S` 绑定到 `this.save()`。

#### 注册方式

```js
const keyBindings = {
  'F11' (c) { c.setOption('fullScreen', !c.getOption('fullScreen')) },
  'Esc' (c) { if (c.getOption('fullScreen')) c.setOption('fullScreen', false) }
}
_.set(keyBindings, `${CtrlKey}-S`, c => { this.save(); return false })
_.set(keyBindings, `${CtrlKey}-B`, c => { this.toggleMarkup({ start: `**` }) })
this.cm.setOption('extraKeys', keyBindings)
```

`_.set()` 动态拼键名字符串，返回 `false` 阻止 CodeMirror 默认行为。

### 10.2 CKEditor 系：内置快捷键

CKEditor 5 的 DecoupledEditor 自带快捷键系统（Bold=Ctrl+B、Italic=Ctrl+I、Link=Ctrl+K 等），Wiki.js **没有额外注册或覆盖**。

CKEditor 的 save 快捷键也是**未注册**的，保存只通过顶部导航栏按钮触发。

### 10.3 编辑器间快捷键不一致问题

| 功能 | Markdown | Code | AsciiDoc | CKEditor |
|------|----------|------|----------|----------|
| Ctrl+S 保存 | ✅ | ❌ | ❌ | ❌ |
| Ctrl+B 加粗 | ✅ `**` | ❌ | ✅ `*` | ✅ (内置) |
| Ctrl+I 斜体 | ✅ `*` | ❌ | ✅ `_` | ✅ (内置) |
| 标题升降级 | ✅ | ❌ | ❌ | ❌ |
| F11 全屏 | ❌ | ✅ | ✅ | ❌ |

各编辑器快捷键**完全独立注册**，没有共享的 keymap 配置层，无法全局统一。

---

## 十一、文件上传与图片嵌入

### 11.1 整体架构

```
前端 Media 弹窗 → FilePond 上传 → POST /u (multer) → assets.upload() → DB
                                                                       ↓
前端选文件插入 → $root.$emit('editorInsert') → 各编辑器组件监听 → 按格式插入到光标位置
```

### 11.2 上传端点：POST /u

`server/controllers/upload.js:13-99`：

1. **multer 中间件**接收文件，存储到 `data/uploads/` 临时目录
2. **权限校验**：`WIKI.auth.checkAccess(req.user, ['write:assets', 'manage:system'])`
3. **单文件限制**：即使 FilePond 配置 `allow-multiple`，后端只接受单文件（多个 file 返回 400）
4. **文件夹元数据**：从 `body.mediaUpload` JSON 中解析 `folderId`
5. **路径权限校验**：拼接 assetPath 后 `checkAccess(user, ['write:assets'], { path: assetPath })`
6. **文件名清洗**：`sanitize()` + 小写 + 空格/逗号替换为下划线
7. **调用 `WIKI.models.assets.upload()`** 处理实际入库

### 11.3 assets.upload() 入库逻辑

`server/models/assets.js:81-100`：

```js
static async upload(opts) {
  const fileInfo = path.parse(opts.originalname)
  const fileHash = assetHelper.generateHash(opts.assetPath)
  // 检查是否已有同 hash 同 folder 的资产
  let asset = await WIKI.models.assets.query().where({ hash, folderId }).first()
  let assetRow = {
    filename: opts.originalname,
    hash: fileHash,
    ext: fileInfo.ext,
    kind: _.startsWith(opts.mimetype, 'image/') ? 'image' : 'binary',
    mime: opts.mimetype,
    fileSize: opts.size,
    folderId: opts.folderId
  }
  // ... 已有则更新，没有则插入
}
```

`kind` 字段按 MIME 前缀判定：`image/*` → `'image'`，其余 → `'binary'`。这个 `kind` 决定了插入编辑器时走 IMAGE 还是 BINARY 分支。

### 11.4 前端 FilePond 上传组件

`client/components/editor/editor-modal-media.vue:139-150`：

```pug
file-pond(
  name='mediaUpload'
  ref='pond'
  :server='filePondServerOpts'
  :instant-upload='false'
  @processfile='onFileProcessed'
)
```

`filePondServerOpts`（`editor-modal-media.vue:317-327`）配置上传目标：

```js
filePondServerOpts() {
  const jwtToken = Cookies.get('jwt')
  return {
    process: {
      url: '/u',
      headers: { 'Authorization': `Bearer ${jwtToken}` }
    }
  }
}
```

上传流程：
1. 用户拖拽/浏览文件 → FilePond 缓存到本地
2. 点击 Upload → `this.$refs.pond.processFiles()` → 逐文件 POST 到 `/u`
3. 每个文件处理完触发 `onFileProcessed` → 自动 5 秒后从 FilePond 列表移除 → 刷新资产列表

### 11.5 资产选择与插入：editorInsert 事件总线

用户在资产列表选中文件点 Insert 时（`editor-modal-media.vue:371-381`）：

```js
insert() {
  const asset = _.find(this.assets, ['id', this.currentFileId])
  const assetPath = this.folderTree.map(f => f.slug).join('/')
  this.$root.$emit('editorInsert', {
    kind: asset.kind,       // 'IMAGE' | 'BINARY'
    path: this.currentFolderId > 0
      ? `/${assetPath}/${asset.filename}`
      : `/${asset.filename}`,
    text: asset.filename,
    align: this.imageAlignment   // 对齐方式
  })
  this.activeModal = ''
}
```

### 11.6 各编辑器对 editorInsert 的处理

`editorInsert` 是一个**全局事件总线**，各编辑器在 `mounted()` 中用 `$root.$on` 监听，`beforeDestroy()` 中 `$root.$off` 取消。

#### Markdown 编辑器（`editor-markdown.vue:825-848`）

```js
this.$root.$on('editorInsert', opts => {
  switch (opts.kind) {
    case 'IMAGE':
      let img = `![${opts.text}](${opts.path})`
      if (opts.align && opts.align !== '') {
        img += `{.align-${opts.align}}`
      }
      this.insertAtCursor({ content: img })
      break
    case 'BINARY':
      this.insertAtCursor({ content: `[${opts.text}](${opts.path})` })
      break
    case 'DIAGRAM':
      this.cm.doc.replaceSelection('```diagram\n' + opts.text + '\n```\n', 'start')
      break
  }
})
```

- IMAGE → `![filename](/path)` + 可选 `.align-xxx` 属性
- BINARY → `[filename](/path)` 普通链接
- DIAGRAM → ``` ```diagram\nbase64...\n``` ``` 代码块

#### Code 编辑器（`editor-code.vue:229-247`）

```js
case 'IMAGE':
  let img = `<img src="${opts.path}" alt="${opts.text}"`
  if (opts.align && opts.align !== '') {
    img += ` class="align-${opts.align}"`
  }
  img += ` />`
  break
case 'BINARY':
  `<a href="${opts.path}" title="${opts.text}">${opts.text}</a>`
```

因为是 HTML 模式，插入的是 `<img>` 和 `<a>` 标签。

#### AsciiDoc 编辑器（`editor-asciidoc.vue:454-474`）

```js
case 'IMAGE':  `image::${opts.path}[${opts.text}]`
case 'BINARY': `link:${opts.path}[${opts.text}]`
case 'DIAGRAM': ```diagram\nbase64...\n```
```

#### CKEditor（`editor-ckeditor.vue:102-119`）

```js
case 'IMAGE':  this.editor.execute('imageInsert', { source: opts.path })
case 'BINARY': this.editor.execute('link', opts.path, { linkIsDownloadable: true })
case 'DIAGRAM': this.editor.execute('imageInsert', { source: `data:image/svg+xml;base64,${opts.text}` })
```

CKEditor 通过自己的命令系统 `execute()` 插入，DIAGRAM 被转为 base64 data URI 当图片插入。

### 11.7 Draw.io 图表嵌入

`editor-modal-drawio.vue` 嵌入 diagrams.net iframe，通过 `postMessage` 协议交互：

```
1. init 事件 → 发送已有 XML 数据加载
2. save 事件（exit=true） → 请求 export 为 SVG
3. export 事件 → 取 base64 部分 → $root.$emit('editorInsert', { kind: 'DIAGRAM', text: svgBase64 })
```

Markdown 代码块内已有 `diagram` 类型的图表，双击会弹出 Draw.io 编辑器（`editor-markdown.vue:697-698`）。

### 11.8 资产权限校验与编辑器的联动

编辑器本身**不直接做资产权限校验**，校验全部在服务端：

| 操作 | 校验点 | 权限要求 |
|------|--------|---------|
| 上传文件 | `controllers/upload.js:22` | `write:assets` 或 `manage:system` |
| 上传到特定路径 | `controllers/upload.js:83` | `write:assets` + 路径匹配 |
| 浏览文件夹 | `graph/resolvers/asset.js:42` | `read:assets` + 路径匹配 |
| 列出资产 | `graph/resolvers/asset.js:28` | `read:assets` + 路径匹配 |
| 重命名 | `graph/resolvers/asset.js:110` | 源路径 `manage:assets` + 目标路径 `write:assets` |
| 删除 | `graph/resolvers/asset.js:162` | `manage:assets` |

前端只负责展示，如果用户无权限，GraphQL 查询返回空列表或 mutation 返回错误。

---

## 十二、权限校验体系与编辑器联动

### 12.1 checkAccess 核心算法

`server/core/auth.js:221` 的 `checkAccess(user, permissions, page)` 是整个权限系统的核心：

```
1. manage:system 权限 → 直接放行（超级管理员）
2. 用户全局权限 ∩ 要求权限 → 交集为空则拒绝
3. 没有页面级规则（page=false）→ 放行
4. 遍历用户所属组的 pageRules：
   - 按 locale 过滤
   - 按匹配类型（START/END/REGEX/EXACT/TAG）匹配路径
   - 按优先级（EXACT > TAG > REGEX > END > START）取最高优规则
   - 规则 DENY → 拒绝；规则 ALLOW → 放行
```

页面级规则支持五种匹配模式，优先级从高到低：EXACT > TAG > REGEX > END > START。

### 12.2 编辑器场景下的权限注入

用户打开 `/e/...` 编辑页面时，`controllers/common.js:133` 计算出完整的 `effectivePermissions`：

```js
const effectivePermissions = WIKI.auth.getEffectivePermissions(req, pageArgs)
```

`getEffectivePermissions()`（`auth.js:496-521`）返回一个对象，包含：

```js
{
  comments: { read, write, manage },
  history:  { read },
  source:   { read },
  pages:    { read, write, manage, delete, script, style },
  system:   { manage }
}
```

这个对象被 base64 编码后作为 `effective-permissions` prop 传入 `editor.pug` → 前端 `editor.vue` 的 `effectivePermissions` prop → 解码后存入 Vuex `page/effectivePermissions`。

### 12.3 权限在编辑器中的消费

前端拿到 `effectivePermissions` 后，**主要用于控制 UI 元素的可见性和保存按钮的可用性**，不做细粒度的功能拦截：

- **保存权限**：顶部导航栏的 Save 按钮始终可见，但服务端 `createPage/updatePage` 内部会再次校验 `write:pages` 权限
- **脚本/样式编辑**：保存时 `createPage/updatePage` 检查 `write:styles` / `write:scripts` 才写入 extra.css/js
- **资产上传**：上传弹窗始终可打开，但 POST `/u` 时校验 `write:assets`

### 12.4 "没有 @提及与权限联动"的现状

Wiki.js 的代码中**不存在 @mention 功能**。搜索整个代码库，`mention` 关键词只出现在：
- CKEditor 的一个被注释掉的 TODO 块（`editor-ckeditor.vue:73-80`），提到 `mention` autocomplete 但未实现
- 第三方 CSS 文件中的无关匹配

因此**没有 @提及与权限校验的联动链路**。当前编辑器不存在 @someone 触发权限检查或通知的逻辑。

如果未来实现 @mention，合理的链路应该是：
1. 编辑器内输入 `@` → 触发 autocomplete 弹窗
2. 弹窗调用 GraphQL 查询用户/组列表（需考虑隐私权限）
3. 选中后插入特殊标记（如 `@[username](userId)`）
4. 保存时解析标记，对被提及者发通知
5. 通知目标用户时需 `checkAccess` 确认其有 `read:pages` 权限

---

## 十三、编辑器组件懒加载边界

### 13.1 三层加载架构

编辑器前端的加载分三个层级，每一层都由 webpack 的动态 `import()` 驱动：

```
client-app.js (全局注册)
  └─ Editor → import(/* webpackChunkName: "editor", webpackPrefetch: -100 */) editor.vue
                    │
                    ├─ 编辑器组件 (lazy) ──────────────────────────────
                    │   editorMarkdown:  import(/* chunk: "editor-markdown",  lazy */)
                    │   editorCkeditor:  import(/* chunk: "editor-ckeditor", lazy */)
                    │   editorCode:      import(/* chunk: "editor-code",     lazy */)
                    │   editorAsciidoc:  import(/* chunk: "editor-asciidoc", lazy */)
                    │   editorApi:       import(/* chunk: "editor-api",      lazy */)
                    │   editorRedirect:  import(/* chunk: "editor-redirect", lazy */)
                    │
                    ├─ 弹窗组件 (eager) ────────────────────────────────
                    │   editorModalEditorselect: import(/* chunk: "editor", eager */)
                    │   editorModalProperties:   import(/* chunk: "editor", eager */)
                    │   editorModalUnsaved:      import(/* chunk: "editor", eager */)
                    │   editorModalMedia:        import(/* chunk: "editor", eager */)
                    │   editorModalBlocks:       import(/* chunk: "editor", eager */)
                    │   editorModalDrawio:       import(/* chunk: "editor", eager */)
                    │
                    └─ 冲突弹窗 (lazy) ────────────────────────────────
                        editorModalConflict:     import(/* chunk: "editor-conflict", lazy */)
```

### 13.2 加载策略对照

| 类别 | webpackMode | webpackChunkName | 含义 |
|------|------------|-----------------|------|
| 编辑器主体 | `lazy` | `editor-<key>` | 每个编辑器独立 chunk，按需加载 |
| 常用弹窗 | `eager` | `editor` | 不额外分 chunk，打包进 editor.vue 的主 chunk |
| 冲突弹窗 | `lazy` | `editor-conflict` | 独立 chunk，只有冲突时才加载（含 diff-match-patch 等重依赖） |

### 13.3 webpackMode: "lazy" vs "eager" 的实际效果

- **`lazy`**：生成独立 JS 文件，只在组件被 Vue 渲染时才下载。6 个编辑器各自独立 chunk，用户用 Markdown 编辑器时不会加载 CKEditor 的 1MB+ 依赖。
- **`eager`**：不生成额外 chunk，模块代码合并到父 chunk（即 `editor.vue` 所属的 `editor` chunk）中。这意味着 properties/media/editorselect 等弹窗的代码在编辑器页面加载时就一起下载了，即使还没打开。

### 13.4 Editor 全局组件的预取优先级

```js
// client-app.js:157
Vue.component('Editor', () => import(/* webpackPrefetch: -100, webpackChunkName: "editor" */ './components/editor.vue'))
```

`webpackPrefetch: -100` 是负数优先级，意味着**编辑器主 chunk 在浏览器空闲时会被优先预取**（负数让它在 prefetch 队列中排前），但不阻塞初始页面渲染。用户在浏览页面时，编辑器代码可能已经悄悄下载好了。

### 13.5 懒加载的边界效应

1. **首屏不加载任何编辑器代码**：`editor.vue` 本身通过 `webpackPrefetch` 预取，6 个编辑器组件按 `lazy` 策略延迟加载。用户打开编辑页面后，只有实际使用的编辑器 chunk 被下载。

2. **编辑器切换时的加载空白**：如果用户从 Markdown 切换到 CKEditor，`<component :is="currentEditor">` 触发 Vue 的异步组件解析 → webpack 动态 import CKEditor chunk → 下载 + 解析 → 渲染。这中间有一个短暂空白期，**没有 loading 占位符**（`editor.vue` 的 `<component :is>` 没有配合 `<Suspense>` 或 loading 状态）。

3. **弹窗 eager 的代价**：media 弹窗引入了 FilePond + 所有 GraphQL 查询，properties 弹窗引入了 Vuetify 的 date-picker 等重组件，这些全部合并进 `editor` chunk，即使用户不打开这些弹窗，代码也已经下载了。

4. **外部复用 media 弹窗**：`admin-security.vue:256` 和 `admin-general.vue:279` 也分别 import 了 `editor-modal-media.vue`，但用的是 `lazy` 模式。这意味着同一组件在编辑器上下文中是 eager 打包的，在 admin 上下文中是 lazy 分 chunk 的。

5. **CodeMirror 的共享依赖**：markdown、code、asciidoc 三个编辑器都依赖 CodeMirror 核心和多个 addon。由于 webpack 的 chunk 共享机制，这些公共依赖会被提取到公共 chunk 中，不会重复下载。

---

## 十四、AsciiDoc / API / Redirect 次要编辑器的协议差异

### 14.1 三种次要编辑器总览

| 维度 | AsciiDoc | API | Redirect |
|------|----------|-----|----------|
| contentType | `asciidoc` | `yml` | `redirect` |
| 内容载体 | CodeMirror（纯文本） | Vuetify 表单控件（结构化表单） | Vuetify 表单控件（结构化表单） |
| 内容同步到 Vuex | `cm.on('change')` → `editor/content` | **不写入** `editor/content` | **不写入** `editor/content` |
| editorInsert 事件 | ✅ 监听 IMAGE/BINARY/DIAGRAM | ❌ 不监听 | ❌ 不监听 |
| 冲突检测 | ✅ 监听 `saveConflict` | ❌ 不监听 | ❌ 不监听 |
| 创建时默认内容 | `'== header\n\ncontent'` | `'<h1>Title</h1>\n\n<p>Some text here</p>'` | `'<h1>Title</h1>\n\n<p>Some text here</p>'` |

### 14.2 AsciiDoc 编辑器：类 Markdown 的完整协议

`editor-asciidoc.vue` 的协议与 Markdown 编辑器几乎一致：

**内容同步**：
```js
this.cm.on('change', c => {
  this.$store.set('editor/content', c.getValue())
  this.onCmInput(this.$store.get('editor/content'))
})
```

**预览渲染**：使用 asciidoctor.js（`require('asciidoctor')()`）在前端实时转换：
```js
let html = asciidoctor.convert(newContent, { standalone: false, safe: 'safe', ... })
const $ = cheerio.load(html, { decodeEntities: true })
// 处理 diagram 代码块
this.previewHTML = DOMPurify.sanitize($.html(), { ADD_TAGS: ['foreignObject'] })
```

预览流程与 Markdown 编辑器的区别：
- Markdown 用 markdown-it + 服务端渲染管线
- AsciiDoc 用 asciidoctor.js 完全在前端渲染（客户端渲染），**不经过服务端渲染管线**
- 但服务端 `render-page.js` 保存时也走渲染管线，此时 AsciiDoc 用 `asciidoc-core` 渲染器（`server/modules/rendering/asciidoc-core/`）

**格式化标记差异**：

| 操作 | Markdown | AsciiDoc |
|------|----------|----------|
| 加粗 | `**text**` | `**text**`（同样语法） |
| 斜体 | `*text*` | `__text__`（双下划线） |
| 标题 | `# text` | `= text`（等号数量 = 级别） |
| 上标 | 不支持 | `^text^` |
| 下标 | 不支持 | `~text~` |
| 引用 | `> text` | 支持 NOTE/TIP/WARNING/CAUTION/IMPORTANT 前缀 |
| 图片 | `![alt](path)` | `image::path[alt]` |
| 链接 | `[text](url)` | `link:url[text]` |
| 内部链接 | `[/locale/path][text]` | `link:/locale/path[text]` |

**Diagram 支持**：与 Markdown 相同的 ` ```diagram\nbase64\n``` ` 格式，双击打开 Draw.io 编辑器。

### 14.3 API 编辑器：表单驱动的结构化编辑

`editor-api.vue` 是一个**完全不同的编辑模式**——没有 CodeMirror、没有文本编辑、没有预览。

**布局**：左侧 sidebar 导航（Info / Servers / Endpoints / Models / Auth）+ 右侧表单区域。

**数据模型**：所有数据存在组件的 `data()` 中：
```js
data() {
  return {
    tab: 'endpoints',
    kind: 'rest',
    info: { title: '', version: '1.0.0', description: '' },
    servers: [{ name: 'Production', url: 'https://api.example.com/v1', icon: 'server', id: '123456' }],
    endpointGroups: [{ id: '345678', name: '', description: '', endpoints: [...] }],
    endpointMethods: [{ key: 'GET', color: 'blue' }, { key: 'POST', color: 'green' }, ...]
  }
}
```

**关键问题：content 断链**

```js
mounted() {
  this.$store.set('editor/editorKey', 'api')
  if (this.mode === 'create') {
    this.$store.set('editor/content', '<h1>Title</h1>\n\n<p>Some text here</p>')
  }
}
```

- `editorKey` 设置为 `'api'`，这是唯一与编辑器框架的交互
- `editor/content` 的默认值是 HTML（`<h1>Title</h1>...`），不是 YAML/JSON——这是个硬编码占位符
- 表单中 info/servers/endpoints 的变化**完全没有同步到 `editor/content`**
- 保存时 `editor.vue` 的 `save()` 取的是 `this.$store.get('editor/content')`，但这个值从未被 API 编辑器更新过

这意味着 **API 编辑器当前处于半成品状态**：表单 UI 已搭建但数据流未闭环。编辑器可以展示，但保存不会把表单内容持久化。

**API 编辑器也不监听 `editorInsert` 和 `saveConflict` 事件**——不需要嵌图功能，冲突检测也缺失。

**服务端渲染管线**：API 编辑器的 contentType 是 `yml`，保存时触发 `render-page.js`，走 `getRenderingPipeline('yml')` 查找匹配的渲染器。当前只有 `openapi-core` 渲染器声明 `input: openapi`，不匹配 `yml`，所以**API 页面的渲染管线实际上为空**——内容直接存入 DB 的 `render` 列但不做任何转换。

### 14.4 Redirect 编辑器：条件重定向配置

`editor-redirect.vue` 也是一个表单驱动编辑器，用于配置页面的条件重定向规则。

**布局**：居中表单，包含：
- 条件规则列表（按用户组匹配 → 跳转到指定页面/URL）
- 兜底规则（不匹配任何条件时跳转到指定页面/URL）

**数据模型**：
```js
data() {
  return {
    fallbackMode: 'page',        // 'page' | 'url'
    fallbackUrl: 'https://'
  }
}
```

**与 API 编辑器同样的问题**：
- `editor/content` 设置了 HTML 占位符但表单变化不同步
- 不监听 `editorInsert` 和 `saveConflict`
- 条件规则部分的 UI 只是骨架（`@click=''` 空处理器），"Add Conditional Rule" 按钮无实际功能

**Apollo 查询**：
```js
apollo: {
  groups: {
    query: gql`{ groups { list { id name } } }`,
    fetchPolicy: 'network-only',
    update: (data) => data.groups.list
  }
}
```

Redirect 编辑器通过 GraphQL 查询用户组列表（用于条件规则的组选择器），这是它唯一的服务端交互。

**contentType 为 `redirect` 的渲染管线**：没有渲染器声明 `input: redirect`，保存时 `render` 列直接存原始 content。用户访问 redirect 页面时由服务端中间件处理跳转逻辑，不需要渲染 HTML。

### 14.5 三种编辑器与编辑器框架的协议完整度对比

| 协议点 | Markdown/Code/CKEditor | AsciiDoc | API | Redirect |
|--------|----------------------|----------|-----|----------|
| 设置 editorKey | ✅ | ✅ | ✅ | ✅ |
| 同步 content 到 Vuex | ✅ 实时 | ✅ 实时 | ❌ | ❌ |
| 监听 editorInsert | ✅ | ✅ | ❌ | ❌ |
| 监听 saveConflict | ✅ | ✅ | ❌ | ❌ |
| overwriteEditorContent | ✅ | ✅ | ❌ | ❌ |
| 创建时默认 content | 空或模板 | AsciiDoc 模板 | HTML 占位 | HTML 占位 |
| 保存时 content 有效 | ✅ | ✅ | ❌ | ❌ |

API 和 Redirect 编辑器虽然注册了 `editorKey`，但**没有实现编辑器框架期望的内容同步协议**，保存时实际上保存的是初始占位符内容。

---

## 十五、API 模式下的 Schema 校验链路

### 15.1 现状：没有前端 Schema 校验

API 编辑器（`editor-api.vue`）**不进行任何 OpenAPI/YAML schema 校验**。

前端表单中：
- `info.title` 和 `info.version` 标注了 "Required" hint，但只是文本提示，没有 `required` 验证规则
- `servers[].url` 和 `endpoints[].path` 同理
- Vuetify 的 `v-text-field` 组件的 `rules` 属性均未设置

### 15.2 服务端：没有 API 内容的 Schema 校验

服务端保存页面时（`pages.createPage` / `pages.updatePage`），对 content 字段的处理：

```js
// pages.js:316
if (opts.content.length < 1) {
  throw new WIKI.Error.PageEmptyContent()
}
```

唯一校验是**内容非空**，不区分 contentType，不做 OpenAPI schema 校验。API 编辑器的 contentType 是 `yml`，但服务端不会尝试 `yaml.safeLoad()` 解析或校验 YAML 格式。

### 15.3 渲染管线中的 OpenAPI 处理

`server/modules/rendering/openapi-core/` 是唯一与 OpenAPI 相关的服务端模块：

**definition.yml**：
```yaml
key: openapiCore
title: Core
description: Basic OpenAPI Parser
input: openapi
output: html
```

**renderer.js**：
```js
async render() {
  let output = this.input
  for (let child of this.children) {
    const renderer = require(`../${_.kebabCase(child.key)}/renderer.js`)
    output = await renderer.init(output, child.config)
  }
  return output
}
```

这个渲染器声明 `input: openapi`，但当前 API 编辑器的 contentType 是 `yml` 而不是 `openapi`。渲染管线选择器（`renderers.js:141`）按 `contentType` 匹配 `input`：

```js
let activeCoreKeys = _.filter(rawCores, ['input', contentType]).map(core => core.key)
```

`yml` ≠ `openapi`，所以 openapi-core 渲染器**不会被激活**。

### 15.4 渲染管线的完整工作原理

当页面保存后，`render-page.js` 被触发：

```
1. 从 DB 读取 page.content + page.contentType
2. 调用 getRenderingPipeline(contentType) 构建渲染管线
3. 管线构建逻辑（renderers.js:110-171）：
   a. 取所有 isEnabled 的渲染器
   b. 找没有 dependsOn 的核心渲染器（core）
   c. 给每个 core 挂载有 dependsOn = core.key 的子渲染器
   d. 用 DepGraph 按 input/output 依赖关系排序
   e. 过滤出 input === contentType 的起点 + 其所有下游依赖
   f. 按拓扑排序返回有序渲染器列表
4. 逐个执行 renderer.render()，上一个的 output 是下一个的 input
5. 最终 output 存入 pages.render 列
```

各 contentType 的典型渲染管线：

| contentType | 管线 |
|-------------|------|
| `markdown` | markdown-core → html-core → html-security → html-codehighlighter → ... |
| `asciidoc` | asciidoc-core → html-core → html-security → ... |
| `html` | html-core → html-security → ... |
| `yml` | （无匹配渲染器，content 直接存为 render） |
| `redirect` | （无匹配渲染器，content 直接存为 render） |

### 15.5 API 编辑器"如果完整实现"应有的校验链路

假设 API 编辑器完成闭环，校验应该发生在三个层次：

1. **前端表单层**：Vuetify `rules` 做 required/format 校验，确保 title/version 非空，URL 格式合法
2. **前端序列化层**：将 info/servers/endpoints 结构序列化为 OpenAPI 3.0 YAML → 写入 `editor/content`
3. **服务端渲染层**：`openapi-core` 渲染器解析 YAML → 用 swagger-parser 或类似库校验 OpenAPI schema → 生成可交互的 API 文档 HTML → 存入 `render` 列

当前三步均未实现：表单无 rules，序列化未写，渲染器 input 不匹配。

### 15.6 为什么 API/Redirect 编辑器处于半成品状态

代码中的多处信号表明这两个编辑器尚未完成：

1. **创建时默认 content 是 HTML**：`'<h1>Title</h1>\n\n<p>Some text here</p>'` 不符合 API（应为 YAML）和 Redirect（应为 JSON 配置）的 contentType
2. **GraphQL 类型标记为 disabled**：API 编辑器中 GraphQL 选项 `disabled`，注释 "Coming soon"
3. **条件规则按钮无处理器**：Redirect 编辑器的 "Add Conditional Rule" 按钮 `@click=''`
4. **没有 content 同步**：两个编辑器都没有在表单变化时更新 `editor/content`
5. **没有 media/conflict 弹窗**：不监听 `editorInsert` 和 `saveConflict`

这些编辑器已注册到 `definition.yml` 和 `editor.vue` 的 components 中，但功能仅为 UI 骨架。
