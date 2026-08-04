# CLAUDE.md — EFEMERA

Shopify storefront theme for EFEMERA. Base theme is **Stiletto v5.2.2** by
Fluorescent (licensed, vendor-maintained). No build system, no tests, no CI —
plain Liquid/JSON/CSS/JS served by Shopify.

## Repo-specific traps

- **`main` is written by Shopify, not just by humans.** Every commit is
  `Update from Shopify for theme efe/main` — the GitHub theme integration syncs
  merchant edits back into `templates/*.json`, `sections/*-group.json`, and
  `config/settings_data.json`. Hand-editing those is racy; they get overwritten.
  Work on a branch, rebase on `origin/main` before pushing.

- **`assets/custom.css` is dead code.** It holds the EFEMERA button overrides
  (text-only primary, outline on hover) but is *not loaded* — `useCustomCSS` in
  `snippets/theme-globals.liquid` is `false`. Editing it changes nothing unless
  that flag is flipped in the same change. Same for `assets/custom-events.js`
  and `useCustomEvents`.

- **`theme.min.js` is what ships**; `assets/theme.js` is the readable twin
  behind the `useUnminThemeJS` flag. Both are vendor build outputs with no
  source in this repo. A patch to only one of them is invisible in production,
  and any patch is lost on a theme upgrade.

- **`polyfill-resize-observer-chunk.js`** is declared on `window.flu.chunks` but
  is missing from `assets/` — it 404s if requested.

## Local customizations (vs. stock Stiletto)

Check these first on any vendor theme upgrade:

- `snippets/gtm-customer-events-storefront.liquid` — GTM click/link tracking,
  rendered as the first thing in `<head>` in `layout/theme.liquid`.
- `assets/custom.css` — EFEMERA button treatment (currently not loaded, above).
- `assets/efemera-*.jpg` — brand imagery committed as theme assets.
- 42 bespoke `templates/product.*.json` templates.
- Design settings in `config/settings_data.json`: `#fcfcf9` / `#3d352f`,
  Fraunces headings, Instrument Sans body, Work Sans logo.

Most imagery is referenced as `shopify://shop_images/*.jpg` — those live in
Shopify Files, not this repo, and can't be added by committing to `assets/`.

## Verification

Nothing runs locally: no linter, no tests, no dev server, no Shopify CLI. You
can re-parse edited JSON, check that section `type`s and `{% render %}` targets
resolve to real files, and confirm new `t:` keys exist in
`locales/en.default.schema.json`. **Do not claim a change is visually verified** —
that requires a Shopify preview.
