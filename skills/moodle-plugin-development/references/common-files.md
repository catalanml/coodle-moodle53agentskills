# Moodle 5.3 Plugin Common Files

Reference for frankenstyle component naming and the files common to every Moodle 5.3 plugin, distilled from the official Moodle 5.3 developer docs and the 5.0–5.3 UPGRADING notes.

## Frankenstyle component names

A component name uniquely identifies a plugin: `{plugintype}_{pluginname}`, e.g. `mod_quiz`, `local_coolreport`, `block_participants`.

- Prefix = plugin type (e.g. `mod` for activity modules); name = the plugin's folder name, always lowercase.
- Core subsystems (not plugins) are referred to as `core_{subsystem}`, e.g. `core_course`, `core_user`, `core_grades`, `core_message`, `core_cache`, `core_form`. Bare `core` is Moodle core itself.

Plugin machine-name rules (validated by `core_component::is_valid_plugin_name()`):

- Starts with a lowercase latin letter; contains only lowercase letters, numbers, underscores; ends with a letter or number; no hyphens.
- Regex: `/^[a-z](?:[a-z0-9_](?!__))*[a-z0-9]+$/`
- Activity modules (`mod`) may NOT contain underscores in the plugin name (legacy).
- Since 5.2: `is_valid_plugin_name()` also rejects `qtype_random` (removed from core; the name stays reserved so old backups restore).

Frankenstyle is used in:

| Where | Convention | Example |
| --- | --- | --- |
| Global function names | Full frankenstyle prefix (prefer autoloaded classes instead) | `local_coolreport_cron()` |
| Classes | Namespace = component name; classes autoloaded from `classes/` | `\mod_forum\discussion` |
| Constants | Uppercase frankenstyle prefix (class constants preferred) | `MOD_FORUM_XXXX` |
| DB table names | Frankenstyle prefix after the Moodle prefix (exception: activities omit `mod_`, e.g. table `forum`) | `local_coolreport_users` |
| Capabilities | `/` instead of `_` between type and name | `mod/quiz:viewattempt` |
| Lang files | `lang/en/{component}.php` (activities use plugin name only: `mod/quiz/lang/en/quiz.php`) | `blocks/participants/lang/en/block_participants.php` |
| Settings | `plugintype_pluginname/settingname`, stored in `config_plugins` | `tool_example/apikey` |
| Composer package (5.2+) | `vendor/moodle-{plugintype}_{pluginname}` | `myplugin/moodle-block_myblock` |
| Also | PHP namespaces, JS module names, template names, `@package` in phpdoc, web service function names, `config_plugins.plugin` column | |

## Where plugin code lives (5.1+ `public/` directory)

Since 5.1 (MDL-83424): almost all Moodle code, including every plugin, moved into a `public/` directory inside the Moodle checkout, and the web server `DocumentRoot` (Apache) / `root` (nginx) must point at `public/`.

