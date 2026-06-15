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
