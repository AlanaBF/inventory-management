---
name: vue-reuse-audit
description: Audit Vue 3 components in client/src/ for code reuse opportunities — repeated state+logic that should be composables, duplicate template blocks, prop drilling chains, inline utilities that belong in shared helpers, and API calls that bypass the centralized client. Report-only; produces a structured findings list with file:line references and extraction sketches. Use when the user asks to find reuse opportunities, extract composables, reduce duplication, or audit the frontend for DRYness.
---

# Vue Reuse Audit

This skill scans the Vue 3 frontend (`client/src/`) for code reuse opportunities and produces a structured report. **It does not modify files** — it reports findings with locations and concrete extraction suggestions for the user to apply manually.

## When to use

Run this audit when the user asks to:
- Find code that should be extracted into composables
- Reduce duplication across components
- Audit the frontend for DRYness or refactoring opportunities
- Identify prop drilling or repeated patterns
- Review reuse before adding a similar component

Do **not** use this skill for: single-file performance tuning, reactivity bug fixes, styling-only changes, or actually implementing the extractions. The user reads the report and chooses what to apply.

## Scope

- **Reads only**: `client/src/views/*.vue`, `client/src/components/*.vue`, `client/src/composables/*.js`, `client/src/api.js`, `client/src/App.vue`, `client/src/main.js`.
- **Skips**: `node_modules/`, `dist/`, build artifacts, the backend (`server/`), tests.
- **Always reads the current state on disk** — do not rely on prior conversation context, since components change between audits.

## Process

1. **Inventory the tree.** List every `.vue` file under `client/src/` and every file in `client/src/composables/`. Record which composables already exist (e.g., `useFilters`, `useAuth`, `useI18n`) so suggestions don't propose duplicates.
2. **Read every `.vue` file in scope, in full.** Partial reads miss duplication that lives at the bottom of a file. Read `api.js` and `App.vue` too.
3. **Scan each finding category below.** For each pattern, record the locations (file path + line number range) and a short snippet showing the duplication.
4. **Sort findings by impact**, not by category. A widely-duplicated fetch trio outranks a CSS color used five times.
5. **Produce the report** using the exact output format at the end of this file.

## Finding categories

### 1. State + logic that should be a composable

Look for the same `ref` / `reactive` / `computed` / `watch` patterns appearing in two or more components:

- **Fetch trio**: `const data = ref(null); const loading = ref(false); const error = ref(null);` plus a try/catch wrapping an `api.*` call → candidate for `useApiResource(fetcher)`.
- **Modal open/close**: `const showModal = ref(false); const selected = ref(null); function open(x) {...} function close() {...}` → candidate for `useModal()`.
- **Local filter shadowing**: components that mirror `useFilters` state in local refs instead of consuming it directly. Almost always a bug or an unnecessary copy.
- **Form state**: same field refs + reset + validate pattern across modals.
- **Pagination / sort state**: repeated `currentPage`, `pageSize`, `sortKey`, `sortDir` blocks.
- **Polling / interval**: `setInterval` + `onUnmounted(clearInterval)` repeated → candidate for `useInterval(fn, ms)`.

Report each as: pattern name → files involved → proposed composable signature.

### 2. Duplicate template blocks

Look for repeated markup that could become a component or slot:

- Card / table / list-item structures that appear in 3+ views with the same outer layout.
- Status badges (color + label mapped from a status string).
- Empty-state placeholders ("No orders found", "No items", etc.).
- KPI / stat tiles repeated across Dashboard, Reports, Spending.
- Inline SVG icons inlined per component (candidate for an `<Icon>` component or shared imports).
- Filter-result counters ("Showing X of Y").

For each: list locations and propose either a new component (e.g. `<StatusBadge>`, `<KpiTile>`, `<EmptyState>`) or a slot/template extension to an existing one.

### 3. Prop drilling and emit chains

Look for:

- The same prop passed through 2+ intermediate components without being read in the middle.
- A child's `emit` immediately re-emitted by its parent (the "pass-through emit").
- Filter state, auth state, or i18n state passed as props instead of consumed via the existing `useFilters` / `useAuth` / `useI18n` composables.

For each: trace the chain (`A.vue → B.vue → C.vue`) and recommend `provide/inject`, a composable, or pulling the existing composable directly into the leaf.

### 4. Inline utilities that belong in a shared helper

Look for:

- Date formatting done inline in templates (`new Date(x).toLocaleDateString(...)` repeated, or manual `getMonth() + 1` patterns).
- Currency formatting (`x.toLocaleString('en-US', { style: 'currency', ... })`).
- Status → color / label mappings inlined in `<script setup>` or templates.
- Number truncation, percentage formatting, pluralization ("1 item" vs. "N items").
- Inline sort or filter helpers duplicated in `<script setup>` blocks.
- Repeated month-name lookups (`['Jan', 'Feb', ...]`).

