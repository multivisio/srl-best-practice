# PDF output (PDFreactor)

## Pipeline

- `src/entries/pdf.ts` is the entry of the pdf build. It imports
  `#imports/pdf.scss` (all pdf styles) and runs the PDF scripts. The result is
  `.output/pdf/pdf.css` + `pdf.js`, referenced by the PDFreactor configuration
  (`userStyleSheets`, `userScripts` with `beforeDocumentScripts`).
- Loading `pdf.js` / `pdf.css` through the `pdf-configuration.xml`
  (`userScripts`, `userStyleSheets`) instead of the design makes the PDF scripts
  run in the nswow PDF preview next to the editor as well, not only in the final
  PDF. Layout scripts (page sides, splitting) then also apply in the preview, so
  no separate no-JavaScript fallback styles are needed for it.
- The other way: upload `pdf.js` and `pdf.css` on their own in ns.wow and leave
  `userScripts` / `userStyleSheets` out of the `pdf-configuration.xml`. Never do
  both, the files would be included twice. Which way a project uses is the
  project's decision; a build script should then write exactly that variant.
- Keep the element order of the live `pdf-configuration.xml` that ns.wow expects,
  another order can cause problems there: `userScripts`
  (`beforeDocumentScripts` + `uri`), `userStyleSheets`, `viewerPreferences`,
  `appendLog`, `addTags`, then debug-only elements such as `inspectableSettings`.
  Elements a project does not use are left out, the rest keeps this order. When
  a build script writes the file, write it in exactly this order.
- `inspectableSettings` (`<enabled>true</enabled>`) makes the PDF inspectable
  (structure and styles can be read out). Good for local renders and for a short
  live check; remove it from the live configuration again afterwards.
- Library helpers in `srl/pdf/`:
  - `PDFNestedContainers({ selector: '.srl-nested-container' })` adds
    `<class>-first` / `<class>-last` to containers based on their first/last child,
    which the spacer `component-margin` rules use in the pdf build.
  - `PDFNotes({ noteClass })` groups notes and marks first/last.
  - `PDFSetPageNumbers({ tocItemClass, tocItemPageNumberClass })` fills the table
    of contents with page numbers.
- Customer builds: `pdf/customers/<name>-<suffix>/` with optional `custom.ts`,
  `custom.scss` and `public/`. `srl build -t pdf --customer <folder|all>` copies
  pdf/word/xbrl output into `.output/ldd/…` and writes `pdf-configuration.xml` and
  `pdf-configuration-debug.xml` pointing to
  `$INTERNAL_LDD_URL/<design>/<version>/pdf/…`. The customer name is the folder
  name up to the first `-`.

## Script timing in PDFreactor

- PDFreactor lays the document out, runs the scripts, and renders the modified
  DOM. Everything must run synchronously in `DOMContentLoaded` / `load`.
- `pdf.js` is not always loaded before the document: as a user script without
  `beforeDocumentScripts`, or through a script tag added later, `DOMContentLoaded`
  and `load` are already over and their listeners never run (nothing of the PDF
  scripts happens). Inline, a static `<script src>` in the head and a user script
  with `beforeDocumentScripts` work. Start the entry and the layout scripts with a
  check of `document.readyState`: `'loading'` → wait for `DOMContentLoaded`,
  otherwise run now; for layout scripts `'complete'` → run now, otherwise wait for
  `load`. Constructors keep their order, so a script that must run before another
  still does.
  `setTimeout` and Promises run too late and are effectively ignored.
- Layout information (`ro.layout.getBoxDescriptions(el)`) is only available in
  `load`. `ro.layout.forceRelayout()` lays the document out again synchronously.
- **Reading layout after any DOM change triggers a full relayout, once per
  read.** Read everything you need first (a compact snapshot: page index,
  left/right, lines), then change the DOM, then read again only after an explicit
  `forceRelayout()`. For page-side logic: read all sides first, then write all
  classes.
- A snapshot of all boxes of a large document can exhaust PDFreactor's memory;
  keep it to the data you need (e.g. tables only down to `tr`).
- Moving DOM subtrees is expensive (roughly 0.3 ms per node). Split by moving
  only what goes to later pages.
