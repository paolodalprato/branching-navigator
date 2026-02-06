# Check Report

**Project**: `D:\GITHUB\choicemap`
**Date**: 2026-02-06
**Checks**: structure, code-review

---

## Summary

| Check | Status | Issues |
|-------|--------|--------|
| Structure | ⚠️ Warning | 1 |
| Code Review | ⚠️ Warning | 8 |

---

## Structure Analysis

### Metrics

| Metric | Value |
|--------|-------|
| Total modules (JS) | 1 |
| Entry points | `shared-utils.js` |
| Total lines (code files) | 3.272 |
| Largest file | `scenario-editor.html` (1.566 lines) |
| Average module size | 394 lines |
| Total imports | 0 |
| Health | **warning** |

### File breakdown

| File | Lines | Role |
|------|-------|------|
| `scenario-editor.html` | 1.566 | Main editor (inline React + JSX) |
| `theme-editor.html` | 496 | Theme configuration editor |
| `navigator.html` | 445 | Runtime viewer / decision tree player |
| `shared-utils.js` | 394 | Shared utility library |
| `shared-styles.css` | 373 | Shared stylesheet |

### Issues

| Severity | Issue | Details |
|----------|-------|---------|
| ⚠️ Warning | **Monolith** | `shared-utils.js` (394 lines, threshold 300). Mixed concerns: sanitization, BFS graph traversal, SVG path calculations, React ErrorBoundary component. |

### Structural Notes

- **No import/export graph**: the project uses global `window.ChoiceMapUtils` pattern (IIFE) instead of ES modules. Structure-agent detects only 1 module because `<script type="text/babel">` blocks embedded in HTML are invisible to static analysis.
- **Real complexity lives inside HTML files**: `scenario-editor.html` alone contains ~1.470 lines of inline JSX with 12 React components. This is the actual architectural bottleneck.
- **No circular dependencies**: the dependency model is trivially linear (`shared-utils.js` → consumed by 3 HTML files via global).
- **No dead exports**: all exported utilities in `ChoiceMapUtils` are consumed.

---

## Code Review

### 🔴 Critical

#### 1. `scenario-editor.html` is a 1.566-line monolith

The scenario editor embeds 12 React components plus all business logic in a single `<script type="text/babel">` block. Components: `TreeMapOverlay`, `NodeList`, `CreateNodePopup`, `ChoiceEditor`, `ResourceEditor`, `ResizableTextarea`, `NodeEditor`, `MetaEditor`, `ScenarioEditor` (main), plus factory functions and helpers.

**Impact**: hard to navigate, test, or maintain independently. Any change requires reading the entire file.

**Suggestion**: extract components into separate `.jsx` files and use a build step (Vite, esbuild), or at minimum split into multiple `<script>` blocks loaded in order.

---

### 🟡 Important

#### 2. Module-level mutable state shared with React

Both `navigator.html` and `scenario-editor.html` use module-level `let` variables (`fontOptions`, `defaultTheme`, `layoutConfig`) that are mutated during `useEffect` and read inside render. The code comments explain this is safe because loading state blocks render, but this pattern is fragile: any future change that allows rendering before fetch completion would cause subtle bugs.

**Suggestion**: move these into React state or context, or at minimum document the invariant more prominently.

#### 3. `dangerouslySetInnerHTML` in MarkdownContent (navigator.html:162-168)

The `MarkdownContent` component uses `dangerouslySetInnerHTML` after running `escapeHtml` + regex-based bold/italic parsing. While `escapeHtml` is applied first, the regex replacements re-introduce raw HTML tags (`<strong>`, `<em>`). This is currently safe because the re-introduced tags are hardcoded, but any future extension of the regex patterns could introduce XSS if not carefully reviewed.

**Suggestion**: consider using `React.createElement` to build inline formatting instead of HTML string injection, or add a comment warning about the security boundary.

#### 4. `getNodeLevel` called per-node without caching in `TreeMapOverlay` connections (scenario-editor.html:413-414)

Inside the `TreeMapOverlay` render, `getNodeLevel` is called for every connection endpoint. Without passing a pre-calculated `levelsMap`, each call triggers a full BFS traversal (`calculateNodeLevels`). For a scenario with N nodes and M connections, this results in O(M × N) BFS traversals.

**Same issue in `NodeList` component** (line 628): `getNodeLevel` called per node in the sidebar list.

**Note**: `navigator.html` (line 205-209) already does this correctly by pre-calculating `levelGroups` once.

**Suggestion**: compute `calculateNodeLevels` once at the top of `TreeMapOverlay` and pass the result as `levelsMap` parameter, same pattern as `navigator.html`.

#### 5. Preview component in `theme-editor.html` uses unsanitized template literals

The `Preview` component (theme-editor.html:133-213) builds HTML via template literals using `theme.brand.name`, `theme.brand.logo`, `theme.brand.website`, and `fileName`. These values come from user-edited JSON but are interpolated into `srcDoc` HTML without escaping.

**Impact**: a crafted theme JSON with `"name": "<img src=x onerror=alert(1)>"` would execute in the iframe context. The iframe is same-origin due to `srcDoc`, so it has full access to the parent page.

**Suggestion**: apply `escapeHtml()` (available via `ChoiceMapUtils`) to all interpolated values in the preview HTML template.

#### 6. Babel standalone in production

All three HTML files load `babel-standalone` (930 KB gzipped) to transpile JSX at runtime. This is acceptable for a static-file-only project but adds significant load time and blocks rendering.

**Suggestion**: if the project grows, consider a build step to pre-compile JSX. For now, document this as an intentional trade-off for zero-build-step simplicity.

---

### 🟢 Suggestions

#### 7. Duplicated layout initialization pattern

Each HTML file independently fetches `defaults.json` and `config.json` with cache-busting, parses layout config, and initializes module-level state. This pattern is repeated 3 times with slight variations.

**Suggestion**: extract a shared `loadConfig()` function into `shared-utils.js` that returns `{ config, defaults, fontOptions, layoutConfig }`.

#### 8. `shared-utils.js` mixes concerns

The file bundles: input sanitization, BFS graph traversal, SVG geometry (connection paths, arrow positions), color logic, and a React class component (ErrorBoundary). These are unrelated domains.

**Suggestion**: if/when a build step is introduced, split into `sanitization.js`, `graph.js`, `svg-paths.js`, `error-boundary.jsx`. For now, the section comments provide adequate navigation.

---

## Next Steps

- [ ] **P1**: Sanitize interpolated values in theme-editor Preview component (issue #5 — XSS risk)
- [ ] **P1**: Pre-calculate `levelsMap` in `TreeMapOverlay` and `NodeList` to avoid O(M×N) BFS (issue #4 — performance)
- [ ] **P2**: Evaluate splitting `scenario-editor.html` into multiple files (issue #1 — maintainability)
- [ ] **P3**: Extract shared config loading pattern into `shared-utils.js` (issue #7 — DRY)
- [ ] **P3**: Consider module-level mutable state refactoring (issue #2 — robustness)
