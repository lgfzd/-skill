---
name: frontend-code-standards
description: Write, review, or refactor clean, maintainable frontend code (Vue, React, or vanilla JS/TS) with clear component architecture, formatting, naming, data/service separation, low duplication, state safety, and accessible UI behavior. Use for frontend components, pages, styles, scripts, multi-step flows, and AI-generated UI code that is monolithic, mixes mock data with views, repeats logic, or needs a code-quality review.
---

# Frontend Code Standards

Enforce formatting, naming, data-separation, redundancy, and safety rules that AI-generated frontend code commonly violates. These rules are framework-agnostic unless noted, and apply to Vue 2/3, React, and vanilla JS/TS.

## Formatting (Non-negotiable)

Never compress multiple statements, elements, or CSS rules onto a single line to save vertical space. This is the single most common AI mistake.

### Template / JSX

- One element per line when it has children or multiple attributes. Break long attributes across lines.
- Never put an entire `v-for` grid + table + form on one line.
- Keep closing tags on their own line for any element with children.

```html
<!-- WRONG: entire grid + table on one line -->
<div class="info-grid"><div><span>名称</span><strong>{{ row.name }}</strong></div></div>

<!-- CORRECT -->
<div class="info-grid">
  <div>
    <span>名称</span>
    <strong>{{ row.name }}</strong>
  </div>
</div>
```

### Script

- One statement per line. One method definition per line.
- Never chain multiple method definitions with `,` on a single line.
- One property per line in object literals with more than 2 keys.

```js
// WRONG
handleQuery() {}, handleReset() { this.form = { a: 1, b: 2 } }, getStatusLabel(s) { return MAP[s] },

// CORRECT
handleQuery() {
  this.pagination.page = 1
},
handleReset() {
  this.form = { a: 1, b: 2 }
},
getStatusLabel(s) {
  return MAP[s]
}
```

### Style / SCSS / CSS

- One selector + one property per line. Never pack multiple CSS rule blocks onto a single line.
- A single CSS line must not exceed ~120 characters. If it does, break it.
- Use nested SCSS for scoping, not for cramming entire stylesheets into one line.

```scss
// WRONG
.panel { display: flex; } .panel-heading { font-size: 16px; } .panel-body { padding: 20px; overflow-y: auto; }

// CORRECT
.panel {
  display: flex;
}
.panel-heading {
  font-size: 16px;
}
.panel-body {
  padding: 20px;
  overflow-y: auto;
}
```

### Indentation and Line Endings

- Follow the project `.editorconfig`. If none exists: 2-space indent, LF endings, UTF-8, trim trailing whitespace, insert final newline.
- Match the project's ESLint/Prettier config. If the project has `semi: never` and `quotes: 'single'`, follow exactly.

## File Size and Component Splitting

| Signal | Action |
|--------|--------|
| A `.vue`/`.tsx` file exceeds **400 lines** | Review its responsibilities; split only when a meaningful ownership or reuse boundary exists |
| Template section exceeds **200 lines** | Look for an independent region or repeated unit; keep it intact when the markup is one cohesive view |
| `<script>` section exceeds **250 lines** | Move only independently owned domain/API behavior to a composable, hook, or service |
| A single `<template v-if/v-else-if>` chain has **3+ branches** | Use a step registry/dynamic component when branches are independent screens; keep simple conditional fragments local |
| Multiple dialogs in one file | Extract dialogs when they have separate state, validation, lifecycle, or reuse needs |

Line counts are review signals, not hard limits. Do not split a cohesive file merely to meet a number: extra files, prop plumbing, and artificial abstractions are worse than a readable, focused component. Split when it makes ownership, testing, navigation, or reuse materially clearer. When splitting, the parent keeps route/page orchestration (state coordination, navigation, layout, guards, and final submit). Children own their local fields, validation, and visual behavior through explicit props and events. Before restructuring a feature, load [references/architecture-checklist.md](references/architecture-checklist.md) for ownership, service, fixture, composable, and shared-component decisions.

## Data and View Separation

### Mock data must not live in component state

Never hardcode large mock objects inside `data()`, `setup()`, `ref()`, or template literals. If the page needs placeholder data during development:

1. Put it in a separate `mock.js` / `__fixtures__/` file.
2. Import it explicitly so removal is a one-line change.
3. Mark it clearly: `// TODO: replace with API response`.

```js
// WRONG — mock data baked into data()
data() {
  return {
    taskList: [
      { id: 1, name: '任务A', status: 'pending', /* ...50 more lines */ },
      { id: 2, name: '任务B', status: 'done', /* ... */ }
    ]
  }
}

// CORRECT — mock data in a dedicated file
import { mockTaskList } from './mock'
data() {
  return {
    taskList: mockTaskList // TODO: replace with API response
  }
}
```

### No hard-coded content in templates

