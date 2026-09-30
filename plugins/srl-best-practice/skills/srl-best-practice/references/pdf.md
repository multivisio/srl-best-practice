# PDF output (PDFreactor)

## Pipeline

- `src/entries/pdf.ts` is the entry of the pdf build. It imports
  `#imports/pdf.scss` (all pdf styles) and runs the PDF scripts. The result is
  `.output/pdf/pdf.css` + `pdf.js`, referenced by the PDFreactor configuration
  (`userStyleSheets`, `userScripts` with `beforeDocumentScripts`).
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
