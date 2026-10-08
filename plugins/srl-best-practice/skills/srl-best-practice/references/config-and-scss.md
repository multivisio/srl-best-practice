# srl.config.json and the `srl` SCSS module

## From config to SCSS

`srl.config.json` is read by `beaver` (scripts/beaver.js) on every Vite start, on
every change of the file in dev, and on every build. It writes:

- `srl/config.scss`: `@use "@simple-reporting/base/scss/<file>/variables.scss" with (…)`
  for each top-level key. Only these top-level keys are used:
  `system`, `fonts`, `meta`, `grid`, `colors`, `typography`, `spacer`, `helpers`.
  Each key inside them becomes a SCSS variable (`typography.typography` →
  `$typography`, `spacer.margins` → `$margins`, …).
- `srl/typography.scss`: one mixin per typography entry.
- `srl/colors.scss`: one function per color.
- `.srl/composables/srlConfig.ts`: the whole JSON as a Vue ref (`useSrlConfig()`).

Arrays of objects with a `name` key become maps keyed by that name, so the order
in the array does not matter and **duplicate names silently overwrite each
other** (the last one wins). Keep names unique.

Strings that contain a `.` or `,` outside of parentheses and are not a size are
quoted automatically (font stacks, `", "`).

`srl/index.scss` forwards everything with prefixes, which is why the API is
`srl.typography-…`, `srl.colors-…`, `srl.spacer-…`, `srl.grid-…`,
`srl.system-…`, `srl.fonts-…`, `srl.helpers-…`, `srl.button-…`; `meta` is
forwarded without prefix (`srl.$meta`). Every SCSS file starts with `@use 'srl';`.

## Units

`srl.system-size-unit($value)` converts **unitless numbers** (design px at 96 dpi,
root size 16) into the target unit: `rem` for app, editor, pdf and xbrl, `pt` for
word. Values with a unit (`"7pt"`, `"118pt"`) are kept. Line heights are unitless
and become `em`. So:

- Screen values: unitless numbers.
- Print-only values: strings with `pt`.
- `0` stays `0`; `false`/`null` become `unset`.
- Never wrap a value that already has a unit: `srl.system-size-unit(12pt)` just
  returns `12pt` (only `number` + `math.is-unitless` + `!= 0` is converted, see
  `size-unit()` in `scss/system/functions.scss`). Write `12pt` directly, or pass
  a unitless value if it should follow the target unit.

## Typography

Entry keys: `name`, `font-family`, `font-size`, `line-height`, `font-style`,
`font-weight`, `color` (a color name), `letter-spacing`, `text-transform`,
`margin-top`, `margin-bottom`, `media`.

`media` overrides per breakpoint: `print`, a breakpoint name (only that range),
`up: { <bp>: {…} }`, `down: { <bp>: {…} }`. `print` values end up in
`@media print { :root { … } }`, which is what PDFreactor and the word build use.
Inside `media` only override `font-family`, `font-size`, `line-height`,
`font-style`, `font-weight`, `margin-top`, `margin-bottom`. The library reads
`color`, `letter-spacing` and `text-transform` of a media block from a variable
that only exists for the base definition, so setting them there breaks the Sass
build.

For every entry the library generates:

| Output | Name |
|---|---|
| Mixin | `@include srl.typography-<name>($margins: false)` |
| Getters | `srl.typography-get-font-family\|font-size\|line-height\|font-style\|font-weight\|font-color\|letter-spacing\|text-transform\|margin-top\|margin-bottom('<name>')` |
| CSS variables | `--srl-typo-<name>-<property>` on `:root` |
| Utility class | `.srl-typo-<name>` (all targets) |

The mixin and most getters throw a Sass `@error` for unknown names; the getters
`get-font-weight`, `get-letter-spacing` and `get-text-transform` do not check the
name and silently return `var(--…, unset)`. Double-check names used with them.

Typographic changes belong in `srl.config.json`: if a look exists as its own
typography entry, change the entry there instead of overriding it in the
component SCSS. `line-height` is a unitless factor like everywhere else in the
config (12pt on 9pt text: `1.333333`, 13pt: `1.444444`). Overriding a single
value in a component is fine for real special cases, preferably with a getter
from another entry (e.g. `line-height: srl.typography-get-line-height('paragraph')`
to put a list on the grid of the running text).

