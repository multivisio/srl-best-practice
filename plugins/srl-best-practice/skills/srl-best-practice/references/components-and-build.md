# Components, properties and builds

## Folder layout

```
livingdocs/
  NNN.Group_Name/
    NNN.component-name/
      component-name.html   # Livingdocs template (exactly one *.html)
      ld-conf.json          # name, label, properties, directives, allowedParents …
      properties.json       # optional, component properties (global namespace!)
      app.ts                # optional, runtime class for data-autoload
      *.vue                 # optional, registered as async component SrlLd<Name>
      scss/{general,app,editor,web,pdf,word,xbrl}.scss
  999.Properties/<property>/properties.json + scss/   # shared properties
```

- Group label = folder name without the number prefix; the first `_and_` becomes
  ` / `, the first `_` becomes a space. Labels must be unique.
- Component name = folder name without prefix = `ld-conf.json` `name`. A folder
  only counts as a component if it has an `*.html` and an `ld-conf.json`.
- Order in the editor and in the aggregated SCSS follows the folder names
  (alphabetical, so keep the numeric prefixes zero-padded). `999.Properties` is
  always imported after all components.
- Renumbering folders is safe: only order changes, names stay the same.
- Scaffold with `npx srl create component <Group>/<NNN.name>` (non-interactive
  when both parts are given). It creates html, ld-conf.json and the SCSS files.

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
- Runtime behaviour in the web app: `data-autoload="<ClassName>"` (or a JSON
  array) plus `data-options='{…}'`; the class comes from the component's `app.ts`
  and is registered automatically under the camelCased folder name.
- Classes the editor may toggle belong in properties, not in the template.

## Properties

- `properties.json` files from **all** folders are merged into one
  `componentProperties` map. Keys must be globally unique; a duplicate key
  silently replaces the earlier one.
- A component lists the properties it offers in `ld-conf.json` `properties`.
- The validator fails for properties used but not declared, and warns for
  declared but unused ones (`component properties […] are unused`). Treat the
  warning as a cleanup hint.
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
- The LDD build maps components and properties into `livingdocs.config.json`,
  sets `name`/`version` from package.json and copies it to `design.json`.
- `srl prepare` (postinstall) deletes and recreates `.srl/` and copies the
  library's `srl/` files; the Vite plugin reruns it when the package version
  changes. Restart dev after updating the package.
- `srl build` exits with 0 even when a Vite build or the validator throws. Check
  the log.

## Aliases (set by the Vite plugin)

`@` → `src/`, `~` → project root, `#srl` → `.srl/`, `#components`,
`#composables`, `#plugins`, `#types`, `#utils`, `#imports` → `.srl/…`,
`#ld` → `livingdocs/`, `assets` → `src/assets`, `srl` → `srl/`,
`fa-source`/`fa-font` → Font Awesome free or pro (pro if
`@fortawesome/fontawesome-pro` is installed).
