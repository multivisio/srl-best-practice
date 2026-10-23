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
   Variable names need no `pdf`/`web` label: customer overlays only reach the PDF.
   Never rename variables customers already set (e.g. `--srl-table-background-NN`).
   Examples in advanced-ld (branch `feature/customer-css-variables-base`, not in main yet):
   borders and paddings around titles per side, `--srl-title-h1..h6-border-<side>` /
   `-padding-<side>` and `--srl-heading-level-7..9-border-<side>` / `-padding-<side>`
   (mixin `src/assets/scss/mixins/_border-padding.scss`, defaults: none / 0). Sub- and
   superscript offsets have no variables on purpose; they stay as customer CSS for now.
3. Customer-specific selector rules sometimes need `!important` to beat base styles.
   Keep them in the matching partial (`table.scss`, `typography.scss`, ...).
4. Fonts: declare `@font-face` in `scss/fonts.scss` with a relative `$font-path: "./fonts"`.
   Only ship the font files that are actually referenced.
5. Customer PDF scripts go in `custom.ts` (e.g. `initializeTablePageSplit` from
   `@/utils/pdf/PDFTablePageSplit` with customer options). They run on top of the base
   `pdf.js`.

## Design handover via Figma

Template for aligning a customer's PDF design: Figma file "SRL advanced-ld – Kunden-Design-Tokens"
(https://www.figma.com/design/i9MXNC8Qa0kSteyYFCVk4f, MULTIVISIO team). It holds every CSS variable the
base exposes for the PDF (489 at creation, built from `.output/ldd/pdf/pdf.css` of main) as Figma variables
in five collections: 1 Farben, 2 Typografie, 3 Abstände & Raster, 4 Seite (PDF), 5 Komponenten. Each
variable has its CSS name as WEB code syntax and two modes: "Kunde" (edited) and "Basis" (reference).
Pages: Anleitung, Design Tokens, Komponenten (bound Figma components with explanations), Beispielseiten
(A4 text flow), Übergabe (mapping table and transfer rules).

- Workflow: duplicate the file per customer, change only the "Kunde" mode, transfer every value that
  differs from "Basis" as `:root` variable into the overlay (values in pt, aliases as `var(--srl-…)`,
  title borders combined as `<width>pt solid <color>`).
- Values are the print (PDF) values in pt; line-height is stored absolute in pt (valid in CSS too).
- When the base gets new or renamed variables, update the template file as well.

## Rendering a customer PDF locally

- Each customer has a `pdf-<name>` script next to `build-<name>`:
  `"pdf-greiner": "pdf-reactor pdf --customer nsp281-greiner"`. It runs the global
  `pdf-reactor` command (repo `multivisio/pdf-reactor`, `~/.local/bin/pdf-reactor`), which starts
  the PDFreactor Docker container on demand.
- Input is the customer's ns.wow PDF export, saved as `pdf/customers/<folder>/pdf.html`. The
  exports differ a lot between customers (especially tables), so each customer keeps its own.
  It is customer data: `.gitignore` has `/pdf/customers/*/pdf.html`. Check with
  `git check-ignore -v` before working with a new export and never commit it.
- The tool runs `npm run build-pdf -- --no-clean --customer <folder>` (project script
  `"build-pdf": "srl build -t pdf"`, no version prompt because there is no `ldd` target), strips
  the export's own `<link>`/`<script>` tags (they point at the deployed NSWOW files), adds local
  `pdf.css`/`pdf.js` and then the customer files in the order of
  `.output/ldd/pdf/nspXXX/pdf-configuration.xml`, served from `.output/ldd/pdf/` so relative
  font and asset URLs resolve as in NSWOW.
- Output: `.output/pdf-reactor/<folder>.pdf` (named after the customer), HTML sent to PDFreactor
  in `.output/pdf-reactor/last-wrapper.html`.
- Options pass through: `npm run pdf-greiner -- --watch` (re-renders on changes in the customer
  folder, `src/`, `livingdocs/`), `-- --skip-build`, `-- --file <other export>`,
  `-- --mode components` (all components with the customer styles, no export needed).
- Without a licence key PDFreactor adds a watermark and an evaluation page after page 1.

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
