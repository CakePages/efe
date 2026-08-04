# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

The Shopify storefront theme for **EFEMERA** (a jewelry brand). It is a licensed
copy of **Stiletto v5.2.2** by Fluorescent Design Inc., with a small number of
store-specific customizations layered on top.

- Vendor docs: https://help.fluorescent.co/v/stiletto
- Theme identity is declared in `config/settings_schema.json` (`theme_info`) and
  echoed in `layout/theme.liquid` and `snippets/theme-setup.liquid`. Keep the
  version strings in those three places consistent if the theme is ever upgraded.

There is **no build system, no package manager, no test suite, and no CI** in
this repo. It is plain Liquid/JSON/CSS/JS served directly by Shopify. Do not add
`package.json`, bundlers, or lockfiles unless explicitly asked.

## Repository layout

```
layout/      theme.liquid (main document), password.liquid
sections/    70 .liquid sections + 3 section-group JSON files
snippets/    132 reusable .liquid partials
templates/   65 JSON templates + gift_card.liquid + templates/customers/ (7)
assets/      compiled CSS/JS bundles, vendor chunks, and store imagery
config/      settings_schema.json (editor schema), settings_data.json (live values)
locales/     *.json storefront strings, *.schema.json theme-editor labels
```

Counts as of this writing: 42 `product.*.json` templates, 6 `collection.*.json`,
9 `page.*.json`. The large number of product templates is intentional — most
products get a bespoke template.

## How changes reach this repo (important)

This repository is connected to Shopify via the **GitHub theme integration**.
Every commit in the history is authored by Shopify with the message
`Update from Shopify for theme efe/main`. That means:

- **`main` is written to by Shopify**, not only by humans. Merchant edits made in
  the Shopify admin theme editor are committed back automatically, mostly to
  `templates/*.json`, `sections/*-group.json`, and `config/settings_data.json`.
- Editing those JSON files by hand is legitimate but **racy** — the theme editor
  can overwrite your changes. Every JSON template carries a banner comment saying
  exactly this. Prefer changing Liquid/CSS/JS in this repo and leaving content
  and merchandising choices to the theme editor.
- Always work on the assigned feature branch, never commit directly to `main`,
  and rebase on the latest `origin/main` before pushing — it moves on its own.

## Working with the JSON files

`templates/*.json`, `sections/*-group.json`, and `config/settings_data.json` are
**JSON with `/* … */` comment blocks** at the top. Standard JSON parsers reject
them. To read one programmatically, strip comments first:

```python
import re, json
src = re.sub(r'/\*.*?\*/', '', open(path).read(), flags=re.S)
data = json.loads(src)
```

Structure of a JSON template: a `sections` map (key → `{type, blocks, settings}`)
plus an `order` array controlling render order. `type` must match a filename in
`sections/` (minus `.liquid`).

Images in these files are referenced as `shopify://shop_images/<name>.jpg` —
those live in Shopify **Files**, not in this repo, so they will not resolve
locally and cannot be added by committing a file to `assets/`. Only a handful of
`efemera-*.jpg` files in `assets/` are true theme assets.

## Section anatomy

A section follows a consistent shape — match it when adding or editing one:

1. A leading `{%- liquid … -%}` block that derives local variables from
   `section.settings` (position splits, opacity math, `section.index == 1` →
   `prioritize_loading`).
2. A wrapper `<div>` carrying:
   - BEM-ish classes plus shared utilities (`section`, `section--full-width`,
     `{{ section.settings.section_padding }}`, `animation animation--<name>`)
   - `data-section-id="{{ section.id }}"` and `data-section-type="<type>"`
   - inline `style="--custom-height: …; --text-vertical-position: …"` — settings
     are passed to CSS through **custom properties**, not generated stylesheets.
3. A `{% schema %}` block at the end, with all user-facing labels as `t:` keys.

`snippets/theme-setting-vars.liquid` and `snippets/theme-globals.liquid` set the
global CSS custom properties and the `window.theme` JS globals; look there before
inventing a new mechanism for passing a setting to CSS or JS.

Product page layout is block-driven: `snippets/product-blocks.liquid` is the
dispatcher that `case`s on `block.type` and renders the matching
`snippets/product-block-*.liquid`. Add a new product block by adding a `when`
branch there, a `product-block-<name>.liquid` snippet, and a block entry in the
schema of `sections/main-product--default.liquid` (and `--quick` if it should
appear in quick view).

## JavaScript

`assets/theme.js` (unminified, ~14k lines) and `assets/theme.min.js` (loaded by
default) are both **vendor build outputs**; the original source is not in this
repo. Consequences:

