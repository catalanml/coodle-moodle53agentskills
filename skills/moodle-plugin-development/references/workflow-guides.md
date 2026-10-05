# Moodle 5.3 Workflow Guides

Distilled from the official Moodle 5.3 developer docs: Git workflow, the plugin Upgrade API, PHP profiling, the Bootstrap 5 migration, the 5.1 code restructure (`public/` directory), platform requirements, and the 5.3 developer update. For a full 4.5 → 5.3 API migration list see `upgrading-to-5.3.md`.

## Git workflow for Moodle developers

Moodle recommends the MDK (Moodle Developer Kit) tool to streamline Git-based development (`mdk update` keeps official remotes synced).

### General workflow

1. Commit changes to your personal (local) repository.
2. Push to your public repository (usually a GitHub fork of `moodle/moodle`) and publish links to your changes in the Moodle Tracker.
3. Request a peer review from another developer.
4. When the peer reviewer is happy, they submit the issue for integration.
5. Moodle integrators pull your changes and, if approved, merge them into Moodle's integration repository.
6. Integrated changes are tested and pushed to Moodle's production repository (moodle.git).
7. You synchronise your local repository with production and the cycle repeats.

The workflow runs in weekly cycles: integration Monday/Tuesday, testing Wednesday, production update Thursday (or Friday). New Tracker users cannot self-assign or send issues for peer review until their first fix is integrated — comment with a link to your branch instead.

### Setup

```bash
git config --global user.name "Your Name"        # real name, proper capitalisation
git config --global user.email yourmail@domain.tld
git config --global core.filemode false          # don't push file-permission changes
git config --global core.autocrlf false          # Moodle uses LF only
```

Fork `github.com/moodle/moodle`, set up your SSH key, then:

```bash
git clone git@github.com:YOUR_GITHUB_USERNAME/moodle.git
git remote add upstream git://git.moodle.org/moodle.git
```

Optionally enable GitHub Actions on your fork so your code is verified automatically before review.

### Syncing your fork

Never commit directly to `main` or `MOODLE_XXX_STABLE` branches — they must stay fast-forwardable. Sync weekly (upstream updates roughly once a week; safe to cron):

```bash
git fetch upstream
for BRANCH in MOODLE_{19..39}_STABLE MOODLE_{310..311}_STABLE MOODLE_{400..405}_STABLE MOODLE_{500..503}_STABLE main; do
    git push origin refs/remotes/upstream/$BRANCH:refs/heads/$BRANCH
done
```

**Note:** the loop on the upstream Git page still stops at `MOODLE_{400..405}_STABLE`; the 5.x stable branches (up to `MOODLE_503_STABLE`) follow the same naming pattern, so extend the range.

If Moodle creates a new stable branch your fork lacks, `git checkout MOODLE_XX_STABLE && git push origin MOODLE_XX_STABLE:MOODLE_XX_STABLE` (then optionally delete the local copy).

### Preparing and submitting a patch

Always work on a topic branch forked off the standard branch it should merge into — stable fixes off `MOODLE_XXX_STABLE` (e.g. `MOODLE_503_STABLE` for a 5.3 fix), new features for the next major version off `main`:

```bash
git checkout -b MDL-xxxxx-main_brief_name origin/main
# ... work ...
git add .
git commit -m "MDL-XXXXX component: Brief description"   # follow coding-style commit rules
git push origin MDL-xxxxx-main_brief_name
```

To fix commits after CI (CiBot) feedback, either `git reset --mixed origin/main` and re-commit, or `git rebase --interactive`, then force-push: `git push -f origin MDL-xxxxx-main_brief_name`.

Since 5.1 the core repository keeps almost all code under `public/` (see "Code restructure" below), so repository-level paths in patches look like `public/mod/forum/lib.php`.

### Rebasing

You may be asked to rebase if your branch is based on an outdated commit: `git rebase main MDL-xxxxx-main_topic_branch`. Rebasing rewrites history — never rebase a branch others may have based work on, and never base your branches on branches submitted for integration (they are potentially rebased). On conflicts: fix files, `git add`, `git rebase --continue`.

### Applying a fix to multiple branches

Stable branches are maintained separately from `main` and never merged into it; contributors typically prepare one branch per affected stable branch plus one for `main`.

Cherry-pick a single commit into another checkout:

```bash
cd ~/public_html/moodle_main
git checkout -b MDL-xxxxx-main_topic origin/main
git fetch ../moodle_400 MDL-xxxxx-400_topic
git cherry-pick FETCH_HEAD        # or: git cherry-pick A^..B for a range
```

