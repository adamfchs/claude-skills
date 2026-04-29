---
name: citizen-sdc-builder
description: Use when the user asks to create a new Single Directory Component (SDC) in the citizen_sdc Drupal 11 theme — e.g. "build a paragraph SDC for X", "create a block SDC", "scaffold a node teaser for content type Y", "add a new layout SDC", "let's do the pager SDC", or otherwise references .component.yml / SDC twig / SDC SCSS in this theme. Covers block/paragraph/node/layout/view SDCs, base-SDC delegation, naming conventions, BEM rules, SCSS @use patterns, library attachment, and the when-to-write-an-.md rule. Pairs with the global drupal-sdc skill — that one covers generic Drupal 11 SDC anatomy; this one covers the citizen_sdc-specific patterns layered on top.
allowed-tools: Read, Edit, Write, Bash, Grep, Glob
---

# Citizen SDC Builder

You're creating a Single Directory Component for the **citizen_sdc** Drupal 11 theme. The theme has its own conventions on top of Drupal core SDC — follow them strictly so the new component composes cleanly with everything already built.

If anything in this skill conflicts with what you find by reading the actual theme code, **trust the code** and report the conflict. The skill is the convention; the codebase is the source of truth.

## Step 1 — Gather requirements

Before writing any files, ask the user for any of these you don't already know:

1. **What kind of SDC?** block, paragraph (per-bundle), node (default/teaser/listing/search-result), layout (theme-region / node-layout / paragraph-section-layout), or view base.
2. **Machine name?** This becomes the folder name + component id (e.g. `cta`, `news-teaser`, `block-search`, `twocol-split-left`).
3. **Bundle / plugin id?** For paragraphs and nodes: which bundle / content type? For blocks: which Drupal block plugin id or `block_content` bundle backs it? Drupal's `clean_class` of that machine name becomes the BEM modifier.
4. **Fields → props vs slots:**
    - Plain values (strings, numbers, booleans, simple objects) → **props**.
    - Rendered Drupal field markup / render arrays → **slots**.
    - Required vs optional? Mark required props in `component.yml`.
5. **Libraries needed?** Co-located `{name}.js` (auto-attached), an external lib (Embla, Select2), a custom-module library, or a sibling SDC's auto-library?
6. **Animation (paragraphs only):** does it opt into the scroll-in observer (`animate: 'yes'`)? Default is `'no'`. **If the user wants animation but hasn't said which elements animate or how, pause and ask** — animation is per-component-specific and guessing wastes a round trip.
7. **Bundle / config exists yet?** If creating a new paragraph or content type that's not in `config/sync` yet, the bundle config has to land before the SDC is useful. Ask whether you need to scaffold the bundle too.

## Step 2 — Pick the right base SDC

This theme uses **base SDCs** that own the wrapper markup. Bundle/type-specific SDCs are thin: they compose the field markup and delegate to a base via `include`. Don't re-implement the wrapper.

| Building… | Extend | Folder | File / id prefix |
|---|---|---|---|
| Block (any) | `citizen_sdc:block-base` (called automatically by `block.html.twig`) | `components/03-composites/blocks/{name}/` | `block-` |
| Block (custom block_content entity, unnamed numeric IDs) | same | `components/03-composites/blocks/custom/{name}/` | `block-` |
| Paragraph (per-bundle) | `citizen_sdc:paragraph-base` | `components/03-composites/paragraphs/{name}/` | none — file name = folder name |
| Node default/full view | `citizen_sdc:node-base` | `components/03-composites/nodes/{type}-default/` | `node-` |
| Node teaser | `citizen_sdc:node-teaser-base` | `components/03-composites/nodes/{type}-teaser/` | `node-` |
| Node listing (view-mode) | `citizen_sdc:node-listing-base` | `components/03-composites/nodes/{type}-listing/` | `node-` |
| Node search-result | `citizen_sdc:node-search-result-base` | `components/03-composites/nodes/{type}-search-result/` | `node-` |
| Theme region wrapper | (no base — self-contained) | `components/03-composites/layouts/theme-regions/region-{name}/` | none |
| LB layout (custom node layout) | (no base) | `components/03-composites/layouts/node-layouts/{name}/` | none |
| Paragraph layout-section layout | (no base) | `components/03-composites/layouts/paragraph-section-layouts/{name}/` | none |
| View base / per-view | `citizen_sdc:views-base` (or `views-form-base`, `views-unformatted-base`, `views-table-base`) | `components/03-composites/views/{name}/` | varies |
| Menu (per-menu) | `citizen_sdc:menu-base` | `components/02-chunks/menus/{name}/` | `menu-` |