- Pure DOM changes that do not read layout (replacing hidden elements, adding
  classes) are cheap. Run them first, before any layout-dependent script.
- Elements with `display: none` do not take part in the layout; replacing them
  does not shift anything.

## CSS support in PDFreactor

- `:has()` is **not** supported. Set a class on the element instead (in the
  template or from the PDF script) and select on that.
- `leader('.')` works in `content` (e.g. dotted ToC leaders up to the page
  number); `target-counter(attr(…), page)` for page references.
- In scripts, `node.replaceWith(a, b)` with several nodes does not work;
  replace with one node and insert the next with `after()`.
- Synthetic bold: a font with only a regular face gets a faux bold from
  `font-weight: 600` on, never at 500. PDFreactor decides by the weight stored in
  the font file, not by the `@font-face` descriptor: registering a Medium file
  for `font-weight: 700` still adds a faux bold on top. So a family with a real
  Medium face (bold = 500) and one with only a regular face (bold >= 600) need
  different bold weights, e.g. a CSS variable `--…-font-weight-bold` that the
  components in the regular-only font switch to 700 (word gets a plain value,
  it knows no variables). Light can never be synthesized.
- Hyphenation limits: `hyphenate-limit-chars: auto auto 3` (word length,
  letters before, letters after the hyphen) works in PDFreactor and Chromium;
  the prefixed `-ro-hyphenate-limit-chars` does not. Without it PDFreactor
  leaves only 2 letters on the next line.

## Page layout

- `@page :left` / `@page :right` with margin boxes (`@top-left-corner`, …);
  PDFreactor-specific properties use the `-ro-` prefix
  (`-ro-truncate-margin-before-break`/`-after-break` only work inside `@page`).
- Running headers: an element with `position: running(<name>)` leaves the flow
  and is shown with `content: element(<name>)` in a margin box. A page shows the
  first running element assigned on it, otherwise the last one from earlier
  pages. A running element keeps its value until the next one appears, so empty
  values must be explicit in the content (e.g. an empty subtitle).
- A pattern that works for headers: keep the editable title/subtitle
  components hidden (`display: none`) and let a small script replace them, in
  document order, with one list element (`ul.srl-pdf-header` with `__title` /
  `__subtitle` items, empty values omitted) that is the running element.
- Page sizes come from `meta.pdf` (`margin`, `image-quality`) and `@page`
  (`-ro-media-size`, bleed, crop).
- Use `break-inside: avoid` on table rows and on cells with `rowspan` to keep
  spanning rows together.
- Split tall grid elements only between lines (text) or rows (tables), clone the
  wrapper path, table `colgroup`/`thead`, and mark continuations with a class.
- Column widths of exported tables come as `<style>` blocks inside each table
  (`.srl-table-tabXXX-colN { width: N% }`, one per column of the table
  definition). The PDF keeps them; a local renderer must not strip body styles.
  The selectors are global: when the same table definition (`tabXXX`) appears
  several times with different widths, the last definition wins for every table
  of that definition in the full PDF, while the editor (one page) looks right.
  Check the export for conflicting definitions before suspecting the design; the
  clean fix is a table definition of its own for the differing tables.

## Document ns.wow sends to PDFreactor

- No doctype, `pdf.css` inline as the first `<style>` (with `@import` of the design's
  `fonts/style.css`), `pdf.js` inline as `<script>` in the head, `<body class="pdf">`.
- First child: `p.srl-pdf-publication-title > span.srl-pdf-publication-title__text`
  (`publicationName` from `settings.json`).
- Then one `div.srl-chapter.ns-chapter.srl-<chapter name>` (`id='srl-chapter-id-<ld id>'`)
  per chapter. Chapter names are not translated (`srl-business-report` in every language).
- Inside, one `div.srl-article.ns-article.srl-<page name>` per article
  (`id='srl-article-id-<ld id>'`, `lang`); the class is the page name lower-cased, spaces to
  `_`, other punctuation dropped (`05_06_14 Property, Plant and Equipment` →
  `srl-05_06_14_property_plant_and_equipment`). Articles of a category also get
  `srl-category-<category page>` (e.g. `srl-category-notes_group`, used by `PDFNotes`).
