---
description: "Design-system gap analysis — cross-reference a project's components against a design system's catalog (16+ systems) and classify each as Present / Missing / Partial / Extra / Mis-sourced. Works on Flutter and Vite. Feeds /generate."
argument-hint: "[:design-system] [:stack] [:report]  e.g. :material3, :tailwind-shadcn, :bootstrap5:vite:report"
---

# /design-system-audit — Design System Gap Analysis

You are an expert UI architect cross-referencing a project's component inventory against a design system's catalog. This is the **design-system gap** concern, split out of the monolithic `/audit` so it can run on its own. Works on both **Flutter** and **Vite**, against 16+ design systems.

This command is deliberately single-concern. For the other audit axes:

- **Component inventory + decomposition** (V1-V3, V5) → `/atomic-design-toolkit:component-audit`.
- **Bundle health** (V4, Vite) → `/atomic-design-toolkit:bundle-audit`.
- **Atomic-design enforcement** (hooks / CI / ownership cure presence) → `/atomic-design-toolkit:enforcement-audit`.
- **Story coverage** (per pillar) → `/atomic-design-toolkit:storybook-audit`.
- **Unified report + the four together** → `/atomic-design-toolkit:audit`.

## Usage

```bash
# Design system gap analysis — auto-detect stack
/atomic-design-toolkit:design-system-audit :material3
/atomic-design-toolkit:design-system-audit :bootstrap5
/atomic-design-toolkit:design-system-audit :tailwind-shadcn
/atomic-design-toolkit:design-system-audit :carbon

# Force a stack
/atomic-design-toolkit:design-system-audit :vite :bootstrap5
/atomic-design-toolkit:design-system-audit :flutter :material3

# Emit the gap section to a report
/atomic-design-toolkit:design-system-audit :vite :tailwind-shadcn :report
```

The design-system flag is required (the gap analysis is meaningless without a catalog to compare against). Stack auto-detects from `pubspec.yaml` / `vite.config.*` unless `:flutter` / `:vite` is supplied.

## Step 0: Resolve the Design System + Inventory

1. **Detect the stack** (`pubspec.yaml` → Flutter, `vite.config.*` or `"vite"` in `package.json` → Vite) so the generated/recommended components match the project's framework.
2. **Get the project's current component inventory.** When invoked by `/atomic-design-toolkit:audit`, the orchestrator passes the inventory in. When run standalone, do a quick inventory pass (or call `/atomic-design-toolkit:component-audit` first) so there is something to compare against the catalog.

## Process

Use Context7 MCP to query current docs for the selected design system. Read `${CLAUDE_PLUGIN_ROOT}/references/design-systems/` for pre-compiled component catalogs (the ones marked ✅ below).

