---
name: srl-migrate-srl2
description: >
  Workflow for migrating a Simple Reporting Library 1 project (@simple-reporting/base,
  Vue 3 + Vite, CLI `srl`) to SRL 2 (@simple-reporting/core, Nuxt 4): project setup,
  data import, livingdocs and config takeover, porting the app shell from src/ to app/,
  builds and parity checks, plus the known pitfalls.

  TRIGGER THIS SKILL WHEN:
  - The user wants to move, port, migrate or upgrade an srl project to SRL 2 / Nuxt /
    @simple-reporting/core
  - A project with @simple-reporting/base in package.json should run on SRL 2
  - Debugging differences between an SRL 1 project and its SRL 2 port
---

# Migrating an SRL 1 project to SRL 2

Learned from migrating a real customer project (133 own components, custom Izanagi
shell) with SRL 2 `2.0.0-alpha.0`. For SRL 1 conventions use the `srl-best-practice`
skill; for SRL 2 itself read the `README.md` of the SRL 2 sources.

Mapping tables (files, aliases, imports, what is dropped):
[references/mapping.md](references/mapping.md).

## Ground rules

1. **Never touch the SRL 1 project.** Create the SRL 2 project in a new folder and copy
   from the old one. The old project stays the reference.
2. **Take the project's components as they are.** Projects are often migrated from older
   systems and have nothing to do with the standard components of
   `@simple-reporting/livingdocs`. Replace the `livingdocs/` folder completely; do not
   compare it with or merge it into the standard set.
3. **Ask before choosing a direction** when there are several plausible ways (where assets
   come from, whether to fix SRL 2 or work around it, how far to port the shell).
4. **Bugs in SRL 2 are fixed in SRL 2**, not worked around in the project: feature branch
   in the package repository (core, livingdocs or create), test, rebuild core (`npm run build`
   in core), push the branch. Projects use the local clones via `file:` dependencies until the
   packages are on npm. Restart `srl dev` afterwards (see pitfall 11).
5. **Commit the migrated project in small steps** (data/livingdocs → shell → assets →
   builds), so each step can be compared.

## Workflow

1. **Prepare SRL 2.** Node 24 (`.nvmrc` of the workspace). In the workspace:
   `npm install` and `npm run build` (core `dist/` must be current).
2. **Create the project.** Before the packages are on npm, use the local packages:
   ```bash
   node <workspace>/packages/create/index.mjs <folder> --livingdocs none --yes --local <abs. workspace path>
   ```
   `--livingdocs none` only means "no alternative livingdocs package"; the 47 standard
   components are installed anyway and are replaced in step 4.
3. **Import the content data** from the downloaded nswow preview (zip or folder):
   `npx srl data import <zip> --yes` → `data/{json,html,images,downloads}`. The rest of the
   zip (old app build, `svg/`, `fonts/`, `css/`, `js/`, `favicons/` of the hosting) is not
   imported; see step 6 for assets the project still needs from it.
4. **Take over the design part** (format unchanged):
   - replace `livingdocs/` with the old folder
   - the SRL 2 project uses `srl.config.ts`. Delete the template `srl.config.ts`, copy
     the old `srl.config.json`, add `"project": { "livingdocs": { "alternative": null } }`
     (otherwise `srl dev` asks for it), then convert it; there is no `srl` command for it:
     ```bash
     node -e "import('@simple-reporting/core/config').then(m => m.migrateJsonConfig(process.cwd(), { removeJson: true }))"
     ```
     This writes `export default defineSrlConfig({...})` and deletes the JSON. Keep only one
     config: if both exist, the `.ts` wins and the `.json` is silently ignored.
   - `livingdocs.config.json`: copy only the hand-maintained keys. Drop the generated
     ones: `name`, `version`, `assets`, `groups`, `components`, `componentProperties`.
   - copy `pdf/pdf-configuration*.xml`; the template `pdf/entry.ts` already replaces
     `src/entries/pdf.ts` (same PDF helpers, new import paths).
   - set `name` in `package.json` to the old package name: the design name in
     `design.json` comes from it, and nswow identifies the design by name.
5. **First milestone: content renders.** `npx srl doctor` (0 errors expected; warnings
   are mostly content issues), then `npx srl dev` with the template shell. Fix SCSS and
   import paths until every page returns 200 (pitfalls 1–6).
6. **Port the app shell** from `src/` to `app/` (details in the mapping reference):
   copy `components/`, `composables/`, `locales/`, `assets/`, `lib/`, helper folders;
   `@/…` imports keep working because `@` is `app/` in Nuxt. Then rewrite the SRL 1
   specific parts: `App.vue` → `app/app.vue`, `RouterView`/`ArticleView` → `PageMain.vue`
   with `<NuxtPage />`, `entries/main.ts` install code → `app/plugins/*.ts`, own i18n →
   SRL 2 i18n, router `scrollBehavior` → `app/router.options.ts`, special pages →
   `app/components/Srl/Page/<Component>.vue`. Install only the npm packages the code
   really imports.
