---
description: "Vite bundle-health audit — the ten signals (duplicate deps, legacy public/ assets, mixed majors, vendored libs, hashless assets, outdated deps, CI audit coverage, HMR, visual regression, dual lockfiles). The V4 axis of /audit, split out."
argument-hint: "[:report]  e.g. :report"
---

# /bundle-audit — Vite Bundle Health

You are a frontend modernization auditor with **one** job: measure the bundle-health of a Vite project across ten signals. This is the **bundle-health** concern (the V4 axis), split out of the monolithic `/audit` so it can run on its own. It is the axis Flutter audits do not cover — **this command is Vite-only**.

This command is deliberately single-concern. For the other audit axes:

- **Component inventory + decomposition** (V1-V3, V5) → `/atomic-design-toolkit:component-audit`.
- **Atomic-design enforcement** (hooks / CI / ownership cure presence) → `/atomic-design-toolkit:enforcement-audit`.
- **Design-system gap** (component catalog cross-reference) → `/atomic-design-toolkit:design-system-audit`.
- **Story coverage** (per pillar) → `/atomic-design-toolkit:storybook-audit`.
- **Unified report + the four together** → `/atomic-design-toolkit:audit`.

## Usage

```bash
/atomic-design-toolkit:bundle-audit            # run the ten signals, print findings
/atomic-design-toolkit:bundle-audit :report    # also emit the bundle-health section to a report
```

## Step 0: Confirm Vite Stack

Bundle health is Vite-specific. Confirm the stack before running:

```bash
ls vite.config.js vite.config.ts vite.config.mjs vite.config.cjs 2>/dev/null
grep -l '"vite"' package.json 2>/dev/null
```

If this is a Flutter repo (`pubspec.yaml` present, no `vite.config.*`), abort with a note: bundle-health does not apply to Flutter — run `/atomic-design-toolkit:component-audit :flutter` for the widget inventory instead.

Detect the package manager from the lockfile / `packageManager` field (`npm` / `pnpm` / `yarn` / `bun`) so the audit commands below use the right tool.

## Step 1: Run the Ten Signals

Read the full methodology at `${CLAUDE_PLUGIN_ROOT}/references/vite-audit-checklist.md` — it carries the detection command, severity tier, remediation, and the Seacrets.Online evidence behind each signal. Run at least these ten:

| # | Check | How to detect |
|---|-------|--------------|
| 1 | **Duplicate dependencies** | `npm ls <lib> --all` — more than one version resolved for the same package (jQuery, lodash, Bootstrap, etc.) |
| 2 | **Legacy `public/` assets** | Files under `publicDir` that are `.js` / `.css` and not referenced by any Vite entry — they bypass hashing and bundling |
| 3 | **Mixed major versions of UI libs** | e.g. `bootstrap@4` and `bootstrap@5` co-existing, `@mui/material@4` and `@mui/material@5` |
| 4 | **Vendorized libraries** | A `vendor/`, `public/lib/`, `public/vendor/`, or `resources/lib/` folder holding libs that already exist on npm — unpatched and invisible to `npm audit`. Do NOT scan `src/` (application code). |
| 5 | **Hashless assets in production HTML** | `<script src="/js/app.js">` without a content hash — breaks cache invalidation across deploys. The Vite dev entry `<script type="module" src="/src/main.tsx">` is expected and gets hashed at build time. |
| 6 | **Abandoned / outdated dependencies** | `npm outdated` (or `bun outdated`) showing majors behind, or packages last published >3 years ago |
| 7 | **No dependency audit in CI** | No `npm audit` / `pnpm audit` / `yarn audit` / `bun audit` step in any workflow under `.github/workflows/` |
| 8 | **No HMR in dev** | `vite.config.*` sets `server.hmr: false`, or dev entry does a full reload |
| 9 | **No visual regression** | No Playwright / Chromatic / Percy / Loki config — regressions only caught by manual QA |
| 10 | **Dual / competing lockfiles** | Two lockfiles coexist (e.g. `bun.lock` + `package-lock.json`), or the surviving lockfile does not match the `packageManager` field in `package.json` |

For each finding, record: severity (blocker / warning / info), evidence (file path or command output), and remediation (one-line action). The checklist reference documents how each signal's severity escalates (e.g. signal #7 becomes a blocker when combined with #4 or #6).

## Step 2: Emit the Bundle Health Section

```markdown
### Bundle Health Findings
| # | Signal | Severity | Evidence | Remediation |
|---|--------|----------|----------|-------------|
| 1 | ... | blocker | ... | ... |
```

See the worked snippet at the end of `${CLAUDE_PLUGIN_ROOT}/references/vite-audit-checklist.md` (§"Report Snippet").

## Report Mode (`:report`)

When `:report` is present, the bundle-health findings become structured findings under the audit-report schema:

1. **Standalone** — emit `.atomic-design-toolkit/reports/bundle-audit-{YYYYMMDD-HHMM}-vite.md` with the findings classified per `${CLAUDE_PLUGIN_ROOT}/references/audit-report-schema.md`: stable IDs (`B`/`W`/`I`, plus `N` for novel signals not in the checklist), Severity / Effort / Risk / Phase fit / Signal (`#1`-`#10`) / Evidence / Remediation / Blocks / Blocked by.
2. **As part of the unified `/audit` report** — when invoked by `/atomic-design-toolkit:audit`, return the `### Bundle Health Findings` block + the `summary.pass` count (of the ten signals that passed) to the orchestrator, which folds it into the single unified report. Do **not** write a separate file in that case.

### Phase fit defaults for bundle findings

| Signal | Default Phase fit |
|--------|-------------------|
| #10 dual lockfiles, dead deps | 0 (Baseline) |
| #8 HMR, vite.config hardening | 1 (Infrastructure) |
| #1, #3, #4, #6 dupes / mixed majors / vendored / outdated | 2 (Dependency consolidation) |
| #2 legacy `public/` assets | 4 (Legacy asset removal) |
| #5, #7, #9 hashless assets / CI audit / visual regression | 5 (Hardening) |

Novel signals (`N{n}`) are candidates for promotion into `vite-audit-checklist.md`. Record them under the report's `### New findings` subsection.

## Relationship to /audit

`/bundle-audit` is the bundle-health slice (V4). `/atomic-design-toolkit:audit` orchestrates this command alongside `/component-audit`, `/enforcement-audit`, and `/design-system-audit`, and composes the unified `:report` that `/migrate` consumes. Run this command alone when you only need bundle health; run `/audit` for the full picture.
