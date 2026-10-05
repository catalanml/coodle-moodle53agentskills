# Moodle CI and Code-Quality Tools (Moodle 5.3)

Reference for phpcs/moodle-cs, GitHub Actions CI (core and moodle-plugin-ci
for plugins), PHPUnit snapshots in CI, grunt/NodeJS tooling, MDK, composer,
Marketplace release automation, plugin prechecks and QA testing, per the
Moodle 5.3 developer docs.

## Moodle 5.3 test matrix

From the 5.3 release notes (5.3 is an LTS):

| Requirement | Moodle 5.3 |
|---|---|
| PHP | 8.3 minimum, 8.4 supported (64-bit only); `sodium` extension required; `max_input_vars` >= 5000 |
| PostgreSQL | 17 (raised in 5.3) |
| MySQL | 8.4 (raised in 5.0) |
| MariaDB | 11.4.0 (raised in 5.3) |
| SQL Server | 2019 (raised in 5.2) |
| Aurora MySQL | 8.0 (configure with MySQL version numbers) |
| Oracle | not supported since 5.0 |
| Upgrade from | 4.4 or later |
| Stable branch | `MOODLE_503_STABLE` |

PHP 8.5 support is still under development for 5.3 (MDL-84684, development
only). A plugin that also supports 4.5 needs PHP 8.1 jobs for that branch
(PHP 8.1 is the 4.4/4.5 minimum). Since 5.1 (MDL-83424) the Moodle code is
under `public/` in the checkout — use tooling versions that support the new
layout (the docs say most tooling has been updated; no versions given).

## PHP CodeSniffer with moodle-cs

Two phpcs rulesets implement the Moodle coding style:

- `moodle` — standard ruleset, meets all variants of the coding style
- `moodle-extra` — adds recommended best practices (recommended, especially
  for new code)

Install globally so one copy serves all code; updates are not automatic:

```console
composer global config minimum-stability dev
composer global require moodlehq/moodle-cs
composer global show -l      # check for updates
composer global update       # apply them
```