If a template contains static text that represents business data (names, descriptions, dates, numbers), it must be data-bound, not written as literal text in the markup. The only exception is immutable UI text (labels, column headers, button text, section titles).

```html
<!-- WRONG — business data as literal text -->
<strong>25人</strong>
<strong>是</strong>
<p>第一党支部成立于2015年，现有党员25人……</p>

<!-- CORRECT — data-bound -->
<strong>{{ row.memberCount }}人</strong>
<strong>{{ row.hasGroup ? '是' : '否' }}</strong>
<p>{{ row.reportContent }}</p>
```

### Configuration data must be external

Status maps, dimension configs, option arrays, and step definitions should be module-level constants or in a `constants/` file — not recreated on every render or buried inside `data()`.

```js
// Module-level constant — good
const STATUS_MAP = {
  pending: { label: '待处理', type: 'warning' },
  done: { label: '已完成', type: 'success' }
}

// Inside data() — bad
data() {
  return {
    statusMap: { pending: { ... }, done: { ... } } // recreated per instance
  }
}
```

## Redundancy and Duplication

### Extract only a real shared contract

- Extract a shared component or composable when two or more consumers have the same invariant behavior and public contract.
- Keep similar-looking code local when its domain language, validation, lifecycle, or data shape differs.
- Prefer domain-named components with explicit props/events over generic `BaseThing` components with many mode flags.
- Put cross-feature primitives in the project's shared UI location only after a second meaningful consumer exists.
- Extract repeated request-state handling, transformations, or validation only when they have the same input/output contract.

### Merge identical methods

If two or more methods have the same logic and differ only by the target property, merge them into one parameterized method. Do not create `handleXxxUploadChange`, `handleYyyUploadChange`, `handleZzzUploadChange` when they all do `this[target] = fileList.slice()`.

```js
// WRONG — three identical methods
handleLegalEntityUploadChange(file, fileList) { this.legalEntityFiles = fileList.slice() },
handleReportUploadChange(file, fileList) { this.reportFiles = fileList.slice() },
handleBranchCommitteeUploadChange(file, fileList) { this.branchCommitteeFiles = fileList.slice() }

// CORRECT — one method
handleUploadChange(target, file, fileList) {
  this[target] = fileList.slice()
}
```

### Avoid string-based dynamic property access

`this[stringVariable]` breaks IDE refactoring, type checking, and is error-prone. Prefer explicit property access or a lookup map.

```js
// AVOID
removeRow(listName, index) {
  this[listName].splice(index, 1)
}

// PREFER — explicit, or use a map
removeDepartmentRow(index) {
  this.departmentRows.splice(index, 1)
}
```

### No empty or placeholder methods

If a method is bound in the template but does nothing (`handleQuery() {}`), either implement it with the intended logic or remove the binding. An empty method misleads readers into thinking functionality exists.

```js
// WRONG
handleQuery() {},  // template binds @click="handleQuery"

// CORRECT — implement it
handleQuery() {
  this.pagination.page = 1
  this.fetchList()
}
```

## State Mutation Safety

### Never mutate a shared object reference directly

When opening a dialog or editing a row, create a copy. Mutating the original object causes unintended side effects on the parent list and makes "cancel" impossible.

```js
// WRONG — direct reference mutation
openEditDialog(row) {
  this.activeRow = row          // reference!
},
saveEdit() {
  this.activeRow.name = newName // mutates the original list item
}

// CORRECT — copy on open, write back on save
openEditDialog(row) {
  this.activeRow = { ...row }   // shallow copy
},
saveEdit() {
  const index = this.list.findIndex(item => item.id === this.activeRow.id)
  if (index > -1) {
    this.$set(this.list, index, { ...this.activeRow })
  }
}
```

### Reset state completely on close/cancel

When a dialog closes, reset all related state (form, steps, file lists, flags). Incomplete resets cause stale data to bleed into the next open.

```js
// WRONG — only resets some fields
handleDialogClose() {
  this.currentStep = 0
  // forgot to reset form, file lists, active task
}

// CORRECT — full reset
handleDialogClose() {
  this.currentStep = 0
  this.currentTask = null
  this.reportForm = this.getDefaultForm()
  this.fileList = []
}
```

## Form Validation

- Every form with a submit action must have validation rules (`:rules` in Element UI, schema validation in React).
- The submit handler must call validation before proceeding. Never go straight from click to success message.
- Mark required fields with both visual indicators and validation rules.