Propose a `client/src/utils/format.js` (or similar) module and list each call site. Before proposing a new util, check whether an existing composable already covers it.

### 5. API calls bypassing `client/src/api.js`

This codebase deliberately centralizes API calls in `client/src/api.js`. Flag:

- Direct `axios` or `fetch` usage in any `.vue` file.
- Repeated query-param building (skipping `'all'` filter values, formatting `YYYY-MM` months, quarter coercion) duplicated outside `api.js`.
- Components that wrap an existing `api.*` call with their own loading/error trio (cross-link to category 1 — usually one composable fixes both).

Each finding should reference both the component(s) and the suggested addition to `api.js`.

### 6. CSS / style duplication

Look for:

- The same hex color repeated 5+ times across components instead of a CSS variable. Slate palette in this project: `#0f172a`, `#64748b`, `#e2e8f0`.
- Identical scoped style blocks across components (candidate for `App.vue` globals or a shared utility class).
- Repeated `display: grid; grid-template-columns: ...` patterns with the same column shape.
- Repeated transition / hover styles.

Propose either CSS variables in `App.vue` or shared utility classes. Do **not** suggest a CSS framework — this project uses hand-written CSS deliberately.

### 7. Oversized components

Flag any `.vue` file > 400 lines. For each:

- Note total line count and rough breakdown (template / script / style).
- Suggest 2–4 concrete sub-components to extract based on cohesive sections of the template.
- Cross-reference category 2 — extractions often line up with patterns already duplicated elsewhere.

## Rules

- **Report only.** Never use Edit, Write, or NotebookEdit on any `.vue`, `.js`, or `.css` file during this skill. Reading is fine.
- **Cite file:line for every finding.** Suggestions without locations are not actionable.
- **Don't propose composables that already exist.** Check `client/src/composables/` before naming a new one. If an existing composable could be extended instead, say so.
- **Prefer extending `api.js` over creating new API modules.** This codebase centralizes there on purpose.
- **Don't propose extracting something used only once.** Two occurrences = mention briefly; three or more = a full finding entry. Single use only earns a finding if it's a clear hotspot (e.g., a leaf component shadowing `useFilters`).
- **Order findings by impact**, not by the order of categories in this file. Highest-reuse-value items first.
- **No emojis in the report** — matches project UI conventions.
- **Don't recommend external libraries** as a way to dedupe. Stick to project conventions: Composition API, hand-written CSS, axios via `api.js`.

## Output format

Produce a single markdown report with this exact structure:

```markdown
# Vue Reuse Audit

**Scope:** `client/src/` — N .vue files scanned, M composables already present (`useFilters`, `useAuth`, `useI18n`, ...).

## Summary
- Composable candidates: X
- Component candidates: Y
- Prop drilling chains: Z
- Inline utilities to extract: W
- API calls bypassing api.js: V
- CSS duplication clusters: U
- Oversized components (>400 lines): T

## Findings

### 1. [Category]: [Short pattern name]
**Locations:**
- `client/src/views/Foo.vue:42-58`
- `client/src/views/Bar.vue:101-118`
- `client/src/components/Baz.vue:12-29`

**What's duplicated:**
\`\`\`vue
<!-- 3-8 lines showing the pattern, taken from one of the locations -->
\`\`\`

**Suggested extraction:**
- Name: `useFetchResource(fetcher)` (composable) / `<StatusBadge :status="...">` (component) / `formatCurrency(value)` (util in `client/src/utils/format.js`)
- Signature / API sketch:
  \`\`\`js
  // very short sketch — not a full implementation
  \`\`\`
- Effort: small | medium | large
- Notes: reactivity caveats, conflicts with existing composables, refactor ordering hints.

---

(repeat per finding, ordered by reuse value: highest impact first)

## Not flagged (and why)
Short bulleted list of patterns that looked like duplication but are intentional or low-value to extract. Keeps the user from re-litigating each scan and prevents future audits from re-flagging the same items.
```

## Worked example (illustrative)

If `Orders.vue`, `Inventory.vue`, and `Backlog.vue` all contain:

```js
const items = ref([])
const loading = ref(false)
const error = ref(null)

async function load() {
  loading.value = true
  error.value = null
  try {
    items.value = await api.getOrders(/* params */)
  } catch (e) {
    error.value = e.message
  } finally {
    loading.value = false
  }
}
```

That's a category-1 finding (fetch trio). The report entry would name `useApiResource(fetcher)`, list all three file:line ranges, sketch the composable signature, mark effort as **medium** (three call sites to migrate), and note that the existing `useFilters` composable should remain the source of filter params passed into `fetcher`.

## Key reminders

- This skill produces a report, not a patch.
- Two occurrences = mention; three or more = full finding.
- Always check `composables/` and `api.js` before suggesting new names.
- Sort by impact, not by category.
- File:line citations are mandatory.
