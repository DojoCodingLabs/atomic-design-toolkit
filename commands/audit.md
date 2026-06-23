---
description: "Orchestrate the focused audit commands (component / bundle / enforcement / design-system) over a Flutter or Vite codebase and compose the unified report that feeds /migrate"
argument-hint: "[:stack] [:design-system] [:report]  e.g. :vite, :flutter, :material3, :bootstrap5, :audit:vite:bootstrap5:report"
---

# /audit — Orchestrator: Component + Bundle + Enforcement + Design System

You are an expert UI architect running a **full codebase audit** by orchestrating the four focused audit commands and composing their output into one unified report. Works for **Flutter** and **Vite** projects (React / Vue / Svelte / Solid / vanilla JS+TS). Optionally cross-references against 16+ design systems. When run with `:report`, emits a structured markdown file that `/atomic-design-toolkit:migrate` consumes to generate a phased remediation plan.

`/audit` is the **front door**. It does not re-implement the individual checks — it delegates to the single-concern commands, then merges their findings into one report with one unified frontmatter, one finding namespace, and one `## Suggested Phases` appendix. Run a focused command directly when you only need that one axis.

### The audit family

| Axis | Focused command | What it covers |
|------|-----------------|----------------|
| Component inventory + decomposition (V1-V3, V5) | `/atomic-design-toolkit:component-audit` | atoms/molecules/organisms inventory, monoliths, misplaced, duplicated, inline, missing components |
| Bundle health (V4, Vite-only) | `/atomic-design-toolkit:bundle-audit` | the ten signals: duplicate deps, legacy `public/` assets, mixed majors, vendored libs, hashless assets, outdated deps, CI audit coverage, HMR, visual regression, dual lockfiles |
| Atomic-design enforcement (V7) | `/atomic-design-toolkit:enforcement-audit` | the four-cure defense-in-depth: hooks, `.atomic-design-rules.json`, agent rules, CI / pre-commit guards (E-findings) |
| Design-system gap | `/atomic-design-toolkit:design-system-audit` | component catalog cross-reference against 16+ design systems (Present / Missing / Partial / Extra / Mis-sourced) |
| Story coverage per pillar | `/atomic-design-toolkit:storybook-audit` | which renderable components lack a `.stories.tsx`, grouped by pillar + atomic level (**cross-referenced, not invoked** — see Step 4) |

## Usage

```bash
# Full audit — auto-detect stack (runs component + bundle + enforcement)
/atomic-design-toolkit:audit

# Force a stack
/atomic-design-toolkit:audit :flutter
/atomic-design-toolkit:audit :vite

# Add design system gap analysis
/atomic-design-toolkit:audit :material3
/atomic-design-toolkit:audit :bootstrap5
/atomic-design-toolkit:audit :tailwind-shadcn
/atomic-design-toolkit:audit :carbon

# Combined: audit + design system lens
/atomic-design-toolkit:audit :audit:material3
/atomic-design-toolkit:audit :audit:vite:bootstrap5

# Emit a structured report file (input to /migrate)
/atomic-design-toolkit:audit :report
/atomic-design-toolkit:audit :vite :report
/atomic-design-toolkit:audit :audit:vite:bootstrap5:report
```

## Step 0: Stack Detection (once, for the whole run)

Detect the stack **once** here, then pass the verdict to every focused command so they skip re-detection.

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

If the argument starts with `:flutter` or `:vite`, honor it and skip auto-detection. For Vite, sub-detect the framework (React / Vue / Svelte / Solid / vanilla) from `package.json` dependencies — the focused commands document the table; detect it here once and pass it down.

## Step 1: Run the Component Audit

Invoke `/atomic-design-toolkit:component-audit` for the detected stack (V1-V3, V5: inventory + classification + missing components). Collect:

- The `## Component Inventory` table + the `components.*` counts (atoms / molecules / organisms / templates / pages / unclassified).
- Decomposition findings (monoliths, misplaced, dead duplicates, inline) as `B`/`W`/`I` tier.
- A **drift verdict** — did V1-V5 surface any component-layer drift? This feeds the enforcement severity matrix in Step 3.

## Step 2: Run the Bundle Audit (Vite only)

If the stack is Vite, invoke `/atomic-design-toolkit:bundle-audit` (V4: the ten signals). Collect the `### Bundle Health Findings` block + the `summary.pass` count. Skip this step entirely for Flutter (bundle health does not apply).

## Step 3: Run the Enforcement Audit

Invoke `/atomic-design-toolkit:enforcement-audit` (V7: the seven cure-presence checks), **passing in the drift verdict from Step 1** so the severity matrix resolves correctly (drift + zero cures → top E-finding escalates to `blocker` per the compounded-risk override). Collect the `### Atomic-Design Enforcement` section + the `summary.enforcement` frontmatter block.

> Execution order: enforcement (V7) is computed after component + bundle so its severity can react to detected drift — its findings still appear in their own section of the report. The report reads V1 → V2 → V3 → V4 → V5 → V7 → V6 (V6 = report composition, this orchestrator's job).

## Step 4: Cross-Reference Story Coverage (delegate, don't re-implement)

Story coverage is owned by `/atomic-design-toolkit:storybook-audit` (the per-pillar backlog-discovery half of the story-coverage cure). `/audit` does **not** re-run a full coverage sweep. Instead:

- In the report's `### Story / Test Coverage` line, point the user at `/atomic-design-toolkit:storybook-audit` for the per-pillar coverage table, and note any story-missing components Step 1 flagged as `Undocumented`.
- If the project has Storybook configured and the user wants coverage folded into this run, recommend they run `/atomic-design-toolkit:storybook-audit :report` and let `/migrate` consume its missing list as a Phase 5 (Hardening) batch — the same `/migrate` contract.

This keeps story coverage single-concern (its CI gate is DOJ-5319) while `/audit` stays the front door for the other four axes.

## Step 5: Run the Design System Gap Analysis (when a `:design-system` flag is present)

If the argument list includes a design-system flag (`:material3`, `:bootstrap5`, `:tailwind-shadcn`, …), invoke `/atomic-design-toolkit:design-system-audit` with that flag and the detected stack, **passing in the component inventory from Step 1** so it has something to compare against the catalog. Collect the `## Design System Gap` appendix + the `design-system-gap` frontmatter block.

## Step 6: Compose

If `:report` is **not** present, print the merged sections to the conversation: inventory, decomposition findings, bundle health (Vite), enforcement E-findings, and the DS gap (if run). Recommend `/atomic-design-toolkit:audit :report` to emit the structured file, and name the focused commands the user can run for a deeper single-axis pass.

If `:report` **is** present, go to Report Mode below.

---

## Report Mode (`:report`)

When `:report` is present in the argument list, after running the selected audit passes, write a structured markdown report to `.atomic-design-toolkit/reports/audit-{YYYYMMDD-HHMM}-{stack}.md`. This file is the contract `/atomic-design-toolkit:migrate` consumes. The focused commands return their sections to this orchestrator; **only `/audit` writes the unified report file** (the focused commands write their own standalone files only when invoked directly, not under orchestration).

### Process

1. Run the selected audit passes via Steps 1-5 (stack detection, component inventory, bundle health, enforcement, design system gap — whatever the flags request).
2. Read `${CLAUDE_PLUGIN_ROOT}/references/audit-report-schema.md` for the required file format.
3. Classify every finding on **four dimensions**, not just severity:
   - **Severity** — `blocker` / `warning` / `info`
   - **Effort** — `S` (≤2h) / `M` (≤1 day) / `L` (>1 day) / `XL` (>1 week)
   - **Risk** — one-line plain-English consequence of leaving the finding unfixed
   - **Phase fit** — integer 0-5 matching the 6-phase migration model (Baseline / Infrastructure / Dependency consolidation / Feature modules / Legacy asset removal / Hardening)
4. Assign stable IDs: `B{n}` (blockers), `W{n}` (warnings), `I{n}` (info), `E{n}` (atomic-design enforcement findings from the enforcement audit). Numbering within each tier starts at 1 and is stable across re-audits of the same project — if a finding disappears, its ID is retired, not reused.
5. Record dependency relations: every finding's `Blocks` and `Blocked by` lines must reference other IDs in the same report (or `(none)`). E-findings frequently block B/W findings because hooks would have prevented those B/W findings from regressing.
6. Surface **new findings** — ones that did not match any known signal — under a `### New findings` subsection with IDs prefixed `N{n}`. These are candidates for future additions to `vite-audit-checklist.md`.
7. Emit a `## Suggested Phases` appendix listing finding IDs by phase number — this is the seed `/migrate` uses to draft the remediation plan. E-findings typically land in Phase 5 (Hardening) unless V1-V5 surfaced active drift, in which case the top E-finding moves to Phase 1 (Infrastructure) per the compounded-risk override.
8. Emit a `## Re-audit Checklist` appendix — one concrete command per remediated finding so the user can confirm the signal flipped to PASS.

### Output Checklist

- [ ] File written to `.atomic-design-toolkit/reports/audit-{YYYYMMDD-HHMM}-{stack}.md`.
- [ ] Frontmatter includes all required fields from the schema reference.
- [ ] TL;DR has exactly three sentences (stack summary → top risk → recommended next action referencing `/migrate`).
- [ ] Every finding has Severity / Effort / Risk / Phase fit / Signal / Evidence / Remediation / Blocks / Blocked by.
- [ ] Passing checks listed one-line each (no details needed).
- [ ] `.atomic-design-toolkit/` added to `.gitignore` if the team has not opted to commit reports.
- [ ] Atomic-Design Enforcement section present with E{n} findings from the enforcement audit (or "(none)" if all four cures are present).
- [ ] If all 4 cures present, no E findings in report — the four V7 passing checks instead appear under "Passing checks".
- [ ] If 0 cures present **and** component-layer drift was detected, the top E-finding is escalated to **`blocker`** severity (overrides per-finding severity because compounded risk is real per premortem §5.2 — $324K forward extrapolation).
- [ ] E-finding remediations link to `references/atomic-design-hooks-setup.md` and (when applicable) the `make-no-mistakes-toolkit` version that ships the hooks.

### Example Invocation

For the **dojo-os** audit that produced 0 blockers, 2 warnings, and 1 new finding:

```bash
/atomic-design-toolkit:audit :vite :report
# → writes .atomic-design-toolkit/reports/audit-20260418-1530-vite.md
# → print the file path + TL;DR + the suggested /migrate command
```

See the full worked example inside `${CLAUDE_PLUGIN_ROOT}/references/audit-report-schema.md`.

---

## Relationship to the focused commands and /migrate

- **`/audit` orchestrates** `/component-audit` + `/bundle-audit` + `/enforcement-audit` + `/design-system-audit`, cross-references `/storybook-audit`, and is the **only** command that writes the unified report.
- **Each focused command runs standalone** too — invoke one directly when you only need that axis. Under orchestration they return sections to `/audit` and do not write their own files.
- **`/migrate` consumes the unified report** — the schema in `${CLAUDE_PLUGIN_ROOT}/references/audit-report-schema.md` is the byte-for-byte contract between `/audit :report` (producer) and `/migrate` (consumer). It is unchanged by this decomposition: the same frontmatter, the same `B`/`W`/`I`/`E`/`N` finding namespace, the same `## Suggested Phases` and `## Re-audit Checklist` appendices.
- **`/generate` fills the gaps** — hand the missing/mis-sourced component list (from `/design-system-audit`) or the decomposition list (from `/component-audit`) to `/atomic-design-toolkit:generate`.