7. **Make the code SSR-safe.** SRL 2 prerenders every page. `window`, `document`,
   `localStorage` and `navigator` may only be used in `onMounted`, event handlers or behind
   `import.meta.client` (pitfalls 7–9).
8. **Build everything.**
   ```bash
   npx srl build -t generate --base /<path>/     # what nswow runs
   npx srl build <version> -t app,editor,pdf,word,xbrl
   ```
   Both must exit with 0. `srl build` cleans `dist/` first, so `dist/generate` from an
   earlier run disappears.
9. **Check parity.**
   - Browser test (Playwright from the workspace) for the start page, a few articles in
     every locale, search, language switch: status 200, no console errors, no failed
     requests, no hydration warnings, no leftover `<srl-ld-*>` tags in the DOM.
   - Serve `dist/generate` under the base path and test it the same way.
   - `design.json` vs. the old generated `livingdocs.config.json`: same groups,
     components and properties. Differences usually mean the old file was stale; compare
     with the component sources (`ld-conf.json`, minified `*.html`).
   - CSS: `node <workspace>/scripts/compare-css.mjs <srl1.css> <srl2.css>`. Concatenate all
     CSS chunks of each build first. Expect differences from components that are no
     longer bundled because nothing uses them (SRL 1 registered all of `src/components`).
10. **Write down what is left**: SRL 2 issues found (with branch), content warnings from
    `srl doctor`, assets that were already missing in SRL 1.

## Updating a migrated project

When the SRL 1 project keeps changing after the migration:

1. Note the SRL 1 commit the port is based on; diff from there (`git diff <old> <new>`).
2. `livingdocs/`: apply the diff as a patch, rewriting SCSS paths into the old shell first,
   e.g. `git diff <old> <new> -- livingdocs/ | sed 's#\.\./src/assets/#../app/assets/#g' | git apply`.
3. Shell files: if the SRL 2 copy still equals the old SRL 1 version, apply the diff with
   `--relative=src --src-prefix=a/app/ --dst-prefix=b/app/`; port the diverged ones by hand.
4. `livingdocs.config.json`: compare only the hand-maintained keys.
5. Content data (`public/json`) of the SRL 1 repo is not the source; import a new nswow zip.

## Known pitfalls

Status: the fixes for pitfalls 2–6 and the hook `srl:article:html` (pitfall 15) are in core
`main` since 2026-11-09 (after `2.0.0-alpha.0`). With older core versions those pitfalls still
apply.

1. **Misleading Sass error.** `Can't find stylesheet to import` pointing at
   `@use "@simple-reporting/core/scss/init-root.scss"` in `.srl/scss/entries/app.scss`
   usually means a *component* SCSS file imports something that no longer exists. Grep
   the livingdocs SCSS for paths into `src/`, e.g. `@use '../../../../src/assets/scss/tools/api'`
   → `app/assets/…`.
2. **`~` alias.** In Nuxt `~` and `@` are `app/`, `~~` is the project root. SRL 1 used
   `~` = project root. Older SRL 2 cores also used `~` = root in the editor/pdf/word/xbrl
   builds, so `url('~/assets/…')` worked on the site but failed in `srl build`
   (fixed: targets use the Nuxt meaning).
3. **Vue files with digits in the name** (`startpage-26-hero.vue`) were registered as
   `<srl-ld-startpage26-hero>` while the nswow HTML uses `<srl-ld-startpage-26-hero>`;
   the component silently stayed an unknown element (fixed: resolver camelizes like Vue).
4. **`.srl-article-root`.** SRL 1 rendered the article into `<article class="srl-article-root">`;
   project CSS and scripts rely on it. Older SRL 2 cores only had `.srl-article` (fixed).
5. **Relative image URLs in inline styles** (`style="background-image: url(./images/…)"`,
   `--large-url: url(../images/…)`) worked in SRL 1 through `<base href>`. SRL 2 has no
   base tag; older cores only rewrote `src`/`href`/`srcset` (fixed: also `style` and
   `<style>`).
6. **Overriding a special page** (`app/components/Srl/Page/Search.vue`) rendered
   `<srlpagesearch>` in older cores, because special pages resolve by name and project
   components are not global (fixed: `SrlPage*` overrides are registered globally).
7. **UMD bundles in `src/lib`** (e.g. a patched `swiper.js` with
   `export default globalThis.Swiper`) crash during SSR with
   `Cannot set property default of [object Module] which has only a getter`, because the
   CommonJS branch sees a read-only `module`. Remove the CommonJS/AMD branch and always
   use the global one.
8. **`document.title` in setup** (search page) breaks SSR → `useHead({ title: computed(…) })`.
9. **Plugins that read `localStorage` on install** (cookie consent) → initialise with
   `import.meta.client && localStorage…`; the banner reads the values in `onMounted`, so
   there is no hydration mismatch.
10. **Custom `v-t` directive.** vue-i18n registers its own `v-t` (keys only). A project
    directive that also accepts plain text must remove it first
    (`delete app._context.directives.t`) to avoid the duplicate warning, and needs
    `getSSRProps: (binding) => ({ textContent: … })` so the text is prerendered.
