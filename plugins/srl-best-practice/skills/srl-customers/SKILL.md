---
name: srl-customers
description: >
  Customer overlays on a shared srl base design (e.g. "advanced-ld" in
  nsfd_report-standard-customer): the pdf/customers/nspXXX-<name>/ folder layout,
  overriding base styles through CSS custom properties, customer fonts and PDF scripts,
  single-customer builds, packaging, and deploying overlays and the base design to NSWOW.

  TRIGGER THIS SKILL WHEN:
  - The project has a pdf/customers/ folder with nspXXX-<name> overlays
  - Adding, changing or debugging a customer's PDF/XBRL styles, fonts or custom.ts
  - Adding a new customer to a shared base design
  - Building, packaging or uploading a customer overlay or the base design.zip
---

# Customer overlays on a shared srl design

One base design (Livingdocs components, `srl.config.json`, base `pdf.scss`/`xbrl.scss`)
serves many customers. Each customer only adds a style overlay for PDF and XBRL. For
the srl library itself (components, tokens, `srl` SCSS module, PDFreactor) use the
`srl-best-practice` skill.

## Folder layout

```
pdf/customers/nspXXX-<name>/
  custom.scss      @uses the partials in scss/ (entry point for the customer CSS)
  custom.ts        imports ./custom.scss, plus optional customer PDF scripts
  fonts/           customer font files, referenced from scss/fonts.scss
  scss/            partials: variables, colors, fonts, typography, table, cover, ...
```

- The folder name is `nsp<number>-<name>`. The build output folder is only the part
  before the first `-`: `.output/ldd/pdf/nspXXX/` and `.output/ldd/xbrl/nspXXX/`.
- Each customer has its own `build-<name>` script in `package.json`
  (`srl build --customer nspXXX-<name>`). `srl build --customer all` builds all of them.
- Adding a customer = new folder under `pdf/customers/` (copy an existing one, e.g. the
  demo customer) + a matching `build-<name>` script.

## Styling rules

1. **Override variables, not selectors.** Base component SCSS exposes tweakable values as
   CSS custom properties via `srl.system-root-style(<var-name>, <default>)`. Customers set
   those variables or the `--srl-color-*` tokens on `:root` in their partials.
2. If a customer needs a value the base does not expose yet, prefer adding a new
   `srl.system-root-style(...)` variable to the base component (with the current value as
   default) over duplicating the selector in the customer overlay. Note: that is a base
   change, so the design.zip must be uploaded too (see Deployment).
3. Customer-specific selector rules sometimes need `!important` to beat base styles.
   Keep them in the matching partial (`table.scss`, `typography.scss`, ...).
4. Fonts: declare `@font-face` in `scss/fonts.scss` with a relative `$font-path: "./fonts"`.
   Only ship the font files that are actually referenced.
5. Customer PDF scripts go in `custom.ts` (e.g. `initializeTablePageSplit` from
   `@/utils/pdf/PDFTablePageSplit` with customer options). They run on top of the base
   `pdf.js`.

## Packaging

`package-designs.mjs` (postbuild, or `npm run package-designs`) zips `.output/ldd` into
`.output/packages/`:

- `<design>-base-<version>.zip`: design.json, assets, shared fonts, base pdf/word/xbrl.
- `<design>-nspXXX-<name>-overlay-<version>.zip` per customer: design.json plus
  `pdf/nspXXX/` and `xbrl/nspXXX/`.

The version comes from `design.json`.

## Deployment (NSWOW)

- **Normally only the overlay of the customer you worked on is uploaded.** NSWOW replaces
  that customer's folder with the new upload; other customers stay untouched.
- NSWOW builds the PDF configuration dynamically from the stored standard XML plus the
  paths to the customer's CSS and JS files. A customer overlay therefore takes effect
  without touching the base.
- **The base design.zip is only uploaded when the shared project really changed**,
  whether the change needs a new version or is just a CSS fix in the core.
- When you finish a task, tell the user which ZIP(s) to upload: only the customer overlay,
  or the base design too.