Use the mixin for full text styles, the getters when only single properties are
needed (e.g. table cells that take the size from one typo and the weight from
another). Editor-only labels use a dedicated typo (`editor-label-text`).

## Colors

`colors.colors[]` with `name` and `color`. Generated: `srl.colors-<name>()`,
`srl.colors-get('<name>')`, `--srl-color-<name>`, `.srl-color-<name>`,
`.srl-bg-<name>` (which also sets the text color from `on-<name>` if that color
exists). A color named `shade` automatically gets `shade-50` … `shade-950` via a
palette generator unless they are defined explicitly.

## Spacer

- `spacer.spacer`: a scale (`"200": { "size": "8pt" }`, optional `media`).
  Use `srl.spacer-get(200)` or the mixins `srl.spacer-margin-top(200)`,
  `margin-block`, `padding-…`, `gap`, `row-gap`, `column-gap`. Unknown keys throw.
  Without a `print` media value, the base size is also used for print.
- `spacer.margins.group`: vertical rhythm between components.
  `"title-h1": { "all": 400, "paragraph": 200 }` produces
  `.srl-margin-group-title-h1 + * { margin-top: … }` and
  `.srl-margin-group-title-h1 + .srl-paragraph { … }`. A component opts in with the
  class `srl-margin-group-<group>` in its HTML and
  `@include srl.spacer-component-margin(<group>)` (or a map of rules, e.g. from a
  local `_spacing-variations.scss` partial). The mixin also handles nested
  containers and aside/content containers; in the pdf build it adds the
  `-first`/`-last` edge classes set by `PDFNestedContainers`.
  Keys in a group must reference existing component classes; entries for removed
  components are dead rules.

## Grid

`grid.breakpoints` (the `print` breakpoint maps to `@media print`),
`grid.columns`, `grid.gutter`, `grid.containers`. API: `srl.grid-media(<bp>)`,
`media-up`, `media-down`, `media-between`, `col`, `offset`, `row`, `container`,
and for PDF `srl.grid-pdf-flex-col($span)`,
`srl.grid-calculate-pdf-col-span-minus-one-gutter($span)`,
`…-plus-one-gutter($span)`, `srl.grid-calculate-pdf-col-start($col)` (based on
`columns.print`). CSS variables: `--srl-gutter-column-gap`,
`--srl-container-max-width`, `--srl-container-padding`, `--srl-breakpoint-…`.

**`srl.grid-col($span, $bp)` and `srl.grid-offset($offset, $bp)` apply only
within the range of `$bp`**, not from `$bp` upwards: they use `grid-media`, which
ends one pixel before the next breakpoint. For "from this breakpoint on" wrap the
call: `@include srl.grid-media-up(desktop) { @include srl.grid-col(8); }`.
`grid-col($span, $start, $end)` uses `media-between`.

`srl.grid-get-breakpoint($bp)` returns `var(--srl--breakpoint-…)` (double hyphen,
the prefix already ends with `-`), a variable that does not exist. Use
`var(--srl-breakpoint-<bp>)` or `map.get` on the breakpoints instead.

## Meta

Free-form project settings (`meta.meta`). Read them with
`map.get(srl.$meta, pdf, margin, top)`; values are not converted, wrap sizes in
`srl.system-size-unit(…)`. Convention: a `-pdf` suffixed key next to the screen
key (`number-width` / `number-width-pdf`).

## CSS variables vs. word

`srl.system-root-style('<var>')` returns `var(--<var>, default)` in every target
except word, where the value is resolved at compile time (print value first) and
pt values are rounded. Word has no custom properties, so never write `var(--srl-…)`
by hand in code that also compiles for word; use the functions.

## Global styles and placeholders

- `src/assets/scss/general.scss` goes into app, ldd, pdf and word; `app.scss`,
  `editor.scss`/`ldd.scss`, `pdf.scss`, `word.scss`, `xbrl.scss` into their target.
- Every file in `src/assets/scss/placeholders/**` is `@use`d by `srl/index.scss`,
  so its `%placeholders` can be `@extend`ed anywhere after `@use 'srl'`
  (e.g. `%srl-regular-width`, `%srl-grid-base`). The files are loaded with
  `@use … as <alias>`, not forwarded: **mixins, functions and variables defined
  in a placeholder file are not available as `srl.…`**. `@use` that file
  directly where you need them.
- Fonts: `src/assets/fonts/**/*.scss` are included automatically.
