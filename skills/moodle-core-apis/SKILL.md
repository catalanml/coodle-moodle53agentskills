---
name: moodle-core-apis
description: Use when working with any Moodle 5.3 core API - database ($DB/DML/XMLDB), forms (moodleform), output/renderers/Mustache templates, events, scheduled or adhoc tasks, caching (MUC), File API, capabilities/contexts, web services (external functions), REST API routes and scopes (routing), Privacy API, Hooks API, Moodle JavaScript (AMD modules, core/ajax), or React components.
---

# Moodle 5.3 Core APIs

Distilled from https://moodledev.io/docs/5.3 and the core 5.0–5.3 upgrade
notes. Router: load ONLY the reference file(s) matching the task.

| Task involves | Reference |
|---|---|
| `$DB`, SQL, recordsets, transactions, install.xml, XMLDB, upgrade steps | `references/database.md` |
| Forms, `moodleform`, form elements, validation, filepicker | `references/forms.md` |
| Pages, `$OUTPUT`/`$PAGE`, renderers, renderables, Mustache templates, html_writer, humandate | `references/output-templates.md` |
| Events (trigger/observe), scheduled tasks, adhoc tasks, cron | `references/events-tasks.md` |
| Caching, MUC, db/caches.php | `references/caching.md` |
| Stored files, file areas, pluginfile.php, draft areas | `references/files.md` |
| Contexts, capabilities, roles, require_login | `references/access-roles.md` |
| Web services, external functions, db/services.php, calling from JS | `references/webservices.md` |
| Routing subsystem (`/api/rest/v2` routes), `#[route]`, `#[scopeset]` scopes, shortlinks, REST style guide, testing external functions, WS file upload/download | `references/webservices-advanced.md` |
| Privacy API, GDPR provider classes | `references/privacy.md` |
| Hooks API, dependency injection, deprecation, clock | `references/hooks-di.md` |
| JavaScript: AMD/ESM modules, grunt, core/ajax, core/str, modals, JS deprecation | `references/javascript.md` |
| JavaScript: reactive library, comboboxsearch, legacy YUI | `references/javascript-advanced.md` |
| React components: build tools, import map, auto-init, theming, testing, profiler | `references/react.md` |
| Backup and restore (activity tasks, stepslib, restore mapping) | `references/backup-restore.md` |
| Editor, plagiarism, xAPI, favourites, time, communication, AI/SMS subsystems, badges, OpenTelemetry, question subsystem, availability/conditional activities, analytics, Check API | `references/more-subsystems.md` |
| File browsing, document converters, file storage internals, DML drivers, privacy CLI utils | `references/files-advanced.md` |
| Strings, messaging, navigation, preferences, report builder, custom fields, comments, grading, calendar, users, locks, admin settings, tags, groups | `references/other-apis.md` |

## Cross-cutting rules

- Never write raw SQL against another component's tables when a core API
  exists for the job; always use `{tablename}` placeholders, never prefixes.
- Server-side capability checks are mandatory in every external function,
  every API route, and before every privileged operation — client-side
  checks are cosmetic.
- All output escaping goes through the renderer/template layer; never echo
  user data directly.
- Anything user-visible is a lang string (`get_string`).
- Upgrading code written for 4.5? Check removed/renamed APIs in
  `moodle-plugin-development` → `references/upgrading-to-5.3.md`.

Pair with `moodle-coding-style` for style and `moodle-testing` for tests.