**Naming rule (important):**

- An SDC that sits on top of a `-base` SDC uses the prefix on the **file name** and **component id** (`block-branding.component.yml` → id `citizen_sdc:block-branding`). The **folder name stays plain** (`branding/`).
- Layouts and other self-contained SDCs **skip the prefix** entirely.
- Node SDCs use `node-{content-type}-{view-mode}`: `node-news-teaser`, `node-basic-page-default`, etc.
- For nodes, the bundle's machine name drives the `modifier_class` (so `.node-content--{machine_name}`). If the folder uses a friendlier name (e.g. `basic-page-default/` for the `page` bundle), the `modifier_class` value still has to match the bundle machine name (`'page'` not `'basic-page'`) — and the Drupal template path under `templates/content/nodes/` also has to use the machine name (`page/node--page--full.html.twig`).

## Step 3 — File structure

Every SDC is one folder:

| File | Required? | What it does |
|---|---|---|
| `{name}.component.yml` | yes | id, label, description, props, slots, optional `libraryOverrides` |
| `{name}.twig` | yes | Markup. Almost always composes a `content` block and `include`s a base SDC |
| `{name}.scss` | yes | Component styles. Compiled to `{name}.css` next to it (`sass --watch components --style compressed` from theme root) |
| `{name}.js` | optional | Co-located JS — Drupal auto-generates a library `core/components.citizen_sdc--{name}` and auto-attaches it when the SDC renders |
| `{name}.md` | **rare** — see "When to write an .md" | Hidden-dependency docs only |

The compiled `.css` and `.css.map` belong next to the `.scss` and are committed.

## Step 4 — `.component.yml` structure

```yaml
component:
  id: citizen_sdc:{name}
  label: '{Title-cased description}'
  description: '{One sentence — what the component does and where it sits.}'
  status: experimental

props:
  type: object
  required:
    - {required_prop}
  properties:
    attr:
      type: object
      description: 'Drupal Attributes object for the wrapper'
    {prop_name}:
      type: string
      description: '{What the prop is, where it comes from in Drupal.}'

slots:
  {slot_name}:
    title: '{Title}'
    description: '{What renders into this slot.}'
```

**Prop description boilerplate (keep verbatim where applicable):**

- `attr` → `'Drupal Attributes object for the wrapper'` (or `for the block` / `for the <article>` etc.)
- `title_prefix` → `'Placeholder for module and theme output before the block title'`
- `title_suffix` → `'Placeholder for contextual links'`
- `title` → `'The name of the block'` (for block-base callers)

**Type tips:**

- Plain text from a field with `value`/`processed` → `type: string`.
- Optional string that may be empty → `type: [string, 'null']` and pass with `|default(null)` in twig.
- Render arrays / Drupal Attributes → `type: object`.
- Don't over-specify enums unless the choice space is genuinely fixed (alignment left/right, urgency low/medium/high, etc.).

## Step 5 — Twig delegation patterns

### Block SDC — content-only (Option C)

The theme's `templates/blocks/block.html.twig` already builds the BEM class array, calls `block-base`, and exposes a `blockContent` Twig block for overrides. Per-block templates only override `blockContent`:

```twig
{# templates/blocks/{name}/block--{plugin-id-with-dashes}.html.twig #}
{% extends 'block.html.twig' %}

{% block blockContent %}
  {{ include('citizen_sdc:block-{name}', {
    {prop}: {value},
  }, with_context = false) }}
{% endblock %}
```

The block SDC twig itself **has no wrapper, no classes, no block-base include** — those are all owned by `block.html.twig`. The SDC is purely the content markup:

```twig
{# components/03-composites/blocks/{name}/block-{name}.twig #}
<div class="{name}__inner">
  {# component-specific markup #}
</div>
```

