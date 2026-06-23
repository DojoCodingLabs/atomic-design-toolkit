---
description: "Atomic-design enforcement audit (V7) — measure the four-cure defense-in-depth pattern: are the PreToolUse/PostToolUse hooks, .atomic-design-rules.json, agent rules, and CI/pre-commit guards installed? Reports E-findings (future-drift exposure)."
argument-hint: "[:report]  e.g. :report"
---

# /enforcement-audit — Atomic-Design Enforcement (Hook Presence)

You are a repo-health auditor with **one** job: measure whether a project will **drift again tomorrow**. Bundle-health and component-inventory checks tell you *what is broken today*; this command tells you whether the four-cure defense-in-depth pattern is installed to keep it from regressing. This is the **V7 axis**, split out of the monolithic `/audit` so it can run on its own.

It reads `.atomic-design-rules.json` and probes for hooks, CI guards, agent rules, and pre-commit guards — emitting `E{n}` findings (a distinct namespace from `B`/`W`/`I` because E-findings track *future-drift exposure*, not current code defects).

Premortem reference: DOJ-3946 component-layer Conway's Law (dojo-os, 2026-05-14, $394K already spent + $324K forward extrapolation if unaddressed). Read the full setup walkthrough at `${CLAUDE_PLUGIN_ROOT}/references/atomic-design-hooks-setup.md` and the enforcement-signal catalog at `${CLAUDE_PLUGIN_ROOT}/references/vite-audit-checklist.md` §"Atomic-Design Enforcement Signals (V7)".

This command is deliberately single-concern. For the other audit axes:

- **Component inventory + decomposition** (V1-V3, V5) → `/atomic-design-toolkit:component-audit`.
- **Bundle health** (V4) → `/atomic-design-toolkit:bundle-audit`.
- **Design-system gap** → `/atomic-design-toolkit:design-system-audit`.
- **Story coverage** (per pillar) → `/atomic-design-toolkit:storybook-audit`.
- **Unified report + the four together** → `/atomic-design-toolkit:audit`.

## Usage

```bash
/atomic-design-toolkit:enforcement-audit            # run the seven cure-presence checks, print E-findings
/atomic-design-toolkit:enforcement-audit :report    # also emit the enforcement section to a report
```

## The Four Cures

1. **Ownership** — pillar list + atomic taxonomy documented (`.atomic-design-rules.json` + CLAUDE.md / AGENTS.md section).
2. **Harness CI** — workflow runs atomic-design / structural guard on every PR.
3. **Agent rule** — CLAUDE.md or AGENTS.md names the rule so agents inherit it via context.
4. **Hooks** — PreToolUse + PostToolUse hooks block the mistake at write-time (the load-bearing cure — CI is too late, agent rules are advisory, ownership docs are passive).

## Step 1: Run the Seven Checks

Each takes <300ms; the whole step should complete in <2s on a typical repo.

| # | Check | How to detect |
|---|-------|--------------|
| 1 | **Repo-level PreToolUse hook for component placement** | `ls .claude/hooks/ 2>/dev/null \| grep -E 'pre-write.*component\|pre-tool-use.*component'`; then `jq '.hooks.PreToolUse' .claude/settings.json` to confirm registration. |
| 2 | **Repo-level PostToolUse hook for drift signals** | `ls .claude/hooks/ 2>/dev/null \| grep -E 'post-write.*component\|post-tool-use'`; then `jq '.hooks.PostToolUse' .claude/settings.json`. |
| 3 | **Toolkit-level hooks installed** | Probe `package.json` / `bun.lock` / `pnpm-lock.yaml` for a toolkit dependency that ships hooks (e.g. `make-no-mistakes-toolkit@>=1.14`). Confirm with `find node_modules/make-no-mistakes-toolkit/.claude/hooks -type f 2>/dev/null` or the toolkit's documented hook path. |
| 4 | **`.atomic-design-rules.json` at repo root** | `test -f .atomic-design-rules.json && jq -e '.pillars\|length > 0' .atomic-design-rules.json`. Without this file, hooks have nothing to enforce — see `atomic-design-hooks-setup.md` for the canonical shape. |
| 5 | **CLAUDE.md / AGENTS.md atomic-design rule** | `rg -i 'atomic.?design\|pillar.{0,20}taxonomy\|atoms/molecules/organisms' CLAUDE.md AGENTS.md 2>/dev/null \| head -3`. Should name the pillar list + atomic taxonomy + exception marker convention. |
| 6 | **CI workflow with atomic-design lint** | `rg -l 'components-guard\|atomic-design-guard\|structural-guard\|atomic-design-rules' .github/workflows/ .gitlab-ci.yml 2>/dev/null`. Bonus: also check `package.json` `scripts` for an `atomic:lint` or `lint:components` target. |
| 7 | **Pre-commit hook with atomic-design guard** | `cat lefthook.yml .lefthook.yml 2>/dev/null \| rg 'atomic\|component-guard'`; or `ls .husky/ 2>/dev/null` + `rg 'atomic\|component-guard' .husky/`; or `rg 'atomic\|component-guard' .pre-commit-config.yaml 2>/dev/null`. |

### Execution sketch

