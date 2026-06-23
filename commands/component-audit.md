---
description: "Inventory and classify components in a Flutter or Vite codebase — flag monoliths, misplaced, duplicated, and inline components by atomic level. The component-inventory half of /audit (V1-V3, V5)."
argument-hint: "[:stack] [:report]  e.g. :vite, :flutter, :vite:report"
---

# /component-audit — Component Inventory + Classification

You are an expert UI architect scanning a codebase for missing, monolithic, or misplaced components. This is the **component-inventory** concern, split out of the monolithic `/audit` so it can run on its own. Works for **Flutter** and **Vite** projects (React / Vue / Svelte / Solid / vanilla JS+TS).

This command is deliberately single-concern. For the other audit axes, run the sibling commands and compose them with `/atomic-design-toolkit:audit`:

- **Bundle health** (Vite — duplicates, hashless assets, vendored libs, lockfiles) → `/atomic-design-toolkit:bundle-audit`.
- **Atomic-design enforcement** (hooks / CI / ownership cure presence) → `/atomic-design-toolkit:enforcement-audit`.
- **Design-system gap** (component catalog cross-reference) → `/atomic-design-toolkit:design-system-audit`.
- **Story coverage** (which components lack a `.stories.tsx`, per pillar) → `/atomic-design-toolkit:storybook-audit`.
- **Unified report + the four together** → `/atomic-design-toolkit:audit`.

## Usage

```bash
/atomic-design-toolkit:component-audit                 # auto-detect stack
/atomic-design-toolkit:component-audit :flutter        # force Flutter
/atomic-design-toolkit:component-audit :vite           # force Vite
/atomic-design-toolkit:component-audit :vite :report   # also emit the inventory section to a report
```

## Step 0: Stack Detection

Before anything, decide which codebase type we are auditing.

### Detection Rules

| Signal | Conclusion |
|--------|-----------|
| `pubspec.yaml` present at repo root | **Flutter** |
| `vite.config.{js,ts,mjs,cjs}` present at repo root | **Vite** |
| `package.json` with `"vite"` in `devDependencies` | **Vite** |
| Both present | Prefer explicit `:flutter` / `:vite` flag; otherwise ask |
| Neither present | Abort — this audit does not support the stack |

Run this single pass:

```bash
ls pubspec.yaml vite.config.js vite.config.ts vite.config.mjs vite.config.cjs 2>/dev/null
# and, if no vite.config.* surfaced, probe package.json
grep -l '"vite"' package.json 2>/dev/null
```

If the argument starts with `:flutter` or `:vite`, honor it and skip auto-detection.

### Sub-Framework Detection (Vite only)

Scan `package.json` dependencies to classify the Vite variant:

| Dep signal | Framework | File extensions |
|-----------|-----------|-----------------|
| `react` + `react-dom` | React | `.tsx` / `.jsx` |
| `vue` | Vue | `.vue` |
| `svelte` | Svelte | `.svelte` |
| `solid-js` | Solid | `.tsx` / `.jsx` |
| None of the above | **Vanilla** (server-rendered + Vite assets) | `.ts` / `.js` modules + template partials |

Note: the **vanilla** variant is very common (Laravel / Django / Rails / plain HTML projects that adopted Vite only for asset bundling). The Seacrets.Online modernization pattern lives here.

---

## Flutter Path

> Applies when stack is Flutter.

### Step F1: Inventory Existing Widgets

```bash
find . -name "*.dart" -type f \
  | xargs grep -l "extends StatelessWidget\|extends StatefulWidget\|extends ConsumerWidget\|extends HookWidget" 2>/dev/null
```

Classify by atomic level based on directory structure and widget complexity.

### Step F2: Classify Existing Widgets

| Classification | Criteria | Action |
|---------------|----------|--------|
| **Well-decomposed** | Correct granularity for its level | Skip |
| **Monolithic** | >150 lines, mixing layout + logic + state | Flag for decomposition |
| **Misplaced** | In atoms/ but has state management | Flag for reclassification |
| **Duplicated** | Similar widgets across features | Flag for extraction to core/ |
| **Inline** | Widget built inline without its own file | Flag for extraction |

### Step F3: Identify Missing Components