The `.scss` root selector is the Drupal-generated block class (`.block--{plugin-id-clean-class}` or `.block--{bundle}`), **not** `.block-{name}`.

### Paragraph SDC — wraps `paragraph-base`

```twig
{# components/03-composites/paragraphs/{name}/{name}.twig #}
{% set body %}
  {# compose field markup using passed props/slots #}
  {% if widget_title %}
    <h2 class="paragraph--{name}__title">{{ widget_title }}</h2>
  {% endif %}
  {{ content_field }}
{% endset %}

{{ include('citizen_sdc:paragraph-base', {
  attr: attr,
  modifier_class: '{name}',
  animate: 'yes',  {# or 'no' / omit — defaults to 'no' #}
  content: body,
}, with_context = false) }}
```

The Drupal template (`templates/paragraphs/{name}/paragraph--{name}.html.twig`) is a thin shim that hands attributes + rendered fields to the SDC:

```twig
{{ include('citizen_sdc:{name}', {
  attr: attributes|without('class'),
  widget_title: content.field_widget_title|render|striptags|trim,
  content_field: content.field_long_text,
}, with_context = false) }}
```

Use `attributes|without('class')` because `paragraph-base` adds its own classes; passing the original would double up.

### Node default — wraps `node-base`

The Drupal template (`templates/content/nodes/{type}/node--{type}--full.html.twig`) imports the layout-section macro and delegates:

```twig
{% import '@citizen_sdc/macros/node-layout-sections.html.twig' as node_macros %}

{{ include('citizen_sdc:node-{type}-default', {
  attr: attributes,
  title_prefix: title_prefix,
  title_suffix: title_suffix,
  content: node_macros.wrap_layout_builder_sections(content),
}, with_context = false) }}
```

The SDC is a thin passthrough to `node-base`:

```twig
{{ include('citizen_sdc:node-base', {
  attr: attr,
  modifier_class: '{type-machine-name}',
  title_prefix: title_prefix,
  title_suffix: title_suffix,
  content: content,
}, with_context = false) }}
```

Per-content-type SCSS scopes itself under `.node-content--{type-machine-name}`.

### Node teaser — wraps `node-teaser-base`

```twig
{% set teaser_content %}
  {% if date %}
    <div class="node-teaser__date">{{ date|date('F j, Y') }}</div>
  {% endif %}
  {% if summary %}{{ summary }}{% endif %}
{% endset %}

{{ include('citizen_sdc:node-teaser-base', {
  attr: attr,
  modifier_class: '{type}',
  title: title,
  url: url,
  title_attr: title_attr,
  image: image,           {# rendered field — slot #}
  content: teaser_content,
}, with_context = false) }}
```

The Drupal template at `templates/content/nodes/{type}/node--{type}--teaser.html.twig` extracts title/url/image from the node and passes them in.

## Step 6 — BEM naming

- **Block SCSS root:** the Drupal class (e.g. `.block--system-branding-block`, `.block--social-links`). Not `.block-{name}`.
- **Block child elements:** component-name BEM (`.branding__logo`, `.branding__site-name`).
- **Paragraph wrapper:** `.paragraph--{bundle}` (set by `paragraph-base`).
- **Paragraph children:** `.paragraph--{bundle}__{element}` (`.paragraph--cta__headline`, `.paragraph--accordions__toggle`).
- **Paragraph variants:** `.paragraph--{bundle}--{variant}` (`.paragraph--layout-section--light-grey`, `.paragraph--fifty-fifty--right`).
- **Node default wrapper:** `.node-content--{type}` (set by `node-base`).
- **Node teaser wrapper:** `.node-teaser--{type}` (set by `node-teaser-base`).
- **Node teaser children:** `.node-teaser__title`, `.node-teaser__image`, `.node-teaser__content`, etc. (already on the base; use these inside per-type SCSS too).
- **Field wrappers** (from `templates/field/field.html.twig`): `.field--{fieldname-without-field-prefix}` for custom fields with `field_` prefix; for base fields like `title`/`body` the prefix is dropped already.

### Dart → SDC selector cheat sheet

When porting styles from `citizen_dart`:

- `.node-{type}` → `.node-content--{type}`
- `.node-teaser-{type}` → `.node-teaser--{type}`
- `.node-listing-{type}` → `.node-listing--{type}`
- Dart field-block wrappers `.block-field-{fieldname}` → SDC `.block-field--{machine_name}` (custom fields keep `field_` prefix; base fields drop it).
- Dart `h2.block-title` → SDC `.block__title`.
- Dart field wrappers `.field-{fieldname}` → SDC `.field--{fieldname}` (leading `field_` stripped).
- Dart section ids `#node-section-1, #node-section-2` → SDC `.node-content__section`.
- Teaser internals `.node-teaser-content` / `.node-field-title` → `.node-teaser__content` / `.node-teaser__title`.

When SDC-ifying, **port the dart styles** — don't ship a bare-wrapper SCSS file when dart already has working styles. Translate the rules into the new BEM selectors.

## Step 7 — SCSS conventions

`@use` paths are **per-file and explicit** — never `@use '../../../00-base' as *`. Pull only what the file needs:

```scss
@use '../../../00-base/00-variables/colors' as *;
@use '../../../00-base/00-variables/component-colors' as *;
@use '../../../00-base/00-variables/units' as *;
@use '../../../00-base/01-mixins/breakpoint-mixins' as *;
@use '../../../00-base/01-mixins/general-mixins' as *;
```

Path depth varies by component depth (e.g. `02-chunks/{name}/` = `../../00-base/...`; `03-composites/{type}/{name}/` = `../../../00-base/...`). Read a sibling SDC's SCSS in the same directory to copy the right relative paths.

**Don't theme `:focus` as a hover-mirror.** Combined `&:hover, &:focus { ... }` patterns and standalone `&:focus` blocks that just duplicate hover are removed theme-wide; the browser default focus indicator is preferred. Functional focus indicators (custom focus rings on form inputs, skip-nav reveal, focus-within for keyboard menu navigation, focus-visible for keyboard-only outlines) **are** kept. When adding focus styling, ask: am I telling the browser to do something it wouldn't do on its own (e.g. reveal a hidden skip-link), or am I just duplicating a hover effect? If the latter, omit it.