- `snippets/theme-globals.liquid` has a `useUnminThemeJS` flag (default `false`).
  Flip it to `true` to serve the readable bundle while debugging — but flip it
  back, and remember any edit made only to `theme.js` is invisible in production
  until it is also applied to `theme.min.js`.
- Prefer not to patch the bundles at all. Any patch is lost on a theme upgrade.

Architecture: a small `Section` class plus a `register('<type>', {…})` registry.
At runtime, elements with `data-section-type` are matched to their registered
handler by that string, and `data-section-id` identifies the instance. Handlers
expose `onLoad/onUnload/onSelect/onBlockSelect/…` so the theme editor can
re-render live. There are ~58 registered types.

Heavy libraries are lazy-loaded as chunks declared on `window.flu.chunks` in
`snippets/theme-globals.liquid`: `photoswipe`, `swiper`, `nouislider`,
`polyfillInert`, `polyfillResizeObserver`. Note that
`polyfill-resize-observer-chunk.js` is declared but **not present in `assets/`**
— it 404s if something requests it.

## CSS

`assets/theme.css` is the primary compiled stylesheet (~19k lines, includes
inlined vendor CSS such as Swiper). `assets/theme-deferred.css` is preloaded
non-blocking. Per-template sheets (`template-article.css`, `template-blog.css`,
`template-customers.css`, `template-gift-card.css`, `template-password.css`) and
partials (`partial-flag-icons.css`, `partial-shopify-product-reviews.css`,
`component-shoppable.css`) are loaded where needed.

**Store-specific styling belongs in `assets/custom.css`**, which already contains
EFEMERA button overrides. ⚠️ It is currently **not loaded**: `useCustomCSS` in
`snippets/theme-globals.liquid` is `false`. If a change to `custom.css` is
supposed to take effect, that flag must be set to `true` in the same change.

The same applies to `assets/custom-events.js` (documented storefront events like
`cart:item-added`, `product:variant-change`, `quickview:loaded`), gated behind
`useCustomEvents`, also `false`.

## Store-specific customizations (vs. stock Stiletto)

Keep these in mind when upgrading the theme or diffing against vendor source:

- `snippets/gtm-customer-events-storefront.liquid` — Google Tag Manager click and
  link tracking that republishes events through `Shopify.analytics.publish`. It is
  rendered as the **first thing in `<head>`** in `layout/theme.liquid`.
- `assets/custom.css` — EFEMERA button treatment (text-only primary, outline on
  hover, underline callouts).
- `assets/efemera-*.jpg` — brand imagery committed as theme assets.
- The 42 bespoke product templates and the design settings in
  `config/settings_data.json` (palette `#fcfcf9` / `#3d352f`, Fraunces headings,
  Instrument Sans body, Work Sans logo).

## Translations

Never hardcode user-facing copy.

- Storefront strings: `locales/en.default.json`, referenced as `{{ 'a.b.c' | t }}`.
- Theme-editor labels (schema `label`, `info`, `content`): `locales/*.schema.json`,
  referenced as `"t:settings_schema…"` / `"t:sections…"`.
- Shipped locales: `en` (default), `de`, `es`, `fr`, `it`. Adding a key to
  `en.default*.json` is the minimum; other locales fall back to English.

## Verification

There is nothing to run locally in this container — no linter, no tests, no dev
server, and Shopify CLI is not installed. Verification is limited to:

- Re-parsing any JSON you edited (with the comment-stripping snippet above).
- Checking that every `type` in a JSON template resolves to a `sections/*.liquid`
  file, and every `{% render 'x' %}` resolves to a `snippets/x.liquid` file.
- Confirming new `t:` keys exist in `locales/en.default.schema.json`.

Do not claim a change is visually verified. Real verification happens by
previewing the theme in Shopify.

## Conventions checklist

- Liquid logic goes in a `{%- liquid -%}` block at the top of the file, not
  interleaved with markup.
- Whitespace-controlling delimiters (`{%-`, `-%}`) are used consistently.
- Filenames are kebab-case; snippet prefixes group related files
  (`product-block-*`, `get-*` for pure computation snippets that only `echo` a
  value, `icon-*`, `filter-*`).
- Snippets document their expected inputs in a leading `{%- comment -%}` block —
  add one when you create a snippet, and pass parameters explicitly via
  `{% render 'name', key: value %}` (`render` does not inherit outer scope).
- Two-space indentation in Liquid, JSON, CSS, and JS.