Since 4.1.0 / 4.0.1 / 3.11.7 a phpcs configuration ships in the Moodle
codebase, so the right ruleset is picked up automatically; `grunt ignorefiles`
generates an extra config ignoring third-party libraries. To layer
`moodle-extra` on core, create `.phpcs.xml` (exclude it via
`.git/info/exclude`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ruleset name="MoodleCore">
  <rule ref="./phpcs.xml"/>
  <rule ref="moodle-extra"/>
</ruleset>
```

For a plugin developed outside the Moodle directory, create `phpcs.xml.dist`
at the repository root (or `echo phpcs.xml.dist >> .git/info/exclude` to keep
it local):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ruleset name="MoodleCore">
  <rule ref="moodle"/>
</ruleset>
```

`phpcs --config-set default_standard moodle` works but is not recommended.

```console
phpcs path/to/file.php                 # single file, full report
phpcs -n index.php                     # -n: errors only, ignore warnings
phpcs --report=summary blocks/html     # recursive, summary per file
```

Report formats: `full` (default), `summary`, `xml`, `csv`, `checkstyle`. The
summary notes how many violations `phpcbf` can fix automatically. moodle-cs
also helps fix PHPUnit 11 issues such as non-static data providers.

IDE integration needs no extra configuration since 3.11.7/4.0.1/4.1.0. On
'ERROR: Referenced sniff "moodle" does not exist', point the IDE at the
composer-installed binary (e.g. `~/.config/composer/vendor/bin/phpcs`) and
select "Automatic" as the standard.

## GitHub Actions

### Moodle core

Since 3.5.16 core ships a GitHub Actions workflow running a matrix of
databases and PHP versions. Enable Actions on your public fork (Settings >
Actions, "Allow all" — it reuses actions from different sources). Every push
to any branch queues a build; results are in the Actions tab, and pass/fail
badges show on Tracker issues whose branch names match. Moodle HQ's Jenkins
(https://ci.moodle.org/) remains the canonical CI; GHA is early warning.

Since 3.11.8 / 4.0.2 a repository secret named `phpunit_options` is passed to
every PHPUnit run as CLI options (list them with `vendor/bin/phpunit --help`;
the page still links the 9.6 option list — 5.3 runs PHPUnit 11.4).

### Moodle plugins: moodle-plugin-ci

The devdocs delegate plugin CI to the Moodle Plugin CI project:
https://moodlehq.github.io/moodle-plugin-ci/ (GitHub Actions instructions and
`.github/workflows/*.yml` templates; Moodle App plugins set
`MOODLE_APP=true`). Its test matrix and `*.dist.yml` templates are updated by
the integration team at each major release. The devdocs give no
moodle-plugin-ci version, command list or flags — take them from that site.

The commands below come from the moodle-plugin-ci project (not mirrored in the
devdocs); verify them against the current template before use:

```bash
composer create-project -n --no-dev --prefer-dist moodlehq/moodle-plugin-ci ci <version-from-upstream-docs>
moodle-plugin-ci install --plugin ./plugin --db-host=127.0.0.1

moodle-plugin-ci phplint
moodle-plugin-ci phpmd
moodle-plugin-ci phpcs          # Moodle coding style (moodle-cs)
moodle-plugin-ci phpdoc
moodle-plugin-ci validate
moodle-plugin-ci savepoints
moodle-plugin-ci mustache
moodle-plugin-ci grunt
moodle-plugin-ci phpunit
moodle-plugin-ci behat
```

For a 5.3 plugin the job matrix should cover branch `MOODLE_503_STABLE`, PHP
8.3 and 8.4, and databases `pgsql` (PostgreSQL 17+) and `mariadb` (11.4+)
and/or `mysqli` (MySQL 8.4+). Add older branches (e.g. `MOODLE_405_STABLE`
with PHP 8.1–8.3) only if `version.php` declares support for them.

### Faster PHPUnit in CI: snapshots (since 5.3, MDL-88495)

Initialising the PHPUnit environment dominates CI time. Cache a snapshot of a
vanilla install instead (how the archive is cached is up to your CI provider):

```bash
# Once per Moodle version (vanilla code):
php public/admin/tool/phpunit/cli/init.php
php public/admin/tool/phpunit/cli/util.php --snapshot
# -> cache <phpunit_dataroot>/snapshots/, keyed by Moodle version

# Every later run:
php public/admin/tool/phpunit/cli/util.php --restore=<name>   # name from --list, no .zip
# install the plugin under test, then:
php public/admin/tool/phpunit/cli/util.php --upgrade
vendor/bin/phpunit --testsuite <component>_testsuite
```

Snapshots are keyed by DB type and Moodle version; after a new Moodle release
fall back to a fresh `init.php` and take a new snapshot. `--restore` needs an
empty DB (`--drop` first; snapshots survive `--drop`).

### CiBot (Tracker prechecks)

Core issues submitted for peer review or integration are checked
automatically by CiBot (unless labelled `ci`). Add the label `cime` to request
a check on demand (polled every 20 min; the label is removed after the run).
Fix style problems on lines your patch touches; don't reformat unrelated code.

## NodeJS and Grunt

Grunt lints, transpiles, minifies and packages JavaScript and CSS, and lints
Behat features. After changing any JS or CSS you must run grunt before the
change is served.

Install NodeJS via NVM (nvm-windows >= 1.1.9). Moodle ships a `.nvmrc`; from
the Moodle directory:

```bash
nvm install && nvm use
npm install                # build dependencies (grunt, eslint, ...)
npm install -g grunt-cli   # optional; or use npx grunt
```

```
grunt amd                          # ignorefiles + eslint:amd + rollup
grunt js                           # amd + yui
grunt css                          # scss + rawcss
grunt stylelint
grunt react                        # build all React components
grunt eslint --show-lint-warnings  # pedantic JS lint warnings
grunt componentlibrary
grunt                              # context-aware: amd in amd/, shifter in yui/src/, else css+js
grunt watch                        # rebuild on change (needs watchman)
```

`cd` into the plugin's `amd/` folder before `grunt amd` to build only that
plugin; `-v` for verbose output; on Windows pass `--root=admin/tool/...`.
Since 5.0.4 / 5.1.1 (MDL-81263) grunt respects local `.stylelintignore` and
`.eslintignore` files. Plugin SCSS compilation needs a local `package.json` +
`Gruntfile.js` in the plugin root (loading core's Gruntfile via
`grunt-load-gruntfile`, with `grunt-contrib-sass`/`watch`/`clean`).

## MDK (Moodle Development Kit) — brief

Python tool managing multiple Moodle instances:

```bash
mdk create -i -v 24 -e pgsql -r dev users     # create + install, dev config, test users
mdk fix 12345 ; mdk push -t                   # branch for MDL-12345, push + update tracker
mdk pull 12345 -t                             # pull a tracker patch into a testing branch
mdk phpunit -r                                # init and run PHPUnit
mdk backport --versions 37 38 --push --update-tracker
```

## Composer — brief

Core's `composer.json`/`composer.lock` pin the PHPUnit (11.4) and Behat
toolchain; run `composer install` in the Moodle root. When changing
`composer.json`, regenerate the lock on the lowest PHP the branch supports:
`rm composer.lock && rm -fr vendor && php composer.phar clearcache && php composer.phar update`,
then verify phpunit/behat still run.

Since 5.3 (MDL-88576) code can inspect Composer dependencies at runtime:
`$composer = \core\di::get(\core\composer::class);` then `is_installed()`, or
`get_status()` returning a `\core\composer\status` (`installed`, `current`,
`current_packages()`, `missing_packages()`).

## Releasing to Moodle Marketplace from CI

The Plugins directory has moved to **Moodle Marketplace**; the old Plugins
directory checklist is legacy guidance (see the Marketplace Plugin Submission
Guidelines for current requirements). New versions can be submitted from CI
with the Marketplace API (for GitHub Actions use
[moodlehq/moodle-plugin-release](https://github.com/moodlehq/moodle-plugin-release)):

```bash
curl -s -X POST \
     -H "Authorization: Bearer $MKPL_TOKEN" \
     -F "file=@mod_quiz.zip" \
     -F "releaseNotes=Fixes the grade calculation for group submissions" \
     -F "vcsTag=v1.4.0" \
     "https://marketplace.moodle.com/api/plugins/mod_quiz/versions"
```

- Token from Marketplace Account > Security > API tokens (shown once, max 10
  active); store it as a CI secret, never in the repo.
- Send **either** `file` or `fileUrl` (https, public) — both/neither is `422`.
  Optional: `releaseNotes`, `vcsRepositoryUrl`, `vcsBranch`, `vcsTag`. Build
  number, release, maturity and supported versions come from `version.php`.
- Success is `201` with `status: in_testing` (queued for prechecks; submitting
  is not publishing). Fail the job on anything else; print `violations`.
  Don't retry `422`; retry only `5xx`/connection errors. `403` = not a
  maintainer, `404` = unknown frankenstyle, `409` = plugin not accepting
  versions, `422` also for duplicate build numbers or invalid ZIPs.

## Plugin code prechecks and checklist

Automated prechecks run on every uploaded version: **phplint**, **phpcs**
(Moodle coding style), **js**, **css** (CSS coding style), **phpdoc**,
**savepoint**, **thirdparty** (`thirdpartylibs.xml`), **grunt** (build output
up to date), **mustache**. Aim for zero errors/warnings.

Checklist essentials (legacy list, still what reviewers look at):

- Repository named `moodle-{plugintype}_{pluginname}`; public source and issue
  tracker URLs, documentation URL, description, screenshots.
- Supports at least one currently maintained Moodle version.
- GPL v3+ boilerplate and `@copyright` in every file; third-party libraries
  GPL-compatible and declared in `thirdpartylibs.xml`.
- Installs from the ZIP via the UI; no composer step for admins; dependencies
  declared in `version.php`.
- No PHP warnings/notices with developer debugging; works on MySQL and
  PostgreSQL (use the DML API).
- Frankenstyle prefixes on tables, settings, functions, classes, constants;
  nothing unprefixed in the global scope.
- Settings in `config_plugins` (`plugintype_pluginname/settingname` in
  `settings.php`, read with `get_config()`).
- All UI text via `get_string()`; ship only `lang/en/`; string files are pure
  `$string['id'] = 'value';` data.
- CSS selectors namespaced (e.g. `.path-mod-mymodule .contentarea`).
- Security: `required_param()`/`optional_param()`, SQL placeholders, `sesskey`,
  `require_login()`, capability checks before display and action; no
  `eval()`/`unserialize()`/`call_user_func()` on user data.
- Privacy API (required for external integrations); backup/restore for
  activity modules.

Approval blockers: no public issue tracker, SQL failing on PostgreSQL,
frankenstyle collisions, security violations, missing privacy API (external
integrations), missing backup/restore (activities), site-policy or trademark
violations, or replicating Moodle commercial products (Commercial Listing
Policy).

## QA testing (pointer)

Separate from automated tests, each major release has a community **QA
testing cycle**: people run written manual test cases (`MDLQA-*` issues on the
Tracker) on a shared site. Docs: `docs/qatesting.md` plus
`qatesting/gettingstarted.md`, `qatesting/faq.md`, `qatesting/commonterms.md`;
developer-side process at `/general/development/process/testing/qa`.

- 5.3 cycle: 31 August – 2 October 2026; dashboard
  https://moodle.atlassian.net/jira/dashboards/10612 (beginner tests in its
  first widget).
- Accounts: Moodle Tracker + moodle.org with the same email (credits/badge).
- Run tests on https://qa.moodledemo.net/ (resets hourly; no personal data).
  Mark `Start testing`, then Pass/Fail with environment details and
  screenshots.
- Labels: `qa_help_needed` (unrelated failure/question),
  `qa_instructions_update` (outdated steps).
- To use your own site it must run the current dev version (e.g. 5.3dev).
