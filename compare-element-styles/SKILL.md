---
name: compare-element-styles
description: >
  Compare the HTML structure and applied styles of two elements across two different
  pages (local or remote URLs). Fetches live HTML via curl, extracts the target
  elements, identifies structural and styling differences, and summarizes gaps to
  close. Use when the user wants to match or reconcile the styling of one element
  to another — e.g. theming a custom menu to match a contrib menu, or syncing a
  microsite component to a main-site component.
---

# Compare Element Styles

This skill fetches two pages, extracts the target elements, compares their HTML
structure and CSS classes, cross-references relevant SCSS source files, and
produces an actionable gap analysis.

---

## Stage 1 — Gather Inputs

Ask the user for all of the following before proceeding:

1. **Reference URL** — the page containing the element to match (the "source of
   truth"). Example: `https://mysite.ddev.site/`
2. **Reference selector** — the CSS selector for the element on that page (ID or
   class). Example: `#144f870b-d2c2-48b7-85df-624c1aae95b7` or `.tbm.tbm-main`
3. **Target URL** — the page containing the element to be styled.
4. **Target selector** — the CSS selector for the element to be styled. Example:
   `#block-eclds-main-menu`
5. **Breakpoint scope** — should the comparison focus on desktop, mobile, or both?
   If mobile, ask for the max-width breakpoint (e.g. `1199px`).
6. **Existing SCSS file(s)** — ask: "Are there specific SCSS files where the
   styling work should be done? If so, provide their paths." Accept multiple files.
   If none provided, you will locate relevant files yourself.

Do not proceed to Stage 2 until all inputs are collected.

---

## Stage 2 — Fetch and Extract

Use `curl -sk <url>` to fetch each page's HTML. Since these are often local DDEV
or dev sites with self-signed certificates, always use `-sk` flags.

For each page, extract the full outer HTML of the target element using Python:

```bash
curl -sk <url> | python3 -c "
import sys, re
html = sys.stdin.read()
selector = '<target-opening-tag-fragment>'  # e.g. 'id=\"block-eclds-main-menu\"'
start = html.find(selector)
if start == -1:
    print('NOT FOUND')
else:
    # Walk back to the opening < of the tag
    start = html.rfind('<', 0, start)
    # Find closing tag by tracking nesting (use tag name from found opening tag)
    print(html[start:start+4000])
"
```

Extract enough HTML (4000–8000 chars) to capture the full element structure
including nested lists, buttons, and any modifier classes.

Collect:
- All classes on the root element and key child elements
- Presence and structure of toggle buttons, submenus, list levels
- Any `data-*` attributes relevant to JS behavior
- Any inline styles

---

## Stage 3 — Read SCSS Source Files

If the user provided SCSS file paths, read each one in full.

If not provided, search the theme for relevant files:
```
Grep pattern="<target-selector-fragment>" path="<theme-root>" glob="*.scss"
```

Also read any contrib module SCSS that styles the reference element if accessible
in the project (e.g. a `tbmm` or `superfish` module scss partial).

Note the breakpoint mixins used (e.g. `onlyMenuBreak`, `mainMenuBreak`) and look
up their definitions in the base variables file if needed.

---

## Stage 4 — Compare and Analyze

Produce a structured comparison covering:

### HTML Structure Differences
- Root element tag, ID, classes
- Toggle button markup and class names
- Menu list hierarchy (ul/li nesting, level classes)
- Submenu toggle mechanism (button vs pseudo-element vs JS class)
- Any elements present in reference but absent in target (e.g. image blocks,
  secondary menu injection, search injection)

### CSS Class Mapping
Create a table or list pairing reference classes to their target equivalents:

| Reference class | Target equivalent | Notes |
|---|---|---|
| `.tbm--mobile` (JS-added) | `.menu--open` (JS-added) | Different trigger mechanism |
| `.tbm-item.level-1` | `.item-level-1` | |
| ... | ... | |

### Style Gaps at Target Breakpoint
List each visual property that differs or is missing in the target SCSS at the
specified breakpoint. For each gap include:
- Property name
- Reference value
- Current target value (or "missing")
- Recommended fix

### JS Behavior Notes
Note any breakpoint mismatches between JS and CSS, class-toggling differences,
or mode-switching logic that may affect styling.

### Out of Scope
List any reference features explicitly excluded from this pass (e.g. mobile
search injection, secondary menu items) so the user can track them for later.

---

## Stage 5 — Confirm Before Acting

Present the full gap analysis to the user and ask:

> "Would you like me to implement these changes now? If so, confirm the SCSS
> file(s) to edit and whether any JS files need updating."

Do not make any file edits during Stages 1–4. All changes happen only after
explicit user confirmation in Stage 5.

---

## Notes and Constraints

- Always use `curl -sk` for local dev sites (self-signed certs).
- `WebFetch` will fail for local DDEV URLs — use `Bash` with `curl` instead.
- When extracting HTML, if Python is unavailable fall back to:
  `curl -sk <url> | grep -o '<nav[^>]*selector-fragment[^>]*>.*' | head -c 4000`
- Never alter toggle button styles unless the user explicitly requests it.
- Never alter desktop styles when the scope is mobile-only.
- Read SCSS files before suggesting edits — never propose changes to code you
  haven't read.
- If the reference element is from a contrib module (e.g. TB Mega Menu), its
  styles may come from both a theme partial and the module's own assets. Note
  which styles are theme-overridable vs. module-locked.
