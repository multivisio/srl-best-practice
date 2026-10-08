# Components, properties and builds

## Folder layout

```
livingdocs/
  NNN.Group_Name/
    NNN.component-name/
      component-name.html   # Livingdocs template (exactly one *.html)
      ld-conf.json          # name, label, properties, directives, allowedParents …
      properties.json       # optional, component properties (global namespace!)
      <name>.vue            # optional, global async component SrlLd<Name> / <srl-ld-name>
      <name>/*.vue          # optional, sub components imported by <name>.vue (not registered)
      scss/{general,app,editor,web,pdf,word,xbrl}.scss
  999.Properties/<property>/properties.json + scss/   # shared properties
```

- Every group folder becomes a group, even without components (`999.Properties`
  shows up as an empty group "Properties").
- Group label = folder name without the number prefix; the first `_and_` becomes
  ` / `, the first `_` becomes a space. Labels must be unique.
- Component name = folder name without prefix = `ld-conf.json` `name`. A folder
  only counts as a component if it has an `*.html` and an `ld-conf.json`.
- Order in the editor and in the aggregated SCSS follows the folder names
  (alphabetical, so keep the numeric prefixes zero-padded). `999.Properties` is
  always imported after all components.
- Renumbering folders is safe: only order changes, names stay the same.
- Scaffold with `npx srl create component <Group>/<NNN.name>` (non-interactive
  when both parts are given). It creates html, ld-conf.json, an empty
  properties.json and the SCSS files (`general`, `web`, `app`, `editor`, `pdf`,
  `word`, `xbrl`), then runs the mapping. **The group must exist**: for a new
  group run `npx srl create group <NNN.Group>` first, otherwise the command fails
  with `EEXIST`.
- `srl add components|groups` compares the exact path `<group>/<folder>` including
  the number prefixes. A component that was renumbered or moved to another group
  in the project is offered again and would be copied a second time. Loose files
  inside a group folder (`README.md`, `.gitkeep`) can show up as choices; do not
  select them.

## Templates

- BEM with `srl-` prefix: block `srl-<name>`, elements `srl-<name>__<part>`.
  Grid components use `srl-grid` on the root and `srl-grid__inner` inside.
- Livingdocs directives: `doc-editable`, `doc-container`, `doc-image`, `doc-link`,
  `doc-include`, … Directive names must be unique per component, `doc-link` may
  appear only once, and every directive configured in `ld-conf.json` must exist
  in the HTML.
- **Whitespace between tags is removed** when the design is built (`>\s+` and
  `\s+<`). Do not rely on spaces between inline elements; put them inside the
  text or use `&nbsp;`.
- Output filtering by the export (nswow): `data-remove-from-<target>` with
  targets `web`, `pdf`, `word`, `xhtml`, `translate-plus`:
  `complete` removes the element with its content, `transient` removes only the
  element and keeps its children. PDF-only components therefore carry
  `data-remove-from-web|word|xhtml="complete"`.
- Runtime behaviour in the web app: put a **Vue component in the component
  root** (e.g. `040.Media/010.table/table.vue`). Every `*.vue` directly in the
  component folder is registered automatically as a **global** async component
  `SrlLd<Name>` (`.srl/plugins/asyncLdComponent.ts`); use it in the template as
  `<srl-ld-<name>>` around the editable markup, which the component renders via
  `<slot />`. `.vue` files in sub folders are not registered; import them.
- **Every `<srl-ld-…>` custom element must carry `data-replace-tag` (e.g.
  `data-replace-tag="div"`) unless it is removed from the XHTML with
  `data-remove-from-xhtml`.** Otherwise the custom element stays in the XHTML
  output and makes it inconsistent.
- Vue components in the project's `src/components/` replace SRL components with
  the same generated name (e.g. `src/components/Srl/Page/Dialog.vue` replaces
  `SrlPageDialog`). This is the intended way to customise SRL runtime components.
- `app.ts` + `data-autoload` / `src/Autoload.ts` is an outdated technique: do not
  use it for new components and do not document it.
- Classes the editor may toggle belong in properties, not in the template.

## How the web app renders content

The app does not render Livingdocs JSON. nswow delivers the rendered article
HTML (`public/html/<locale>/<article name>.html`, plus `public/json/*.json` for
settings, routing, menus and translations). The app fetches that HTML and
compiles it at runtime as a Vue template (`vue3-runtime-template`). Consequences
for component templates:

- Custom elements resolve to the registered components (`<srl-ld-…>`, the SRL
  `<srl-category-accordion>` etc.). Simple Vue syntax works inside the exported
  HTML: `v-slot="{ accordion }"`, `:prop="value"`, `<template-<name>>` becomes a
  named slot `<template #<name>>`.
- Vue's default whitespace handling applies: whitespace-only text between tags
  that contains a line break is dropped, other whitespace runs become one space.
- Inline `<style>` blocks in the HTML are moved into one global style element.
- Links with `data-note-target="popup"` and `data-article-name="<uuid>"` open
  the target article in a dialog instead of navigating. Internal links become
  router links; `./<locale>/home` maps to the start page.
- In table cells, line breaks inside `<span>` become `<br />`.

## Properties

- `properties.js`/`properties.ts` are also read; they need a `default` export.
- `properties.json` files from **all** folders are merged into one
  `componentProperties` map. Keys must be globally unique; a duplicate key
  silently replaces the earlier one.
- A component lists the properties it offers in `ld-conf.json` `properties`.
- The validator fails for properties used but not declared, and warns for
  declared but unused ones (`component properties […] are unused`). Treat the
  warning as a cleanup hint.
- Changing a property from `option` (checkbox) to `select` with the **same key**,
  keeping the old value as one of the options, worked in Livingdocs without
  errors (tested live once, no docs on it). Existing documents keep the class.