11. **Stale dev server after a core rebuild.** `npm run build` in the workspace deletes and
    rewrites `packages/core/dist`; a running `srl dev` then fails with
    `ENOENT … dist/runtime/pages/SrlPage.vue`. Restart it. `srl dev` starts
    `nuxt dev` child processes that survive a plain kill of the parent; stop them too,
    or the new server cannot bind the port and you keep testing the old one.
12. **Hosting assets.** SCSS like `$svg-path: "../svg"` pointed to the `svg/` folder that
    the nswow hosting put next to the SRL 1 app. SRL 2 does not import it. Agreed
    solution: copy the files the project uses into `app/assets/svg/` and set
    `$svg-path: "~/assets/svg"` so Vite bundles them for every target.
13. **macOS NFD file names.** SRL 1 projects sometimes have a Vite plugin that
    normalises `html/` file names to NFD. Check the zip
    (`unicodedata.normalize('NFC', name) != name`); with NFC names the plugin is not needed.
14. **The SRL 1 build in the nswow zip can be stale** (older than the component sources)
    and may hang a headless browser. Use it for orientation only; trust the sources.
15. **Vite plugins that patch the SRL 1 HTML pipeline** (a `transform` on `.srl/utils/html`
    that wraps `prepareHtmlContent`, e.g. to turn `.lc-anchor` into ids, because Vue patched
    DOM-only changes back). SRL 2 prepares articles on the server: register the Nitro hook
    `srl:article:html` in `server/plugins/*.ts` and rewrite `ctx.html` there with `parse5`
    (no `DOMParser` on the server). The result is in the prerendered HTML. The hook does not
    run in preview mode, so keep the client-side fallback if the project has one.
16. **Site search that fetches `./html/<lang>/<name>.html`** in the browser. SRL 2 does not
    ship the raw article HTML. Use the prebuilt index of `useSrlSearch({ locale })`
    (`index` = `{ uuid, path, title, text }`, text of h1–h3 and p).
17. **`index.html` head** (tag manager, favicons, title) → `app.head` in `nuxt.config.ts`.
    Links that SRL 1 resolved via `<base href>` need the base URL: `useHead` in `app.vue`
    with `useRuntimeConfig().app.baseURL`. Favicons come from the nswow zip (`favicons/`),
    which `srl data import` skips; copy them to `public/favicons/`.
18. **Project components with core names.** A file under the same path in `app/components/`
    replaces the core component everywhere, also inside other core components:
    `Srl/Menu/Item.vue` → `SrlMenuItem`, `Srl/Page/Search.vue` → `SrlPageSearch`. Usually this
    only happens by accident in a migration when old components happen to have these names;
    otherwise override on purpose only when there is no other way. `app/app.vue` is never
    affected (core ships no `app.vue`). When copying a core component as a template, fix its
    imports: relative paths no longer resolve, and core does not export its internal files
    (`@simple-reporting/core/dist/shared/uri` fails with "not exported"). Older cores
    registered all components globally, which printed `INEFFECTIVE_DYNAMIC_IMPORT` warnings
    for `SrlMenu`/`SrlMenuItem` on every build (fixed: only `SrlPage*` are global).
19. **404 page.** The nswow article with the slug `404` (per locale) is the error page; without
    one, the special page `ErrorView`. Unknown paths render it with status 404 and `noindex`,
    and `nuxt generate` replaces Nuxt's client-only `404.html` with the prerendered 404 page of
    the default locale (log: `404.html written from /<locale>/404`). On the client Nuxt then
    switches the URL to that path. Shell code that derives classes or links from the current
    page must use `useSrlCurrentEntry`/`useArticle` (they return the 404 page on unknown
    paths), not `byPath[route.path]`, or hydration of `404.html` mismatches. Older cores showed
    the Nuxt error page and wrote an empty `404.html`.
20. **`srl/` folder and IDE code completion.** SRL 2 writes the generated SCSS module to `srl/`
    in the project (gitignored, every file has a do-not-edit header), one folder for all
    targets; only `srl/config.scss` is rewritten per target build. Because it is a real
    folder, PhpStorm resolves `@use 'srl'`, `srl/typography`, `srl/fa` without setup; VS Code
    needs the Some Sass extension with `"somesass.workspace.loadPaths": ["."]` (new projects
    get it in `.vscode/`). Delete the SRL 1 `srl/` folder before the first `srl dev` and add
    `/srl/` to `.gitignore`. Project files with `@use "fa-font"`/`"fa-source"` still build, but
    IDEs only follow `srl/fa/font`/`srl/fa/source`. Older SRL 2 cores wrote the module to
    `.srl/scss/<target>/`, reachable only through the Vite alias (no IDE completion).
    TypeScript: Nuxt writes the aliases (`#composables`, `#imports`, `~`, `@`) to
    `.nuxt/tsconfig.app.json` (after `npm install`/`nuxt prepare`); core adds `livingdocs/`, so
    aliases and auto-imports also resolve in the livingdocs components (VS Code: Vue extension).
    Older cores left `livingdocs/` out. With `file:`-linked SRL packages, IDEs may not find
    `vue-i18n` types in the project (not hoisted); installed from npm they are.
