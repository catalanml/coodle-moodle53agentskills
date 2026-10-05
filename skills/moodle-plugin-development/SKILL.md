---
name: moodle-plugin-development
description: Use when creating or modifying any Moodle 5.3 plugin - activity module, block, local plugin, theme, filter, course format, repository, enrolment method, question type, quiz access rule, or any other type - or when upgrading a plugin from Moodle 4.5 to 5.x. Covers plugin structure, version.php, db/ files, lang strings, and per-type requirements.
---

# Moodle 5.3 Plugin Development

Distilled from https://moodledev.io/docs/5.3 (official Moodle developer docs)
and the core 5.0–5.3 upgrade notes.

## Workflow

1. **Always read `references/common-files.md` first** — frankenstyle naming and
   the files every plugin shares (version.php, lang/, db/access.php,
   db/install.xml, db/upgrade.php, db/tasks.php, db/events.php,
   db/services.php, db/hooks.php, settings.php, lib.php, classes/,
   composer.json).
2. Then read the reference for the plugin type you are building:

| Plugin type | Prefix | Reference |
|---|---|---|
| Activity module | `mod_` | `references/activity-modules.md` |
| Block | `block_` | `references/blocks.md` |
| Local plugin | `local_` | `references/local-plugins.md` |
| Theme | `theme_` | `references/themes.md` |
| Text filter | `filter_` | `references/filters.md` |
| Course format | `format_` | `references/course-formats.md` |
| Repository | `repository_` | `references/repositories.md` |
| Enrolment method | `enrol_` | `references/enrolment.md` |
| Question type / bank / quiz access rule | `qtype_` / `qbank_` / `quizaccess_` | `references/question-types.md` |
| Everything else (ai, tiny, antivirus, availability, customfield, logstore, …) | varies | `references/other-plugin-types.md` |
| Subplugins of activities (booktool, datafield, forum reports, tiny testing, format migration) | varies | `references/subplugins.md` |

Also available:

- `references/upgrading-to-5.3.md` — **load first when porting a 4.5 plugin**:
  breaking-change checklist and removed/renamed APIs across 5.0, 5.1, 5.2, 5.3.
- `references/workflow-guides.md` — git workflow, the `public/` directory
  restructure, plugin upgrade API, profiling, Bootstrap 5 migration, 5.3
  platform requirements. Load it for environment or contribution questions.

3. When using core APIs (database, forms, output, events, …), also load the
   matching reference in the sibling skill `moodle-core-apis`.
4. Follow `moodle-coding-style` for all code you write, and `moodle-testing`
   when adding PHPUnit/Behat tests.

## Non-negotiables for every plugin

- Component name is frankenstyle (`plugintype_pluginname`) and must match the
  directory, `version.php` `$plugin->component`, and lang file name.
- Since 5.1 core code lives under `public/`: a plugin installs to
  `public/<plugintype dir>/<name>` in a git checkout. Code paths stay
  relative to `$CFG->dirroot`.
- Every user-visible string goes through `get_string()` with an entry in
  `lang/en/plugintype_pluginname.php` — never hardcode UI text.
- Schema changes require BOTH `db/install.xml` (fresh installs) and a matching
  `db/upgrade.php` step with a savepoint (upgrades), plus a version bump.
- New classes go under `classes/` with the plugin's namespace so the
  autoloader finds them; no manual require for autoloadable classes.
- Capabilities are declared in `db/access.php` and checked with
  `has_capability()`/`require_capability()` before privileged actions.