```js
// WRONG — no validation
handleSubmit() {
  this.$message.success('提交成功')
}

// CORRECT — validate first
async handleSubmit() {
  try {
    await this.$refs.form.validate()
  } catch {
    this.$message.warning('请完善必填项')
    return
  }
  // proceed with submission
}
```

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Component files | PascalCase | `TaskBoard.vue`, `UserList.tsx` |
| Directories | kebab-case | `task-board/`, `user-management/` |
| Components (registered name) | PascalCase | `TaskBoard`, `UserList` |
| Methods / functions | camelCase, verb-first | `fetchList`, `handleQuery`, `openDialog` |
| Computed / getters | camelCase, noun or is/has prefix | `filteredList`, `isViewMode`, `hasPermission` |
| Data properties | camelCase, noun | `taskList`, `searchForm`, `dialogVisible` |
| Constants (module-level) | UPPER_SNAKE_CASE | `STATUS_MAP`, `MAX_PAGE_SIZE` |
| CSS classes | kebab-case, BEM when scoped | `.task-board__header`, `.is-active` |
| Events | kebab-case (Vue) / camelCase (React) | `@update:content`, `onChange` |

### Event handler naming

- `handle` prefix for DOM/component event handlers: `handleQuery`, `handleSubmit`, `handleSizeChange`.
- `on` prefix only when passing as props in React: `onSubmit`, `onChange`.
- `before`/`after` prefix for lifecycle-adjacent logic: `beforeOpen`, `afterClose`.

## Style Guidelines

### SCSS/CSS rules

- Always use `scoped` (Vue) or CSS Modules (React) unless the styles are intentionally global.
- Use `::v-deep` (Vue 2) or `:deep()` (Vue 3) for penetrating scoped styles into child components. Do not mix both syntaxes in one project.
- Extract colors to variables or theme files. Never hardcode the same hex color in 10+ places.
- Shared visual patterns (status tags, card containers, table headers) should be in a common style file, not copy-pasted across every page.
- Responsive breakpoints: use `@media` queries for tablet/mobile. Test at 768px and 480px breakpoints minimum.

### Color consistency

If the project uses a theme color (e.g. `#d2363b` for a red-themed dashboard), define it once as a SCSS variable and reference it everywhere. Do not hardcode the hex value in every component.

```scss
// In a shared _variables.scss
$--color-primary: #d2363b;

// In component styles
::v-deep .el-button--primary {
  background-color: $--color-primary;
  border-color: $--color-primary;
}
```

## API and Async

### Discover API conventions before writing an endpoint

Before adding or changing an API call, search the whole repository for:

1. The existing HTTP client and its configured base URL, interceptors, error handling, and authentication behavior.
2. API/service directories and files for the same business domain, resource name, route fragment, or endpoint method.
3. Existing request/response types, query hooks/composables, fixtures, and callers for related data.
4. The project's pagination, upload, response-envelope, and error-message conventions where relevant.

Add the endpoint to the existing domain API/service module used by that feature, such as `src/api/users.ts` or `features/users/services/userService.ts`. Reuse its client, types, naming, and response mapping. Do not create a random new API file or call `fetch`/`axios` directly from a component when the project already has an API layer. Create a new domain module only when the search confirms no suitable module exists; mirror the closest existing layout.

### Never hardcode API roots

API base URLs belong in environment files (`.env.development`, `.env.production`), not in component code. Use the project's configured HTTP client (axios instance, fetch wrapper).

### Error handling pattern

Every async operation (API call, file upload, data fetch) must have:
1. A loading state flag
2. A try/catch or equivalent error boundary
3. A user-visible error message (not just `console.error`)
4. A finally block to reset the loading state

```js
async fetchList() {
  this.loading = true
  try {
    const res = await getList(this.queryParams)
    if (res.code === 200) {
      this.tableData = res.data.records || []
      this.pagination.total = res.data.total || 0
    } else {
      this.$message.error(res.msg || '获取数据失败')
    }
  } catch (error) {
    console.error('获取列表失败:', error)
    this.$message.error('获取数据失败，请稍后重试')
  } finally {
    this.loading = false
  }
}
```

## Pre-delivery Checklist

Before reporting a frontend task as complete, verify every applicable item:

- [ ] **Formatting**: No multi-statement lines, no compressed CSS, no truncated single lines. File passes `npm run lint`.
- [ ] **Component boundaries**: Files over 400 lines were reviewed for mixed responsibilities; cohesive files were not split only to satisfy a line count, while independent dialogs and step views have clear owners.
- [ ] **Data separation**: No large mock data in `data()`/`setup()`. No business data as literal template text. Config constants are module-level.
- [ ] **API placement**: Searched existing API/client/types/callers first. The endpoint uses the established domain API/service module and HTTP client.
- [ ] **No redundancy**: No identical methods differing only by target property. No copy-pasted upload/change handlers.
- [ ] **Mutation safety**: Dialog editing uses copies, not direct references. Close/cancel fully resets state.
- [ ] **Form validation**: Submit actions call validation first. Required fields have rules.
- [ ] **Naming**: Follows the convention table above. No mixed casing styles.
- [ ] **Styles**: Scoped, no hardcoded repeated colors, responsive breakpoints tested.
- [ ] **Error handling**: Every async has loading + try/catch + user-visible error + finally.
- [ ] **No dead code**: No empty methods, no unused imports, no commented-out blocks left behind.