- ns.wow inserts a `div.srl-chapter-navigation` (`lc-` classes, `--current` item) at the
  start of the first article of every chapter that has a chapter cover
  (`srl-pdf-chapter-cover`); the items are those chapters with the cover titles. `pdf.css`
  turns it into the running header.
- The app menu (`menu.menuMain` in `routing_<lang>.json`) has nothing to do with the PDF.
  The PDF order matches the page names sorted numerically (`01_02 …` before `01_10 …`), and
  the chapters match the leading number (`00`, `01`, …, `99`).
- ns.wow applies `data-remove-from-pdf` and `data-replace-tag` to the article HTML; the
  other `data-remove-from-*` attributes stay in the document.
- Chapters without a chapter cover (e.g. `cover-homepage-toc`, `imprint-backcover`, `hosting`)
  get no chapter navigation. Their names are not in the export.
- Article links in the export: `href="./<lang>/<slug>" data-li-document-ref="<ld id>"
  data-article-name="<uuid>"` (`data-li-document-ref` only on some links). In the PDF the
  same link is `href="#srl-article-id-<ld id>"`, `data-li-document-ref` stays.
- The app does not apply `data-remove-from-web` itself (`.srl/utils/html.ts` does not read
  it): ns.wow removes it before export. The proper place for it is the srl's HTML rewrite,
  not the project. As a local, uncommitted stopgap for raw HTML: hide
  `[data-remove-from-web='complete']` and set `'transient'` to `display: contents` in
  `app.scss` (not `web.scss`, the editor shares it).
- Links to other articles become `#srl-article-id-<id>` or `#<anchor>`, `data-article-name`,
  `data-note-target` and `loading="lazy"` are dropped, images are absolute imageservice URLs.
- The web export in `public/` (`json/routing_<lang>.json`, `json/settings.json`,
  `html/<lang>/*.html`, `images/`) is the **web** version: `data-remove-from-web` is already
  applied, `data-remove-from-pdf` is still in it. PDF-only articles (covers, TOC, chapter covers,
  imprint, backcover) and PDF-only components (`srl-pdf-pagebreak`,
  `srl-pdf-landscape-container`) are not in it. Category overview pages (`notes_group`) and
  pages with `excludeFromPdf` are not in the PDF. A local renderer can rebuild this
  document from `public/`.
- ns.wow adds two attributes the srl does not set or read (nothing in `pdf.js`/`pdf.css`
  uses them, they stay in the PDF):
  - `data-li-document-ref="<ld id>"`: on `span.srl-key-figure` (key figures embedded from
    their own Livingdocs documents) and, in the PDF, on every `a.srl-pdf-toc__item` with the
    id of the target article (`href="#srl-article-id-<same id>"`). Article ids are Livingdocs
    document ids, but the web export only has them on key figures, not on articles.
  - `data-linked-category="<category page name>"`: on the `th`/`td` of the notes column
    (class `notes`) in financial tables; the cells link to the notes with
    `a.note-link[data-note-target="popup"]`. Names the category page (`notes_group`), not
    its id.

## Testing PDFs

- Render locally with a PDFreactor Docker container and the debug configuration
  (`appendLog`: `console.log` output is appended to the PDF). Use it to find
  root causes instead of guessing.
- Without a licence PDFreactor adds a watermark and an "Evaluation Version" info
  page after page 1 that is not counted by the page counter. For spread checks,
  insert two blank pages at the start of the test export
  (`<div class="srl-pdf-pagebreak" style="height: 1pt;"></div>` twice after
  `<body>`), so the info page sits between them and page 3 starts a spread.
  Keep a copy of the untouched export.
- Render every experiment into its own PDF, compare spreads visually and diff the
  extracted text (`gs -sDEVICE=txtwrite`) against the reference so no content is
  lost. Large reports take minutes: run them in the background.
- State page-side rules concretely ("content on the side opposite the page
  number") and confirm them before changing code; inner/outer is ambiguous when
  the evaluation page shifts physical page numbers.