For multiple commits, export and re-apply with three-way merge fallback:

```bash
git format-patch -o .patches MOODLE_400_STABLE..MDL-xxxxx-400_topic
git am -3 ../moodle_400/.patches/*    # git am --continue / --abort on conflicts
```

Backporting across the 5.1 restructure: a patch made on a branch with `public/` paths will not apply cleanly to `MOODLE_405_STABLE`/`MOODLE_500_STABLE` (no `public/` directory) without path adjustment. The docs do not describe a tool for this.

### Reviewing someone else's branch

```bash
git fetch git://github.com/alice/moodle.git wip-feature
git log -p FETCH_HEAD                     # inspect
git checkout -b alice-wip-feature FETCH_HEAD   # or git merge FETCH_HEAD
git diff ...FETCH_HEAD                    # preview merge without changing anything
```

### Housekeeping

List merged (safe-to-delete) branches with `git branch --merged upstream/main` (add `-r` for remote-tracking). Delete local: `git branch -d NAME`; delete on origin: `git push origin :NAME`.

## Plugin upgrades (Upgrade API)

The Upgrade API lets a plugin manage its own installation and upgrades. Plugins can only be upgraded, never downgraded. In a 5.1+ git checkout these files live under `public/<plugin path>/` (e.g. `public/mod/example/db/upgrade.php`); `$CFG->dirroot` still points at the code directory, so `$CFG->dirroot . '/mod/example/db/upgradelib.php'` is unchanged.

### Key files

- `version.php` — plugin version. Bump it after any change to `db/`, JavaScript, new autoloaded classes, new settings, or language packs: a new version triggers the upgrade procedure and resets all caches.
- `db/install.xml` — database tables created on initial installation only. Must always describe the up-to-date structure. Must be created/maintained with the XMLDB Editor.
- `db/upgrade.php` — steps to migrate from an older installed version. Database field definitions here must also come from the XMLDB Editor. Since 5.2 (MDL-87158), XMLDB-generated `rename_field(...)` step code checks for field existence so it can run more than once.
- `db/upgradelib.php` (optional) — helper functions for non-trivial upgrade steps; these can be unit-tested.

### How it works

On first install (version.php exists on disk, no version row in `config_plugins`), Moodle uses `install.xml`. On upgrade (disk version newer than the stored one), Moodle runs `xmldb_[plugintype]_[pluginname]_upgrade($oldversion)` in `db/upgrade.php`:

```php
function xmldb_[plugintype]_[pluginname]_upgrade($oldversion): bool {
    global $CFG, $DB;
    $dbman = $DB->get_manager(); // DDL manager and xmldb classes.

    if ($oldversion < 2019031200) {
        // Perform the upgrade step, then mark the savepoint:
        // upgrade_plugin_savepoint(true, 2019031200, 'plugintype', 'pluginname');
    }
    return true;
}
```

Helpers callable from upgrade.php:

- `upgrade_set_timeout()` — increase timeout before a long-running step.
- `upgrade_(main|mod|block|plugin)_savepoint()` — mark a step completed (runs once only) and reset timeouts.

Related: `xmldb_..._install()` in `db/install.php`, `xmldb_..._uninstall()` in `db/uninstall.php` (use `main` for core, frankenstyle name for plugins, e.g. `xmldb_block_example_install()`).

Since 5.2 (MDL-87107): `upgrade_ensure_not_running()` is deprecated — use `\core\setup::ensure_upgrade_is_not_running()`, `\core\setup::warn_if_upgrade_is_running()` or `\core\setup::is_upgrade_running()`.

Since 5.3 (MDL-89289): uninstalling a block no longer deletes its instances synchronously; `\core\plugininfo\block::uninstall_cleanup()` queues `\core\task\delete_block_instances_task`, which deletes instances and related data in batches.

### Upgrade code restrictions

- Upgrade code may always use the basic DML database API.
- Plugin upgrade code must not call any of the plugin's own functions (plugin data is still old) — manipulate the database directly instead. Calling core functions is allowed (core data is already current, since core upgrades first).
- Core upgrade code must not call core functions either (both core and plugin data are old), except functions explicitly marked safe such as `set_config`/`get_config`.

### Delaying upgrade work

When a step needs structures not yet present, needs operations unavailable during upgrade (e.g. course cache rebuild throws), or is long-running, queue an ad-hoc task instead:

```php
if ($oldversion < 2020031001) {
    $task = new \plugintype_pluginname\task\migrate_course_completion();
    \core\task\manager::queue_adhoc_task($task, true);
    upgrade_plugin_savepoint(true, 2020031001, 'plugintype', 'pluginname');
}
```

Since 5.3 (MDL-86422), `queue_adhoc_task()` always returns the id of the newly inserted task or of the existing one (depending on `$checkforexisting`), or `false` if the component is deprecated or the insert failed.

### Core stable-branch DB changes (advanced, rare)

Because users can upgrade via different paths (stable-to-stable, stable-to-next-major), any DB change made on a stable branch must be idempotent, e.g. `if (!$dbman->table_exists($table)) { $dbman->create_table($table); }` instead of an unconditional `create_table`. Choose version numbers on each branch carefully and test all upgrade paths.

## Profiling PHP

Two profiler families:

- **XHProf** — function-level hierarchical profiler (C extension), suitable for production. Reports inclusive/exclusive wall time, memory, CPU, call counts; supports run comparison and aggregation. Built into Moodle.
- **Xdebug** — profiler outputting cachegrind files; not recommended for production.

### XHProf in Moodle

With an xhprof extension installed, a "Profiling" option appears under Site administration > Development. Enable "Enable profiling" and specify paths to profile, relative to the Moodle directory (wildcards allowed, e.g. `mod/*/view.php`). Visit the pages, then view runs under Site administration > Development > Profiling runs, as a data table or a callgraph (callgraph requires Graphviz and `pathtodot` configured). Runs on the same page can be compared. (The profiling page is unchanged since 4.5 and does not say whether paths are relative to the install root or to `public/` after the 5.1 restructure; URL-style paths such as `mod/*/view.php` match what the web server serves.)

Since 3.6 (MDL-63031), a plugin can handle XHProf data instead of the database: set `$CFG->disableprofilingtodatabase = false;` in config.php and provide a `store_profiling_data` function.

### Xdebug profiler

Install the Xdebug 3 extension (typically via pecl), restart the webserver, then in php.ini:

```ini
xdebug.mode = profile
xdebug.start_with_request = trigger    ; omit to profile every request
xdebug.trigger_value = ProfileMe       ; optional: require a specific value
```

With trigger mode, append `XDEBUG_PROFILE` to the URL (e.g. `...view.php?id=5&XDEBUG_PROFILE=ProfileMe`); the trigger variable is checked in `$_ENV`, `$_GET`, `$_POST`, and cookies (browser extensions can set the cookie for POST/AJAX profiling). Output goes to `xdebug.output_dir` as `cachegrind.out.*` files; analyse with KCachegrind, WinCacheGrind, xdebugtoolkit, or Webgrind.

## Code restructure: the `public/` directory (since 5.1, MDL-83424)

Since 5.1 almost all code, including every plugin, lives in a `public/` directory below the install root, so that Moodle can later ship content that is not web accessible.

- `$CFG->wwwroot` and `$CFG->dirroot` behave as before — `$CFG->dirroot` points at the code (`public`) directory. Keep writing paths as `$CFG->dirroot . '/mod/example/lib.php'`.
- New read-only `$CFG->root` points at the root of the Moodle installation.
- In a git checkout, `mod/forum` is at `public/mod/forum`; most existing tooling has already been updated to the new plugin locations.
- The web server must serve `public/`:

```diff
-    DocumentRoot /srv/moodle
+    DocumentRoot /srv/moodle/public       # Apache
-    root /srv/moodle;
+    root /srv/moodle/public;              # nginx
```

- Since 5.2 (MDL-87072) the `$CFG->wwwrootendsinpublic` flag is removed, forcing servers to be configured this way.
- Requests for paths that are not real files fall back to the Moodle router, `public/r.php`. The restructure guide includes full Apache/nginx developer configurations that serve several Moodle versions from sub-folders (e.g. `/srv/moodle/m405` vs `/srv/moodle/m501/public`), choosing `public/` when it exists and falling back to `r.php`; copy them from the guide when needed rather than re-deriving them.

## Bootstrap 5 migration

Moodle migrated from Bootstrap 4 to 5 in seven steps: (1) PopperJS v2, (2) SCSS deprecation process, (3) refactoring BS4 features dropped in BS5 (4.4, MDL-79914), (4) a BS5 "bridge" (4.5, MDL-79917), (5) the BS5 upgrade itself — Bootstrap 5.3 in theme_boost (5.0, MDL-75669; the bridge file is removed), (6) a BS4 backwards-compatibility layer, (7) final deprecation in **Moodle 6.0**. Third-party code should now use BS5 syntax everywhere; the BC layer only buys time.

