# srl-best-practice

A Claude Code plugin with best practices for projects built on the
[Simple Reporting Library](https://github.com/mmssolutionsio/simple-reporting-library)
(`@simple-reporting/base`, CLI `srl`).

The skill loads automatically in srl projects and covers:

- the `srl.config.json` design tokens and what the library generates from them
  (SCSS mixins and functions, CSS variables, utility classes)
- Livingdocs component folders, `ld-conf.json`, properties and the design validator
- the per-target SCSS files and builds (app, Livingdocs editor, PDF, Word, XBRL)
- PDFreactor script timing, performance and page layout
- how to check safely whether a token or component is unused before removing it

A second skill, `srl-migrate-srl2`, describes how to migrate an SRL 1 project to SRL 2
(`@simple-reporting/core`, Nuxt) and lists the known pitfalls.

Written against `@simple-reporting/base` 1.0.53.

## Installation

```
/plugin marketplace add multivisio/srl-best-practice
/plugin install srl-best-practice@srl-best-practice
```

## Structure

```
.claude-plugin/marketplace.json
plugins/srl-best-practice/
  .claude-plugin/plugin.json
  skills/srl-best-practice/
    SKILL.md                          core rules
    references/config-and-scss.md     srl.config.json and the srl SCSS module
    references/components-and-build.md components, properties, builds
    references/pdf.md                 PDF target and PDFreactor
  skills/srl-migrate-srl2/
    SKILL.md                          migration workflow and pitfalls
    references/mapping.md             SRL 1 → SRL 2 files, aliases, imports
```