- Property values are CSS classes on the component root; style them in the
  property's own scss folder (for shared properties) or in the component.

## Design validator (runs after the LDD build)

Fails on: duplicate component names, duplicate group labels, undeclared
properties, `allowedParents` that do not exist (`root` is allowed), components
without a group, duplicate/undeclared directives, `allowedChildren` or
`defaultContent` referencing missing components. Because `srl build` catches
errors, the failure only shows up in the log.

## Per-target SCSS aggregation

Generated into `.srl/imports/<target>.scss` in this order: srl config and root
variables → global fonts → `src/assets/scss/general.scss` (not xbrl) → global
target file → per component (alphabetical): `general.scss` (not xbrl), then the
target file → properties → `core-styles` (`xbrl-core-styles` for xbrl).

| Component file | app | ldd (editor) | pdf | word | xbrl |
|---|---|---|---|---|---|
| general.scss | ✓ | ✓ | ✓ | ✓ | – |
| app.scss | ✓ | | | | |
| editor.scss / ldd.scss | | ✓ | | | |
| pdf.scss | | | ✓ | | |
| word.scss | | | | ✓ | |
| xbrl.scss | | | | | ✓ |
| web.scss | via `@use "web"` in app.scss and editor.scss | | | | |

Consequences: styles that should appear in xbrl need `@use 'general';` in
`xbrl.scss`. After the build, **every `@media` block is removed from `xbrl.css`,
including `print`** (the log lists each removed block), so xbrl always renders
with the base values of typography, spacer and grid, never the print or
breakpoint overrides. Styles for
web and editor go into `web.scss`. Editor-only helpers (labels, borders, the
`srl.helpers-editor-label(…)` mixin) go into `editor.scss`.

## Builds

| Command | Output | `srl.$system-build` | unit |
|---|---|---|---|
| `npm run dev` | Vite dev server | app (development) | rem |
| `srl build -t app` | `.output/app`, `app.zip` | app | rem |
| `srl build <version> -t ldd` | `.output/ldd` (`design.json`, assets, fonts), `design.zip`, `v<version>.txt` | editor | rem |
| `srl build -t pdf` | `.output/pdf/pdf.{css,js}` from `src/entries/pdf.ts` | pdf | rem |
| `srl build -t word` | `.output/word/word.css` | word | pt |
| `srl build -t xbrl` | `.output/xbrl/xbrl.css` | xbrl | rem |

- `srl build` without `-t` builds everything. Targets can be combined: `-t pdf,word`.
- `srl build` empties `.output` first. `--no-clean` keeps it, e.g. for a quick
  `srl build --target pdf --no-clean` before a local test render that must not
  delete the `design.zip` of the last full build.
- **Raise the Livingdocs version only when `livingdocs.config.json` changes in
  content.** That comes from component HTML, `ld-conf.json` or `properties.json`.
  CSS/SCSS and `srl.config.json` token changes need no new version: build again
  with the current version from package.json. Compare the parsed config with the
  committed one (ignoring `version`) before deciding.
- The LDD build maps components and properties into `livingdocs.config.json`,
  sets `name`/`version` from package.json and copies it to `design.json`.
- `srl prepare` (postinstall) deletes and recreates `.srl/` and copies the
  library's `srl/` files; the Vite plugin reruns it when the package version
  changes. Restart dev after updating the package.
- `srl build` exits with 0 even when a Vite build or the validator throws. Check
  the log.
- A validator error stops everything after the LDD step: customer builds, XBRL
  `@media` stripping and both zips. A run that "finishes" without a fresh
  `.output/design.zip` failed.
- `-c <customer>|all` only runs together with the `pdf` target. It copies the
  whole `.output/pdf`, `.output/word` and `.output/xbrl` into `.output/ldd/`;
  when `ldd` is built in the same run they end up in `design.zip`. The XBRL copy
  is taken **before** the `@media` blocks are stripped, so that `xbrl.css` still
  contains them.
- `-c all` treats every entry of `pdf/customers/` as a customer, also plain
  files like `.gitkeep`; keep only customer folders there. `custom.ts` is the PDF
  entry of a customer, `custom.scss` is only built for XBRL (together with the
  base `xbrl.scss`).
- Font SCSS in `src/assets/fonts/**` is compiled into app and xbrl. ldd, pdf and
  word instead `@import "<INTERNAL_LDD_URL>/<package name>/<version>/fonts/style.css"`,
  so their fonts load from the deployed design version. `INTERNAL_LDD_URL`
  defaults to `https://nswow-ld.nswow.ch/designs`; it is also used for the URLs in
  the customer `pdf-configuration*.xml`.
- The build reads env files in this order, the first value wins: `.env.nswow`,
  `.env.production.local`, `.env.production`, `.env`.

## Project setup checks

- Projects generated by `srl init` need Node `>=22.14` (`engines` in the
  generated package.json; the CLI itself accepts 22.12).
- The generated `.gitignore` contains the typo `/**/*.lcoal`, so
  `.env.production.local` (read by the build) is **not** ignored. Add
  `/.env*.local` to the project's `.gitignore`.
- `public/{json,html,images,downloads}` hold the nswow content export and are
  gitignored; without them the dev app has no articles.

## Aliases (set by the Vite plugin)

`@` → `src/`, `~` → project root, `#srl` → `.srl/`, `#components`,
`#composables`, `#plugins`, `#types`, `#utils`, `#imports` → `.srl/…`,
`#ld` → `livingdocs/`, `assets` → `src/assets`, `srl` → `srl/`,
`fa-source`/`fa-font` → Font Awesome free or pro (pro if
`@fortawesome/fontawesome-pro` is installed).
