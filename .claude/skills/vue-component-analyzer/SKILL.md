---
name: vue-component-analyzer
description: Analyzes Vue 3 component structure and suggests optimizations for performance and code reuse. Use this skill when reviewing or refactoring .vue files in client/src/views or client/src/components.
---

# Vue Component Analysis

This skill provides a checklist for analyzing Vue 3 Composition API components in this project and surfacing concrete performance and reuse opportunities.

## When to Use

- Before refactoring a `.vue` file
- After a new view/component is added, as a lightweight self-review
- When a component feels slow, duplicated, or hard to follow

## Analysis Checklist

### 1. Reactivity Correctness
- `ref()` for mutable primitives/objects, `computed()` for derived values — flag any derived value stored in a `ref` and recalculated manually in multiple places
- Flag `v-for` using `index` as `:key` instead of a stable id (see root `CLAUDE.md` Common Issues #1)
- Flag props being mutated directly instead of emitting an event up

### 2. Performance
- Heavy calculations inside methods (re-run every render) that should be `computed` instead
- Repeated array `.filter()`/`.map()` chains over the same source inside the template — move to a computed
- Missing `v-show` for frequently toggled elements where `v-if` causes repeated mount/unmount
- Watchers without `debounce` on inputs that trigger API calls (see `watchDebounced` pattern in client `CLAUDE.md`)

### 3. Code Reuse
- Duplicated data-loading pattern (`loading`/`error`/try-catch-finally`) across views — candidate for a shared composable
- Duplicated currency/date formatting logic — should use `client/src/utils/currency.js` and existing locale helpers instead of ad hoc formatting
- Filter state duplicated locally instead of using `useFilters()`
- i18n strings hardcoded instead of using `t()` — check both `en.js` and `ja.js` have matching keys

### 4. Error Handling
- A single `error` ref that, when set, replaces the entire template (`v-else-if="error"`) — this hides the whole page for any failure, including non-fatal ones like a failed submit. Prefer a dedicated error ref per action (e.g. `submitError`) rendered inline near the relevant control.

### 5. Structure
- Component >150 lines of script logic → consider extracting a composable
- Repeated markup blocks (e.g. stat cards, table sections) → consider a small reusable component

## Output Format

When analyzing a component, report findings grouped by section above, each with:
- File and line reference
- What's suboptimal
- A concrete before/after suggestion

Keep suggestions scoped to the component under review — don't recommend unrelated refactors.
