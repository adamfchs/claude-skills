---
name: ec-scss
description: >
  SCSS conventions for citizen_ Drupal themes. Apply these rules whenever
  editing any .scss file in a citizen_ theme (citizen_dart, citizen_sdc,
  citizen_patterns, or any other project theme). These rules govern selector
  structure, nesting, and context-specific overrides.
---

# EC SCSS Conventions

Rules to follow when writing or editing SCSS in citizen_ Drupal themes.
Add to this skill over time as new conventions are established.

---

## 1. Node/Content Type Classes

When the user refers to a node type or content type by name, the class is
almost always `.node-[content-type-name]` (e.g. `.node-data-report`,
`.node-article`, `.node-landing-page`).

- This class lives on the `<article>` element rendered by Drupal for that
  node — **not** on `<body>` or any other ancestor.
- Never assume the class is on `body` or `html` unless the user explicitly
  says so or you can confirm it from the template/markup.
- In JavaScript, check for the article: `document.querySelector('article.node-data-report') !== null`
  rather than `document.body.classList.contains('node-data-report')`.

---

## 2. No Unnecessary Element Prefixes

Do not prefix selectors with `html`, `body`, or `div` (e.g. `body.node-foo`,
`html .block`, `div.wrapper`) unless:

- The user explicitly asks for it, or
- That exact pattern already exists in the specific `.scss` file being edited.

---

## 3. Keep Styles Co-located — Use `&` for Context Variations

Keep all styles for a given selector within or close to that selector, including
any context-specific overrides. Use the Sass `&` parent selector to express
"this element, when it appears inside context X" rather than writing a separate
top-level block for context X.

**Preferred pattern:**
```scss
.block-title {
  // default styles

  .node-data-report & {
    // styles specific to when .block-title appears inside .node-data-report
  }
}
```

**Avoid:**
```scss
.block-title {
  // default styles
}

.node-data-report {
  .block-title {
    // context-specific styles separated from the main block
  }
}
```

This applies in both directions:

- **Additions / specificity** — adding a modifier on the element itself:
  ```scss
  li {
    &.is-active { }   // li.is-active
  }
  ```

- **Parent context overrides** — element behaves differently inside a parent:
  ```scss
  li {
    .node-data-report & { }  // .node-data-report li
  }
  ```

The goal is that someone reading the code for `.block-title` (or `li`, etc.)
sees all of its visual states in one place, rather than having to hunt for a
separate parent-wrapper block.

---

## 4. Formatting Rules

**Brackets always preceded by a space:**
```scss
// Correct
.selector {
  color: red;
}

// Wrong
.selector{
  color: red;
}
```

**Blank line between the last property of a selector and any nested selector:**
```scss
// Correct
.parent {
  color: red;

  .child {
    color: blue;
  }
}

// Wrong
.parent {
  color: red;
  .child {
    color: blue;
  }
}
```

**Properties after a mixin belong in `& {}`:**
When a mixin is used and there are style properties after it that apply to
the same selector, wrap those properties in `& {}`:
```scss
// Correct
#node-section-3 {
  max-width: $headerFooterStop;
  @include auto;

  & {
    margin-bottom: $spaceXs;
  }

  .paragraph--type--layout-section .widget-section {
    max-width: 100%;
  }
}

// Wrong
#node-section-3 {
  max-width: $headerFooterStop;
  @include auto;
  margin-bottom: $spaceXs;

  .paragraph--type--layout-section .widget-section {
    max-width: 100%;
  }
}
```