- `$CFG->dirroot` still points at the code directory (the `public/` dir) and `$CFG->wwwroot` is unchanged, so paths you write in code stay relative to `$CFG->dirroot` exactly as before: `$CFG->dirroot . '/local/myplugin/lib.php'`, `new moodle_url('/local/myplugin/view.php')`.
- New read-only `$CFG->root` points at the root of the Moodle installation (the directory that contains `public/`).
- Repository-level paths gain the prefix: a plugin installed as `local/myplugin` sits at `public/local/myplugin` in a git checkout (e.g. core's `public/mod/assign/submission/onlinetext`). Use the `public/` form only in repository paths, CI scripts, README install instructions and CLI invocations (e.g. `php public/admin/tool/phpunit/cli/util.php`).
- In this suite, paths written like `/mod/forum` or `mod/forum/...` are relative to `$CFG->dirroot`.
- Since 5.2: the `$CFG->wwwrootendsinpublic` flag was removed; servers must be configured to serve `public/`.

## Plugin directory anatomy

Typical layout (based on `mod_folder` in the docs; presence varies by plugin type):

```
myplugin/
├── version.php            (required) version/dependency metadata
├── lang/en/<component>.php (required) English language strings
├── composer.json          (optional, 5.2+) Composer/Packagist metadata
├── db/
│   ├── install.xml        DB schema created at install (XMLDB editor)
│   ├── install.php        post-install hook (xmldb_..._install)
│   ├── upgrade.php        upgrade steps with savepoints
│   ├── uninstall.php      pre-uninstall hook (xmldb_..._uninstall)
│   ├── access.php         capability definitions
│   ├── events.php         event observer subscriptions
│   ├── tasks.php          scheduled task default schedules
│   ├── services.php       external (web service) function declarations
│   ├── caches.php         MUC cache definitions
│   ├── hooks.php          hook callback registrations
│   ├── messages.php       message providers
│   ├── tag.php            tag area definitions
│   ├── subplugins.json    subplugin types (only plugins that host subplugins)
│   ├── renamedclasses.php / legacyclasses.php  class-rename maps
├── classes/               autoloaded, namespaced PHP classes
│   ├── task/              scheduled/adhoc task classes
│   ├── external/          external function classes
│   ├── event/             event classes
│   └── privacy/provider.php
├── cli/                   CLI scripts
├── amd/src/               JavaScript (ESM source, built to amd/build)
├── templates/             Mustache templates
├── pix/icon.svg           icons (svg preferred, png fallback; activity modules ship pix/monologo.svg)
├── backup/                backup & restore support
├── lib.php                legacy callbacks bridge to core
├── locallib.php           (legacy, discouraged) plugin-internal functions
├── settings.php           admin settings
├── styles.css             plugin CSS
├── thirdpartylibs.xml     bundled third-party library declarations
├── readme_moodle.txt, upgrade.txt, environment.xml, README, CHANGES
```

Rule for the whole `db/` folder: files inside `db/` (install.xml, upgrade.php, access.php, tasks.php, ...) must **never** contain `require_once` or any file inclusion — they are parsed in performance-sensitive contexts and must be self-contained definition files with no side effects. The same rule applies to `version.php` and to lang files.

## version.php (required)

Located in the plugin root. It is a plain data file: never `require_once` anything from it. Properties:

- `$plugin->component` (required, string) — full frankenstyle name; validates deploy location.
- `$plugin->version` (required, int) — `YYYYMMDDXX`; must always increase. On stable branches freeze `YYYYMMDD` and bump `XX`; newer branches must have higher `YYYYMMDD00` than older ones.
- `$plugin->requires` (recommended) — minimum Moodle core version: copy the `$version` value from Moodle core's own `version.php` for the oldest release you support. The plugin cannot be installed on an earlier core. Moodle 5.3.0 is `2026100500` (from `public/version.php` on `MOODLE_503_STABLE`; the devdocs do not list core version numbers).
- `$plugin->supported` (int[], since 3.9) — inclusive branch range. Branch numbers follow the pattern `311`, `400`, `405`; Moodle 5.3 is `503`.
- `$plugin->incompatible` (optional, int) — earliest incompatible branch; blocks installation from that branch on.
- `$plugin->maturity` — `MATURITY_ALPHA` | `MATURITY_BETA` | `MATURITY_RC` | `MATURITY_STABLE`; affects update notifications.
- `$plugin->release` (string) — human-readable release name, e.g. `'1.0'`.
- `$plugin->dependencies` (array) — `['frankenstyle' => version]` of other required plugins; unmet dependencies block install/upgrade. Keep declaring them here even if you also ship a `composer.json`.

```php
<?php
// This file is part of Moodle - http://moodle.org/
//
// Moodle is free software: you can redistribute it and/or modify
// it under the terms of the GNU General Public License as published by
// the Free Software Foundation, either version 3 of the License, or
// (at your option) any later version.
//
// Moodle is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU General Public License for more details.
//
// You should have received a copy of the GNU General Public License
// along with Moodle.  If not, see <http://www.gnu.org/licenses/>.

/**
 * Version details.
 *
 * @package   local_myplugin
 * @copyright 2026 Your Name <you@example.com>
 * @license   http://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */

defined('MOODLE_INTERNAL') || die();

$plugin->component = 'local_myplugin';
$plugin->version = 2026100500;          // YYYYMMDDXX.
$plugin->requires = 2026100500;         // Moodle 5.3.0.
$plugin->supported = [503, 503];        // Moodle 5.3 branch only.
$plugin->maturity = MATURITY_STABLE;
$plugin->release = '1.0';
$plugin->dependencies = [
    'mod_forum' => 2022041900,          // Minimum mod_forum version (YYYYMMDDXX of that plugin).
];
```

5.3 platform requirements that affect what plugin code may use: PHP 8.3 minimum (8.4 supported), upgrade path from 4.4+, Oracle unsupported since 5.0.

## composer.json (optional, since 5.2)

Since 5.2 (MDL-87473): plugins can be distributed and installed with Composer. The `moodle/composer-installer` Composer plugin places each package in the right Moodle directory based on its `type`. Key fields:

- `name` — `vendor/moodle-{plugintype}_{pluginname}`; the vendor part is your Packagist vendor name, the package part follows Moodle convention.
- `type` — `moodle-{plugintype}`, e.g. `moodle-block` installs into `blocks/myblock/`.
- `require.moodle/moodle` — supported Moodle version range.
- `require.moodle/composer-installer` — a **production** dependency (`require`, not `require-dev`).
- Other Moodle plugins may be required as Composer packages.

```json
{
    "name": "myplugin/moodle-block_myblock",
    "description": "A description of my Moodle block plugin",
    "type": "moodle-block",
    "require": {
        "moodle/moodle": "^5.2",
        "moodle/composer-installer": "*",
        "abgreeve/moodle-block_stash": "^5.2"
    },
    "license": "GPL-3.0-or-later"
}
```

Gotchas:

- Composer runtime dependencies are only guaranteed when the plugin itself is installed via Composer. Avoid Composer-only runtime dependencies that would break a plain zip/git install, and keep `$plugin->dependencies` in `version.php`.
- Publish by submitting the public repository to Packagist (plus a webhook to keep it in sync); admins then run `composer require myplugin/moodle-block_myblock`.
- Development site: `composer create-project moodle/seed [yourlocation]` creates a site with `composer.json`, `vendor/` and a `moodle/` directory; `composer require "moodle/moodle:~5.2.0"` pins core.
- Local development: add a `path` repository to the **site's** `composer.json` with `"options": {"symlink": false}` and require the package as `"*"` or `"@dev"`. Never symlink plugins — entry points use relative includes such as `require_once('../../config.php')`, which resolve outside the site through a symlink. Re-run `composer update vendor/package` after local changes to refresh the mirrored copy.

## lang/en/{component}.php (required)

Location: `{plugindir}/lang/en/{frankenstyle}.php`, e.g. `auth/ldap/lang/en/auth_ldap.php`. Exception: activity modules use the plugin name only, e.g. `mod/forum/lang/en/forum.php`.

- Every plugin **must** define `$string['pluginname']`.
- Retrieve with `get_string('identifier', 'plugintype_pluginname')`.
- Lang files must never contain `require_once` or any inclusion — simple key/value definitions only.
- Capability names need strings named `pluginname:capabilityname`, e.g. `$string['folder:managefiles'] = 'Manage files in folder module';`
- Cache definitions need `$string['cachedef_<definition>']`.
- Plugins that declare subplugin types also need plugin-type name strings (see subplugins.md, 5.1 change MDL-84948).

## db/access.php — capabilities

Capability names are `plugintype/pluginname:capabilityname`. Bump the plugin version after any change. Changes to *existing* capability definitions only apply to new installs.

```php
$capabilities = [
    'mod/folder:managefiles' => [
        'riskbitmask' => RISK_SPAM,          // Associated risks.
        'captype' => 'write',                // 'read' or 'write' (write blocked for guests).
        'contextlevel' => CONTEXT_MODULE,    // Typical context level checked.
        'archetypes' => [                    // Defaults per role archetype (prefer CAP_ALLOW only).
            'editingteacher' => CAP_ALLOW,
        ],
        // Optional: 'clonepermissionsfrom' => 'moodle/quiz:attempt',
    ],
];
```

Deprecate a capability by removing it from `$capabilities` and adding it to `$deprecatedcapabilities` with optional `replacement` and `message` keys (core did this in 5.0 for `mod/data:comment` and `mod/data:managecomments`).

## db/install.xml — initial schema

Defines tables, fields, keys, and indexes created on first install. **Must** be created and maintained with the built-in XMLDB editor, and must always match the schema produced by running all of upgrade.php. Table names start with the frankenstyle name (activities omit the `mod_` prefix). Example table definition:

```xml
<TABLE NAME="local_myplugin_things" COMMENT="Stores things">
    <FIELDS>
        <FIELD NAME="id" TYPE="int" LENGTH="10" NOTNULL="true" SEQUENCE="true"/>
        <FIELD NAME="prompt" TYPE="text" NOTNULL="false" SEQUENCE="false"/>
        <FIELD NAME="quality" TYPE="char" LENGTH="21" NOTNULL="true" SEQUENCE="false"/>
    </FIELDS>
    <KEYS>
        <KEY NAME="primary" TYPE="primary" FIELDS="id"/>
    </KEYS>
</TABLE>
```

## db/upgrade.php — upgrade steps

Runs when the version in `version.php` exceeds the stored version in `config_plugins`. Only upgrades are supported — never downgrades. Schema-change steps must be generated with the XMLDB editor. Bump `version.php` after any change in `db/`, JavaScript, any new autoloaded class, any new setting, and any lang string change — a new version triggers the upgrade and resets all caches.

```php
<?php

function xmldb_local_myplugin_upgrade($oldversion): bool {
    global $CFG, $DB;

    $dbman = $DB->get_manager(); // Loads ddl manager and xmldb classes.

    if ($oldversion < 2026100501) {
        // Perform the upgrade steps for this version...

        // Savepoint: marks the step complete and resets timeouts.
        upgrade_plugin_savepoint(true, 2026100501, 'local', 'myplugin');
    }

    return true;
}
```

Restrictions: upgrade code may use the DML database API and core functions, but must **not** call the plugin's own functions (plugin data is still in the old state). For work that cannot run during upgrade (cache rebuilds, long migrations), queue an adhoc task:

```php
if ($oldversion < 2026100502) {
    $task = new \local_myplugin\task\migrate_data();
    \core\task\manager::queue_adhoc_task($task, true);
    upgrade_plugin_savepoint(true, 2026100502, 'local', 'myplugin');
}
```

Since 5.3 (MDL-86422): `queue_adhoc_task()` always returns the id of the new (or, with `$checkforexisting`, the existing) task, or `false` if the task's component is deprecated or the insert failed.

Related: `db/install.php` defines `xmldb_{frankenstyle}_install()` (runs after first install; not a substitute for install.xml) and `db/uninstall.php` defines `xmldb_{frankenstyle}_uninstall()`.

## db/events.php — event observers

Defines the event subscriptions your plugin listens for (Events API "event observers"). Do not subscribe to another plugin's events without declaring a dependency on that plugin in `version.php`.

## db/tasks.php — scheduled tasks

Two items are required: a class extending `\core\task\scheduled_task` in `classes/task/`, and the initial schedule in `db/tasks.php`. Adhoc tasks are not listed here. Edits to an existing task's schedule only apply if the admin has not customised it.

```php
$tasks = [
    [
        'classname' => 'mod_example\task\do_something',  // Required, fully-qualified.
        'blocking' => 0,          // Strongly discouraged; never accepted in core.
        'minute' => '30',         // Fields: month, day, dayofweek, hour, minute.
        'hour' => '17',           // Values: '*', int, min-max, lists 'a,b', steps '/n'.
        'day' => '*',             // 'R' = fixed random value chosen at install/update
        'month' => '1,7',         // (accepted for minute, hour, day, dayofweek).
        'dayofweek' => '0',
        // 'disabled' => 1,       // Ship disabled until an admin enables it.
    ],
];
```

Defaults when a field is omitted: `*` (every month/day/hour/minute/day-of-week). If either `day` or `dayofweek` is `*` the other one is used; otherwise the soonest match wins. Task class skeleton:

```php
namespace mod_example\task;

class do_something extends \core\task\scheduled_task {
    public function get_name() {
        return get_string('dosomething', 'mod_example');
    }

    public function execute() {
        // Call your own API.
    }
}
```

## db/renamedclasses.php and db/legacyclasses.php

Use `db/renamedclasses.php` to preserve autoloading and backwards compatibility
when a public class is renamed or moved. Add only classes that form the
component's public API, especially classes that third-party plugins may use;
private/internal classes do not need a map.

Use `db/legacyclasses.php` for public classes moved under `classes/` whose old
names cannot yet be changed to the namespaced convention. Do not add private
classes to either map. Keep both files as self-contained definition files and
bump the plugin version after changing them.

## db/services.php — external functions

Declares external functions for web services and AJAX. Function names follow
`{frankenstyle}_{verb}_{noun}`, e.g. `local_groupmanager_create_groups`.

```php
$functions = [
    'local_groupmanager_create_groups' => [
        // Namespaced external function class (in classes/external/).
        'classname'   => 'local_groupmanager\external\create_groups',
        'description' => 'Creates new groups.',
        'type'        => 'write',          // 'read' or 'write'.
        'ajax'        => true,             // Available to AJAX calls from the web.
        'services'    => [                 // Optional list of predeclared services including it.
            'myintegration',
        ],
        // Advisory list shown to admins configuring custom services:
        'capabilities' => 'moodle/course:creategroups,moodle/course:managegroups',
        // Legacy (avoid with namespaced classes): 'methodname' (defaults to 'execute').
    ],
];
```

After adding/changing services, bump the plugin version to trigger the upgrade that registers them.

## db/caches.php — MUC cache definitions

```php
$definitions = [
    'somedata' => [
        'mode' => cache_store::MODE_APPLICATION,  // APPLICATION (shared), SESSION, or REQUEST.
    ],
];
```

Each definition requires a lang string `cachedef_<name>`. Bump the plugin version so the definitions file is processed. Use via `cache::make('mod_myplugin', 'somedata')`.

## db/hooks.php — hook callbacks (Hooks API)

Registers callbacks for core/plugin hooks. Callbacks are `some\class::static_method` (string) or `[\some\class::class, 'method']` (array notation, Moodle 4.4+). Executed highest priority first. Callbacks run during install/upgrade, so verify plugin state (e.g. `during_initial_install()`).

```php
// /local/stuff/classes/local/hook_callbacks.php
namespace local_stuff\local;

use mod_activity\hook\installation_finished;

class hook_callbacks {
    public static function activity_installation_finished(installation_finished $hook): void {
        if (during_initial_install()) {
            return;
        }
        // Do something...
    }
}
```

```php
// /local/stuff/db/hooks.php
$callbacks = [
    [
        'hook' => mod_activity\hook\installation_finished::class,
        'callback' => [\local_stuff\local\hook_callbacks::class, 'activity_installation_finished'],
        'priority' => 500,
    ],
];
```

Registrations are cached — bump the plugin version or purge caches. Since 5.1, `core\hook\manager::is_deprecated_plugin_callback()` is gone; use `get_hooks_deprecating_plugin_callback()`.

The Moodle App remote add-on API (`db/mobile.php`) is intentionally outside
the scope of this skill.

## db/tag.php — tag areas

Lists the tag areas a component provides in `$tagareas`. Each entry: `itemtype` (required; name of an existing DB table), `component` (full frankenstyle name), optional `collection`, `searchable`, `showstandard` (e.g. `core_tag_tag::HIDE_STANDARD`; only applied when the area is first added), `customurl`, `callback` (returns a `core_tag\output\tagindex`), `callbackfile` (only when the callback is not autoloaded), `multiplecontexts`. Lang strings: `tagarea_<itemtype>` and `tagcollection_<name>`.

```php
$tagareas = [
    [
        'itemtype' => 'local_myplugin_things',
        'component' => 'local_myplugin',
    ],
];
```

## classes/ — autoloading and namespaces

Moodle autoloads classes placed under `classes/` with the correct namespace — no `require_once` needed. Formal namespaces are required for all new classes.

- Level-1 namespace = the component name: `\mod_forum\discussion` lives in `mod/forum/classes/discussion.php`.
- Sub-namespaces map to subdirectories: `\mod_porridge\local\equipment\spoon` → `mod/porridge/classes/local/equipment/spoon.php`.
- One namespace declaration per file; namespace is the first non-comment line, followed by a blank line and any `use` statements (one per line).
- Frankenstyle-prefixed class names without namespaces (`mod_forum_example`) are deprecated — write `mod_forum\example`.

## settings.php

Defines admin-configurable settings. Setting names must be `plugintype_pluginname/settingname` so they are stored in `config_plugins`. `settings.php` must not execute DB queries at include time — use the lazy-loading of `admin_setting_configselect` and friends for DB-driven options, and autoloaded classes (not `require_once`) for shared code.

Since 5.3 (MDL-81935): the admin tree and setting classes have autoloadable names, e.g. `\admin_settingpage` → `\core_admin\setting\settingpage\settingpage`, `\admin_category` → `\core_admin\setting\tree\category`, `\admin_externalpage` → `\core_admin\setting\tree\externalpage`, `\admin_setting_configtext` → `\core_admin\setting\setting\configtext`, `\admin_setting_configcheckbox` → `\core_admin\setting\setting\configcheckbox`, `\admin_setting_heading` → `\core_admin\setting\setting\heading`. The old class names are unaffected for now, so a plugin that also supports pre-5.3 releases should keep the old names. Since 5.1, custom `admin_settings` / `admin_externalpage` subclasses are encouraged to set the new `searchmatchtype` property.

## lib.php and locallib.php

`lib.php` is a legacy bridge between core and the plugin: define only callbacks that are not yet supported as autoloadable classes or hooks. Core often loads all `lib.php` files of a plugin type, so keep it as small as possible; put internal logic in autoloaded classes. `locallib.php` is a legacy pattern that is no longer recommended — use namespaced classes in `classes/` instead.

## Other directories and files

- **cli/** — CLI scripts by convention. Every CLI script must `define('CLI_SCRIPT', true);` before including `config.php`.
- **pix/** — icons; `icon.svg` preferred with `icon.png` fallback. Most plugin types require a default icon.
- **templates/** — Mustache templates in `templates/*.mustache`, referenced as `component/templatename` (e.g. `mod_lesson/timer` → `mod/lesson/templates/timer.mustache`). Subdirectories allowed since 3.8 (first level follows namespace level-2 naming rules). Themes can override any template.
- **amd/** — JavaScript written as ES modules, transpiled to AMD for deployment (`amd/src` → `amd/build`). All new JS uses ESM format.
- **yui/** — legacy YUI JavaScript (deprecated in favour of AMD/ESM).
- **backup/** — backup and restore support (required for activity modules).
- **styles.css** — plugin CSS, included in the theme stylesheet; avoid custom styles where possible. Since 5.3 (MDL-68037) Boost has an experimental dark colour mode, so never assume a light page: take colours from Bootstrap custom properties (`--bs-body-bg`, `--bs-body-color`, `--bs-secondary-bg`, `--bs-tertiary-bg`, `--bs-border-color`, `--bs-emphasis-color`, `--bs-link-color`) or design-system tokens (`--mds-bg-surface-*`, `--mds-text-*`, `--mds-border-*`), prefer utility classes such as `bg-body`/`text-body` over `bg-white`/`text-dark`, and scope mode-specific colours with `[data-bs-theme="dark"]`. Render icons through the icon API so they inherit `currentColor`.
- **thirdpartylibs.xml** — declares bundled third-party libraries and their licences.
- **readme_moodle.txt** — notes on how third-party code was imported/modified.
- **upgrade.txt** — human-readable API change notes per version.
- **environment.xml** — extra environment requirements checked at install/upgrade.
- **README / CHANGES** — plugin readme and changelog.