Cross-reference with component.gallery (https://component.gallery/components/):

**Essential Components Checklist:**
- [ ] AppBar variants (standard, search, contextual)
- [ ] Bottom sheets (modal, persistent)
- [ ] Cards (info, action, selection)
- [ ] Chips (filter, input, suggestion)
- [ ] Dialogs (alert, confirmation, full-screen)
- [ ] Empty states (no data, error, offline)
- [ ] Form fields (text, dropdown, date, checkbox, radio, switch, slider)
- [ ] Loading indicators (skeleton, shimmer, circular, linear)
- [ ] Navigation (bottom bar, rail, drawer, tabs)
- [ ] Progress indicators (determinate, indeterminate, step)
- [ ] Search (bar, full-screen, with suggestions)
- [ ] Snackbars / Toasts
- [ ] Tags / Labels / Badges

For a full design-system catalog cross-reference, run `/atomic-design-toolkit:design-system-audit :material3` (or your DS flag) instead — this checklist is the minimal built-in.

### Step F4: Emit the Flutter Inventory Section

```markdown
## Widget Audit Report (Flutter)

### Current Inventory
| Level | Count | Files |
|-------|-------|-------|
| Atoms | N | ... |
| Molecules | N | ... |
| Organisms | N | ... |
| Feature Pages | N | ... |
| Loose Widgets | N | ... |

### Widgets to Decompose
| Widget | File | Lines | Issue | Recommendation |
|--------|------|-------|-------|----------------|

### Missing Components
| Component | Priority | Design System | Needed For |
|-----------|----------|--------------|-----------|

### Duplicated Widgets
| Widget A | Widget B | Similarity | Extract As |
|----------|----------|-----------|-----------|
```

---

## Vite Path

> Applies when stack is Vite. This command covers the **component axes** only: inventory (V1-V3) and missing components (V5). Bundle health (V4) lives in `/bundle-audit`; enforcement (V7) lives in `/enforcement-audit`.

### Step V1: Map Vite Entry Points

Parse `vite.config.{js,ts,mjs,cjs}` to discover:
- `build.rollupOptions.input` — every entry bundle (may be an object keyed by feature)
- `resolve.alias` — path aliases that rewrite imports
- `publicDir` — the un-hashed passthrough folder (default `public/`)
- `base` — deployment base URL
- Plugin list — React, Vue, Svelte, Solid, Tailwind, Vitest, Storybook, Playwright

Every entry listed in `build.rollupOptions.input` is a boundary for de-duplication analysis. (The de-dupe / code-split bundle checks themselves live in `/bundle-audit`.)

### Step V2: Inventory Components

Scan by the framework detected in Step 0:

```bash
# React / Solid
rg -l "export default (function|const) \w+" --type-add 'jsxlike:*.{tsx,jsx}' -tjsxlike

# Vue
rg -l '<script' -g '*.vue'

# Svelte
rg -l '<script' -g '*.svelte'

# Vanilla (Seacrets pattern) — ESM modules are the unit
rg -l "^export " --type ts --type js src/ resources/js/
# Template partials live alongside — discover them by include/extends directives
rg -l "@extends\|@include\|{% extends\|{% include\|<%- include" templates/ views/ resources/views/ 2>/dev/null
```

For the **vanilla** variant, a "component" is typically a triple: partial HTML + ESM module + SCSS/CSS. Flag any module that exists without a corresponding partial (or vice versa) as a decomposition smell.

### Step V3: Classify Existing Components

| Classification | Criteria | Action |
|---------------|----------|--------|
| **Well-decomposed** | Correct granularity + co-located styles + story/test | Skip |
| **Monolithic** | >250 lines (JSX/SFC) or >400 lines (vanilla partial) mixing layout + data + state | Flag for decomposition |
| **Misplaced** | Under `atoms/` but imports a store/context/fetch | Flag for reclassification |
| **Duplicated** | Similar JSX/templates across features | Flag for extraction to `src/components/` |
| **Inline** | Built inline in a page file without its own module | Flag for extraction |
| **Undocumented** | No Storybook/Histoire story and not a page | Flag as story-missing (delegate the full per-pillar coverage sweep to `/atomic-design-toolkit:storybook-audit`) |

Atomic-level file conventions (adopt whatever the project already uses; these are defaults):

| Level | React / Vue / Svelte / Solid | Vanilla + Vite |
|-------|------------------------------|----------------|
| Atom | `src/components/atoms/` | `resources/views/atoms/` + `resources/js/atoms/` |
| Molecule | `src/components/molecules/` | `resources/views/molecules/` + `resources/js/molecules/` |
| Organism | `src/components/organisms/` | `resources/views/organisms/` + `resources/js/modules/` |
| Template | `src/layouts/` or `src/templates/` | `resources/views/layouts/` |
| Page | `src/pages/` or `src/routes/` | `resources/views/pages/` (or controller-rendered) |

### Step V5: Identify Missing Components

Cross-reference with component.gallery (https://component.gallery/components/). For a full design-system catalog cross-reference, run `/atomic-design-toolkit:design-system-audit` with your DS flag — this checklist is the minimal built-in.

**Essential Components Checklist (Vite projects):**
- [ ] App shell (header + nav + main + footer layout)
- [ ] Navigation (top bar, sidebar, mobile drawer, breadcrumbs)
- [ ] Cards (info, action, selection, media)
- [ ] Form controls (text, textarea, select, multi-select, date, time, file, checkbox, radio, switch, slider)
- [ ] Form layout (fieldset, form group, inline validation, error summary)
- [ ] Modals / Dialogs (alert, confirm, full-screen sheet)
- [ ] Toasts / Snackbars / Inline notifications
- [ ] Tables (sortable, filterable, paginated, virtual-scroll)
- [ ] Lists (flat, grouped, virtualized)
- [ ] Empty / Error / Loading states (skeletons, spinners, retry)
- [ ] Tabs / Accordion / Disclosure
- [ ] Tooltip / Popover / HoverCard
- [ ] Badges / Tags / Chips / Pills
- [ ] Avatars / User identity primitives
- [ ] Progress (determinate, indeterminate, stepper)
- [ ] Search (inline, command-palette, with suggestions)
- [ ] Pagination

### Step V6: Emit the Vite Inventory Section

```markdown
## Component Audit Report (Vite)

### Stack Detected
- Framework: {React | Vue | Svelte | Solid | Vanilla}
- Package manager: {npm | pnpm | yarn | bun}
- Build entries: {N entries from vite.config}

### Current Inventory
| Level | Count | Files |
|-------|-------|-------|
| Atoms | N | ... |
| Molecules | N | ... |
| Organisms | N | ... |
| Templates | N | ... |
| Pages | N | ... |
| Loose / unclassified | N | ... |

### Components to Decompose
| Component | File | Lines | Issue | Recommendation |
|-----------|------|-------|-------|----------------|

### Missing Components
| Component | Priority | Design System | Needed For |
|-----------|----------|--------------|-----------|

### Duplicated Components
| Component A | Component B | Similarity | Extract As |
|-------------|-------------|-----------|-----------|
```

---

## Report Mode (`:report`)

When `:report` is present, write the inventory section to a report file under `.atomic-design-toolkit/reports/`. Two ways this is consumed:

1. **Standalone** — emit `.atomic-design-toolkit/reports/component-audit-{YYYYMMDD-HHMM}-{stack}.md` with the inventory tables above plus any `B`/`W`/`I`/`N` findings for decomposition work (monoliths → typically `W`, phantom hierarchies → `W`, dead duplicates → `W`). Follow the finding schema in `${CLAUDE_PLUGIN_ROOT}/references/audit-report-schema.md` so `/migrate` can consume it.
2. **As part of the unified `/audit` report** — when invoked by `/atomic-design-toolkit:audit`, return the `## Component Inventory` appendix + the `components.*` frontmatter counts + decomposition findings to the orchestrator, which folds them into the single unified report. Do **not** write a separate file in that case.

Decomposition findings (monoliths, misplaced, dead duplicates) carry `Phase fit: 3` (Feature modules) by default. See the schema reference for the full finding-field contract.

## Relationship to /audit

`/component-audit` is the component-inventory slice. `/atomic-design-toolkit:audit` orchestrates this command alongside `/bundle-audit`, `/enforcement-audit`, and `/design-system-audit`, cross-references `/storybook-audit`, and composes the unified `:report` that `/migrate` consumes. Run this command alone when you only need the inventory; run `/audit` for the full picture.