1. **Resolve library**: Use Context7 `resolve-library-id` for the design system.
2. **Query components**: Get the full component catalog.
3. **Cross-reference**: Compare against the project's existing components.
4. **Classify gaps**: For each catalog component, classify as **Present** / **Missing** / **Partial** (exists but missing variants/states) / **Extra** (project has it, catalog doesn't) / **Mis-sourced** (hand-rolled where the DS ships an equivalent). Propose an atom / molecule / organism classification for each missing one.
5. **Generate** (via `/atomic-design-toolkit:generate`): Create the missing components using the design system's patterns adapted to the project's framework.

The **Partial** and **Mis-sourced** classes are what catch design-system conformance debt — a hand-rolled `<button>` where a DS `Button` exists, or a `Card` missing the catalog's `loading` / `disabled` variants. Surface those explicitly; they are the highest-value gap findings.

## Supported Design Systems

| System | Stack fit | Key Unique Components | Pre-compiled |
|--------|-----------|----------------------|--------------|
| Material 3 | Flutter, Vite (Material Web) | Badge, BottomSheet, Chip, NavigationRail, SearchBar | ✅ |
| MUI | Vite + React | SpeedDial, Stepper, DataGrid (X), Stack, Autocomplete | ✅ |
| Cupertino | Flutter | ActionSheet, ContextMenu, DatePicker, SegmentedControl | ✅ |
| Fluent 2 | Vite (Fluent UI React) | Persona, Toolbar, DataGrid, InfoBar, TeachingBubble | via Context7 |
| Carbon | Vite (React / Web Components), Flutter | StructuredList, ContentSwitcher, InlineNotification, TreeView | ✅ |
| Bootstrap 5 | Vite (vanilla / jQuery / Blazor) | Offcanvas, Toast, Accordion, Carousel, ScrollSpy | ✅ |
| Tailwind + shadcn | Vite (React) | Command, DataTable, InputOTP, Sheet, Resizable | ✅ |
| Vuetify | Vite + Vue 3 | VDataTable, VStepper, VExpansionPanels, VVirtualScroll | ✅ |
| Headless UI | Vite (React / Vue) | Accessibility-first patterns (ARIA, keyboard nav, focus trap) | via Context7 |
| Atlassian | Vite (React) | InlineEdit, DynamicTable, Flag notifications | via Context7 |
| Cloudscape | Vite (React) | AppLayout, SplitPanel, Wizard | via Context7 |
| Primer | Vite (React / CSS) | Timeline, StateLabel, Blankslate | via Context7 |
| Polaris | Vite (React) | AccountConnection, CalloutCard, IndexTable | via Context7 |
| Spectrum | Flutter, Vite (React Aria) | CoachMark, IllustratedMessage, StatusLight | via Context7 |
| Lightning | Vite (React / Web Components) | ActivityTimeline, DataTable, Path | via Context7 |
| Ant Design | Vite (React / Vue) | Cascader, Transfer, TreeSelect, Watermark, Tour | ✅ |
| Chakra UI | Vite (React) | Editable, PinInput, Show/Hide | via Context7 |
| Radix UI | Vite (React) | HoverCard, NavigationMenu, ScrollArea | via Context7 |

Catalogs marked ✅ are pre-compiled in `references/design-systems/`. The rest resolve through Context7 MCP at query time.

### Adaptive Components (Flutter)

For platform-adaptive widgets, generate both Material and Cupertino variants:

```dart
class AdaptiveSwitch extends StatelessWidget {
  const AdaptiveSwitch({super.key, required this.value, required this.onChanged});
  final bool value;
  final ValueChanged<bool> onChanged;

  @override
  Widget build(BuildContext context) {
    return switch (Theme.of(context).platform) {
      TargetPlatform.iOS || TargetPlatform.macOS =>
        CupertinoSwitch(value: value, onChanged: onChanged),
      _ => Switch(value: value, onChanged: onChanged),
    };
  }
}
```

### Component Gallery (master reference)
- **All Components**: https://component.gallery/components/
- **All Design Systems**: https://component.gallery/design-systems/

## Emit the Gap Section

```markdown
## Design System Gap
- System: {Material 3 | Bootstrap 5 | Tailwind + shadcn | MUI | ...}
- Present: N   ·   Missing: N   ·   Partial: N   ·   Extra: N   ·   Mis-sourced: N

| Component | Status | Atomic level | Notes |
|-----------|--------|--------------|-------|
| Button | Present | atom | — |
| Toast | Missing | molecule | hand-rolled alerts found in 4 files → mis-sourced |
| Card | Partial | molecule | missing loading / disabled variants |
```

## Report Mode (`:report`)

When `:report` is present:

1. **Standalone** — emit `.atomic-design-toolkit/reports/design-system-audit-{YYYYMMDD-HHMM}-{stack}.md` with the `## Design System Gap` appendix + a `design-system-gap` frontmatter block (`system`, `present`, `missing`, `partial`) per `${CLAUDE_PLUGIN_ROOT}/references/audit-report-schema.md`. Mis-sourced components become `W`-tier findings (behavior-preserving DS-conformance refactors, default `Phase fit: 3`).
2. **As part of the unified `/audit` report** — when invoked by `/atomic-design-toolkit:audit`, return the `## Design System Gap` appendix + the `design-system-gap` frontmatter block to the orchestrator, which folds it into the single unified report. Do **not** write a separate file in that case.

## Relationship to /audit and /generate

`/design-system-audit` is the design-system-gap slice. `/atomic-design-toolkit:audit` orchestrates this command alongside `/component-audit`, `/bundle-audit`, and `/enforcement-audit`, and composes the unified `:report` that `/migrate` consumes. To *fill* the gaps it finds, hand the missing/mis-sourced list to `/atomic-design-toolkit:generate`. Run this command alone when you only need the DS gap; run `/audit` for the full picture.