### Class and markup renames

| BS4 (don't) | BS5 (do) |
| --- | --- |
| `.badge-danger`, `.badge-pill` | `.text-bg-danger` (or `.bg-danger .text-white`), `.rounded-pill` |
| `.media`, `.media-left`, `.media-body` | `.d-flex`, `.flex-shrink-0`, `.flex-grow-1 .ms-3` |
| `.form-group` / `.form-inline` | `.mb-3` / `.d-flex .flex-wrap .align-items-center` |
| `.card-deck` | `.row .row-cols-1 .row-cols-sm-2` with `.col` wrappers |
| `.no-gutters` | `.g-0` |
| `.close` | `.btn-close` |
| `.float-left/right`, `.border-left/right`, `.rounded-left/right`, `.text-left/right` | `-start` / `-end` variants |
| `.ml-*`/`.mr-*`, `.pl-*`/`.pr-*` | `.ms-*`/`.me-*`, `.ps-*`/`.pe-*` |
| `.rounded-sm`, `.rounded-lg` | `.rounded-1`, `.rounded-3` |
| `.sr-only`, `.sr-only-focusable` (and mixins `sr-only()`, `sr-only-focusable()`) | `.visually-hidden`, `.visually-hidden-focusable` (mixins `visually-hidden()`, `visually-hidden-focusable()`) |
| `.font-weight-*`, `.font-italic` | `.fw-*`, `.fst-italic` |
| `.dropdown-menu-left/right` | `.dropdown-menu-start/end` |
| `.custom-check`, `.custom-check.custom-switch`, `.custom-select`, `.custom-range` | `.form-check`, `.form-check.form-switch`, `.form-select`, `.form-range` |
| `.custom-file`, `.form-file` | custom styles on top of `.form-control` |
| `.input-group-append` / `.input-group-prepend` wrappers | dropped: buttons and `.input-group-text` are direct children of `.input-group` |
| `data-toggle`, `data-target`, `data-html` (all Bootstrap JS data attributes) | `data-bs-toggle`, `data-bs-target`, `data-bs-html` |

```html
<div class="input-group">
    <input type="text" class="form-control">
    <button type="submit" class="btn btn-primary search-icon">
        {{#pix}} a/search, core {{/pix}}
        <span class="visually-hidden">{{#str}} search, core {{/str}}</span>
    </button>
</div>
<button class="btn btn-outline-secondary" type="button" data-bs-toggle="tooltip" data-bs-html="true"
        title="{{#str}} string_with_html, block_my_block {{/str}}">{{#pix}} i/info, core {{/pix}}</button>
```

### SCSS

- Dropped mixins (replace with plain CSS): `hover`, `hover-focus`, `plain-hover-focus`, `hover-focus-active`, `float-left/right/none`, `nav-divider`, `img-retina`, `text-hide`, `invisible`, `form-control-focus`, `text-emphasis-variant`, `size`, `make-container-max-widths`, `g-variant`, `bg-gradient-variant`. E.g. `@include hover-focus() { ... }` → `&:hover, &:focus { ... }`.
- `theme-color-level($name, $level)` → `shift-color($color, $weight)`: pass the colour variable, not the name, and percentages instead of levels (each level = 10%): `theme-color-level('primary', 1)` → `shift-color($primary, 10%)`; `-2` → `-20%`.
- **Breakpoint semantics changed**: `media-breakpoint-down()` and the second argument of `media-breakpoint-between()` now take the breakpoint itself, not the next one. To target viewports smaller than md: BS4 `media-breakpoint-down(sm)` → BS5 `media-breakpoint-down(md)`; `between(sm, md)` → `between(sm, lg)`.

### JavaScript

Bootstrap 5 dropped jQuery; core Bootstrap-related code is vanilla JS. Use the Bootstrap component classes and native events:

```js
import {Dropdown} from 'bootstrap';

document.querySelectorAll('[data-bs-toggle="tab"]').forEach((tab) => {
    tab.addEventListener('shown.bs.tab', () => {
        const bootstrapDropdown = new Dropdown('#my-dropdown');
        bootstrapDropdown.toggle();
    });
});
```

**Note (stale docs):** the 5.3 Bootstrap 5 migration page still shows `import Dropdown from 'theme_boost/bootstrap/dropdown'`. The 5.3 upgrade notes (theme_boost, MDL-88766) deprecate that: AMD modules must not depend on `theme_boost/bootstrap/*`. Use `import {Tooltip} from 'bootstrap'` on 5.3+, or `import {Tooltip} from 'theme_boost/index'` to also support 5.2 and earlier (supported until 7.0). The non-public `util` and `dom` helpers still need direct paths: `bootstrap/dom/event-handler` (5.3+) vs `theme_boost/bootstrap/dom/event-handler` (5.2 and earlier).

### BS4 backwards-compatibility layer (5.0, removed in 6.0)

- **jQuery support** (MDL-84324): `$('[data-bs-toggle="tooltip"]').tooltip();` and `$('#myTab a').on('shown.bs.tab', ...)` still work.
- **SCSS helpers** (MDL-80519, initially deprecated `bs4-compat` SCSS): `.media`, `.badge-success` etc., `.form-inline`, `.mr-1`/`.pl-2` spacing, `.sr-only*`, `.font-weight-bold`/`.font-italic`, `.custom-radio`/`.custom-switch` keep working until 6.0.
- **Old data attributes** (MDL-84450): the directly-deprecated `theme_boost/bs4-compat` AMD module rewrites `data-toggle` → `data-bs-toggle`, `data-target` → `data-bs-target`, etc. It does not touch content generated dynamically after it runs.

```js
import * as BS4compat from 'theme_boost/bs4-compat';
BS4compat.init(document.querySelector('[data-region="my-plugin-region"]')); // or BS4compat.init() for the whole document
```

```php
$PAGE->requires->js_call_amd('theme_boost/bs4-compat', 'init');
```

### Dark colour mode (since 5.3, MDL-68037)

Boost has an experimental light/dark colour mode (setting `theme_boost | enablecolourmodes`, built on the Bootstrap 5.3 colour modes API; the mode is written to `data-bs-theme` on `<html>`). A plugin's `styles.css` can no longer assume a light page: take colours from Bootstrap custom properties (`--bs-body-bg`, `--bs-body-color`, `--bs-secondary-bg`, `--bs-tertiary-bg`, `--bs-border-color`, `--bs-emphasis-color`, `--bs-link-color`) or design-system tokens (`--mds-bg-surface-*`, `--mds-text-*`, `--mds-border-*`), or utilities (`bg-body`, `bg-body-secondary`, `bg-body-tertiary`, `text-body`, `text-body-secondary`, `text-bg-*`, `border`) rather than `bg-white`/`text-dark`. Scope genuinely mode-specific colours with `[data-bs-theme="dark"]`. Never fix only half of a background/text pair. Render icons through the icon API so they inherit `currentColor` (an SVG in `<img>` keeps its baked-in fill).

## Platform requirements (Moodle 5.3)

From the 5.3 release notes:

- Upgrade path: from Moodle **4.4 or later**.
- PHP: minimum **8.3.0**; 8.4.x supported. Only 64-bit PHP. Extension **sodium** required. `max_input_vars` must be >= 5000.
- Other required PHP extensions (from the requirements page): iconv, mbstring, curl, openssl, ctype, zip, zlib, simplexml, spl, pcre, dom, xml, xmlreader, intl, json, hash, fileinfo.

| Database | Minimum | PHP extension |
| --- | --- | --- |
| PostgreSQL | 17 (raised in 5.3) | pgsql |
| MySQL | 8.4 (raised in 5.0) | mysqli |
| MariaDB | 11.4.0 (raised in 5.3) | mysqli |
| Aurora MySQL | 8.0 (configure with MySQL version numbers, not Aurora's 3.x) | mysqli |
| Microsoft SQL Server | 2019 (raised in 5.2) | sqlsrv |

Oracle is not supported since Moodle 5.0. History: 5.0 raised PHP to 8.2 and the upgrade path to 4.2.3; 5.2 raised PHP to 8.3 and the upgrade path to 4.4.

**Note:** the "Technical requirements" page bundled with the 5.3 docs is stale — it still says "Moodle 4.0 supports PHP 7.3/7.4/8.0" and lists Oracle and 4.0-era database minimums. Trust the release notes.

## Moodle 5.3 developer update highlights

### Course formats and activities

- **Linear navigation** (MDL-84921, MDL-87302): course views can show "Previous"/"Next" controls in a sticky footer. Third-party formats are opted out by default; override `\core_courseformat\base::uses_linear_navigation()` (e.g. `public static function uses_linear_navigation(): bool { return true; }`, optionally driven by a format setting).
- Activities can steer it via `\core_course\cm_info::get_navigation_url()`, `set_navigation_url(?url $url)` and `reset_navigation_url()` (added 5.2, MDL-87984), typically from the `cm_info_dynamic` callback; setting `null` removes the module from the navigation flow.
- **Sticky footer content** (MDL-88601): `moodle_page::set_supplementary_content()` / `get_supplementary_content()` inject an `action_link` into the sticky footer whether or not the format uses linear navigation (mod_forum uses it for "Go to all discussions"). Related `moodle_page` methods: `set_show_navigation_footer(bool $show)` (MDL-87575), `set_has_sticky_footer()` / `has_sticky_footer()` (MDL-87302).
- `core_courseformat\base::get_return_section()` is deprecated → `get_page_section()`; the `sr` option of `get_view_url()` → `pagesectionid` (MDL-86284).
- Activity chooser items can be disabled via `disabled` and `disabledreason` on `content_item` (MDL-87373).
- `FEATURE_GROUPMEMBERSONLY` is deprecated and no longer supported — remove it from `*_supports()` (MDL-83231).

### Routing and web services

- **Route scopes** (MDL-89710): every route in a `route\api` namespace must carry one or more `#[\core\router\scope\scopeset(...)]` attributes (AND within one, OR across repeats) or an explicit `#[\core\router\scope\unscoped_resource]`; a core PHPUnit test fails otherwise. Tokens (OAuth2 or Personal Access Token) are checked against the scopes in addition to capability checks.
- Routes accept a `cookies: false|true` option; `\core\session\manager::supports_cookies()` / `set_cookies_supported()` replace checking `NO_MOODLE_COOKIES` (MDL-87174).
- Legacy `\external_api`, `\external_value`, `\external_single_structure`, `\external_multiple_structure`, `\external_function_parameters`, `\external_util`, etc. are deprecated in favour of the `\core_external\…` classes (MDL-81225).
- New `allowcorsrequests` property in external function definitions (MDL-87150).

### Core APIs

- `user/lib.php` functions deprecated and moved to `\core\user` static methods, e.g. `user_create_user()` → `\core\user::create_user()`, `user_update_user()` → `\core\user::update_user()`, `user_get_user_details()` → `\core\user::get_user_details()` (MDL-82650).
- Admin setting classes are autoloadable under `\core_admin\setting\…`, e.g. `\admin_setting_configtext` → `\core_admin\setting\setting\configtext`, `\admin_settingpage` → `\core_admin\setting\settingpage\settingpage`, `\admin_category` → `\core_admin\setting\tree\category`; old names still work (MDL-81935).
- `\core_user\hook\extend_user_menu`: use `add_menu_item()` with a `\core_user\output\user_action_menu\base` item; `add_navitem` is deprecated (MDL-88938).
- Hook `before_email_to_user` emitted by `email_to_user()` can modify or block email (MDL-69724).
- Custom fields: MDL-89238 reverses 5.2's advice — `\core_customfield\handler::create()` again returns a new instance by default; opt into caching with `protected const CACHE_HANDLER_INSTANCES = true;`.
- Adhoc tasks: `set_soft_retry_delay()` inside `execute()` requests a retry via `\core\task\manager::adhoc_task_delayed()` without marking failure (MDL-79763).
- Attribute-based DI: `#[\DI\Attribute\Inject]` on properties, resolved via `di::get()` / `di::make()` (MDL-89528).
- React: `\core\output\react_component_renderable` interface and `html_writer::react_component()` (MDL-89296).

### Output and themes

- Classic theme removed from core (MDL-88351); sites wanting it must install it from the Moodle GitHub repository before upgrading, and a `$CFG->theme = 'classic'` in config.php must be changed manually.
- `core/modal` titles are now `<h2 class="modal-title fs-5">` (MDL-75699): start headings inside modal content at `<h3>`.
- Primary/secondary navigation now rendered by React components (`core/nav/PrimaryNav`, `core/nav/Nav`); CSS/JS targeting `.moremenu` or `.nav-link` there must be checked (`.mds-nav-pill` instead) (MDL-87830, MDL-89294).
- Boost default font is now self-hosted Noto Sans (MDL-88412); `theme_boost/drawer` block `drawerheadercontent` replaced by `drawercontrols` (MDL-89050).

### Other docs index changes

The 5.3 guides index adds a "Distribute and install Moodle plugins using Composer" guide (`guides/composer/`), and the APIs index adds an Open Telemetry integration entry (`apis/telemetry/`).