The `ec-scss` skill (in the user's global skill set) covers per-citizen-theme SCSS conventions in more depth — check there for selector structure and nesting rules.

## Step 8 — Library attachment

Drupal SDC auto-generates one library per component at `core/components.{theme}--{component-id}`. So `citizen_sdc:cta` → `core/components.citizen_sdc--cta`. Co-located CSS and JS attach automatically when the SDC renders.

### When you need an extra library

```yaml
# component.yml
libraryOverrides:
  dependencies:
    - core/components.citizen_sdc--node-alert-teaser   # another SDC's auto-library
    - citizen_custom/alert                              # a manual library from a module/theme
    - citizen_sdc/embla                                 # a manual library from this theme's libraries.yml
```

**Naming, exactly:**

- Auto-library of another SDC: `core/components.{theme}--{component-id}` — note the `core/` prefix and the `--` separator. **`{theme}/{component-id}` silently no-ops** and the dep won't attach.
- Manual library defined in a `*.libraries.yml`: `{theme-or-module}/{lib-name}` — e.g. `citizen_sdc/embla`, `citizen_custom/alert`.

To verify a dep is wiring up:

```php
ddev drush ev '$libs = \Drupal::service("library.discovery")->getLibrariesByExtension("core"); print_r($libs["components.citizen_sdc--{component-id}"]["dependencies"] ?? "MISSING");'
```

If the dep doesn't appear, the name is wrong. Fix it and `ddev drush cr`.

### Block plugin `#attached` doesn't bubble through template overrides

If a block plugin returns `['#attached' => ['library' => [...]]]` and the theme's `block--{plugin-id}.html.twig` overrides `blockContent` without ever rendering `{{ content }}`, the library **never attaches**. The render-array metadata is on `content`, which the override skips.

In order of cleanliness:

1. **Co-locate the JS in the SDC folder** (`{name}.js`) — auto-attached when the SDC renders. Use this when the JS is conceptually part of the component.
2. **`libraryOverrides.dependencies` in `.component.yml`** — declarative, attaches when the SDC renders. Use this when the library lives elsewhere (e.g. a module).
3. **`{{ attach_library('module/lib') }}` inside the SDC twig** — fallback for conditional attachment based on props.
4. **Don't rely on block plugin `#attached`** when the template override skips `{{ content }}`.

### AJAX-rendered children

If an SDC renders inside a controller `Response` (e.g. `/ajax/{thing}` returning teaser HTML), libraries attached during that render die with the AJAX response — they never reach the parent page's `<head>`. Add those libraries to the **parent-page SDC's** `libraryOverrides.dependencies` so they load up front (this is why `block-alert` depends on `core/components.citizen_sdc--node-alert-teaser`).

### Don't use `libraryDependencies`

It's not in the SDC schema and is silently ignored. Only `libraryOverrides` works.

## Step 9 — Animation (paragraph SDCs only)

Three pieces wire together:

1. The SDC passes `animate: 'yes'` to `paragraph-base`, which writes `data-animate="yes"` onto the wrapper.
2. The paragraph field template (`templates/field/paragraphs/field--node--field-paragraphs.html.twig`) attaches `citizen_sdc/paragraph-animate` and tags the wrapper with `data-paragraph-animate-field`.
3. JS creates one `IntersectionObserver` per field wrapper, watches descendant `[data-animate="yes"]` elements, adds `is-in-view` when they cross the threshold (then unobserves).

Per-bundle SCSS defines the initial and final states:

```scss
.paragraph--{bundle}[data-animate="yes"] {
  // initial (pre-animation) state, with transitions
  .paragraph--{bundle}__title {
    opacity: 0;
    transform: translateY(16px);
    transition: opacity 0.5s ease-out 0.25s, transform 0.5s ease-out 0.25s;
  }

  &.is-in-view {
    .paragraph--{bundle}__title {
      opacity: 1;
      transform: translateY(0);
    }
  }
}

@media (prefers-reduced-motion: reduce) {
  .paragraph--{bundle}[data-animate="yes"] {
    .paragraph--{bundle}__title {
      opacity: 1;
      transform: none;
      transition: none;
    }
  }
}
```

The reduced-motion override is **required** — JS only handles the class toggle, not transitions or initial states. Without the override, users with the preference see the pre-animation state.

`animate: 'no'` is the default — `paragraph-base` opts out unless the SDC explicitly opts in. Layout-wrapping bundles (`layout-section`, `layout-section-simple`) explicitly pass `animate: 'no'` for readability; their children animate, not them.

See `components/03-composites/paragraphs/00-paragraph-base/paragraph-base.md` for the full animation contract, observer thresholds, and gotchas.

## Step 10 — When to write an `.md` file

Most SDCs **don't** need an `.md`. Only write one when the component depends on something a dev wouldn't find by reading the SDC folder alone:

- A `.inc` preprocess hook sets variables the SDC consumes.
- A custom module (`citizen_custom`) provides plugins / forms / services / JS the SDC relies on.
- A library lives outside the SDC's own folder (e.g. `components-addons/embla/`).
- There's a non-obvious cross-SDC or cross-file gotcha (e.g. slider's "transition has to live on the track, not the viewport").

**Don't** document fields (they're in `config/sync/`), props/slots (they're in `.component.yml`), markup (it's in `.twig`), styles or BEM (they're in `.scss`), or Drupal template hand-off (it's the one-liner template). Docs that repeat code rot fast.

Reference SDCs that legitimately need an `.md`: `paragraph-base` (animation contract), `block-alert` (citizen_custom AJAX architecture), `block-search` (citizen_custom SiteSearch service), `slider` (Embla + adaptive-height coupling), `form-placer` (preprocess), `block-social-share` (custom theme hook).

## Step 11 — Pitfalls / gotchas

- **`drawer` translates internally.** When including `citizen_sdc:drawer`, pass `toggle_open_text` / `toggle_close_text` as **plain strings**, not `'text'|t`. The drawer twig applies `|t` itself; pre-translating throws `InvalidArgumentException`. Same rule for any other SDC that runs `|t` on a prop.
- **Empty list-field values come back as `[]`, not empty string.** Don't pipe `field_x.value` through `Html::getClass` — use `field_x.0.value|default('fallback')` to avoid "Array to string conversion" warnings.
- **Don't clone an existing module's `.info.yml` into the theme.** Drupal's extension discovery picks one arbitrarily and silently shadows the real module — newly added plugins won't register. Rename reference copies to `.info.yml.example`.
- **Slot content arrives as a render array — don't wrap it.** Bare `{{ slot_name }}` renders correctly. Wrapping in a BEM div is fine when the SDC owns the wrapper styling (e.g. `<div class="paragraph--cta__actions">{{ link }}</div>`); just don't double-wrap rendered fields that already bring their own markup.
- **Path depth in SCSS `@use` matters.** Copy from a sibling SDC at the same depth — don't guess.

