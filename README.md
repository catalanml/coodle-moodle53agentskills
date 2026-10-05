<h1 align="center">coodle-moodle53agentskills</h1>

<p align="center">
  <em>Your coding agent, fluent in Moodle 5.3.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/moodle-5.3%20LTS-orange?style=flat-square" alt="Moodle 5.3 LTS">
  <img src="https://img.shields.io/badge/skills-4-blue?style=flat-square" alt="4 skills">
  <img src="https://img.shields.io/badge/references-36-blue?style=flat-square" alt="36 reference files">
  <img src="https://img.shields.io/badge/license-GPL--3.0-green?style=flat-square" alt="GPL-3.0 license">
</p>

<p align="center"><a href="README.md">English</a> · <a href="README.es.md">Español</a></p>

---

Ask a stock coding agent for a Moodle plugin and you get a generic PHP package: hardcoded strings, raw SQL against prefixed table names, no capability checks, no `db/upgrade.php`, a style that fails phpcs on the first line. Ask it for a *Moodle 5.3* plugin and it gets worse: its training data predates the `public/` directory restructure, Bootstrap 5, PHPUnit 11, mandatory route scopes, and a long list of renamed and removed APIs.

This plugin fixes that. It is a suite of **agent skills** distilled offline from the official Moodle 5.3 developer documentation — [moodledev.io/docs/5.3](https://moodledev.io/docs/5.3) and the [coding style policy](https://moodledev.io/general/development/policies/codingstyle) — plus the core `UPGRADING.md` notes for 5.0, 5.1, 5.2, and 5.3. It is packaged so Claude Code, Codex, and any AGENTS.md-aware agent can use it. Your agent loads a compact router per topic and only the reference files the task actually needs: no network fetches, no Docusaurus boilerplate, no guessing from stale training data.

This is the 5.3 edition of [coodle-moodle45agentskills](https://github.com/catalanml/coodle-moodle45agentskills). It is a separate repository and plugin, so both editions can be installed side by side.

## Table of Contents

- [How it works](#how-it-works)
- [What's inside](#whats-inside)
- [What changed from the 4.5 edition](#what-changed-from-the-45-edition)
- [Installation](#installation)
- [Example session](#example-session)
- [Design principles](#design-principles)
- [Accuracy and verification](#accuracy-and-verification)
- [Known gaps](#known-gaps)
- [Updating the content](#updating-the-content)
- [Contributing](#contributing)
- [License](#license)

## How it works

Each skill is a folder with a `SKILL.md` router and a `references/` directory. The router's frontmatter description tells the agent *when* to load it; the router body tells it *which* reference file covers the task at hand. A question about `$DB->get_records()` pulls in one ~370-line database reference, not the whole suite.

```
You: "Add a REST API route to my local plugin that lists a user's badges."

Agent loads: moodle-plugin-development  → references/common-files.md          (plugin skeleton)
             moodle-core-apis           → references/webservices-advanced.md  (#[route], #[scopeset])
                                        → references/access-roles.md          (capability checks)
             moodle-coding-style        → SKILL.md                            (while writing)
```

The skills trigger automatically; you never invoke them by hand.

## What's inside

### `moodle-plugin-development`

Plugin anatomy, per-type requirements, and the 4.5 → 5.3 migration guide.

| Reference | Covers |
|---|---|
| `common-files.md` | Frankenstyle naming, version.php, lang/, every db/ file, classes/ autoloading, settings.php, lib.php, composer.json support, the 5.1 `public/` layout |
| `upgrading-to-5.3.md` | **New.** Must-do checklist and Old → New tables of removed/renamed APIs across 5.0, 5.1, 5.2, 5.3 |
| `activity-modules.md` | mod plugins: lib.php callbacks, FEATURE_* flags (incl. `FEATURE_MOD_OTHERPURPOSE`), activity overview page integration, `modulename_summary`/`_tip` strings |
| `blocks.md` / `local-plugins.md` | block_base API, instance config; local plugin structure and when to use one |
| `themes.md` | config.php, Boost inheritance, SCSS, Bootstrap 5, activity icon colour variables, template relocations, Classic removal |
| `filters.md` / `course-formats.md` | `\core_filters\text_filter`; `core_courseformat\base`, output classes, 5.3 linear navigation, activity chooser changes |
| `repositories.md` / `enrolment.md` | repository class contract; enrol_plugin workflow and subsystem API |
| `question-types.md` | qtype and qbank plugins, qtype restore, **quiz access rule (`quizaccess_`) plugins**, module-level question banks |
| `other-plugin-types.md` | 5.3 plugin-type catalogue, AI providers/placements, tiny, antivirus, availability, customfield, sms, plugin-type deprecation |
| `subplugins.md` | db/subplugins.json with `subplugintypes`, database field types, book tools, TinyMCE Behat testing, course-format migration |
| `workflow-guides.md` | Git workflow, `public/` restructure, plugin upgrade API, profiling, Bootstrap 5 migration, 5.3 requirements |

### `moodle-core-apis`

One reference per core API — the router maps task → file.

| Reference | Covers |
|---|---|
| `database.md` | $DB/DML methods, placeholders, transactions, sql_* helpers, XMLDB/DDL (Oracle removed) |
| `forms.md` | moodleform lifecycle, element catalogue, validation, file elements |
| `output-templates.md` | $PAGE/$OUTPUT, renderers, renderables, Mustache, humandate/humantimeperiod |
| `events-tasks.md` | Event classes and observers; scheduled and adhoc tasks |
| `caching.md` / `files.md` | MUC definitions and versioned caches; File API, pluginfile serving, draft areas |
| `access-roles.md` | Contexts, capabilities, has_capability/require_login |
| `webservices.md` | `\core_external\external_api` functions, db/services.php, core/ajax |
| `webservices-advanced.md` | Routing subsystem, **mandatory `#[scopeset]` / `#[unscoped_resource]`**, scope classes, shortlinks, REST style guide, WS testing and files |
| `react.md` | **New.** React components: build tools, import map, auto-init, theming, testing, profiler |
| `privacy.md` | GDPR provider classes, metadata, export/delete |
| `hooks-di.md` | Hooks API, dependency injection, \core\clock, deprecation attributes |
| `javascript.md` | AMD/ESM modules, grunt, core/str, `Modal.create()`, JS deprecation process |
| `javascript-advanced.md` | Reactive library, comboboxsearch, legacy YUI recognition |
| `backup-restore.md` | Backup/restore tasks and stepslib for activities, blocks, and admin tools |
| `more-subsystems.md` | Editor, plagiarism, xAPI, favourites, time, communication, AI, SMS, badges, OpenTelemetry, availability, analytics, Check API |
| `files-advanced.md` | File browsing, document converters, storage internals, DML drivers, privacy CLI utils |
| `other-apis.md` | Strings, messaging, navigation, report builder, custom fields, comments, admin settings, calendar, tags, groups, locks |

### `moodle-coding-style`

The official coding style for PHP, JavaScript, and SQL in `SKILL.md`, plus **new** `references/css.md` for the CSS/SCSS style policy.

### `moodle-testing`

| Reference | Covers |
|---|---|
| `phpunit.md` | PHPUnit 11 (since 5.0), attributes vs annotations, generators, sinks, upgrading broken tests |
| `behat.md` | .feature structure, tags, custom steps, entity generators |
| `ci-tools.md` | phpcs + moodle-cs, moodle-plugin-ci on GitHub Actions with the 5.3 PHP/DB matrix, QA testing pointers |

## What changed from the 4.5 edition

- **3 new references:** `upgrading-to-5.3.md`, `react.md`, `moodle-coding-style/references/css.md`.
- **32 new upstream pages** folded in: quiz access rules, activity overview, linear navigation, route scopes and shortlinks, the REST style guide, React (7 pages), Composer support, the code restructure, frontend and JS deprecation guides, humandate, comments, badges, OpenTelemetry, qtype restore, the PHPUnit writing guide, and QA testing.
- **Every existing reference re-distilled** against the 5.3 tree and cross-checked with the 5.0–5.3 `UPGRADING.md` notes. Many devdocs pages are unchanged since 4.5 even where the API has changed, so the notes are often the only source of a change. Each reference opens with a "changes since 4.5" summary where relevant.

## Installation

The repository uses one shared `skills/` directory and thin manifests for its supported agent integrations.

### Claude Code

```bash
/plugin marketplace add catalanml/coodle-moodle53agentskills
/plugin install coodle-moodle53agentskills@coodle53
```

The marketplace is named `coodle53` so it does not collide with the 4.5 edition's `coodle` marketplace. Or, without publishing, copy the skills into a project's skills directory (use a project-level directory if you also keep the 4.5 skills, because the skill names are the same):

```bash
cp -r skills/* /path/to/your/project/.claude/skills/
```

### Codex

`.codex-plugin/plugin.json` declares the shared `skills/` directory. Install the repository as a plugin from the Codex plugin interface (`/plugins`), or point Codex at a local clone. `AGENTS.md` gives Codex the routing instructions even without plugin support.

### Other agents (AGENTS.md)

Any agent that reads `AGENTS.md` (OpenCode, Cursor, Copilot CLI, …) can use the suite: clone this repo somewhere stable and add a line to your project's agent instructions:

```markdown
Before writing Moodle 5.3 code, read /path/to/coodle-moodle53agentskills/AGENTS.md
and follow the skill routing it describes.
```

Skills are plain markdown — there are no hooks, no scripts, no runtime dependencies.

## Example session

> **You:** My 4.5 course format plugin breaks on 5.3. Fix it and add linear navigation.
>
> **Agent:** *loads `moodle-plugin-development` (upgrading-to-5.3, course-formats) — replaces array-based section menus with `core_courseformat\output\local\content\section\controlmenu` and `action_menu_link` objects, switches `addModule` links to `newModule` with `data-sectionid`, drops its reliance on the deprecated `maxsections` setting, overrides `uses_linear_navigation()`, and bumps `$plugin->requires` to the 5.3.0 core version.*

## Design principles

- **Distilled, not vendored.** Each reference file is a compact, self-contained rewrite (100–470 lines) with the load-bearing facts: class names, signatures, file paths, real code examples.
- **Router + references.** `SKILL.md` files stay small and point to exactly one file per topic.
- **Docs-only facts.** Content comes from the official documentation and core upgrade notes, not model memory. Where both are silent, the files say so rather than inventing an API.
- **5.3-accurate over doc-faithful.** Where devdocs pages are stale (for example ModalFactory, removed in 5.2; Oracle, removed in 5.0; `set_section_number()`; Atto), the skills teach the current API and note the discrepancy inline.

## Accuracy and verification

Sources:

- [moodle/devdocs](https://github.com/moodle/devdocs) `docs/` (the 5.3 tree, labelled `main (5.3)`) at commit `0367ab8911f86646e1fefdf07a59237c43057be7`, 5 October 2026: 220 Markdown/MDX files (the 188 files of the 4.5 tree plus 32 new). 75 files changed or were added since the 4.5 tree (including images and code snippets).
- The unversioned coding-style, PHP, PHPUnit, and Behat pages under `general/development/` at the same commit.
- `UPGRADING.md` from `moodle/moodle` branch `MOODLE_503_STABLE` (5.3 released 5 October 2026), sections 5.0 to 5.3.

Every content page in the 5.3 tree (excluding the Moodle App docs) is routed to a reference. Load-bearing claims were checked against these sources. Where devdocs and UPGRADING disagree on a class name, namespace, signature or path, the conflict was resolved against the `MOODLE_503_STABLE` source code (for example `\core_course\cm_info`, not UPGRADING's `\course\cm_info`; `\core\router\route`; static `uses_linear_navigation()`; `public/admin/tool/behat/cli/`). Retrieval was tested with fresh agents on 5.3-specific tasks for each skill.

## Known gaps

- Many 5.3 devdocs pages are unchanged since 4.5. Where UPGRADING names a removal but no replacement, the files say "none listed" instead of guessing.
- The Gradebook API (`grade_update()`), the full Events API, and module-level question banks (`mod_qbank`) have little or no devdocs coverage. They are sourced from UPGRADING notes or flagged.
- A few APIs appear only by name in UPGRADING (for example the soft-retry task methods and some hook namespaces). The files flag them as unverified.
- The Moodle App (mobile) docs are intentionally out of scope.

## Updating the content

To refresh: re-clone devdocs. Once 5.3 is versioned, its pages move from `docs/` to `versioned_docs/version-5.3/`. Diff against commit `0367ab8`, re-read the changed pages and any new `UPGRADING.md` entries on `MOODLE_503_STABLE`, and re-verify. 5.3 is an LTS, so drift after release should be slow.

## Contributing

1. Fork the repository and branch from `main`.
2. Keep the rules: plain GitHub markdown, no Docusaurus syntax, facts sourced from the official docs or upgrade notes (cite the source page in your PR), routers stay short.
3. One topic per reference file; put breadth in the `other-*` files.

## License

GPL-3.0-or-later. Content derived from the Moodle developer documentation and Moodle core, which are part of the Moodle project (GPL).
