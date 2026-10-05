# Moodle 5.3 Dev Skills

Skills for developing Moodle 5.3 (LTS) plugins and working with Moodle core
APIs, distilled from the official documentation at https://moodledev.io/docs/5.3,
https://moodledev.io/general/development/policies/codingstyle, and the core
`UPGRADING.md` notes for 5.0–5.3.

## When to use which skill

Working on ANY Moodle code? Start here:

- **moodle-coding-style** — read BEFORE writing or editing any Moodle PHP,
  JavaScript, SQL, or CSS/SCSS. Naming, whitespace, PHPDoc, file boilerplate.
- **moodle-plugin-development** — creating or modifying a plugin of any type
  (activity module, block, local, theme, filter, course format, repository,
  enrolment, question type, quiz access rule, …). Covers version.php, db/
  files, lang strings, per-type requirements, and upgrading plugins from
  Moodle 4.5 to 5.3 (`references/upgrading-to-5.3.md`).
- **moodle-core-apis** — using a Moodle core API: database ($DB), forms,
  output/templates, events, tasks, caching, files, capabilities, web
  services, routing (REST API routes and scopes), privacy, hooks,
  JavaScript modules, React. One reference file per API.
- **moodle-testing** — writing or running PHPUnit (11) tests, Behat
  scenarios, or setting up CI (moodle-plugin-ci, phpcs/moodle-cs).

Each skill's SKILL.md is a compact router; load only the reference files you
need for the task at hand.

All content targets **Moodle 5.3 (LTS)**. Since 5.1 the web-accessible code
lives in a `public/` directory: plugin paths such as `mod/forum` are
`public/mod/forum` in a git checkout, and remain relative to `$CFG->dirroot`
in code. For 4.5 code, use the separate coodle-moodle45agentskills suite.