## Step 12 — Workflow checklist

Once the user has confirmed requirements:

1. **Bundle / config** — confirm the bundle / content type / block plugin already exists in `config/sync` (or scaffold it first if asked).
2. **Folder + files** — create `{name}/` and the four-or-five files inside. Reference an existing similar SDC for SCSS `@use` depth and twig structure.
3. **Drupal template hand-off** — add or update the matching template in `templates/{blocks|paragraphs|content/nodes}/{name}/`. For blocks: `{% extends 'block.html.twig' %}{% block blockContent %}...{% endblock %}`. For paragraphs: thin include of the SDC. For nodes: include with the layout-section macro for content types using Layout Builder.
4. **Compile SCSS** — `sass components --style compressed` from the theme root, or rely on the user's `--watch` if they have it running. Confirm no errors (deprecation warnings about `mixed-decls` are pre-existing).
5. **Drush cr** — `ddev drush cr` to register the new SDC and pick up template changes.
6. **Verify the SDC is registered:** `ddev drush ev '$d = \Drupal::service("plugin.manager.sdc")->getDefinition("citizen_sdc:{name}", FALSE); print_r($d ? "OK" : "MISSING");'`
7. **Render check** — load a page that uses the new SDC and check for layout / styling / no errors. Visual / browser test is the user's call.

Don't claim done until at least drush cr succeeds and the SDC definition resolves. UI/visual confirmation is the user's job unless they ask you to test it.

## Examples

### Minimal block SDC

```yaml
# components/03-composites/blocks/foo/block-foo.component.yml
component:
  id: citizen_sdc:block-foo
  label: 'Block Foo'
  description: 'Renders the foo block content. Backed by the {plugin_id} block plugin.'
  status: experimental

props:
  type: object
  properties:
    headline:
      type: string
      description: 'Plain-text heading.'
slots:
  body:
    title: 'Body'
    description: 'Rendered body field.'
```

```twig
{# components/03-composites/blocks/foo/block-foo.twig #}
{% if headline %}
  <h2 class="block--foo__headline">{{ headline }}</h2>
{% endif %}
{{ body }}
```

```twig
{# templates/blocks/foo/block--foo.html.twig (Drupal block plugin id with dashes) #}
{% extends 'block.html.twig' %}

{% block blockContent %}
  {{ include('citizen_sdc:block-foo', {
    headline: content.field_headline|render|striptags|trim,
    body: content.field_body,
  }, with_context = false) }}
{% endblock %}
```

```scss
// components/03-composites/blocks/foo/block-foo.scss
@use '../../../00-base/00-variables/component-colors' as *;
@use '../../../00-base/00-variables/units' as *;

.block--foo {
  .block--foo__headline {
    color: $headerColor;
    margin-bottom: $spaceXs;
  }
}
```

### Minimal paragraph SDC

```yaml
# components/03-composites/paragraphs/foo/foo.component.yml
component:
  id: citizen_sdc:foo
  label: 'Paragraph Foo'
  description: 'Single-field foo paragraph. Wraps paragraph-base.'
  status: experimental
props:
  type: object
  properties:
    attr:
      type: object
      description: 'Drupal Attributes object from the paragraph template.'
slots:
  text:
    title: 'Text'
    description: 'Rendered field_long_text.'
```

```twig
{# components/03-composites/paragraphs/foo/foo.twig #}
{{ include('citizen_sdc:paragraph-base', {
  attr: attr,
  modifier_class: 'foo',
  content: text,
}, with_context = false) }}
```

```twig
{# templates/paragraphs/foo/paragraph--foo.html.twig #}
{{ include('citizen_sdc:foo', {
  attr: attributes|without('class'),
  text: content.field_long_text,
}, with_context = false) }}
```
