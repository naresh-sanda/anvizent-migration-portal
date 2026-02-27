# Code Review: `index.html` -- Anvizent Migration Portal v2

## Overview

`index.html` is a ~4,930-line single-file SPA that implements a data pipeline migration portal with workspace configuration, a 6-step dataset porting wizard, and a migration details overview. It replaces `index_v1.html`, which was a simpler prototype using Chart.js/D3 and a green-themed UI.

The UI design is polished -- the dark sidebar, accordion workspace cards, wizard stepper, dataset browser, and lineage visualization all come together into a cohesive admin-panel experience. That said, there are structural and maintainability concerns worth addressing before this grows further.

---

## Strengths

1. **Clean visual design system** -- CSS custom properties (`--accent`, `--surf`, `--border`, etc.) provide a consistent palette. The indigo/slate theme is well executed.

2. **Well-structured wizard flow** -- The 6-step migration wizard (Pick Dataset -> Lineage -> Target/DDL -> Column Map -> Schedule -> Review) is logically organized with clear step navigation and visual progress indicators.

3. **Smart UX patterns** -- Collapsible sidebar with localStorage persistence, accordion cards, FQN bars that auto-update, cron expression builder, and the DatasetSource "inherit from source/target/custom" mode selector are all thoughtful touches.

4. **Responsive foundation** -- The `@media(max-width:800px)` breakpoint handles mobile layout gracefully.

5. **Self-contained** -- Zero build tooling needed. Open the file in a browser and it works. For a prototype/demo, this is a real advantage.

---

## Issues and Recommendations

### Critical: Single-File Architecture Won't Scale

The entire application -- 2,276 lines of CSS, 2,050 lines of HTML, and 600+ lines of JavaScript -- lives in one file. This is the single biggest risk to maintainability.

**Recommendation:** Split into at least three files:
- `styles.css` -- all CSS
- `index.html` -- markup only
- `app.js` -- all JavaScript

Even without a build system, this separation makes the codebase navigable and diffable.

### High: No State Management

Application state is scattered across DOM reads, a single `selectedDataset` variable, and hardcoded values. Functions like `confirmDatasetAndNext()` are 80+ lines long because they have to manually update dozens of DOM elements.

**Recommendation:** Introduce a simple state object (even a plain JS object with a render function) so that data flows in one direction: state -> DOM. This would cut function sizes in half and eliminate entire categories of bugs.

### High: Hardcoded Demo Data Mixed with Logic

Sample data for datasets, lineage graphs, DM metadata, column mappings, and RAG-fetched code are all embedded directly in JavaScript functions. For example, `confirmDatasetAndNext()` contains inline objects like `sampleLineage` and `dmMeta` with dataset-specific configurations.

**Recommendation:** Extract all sample/mock data into a separate `mockData.js` file or a JSON structure at the top of the script. This makes it trivial to swap in real API calls later.

### High: Inline Styles Throughout HTML

There are 80+ instances of `style="..."` in the HTML markup. Examples:

- Line 2336: `style="width:auto;padding:4px 24px 4px 8px;font-size:11px;..."` on a `<select>`
- Line 2927: `style="font-size:28px"` on a span
- Line 3334: `style="display:grid;grid-template-columns:1fr 1fr;gap:10px;margin-top:10px"` for layout

**Recommendation:** Move these to named CSS classes. Inline styles defeat the purpose of having a design system and make global changes painful.

### Medium: Accessibility Gaps

- **No ARIA attributes** -- The wizard stepper, accordion cards, sidebar toggle, and tab-like navigation lack `role`, `aria-expanded`, `aria-selected`, and `aria-label` attributes.
- **Keyboard navigation** -- Sidebar links use `<a>` tags without `href` attributes (just `onclick`), making them unreachable via keyboard tab.
- **Color-only status indicators** -- Status dots (`.cc-status.ok`, `.vd-ok`, etc.) rely solely on color to convey state.

**Recommendation:** Add ARIA roles to interactive components. Use `<button>` elements where appropriate instead of `<a>` or `<div>` with `onclick`. Add screen-reader-only text for color-coded indicators.

### Medium: CSS Class Naming is Overly Abbreviated

Class names like `.sw`, `.sl`, `.fg`, `.c2`, `.ph`, `.ibox`, `.vd-ok` require reading the CSS to understand. This hurts onboarding for other developers.

**Recommendation:** Use more descriptive names (e.g., `.switch`, `.slider`, `.form-grid`, `.cols-2`, `.page-header`, `.info-box`, `.validation-dot-ok`). At 2,200 lines of CSS, grep-ability matters.

### Medium: Missing `status-pill` CSS Class

The Migration Details table references `.status-pill.ok` and `.status-pill.warn` classes (lines 4031, 4043, 4055, 4067, 4079) but these classes are never defined in the `<style>` block. Those pills will render without any styling.

**Recommendation:** Add the missing CSS definitions:
```css
.status-pill {
  font-size: 10px;
  padding: 3px 10px;
  border-radius: 12px;
  font-weight: 600;
}
.status-pill.ok {
  background: rgba(56, 203, 137, .12);
  color: var(--green);
}
.status-pill.warn {
  background: rgba(255, 171, 0, .12);
  color: var(--amber);
}
```

### Medium: `toggleSchedType()` Function Referenced but Not Defined

Line 3543 references `onchange="toggleSchedType()"` on the trigger type select, but this function does not exist in the `<script>` block. This will throw a console error when the user changes the trigger type dropdown.

### Medium: Commented-Out RAG/MCP Button

Lines 3128-3140 contain a commented-out "AI-Assisted Logic Discovery" button section. The RAG progress and results UI still exist and the `fetchLogicViaMCP()` function is fully implemented, but there is no way to trigger it from the UI.

**Recommendation:** Either restore the button or remove the dead code (progress indicators, results panel, and the ~150 lines of related JS).

### Low: `event` Used as Implicit Global

In `scanDatasets()` (line 4200) and `copyRagLogic()` (line 4828), the code uses `event.target` without receiving `event` as a function parameter. This relies on the deprecated implicit `window.event` and will fail in strict mode or in some browsers.

**Recommendation:** Pass `event` explicitly via `onclick="scanDatasets(event)"` and update function signatures accordingly.

### Low: `var(--heading)` CSS Variable Not Defined

Line 542 references `font-family: var(--heading)` in the `.ph-title` class, but `--heading` is never defined in `:root`. The browser will fall back to the inherited font, so it still works, but it is likely an oversight.

### Low: Border Color in Light Theme Uses Dark-Theme Values

Several `border-bottom` rules use `rgba(37, 43, 58, .5)` and `rgba(37, 43, 58, .6)` (lines 1607, 2023, 2137), which are dark-theme border colors. In the current light theme, these render as faint gray borders that are barely visible. Consider using `var(--border)` instead for consistency.

### Low: `index_v1.html` Should Be Cleaned Up

The old v1 file (4,097 lines) is still in the repo root. If it is no longer the active version, it should either be deleted or moved to a `legacy/` directory to avoid confusion about which file is canonical.

---

## Summary

| Severity | Count | Key Areas |
|----------|-------|-----------|
| Critical | 1 | Single-file architecture |
| High | 3 | No state management, hardcoded data, inline styles |
| Medium | 4 | Accessibility, CSS naming, missing CSS/JS definitions |
| Low | 4 | Implicit event, undefined CSS var, theme colors, stale file |

The portal is an impressive prototype with a polished UI and thoughtful UX. The main investment needed is structural -- splitting the file, introducing minimal state management, and extracting mock data -- which would make this production-ready without changing any user-facing behavior.