```bash
# Probes run from repo root
test -f .atomic-design-rules.json && echo "E4 pass" || echo "E4 missing"
rg -l 'atomic.?design|atoms/molecules' CLAUDE.md AGENTS.md 2>/dev/null | head -1 || echo "E5 missing"
ls .claude/hooks/ 2>/dev/null | grep -E 'pre-write.*component' || echo "E1 missing"
ls .claude/hooks/ 2>/dev/null | grep -E 'post-write.*component' || echo "E2 missing"
rg -l 'components-guard|atomic-design-guard|structural-guard' .github/workflows/ 2>/dev/null || echo "E6 missing"
cat lefthook.yml .lefthook.yml 2>/dev/null | rg -q 'atomic|component-guard' || \
  ls .husky/ 2>/dev/null && rg -l 'atomic|component-guard' .husky/ 2>/dev/null || \
  echo "E7 missing"
grep -q 'make-no-mistakes-toolkit' package.json 2>/dev/null && \
  find node_modules/make-no-mistakes-toolkit/.claude/hooks -type f 2>/dev/null | head -1 || \
  echo "E3 missing"
```

## Step 2: Severity Classification

For each missing check, severity depends on whether **component-layer drift** was already detected. When this command runs standalone, ask the caller (or detect via a quick `/atomic-design-toolkit:component-audit` pass) whether V1-V5 drift exists. When invoked by `/atomic-design-toolkit:audit`, the orchestrator passes the drift verdict in.

| State | Severity |
|-------|----------|
| All 4 cures present | `info` — do not emit E-findings; record under "Passing checks" |
| 1-3 cures missing, no V1-V5 drift | `warning` per missing cure |
| 1-3 cures missing, V1-V5 drift detected | `blocker` per missing cure (drift will recur) |
| 0 cures present, no V1-V5 drift | `warning` per missing cure |
| 0 cures present, V1-V5 drift detected | **top E-finding escalated to `blocker`**; compounded risk per premortem §5.2 ($324K forward extrapolation) overrides per-finding severity |

The compounding-risk override exists because a repo with active drift AND no enforcement is the exact failure mode the premortem describes. Do not soften it. When the override fires, the remediation line must include the phrase "compounded risk override per premortem §5.2" so `/migrate` can route it to Phase 1.

## Step 3: What to Record per Finding

Each E-finding takes a stable ID `E{n}` (numbering starts at 1, within tier, stable across re-audits — same convention as `B`/`W`/`I`):

- **Cure Layer** — one of: `Repo PreToolUse hook`, `Repo PostToolUse hook`, `Toolkit hooks`, `.atomic-design-rules.json`, `Agent rule (CLAUDE.md/AGENTS.md)`, `CI workflow`, `Pre-commit hook`.
- **Severity** — per the matrix above.
- **Effort** — `S` / `M` / `L` / `XL` — installing a hook is typically `S`.
- **Risk** — one-liner naming the failure mode the missing cure enables.
- **Phase fit** — `5` (Hardening) by default; `1` (Infrastructure) when the compounded-risk override fires.
- **Signal** — `V7 / E{n}`.
- **Evidence** — file path that was probed and the result (e.g. `ls .claude/hooks/ → no matches for pre-write*component*`).
- **Remediation** — "Install from make-no-mistakes-toolkit v1.14+ per `references/atomic-design-hooks-setup.md`" (link to the relevant section).
- **Blocks / Blocked by** — space-separated finding IDs or `(none)`. E4 typically blocks E1/E2/E3 (the rules file must exist before hooks can enforce anything). E-findings frequently block B/W findings because hooks would have prevented those from regressing.

## Step 4: Emit the Enforcement Section

```markdown
### Atomic-Design Enforcement Findings
| ID | Cure Layer | Severity | Evidence | Remediation |
|----|-----------|----------|----------|-------------|
| E1 | Repo PreToolUse hook (component placement) | warning | (not found at .claude/hooks/) | Install from make-no-mistakes-toolkit v1.14+ — see `references/atomic-design-hooks-setup.md` |
```

The full per-finding schema (`#### E1 — …` heading + the nine fields) is in `${CLAUDE_PLUGIN_ROOT}/references/audit-report-schema.md` §"Atomic-Design Enforcement (E-findings)".

## Report Mode (`:report`)

When `:report` is present:

1. **Standalone** — emit `.atomic-design-toolkit/reports/enforcement-audit-{YYYYMMDD-HHMM}.md` with the E-findings per the schema, plus the `summary.enforcement` frontmatter block (`cures-present`, `cures-missing`, `drift-detected`, `compounded-override`).
2. **As part of the unified `/audit` report** — when invoked by `/atomic-design-toolkit:audit`, return the `### Atomic-Design Enforcement` section + the `summary.enforcement` block to the orchestrator, which folds it into the single unified report. Do **not** write a separate file in that case.

If all four cures are present, emit no E-findings — the four V7 checks instead appear as one-liners under the report's `### Passing checks` section.

## Why this matters

The premortem's claim, in one line: removing any of the four cures leaves a hole the chain of causation will find. This command is the audit's way of telling consumers which holes are open. A project can have perfect inventory, zero monoliths, and clean atomic levels — and still be one quarter away from re-introducing all of them if no hooks are installed.

## Relationship to /audit

`/enforcement-audit` is the enforcement slice (V7). `/atomic-design-toolkit:audit` orchestrates this command alongside `/component-audit`, `/bundle-audit`, and `/design-system-audit`, passes in the drift verdict so the severity matrix resolves correctly, and composes the unified `:report` that `/migrate` consumes. Run this command alone when you only need the enforcement posture; run `/audit` for the full picture.
