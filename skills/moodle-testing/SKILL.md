---
name: moodle-testing
description: Use when writing or running tests for Moodle 5.3 code - PHPUnit 11 unit tests, Behat acceptance tests, data generators, upgrading tests broken by PHPUnit 11, or setting up CI with moodle-plugin-ci, phpcs/moodle-cs, and GitHub Actions.
---

# Moodle 5.3 Testing & CI

Distilled from the official Moodle developer docs (moodledev.io).

| Task | Reference |
|---|---|
| PHPUnit tests (PHPUnit 11 since 5.0), generators, dataproviders, event/message sinks, PHPUnit 11 upgrade fixes | `references/phpunit.md` |
| Behat features, scenarios, step definitions | `references/behat.md` |
| CI: moodle-plugin-ci, phpcs (moodle-cs), grunt, GitHub Actions, plugin validation, QA testing pointers | `references/ci-tools.md` |

## Ground rules

- Plugin unit tests live in `PLUGINDIR/tests/`, named `*_test.php`, class in
  the plugin namespace; Behat features live in `PLUGINDIR/tests/behat/`.
- Any test that touches the DB or globals extends `advanced_testcase` and
  calls `$this->resetAfterTest()`.
- Create fixtures with data generators (`$this->getDataGenerator()`), never
  by inserting records by hand.
- Run phpcs with the moodle-cs standard before considering work done.
