# srl-best-practice

> [!IMPORTANT]
> **This repo has moved to [mmssolutionsio/srl-skillbox](https://github.com/mmssolutionsio/srl-skillbox).**
> It is no longer maintained here; all changes to the plugin go to the new repo only.
> The plugin and skill names stay the same, only the marketplace is now called `srl-skillbox`.

## Switching to the new repo

You need read access to `mmssolutionsio/srl-skillbox` (private repo) and a working GitHub
login for git, e.g. `gh auth login`. Then, in a terminal:

```
claude plugin uninstall srl-best-practice@srl-best-practice
claude plugin marketplace remove srl-best-practice
claude plugin marketplace add mmssolutionsio/srl-skillbox
claude plugin install srl-best-practice@srl-skillbox
```

Inside Claude Code the same works with `/plugin marketplace remove srl-best-practice`,
`/plugin marketplace add mmssolutionsio/srl-skillbox` and
`/plugin install srl-best-practice@srl-skillbox`. Restart Claude Code afterwards.

Check with `claude plugin list`: `srl-best-practice@srl-skillbox` should be listed, and no
entry `@srl-best-practice` any more. Future updates:

```
claude plugin marketplace update srl-skillbox
claude plugin update srl-best-practice@srl-skillbox
```

If you have a local clone of this repo, point it at the new one (the history is the same):

```
git remote set-url origin https://github.com/mmssolutionsio/srl-skillbox.git
git pull
```

---

*The content below is the last state in this repo and is kept for reference only.*

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

A third skill, `srl-customers`, covers customer overlays on a shared base design
(`pdf/customers/nspXXX-<name>/`): overriding base styles, fonts, customer PDF scripts,
packaging and deploying overlays and the base design to NSWOW.

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
  skills/srl-customers/
    SKILL.md                          customer overlays and NSWOW deployment
```
