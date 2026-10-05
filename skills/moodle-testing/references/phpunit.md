# Moodle PHPUnit Reference (Moodle 5.3)

Setup, structure, and patterns for writing and running PHPUnit tests in Moodle
5.3, per the Moodle 5.3 developer docs. Core uses **PHPUnit 11.4** since 5.0
(MDL-83468); 4.4–4.5 used PHPUnit 9.6.

| Moodle | PHPUnit |
|---|---|
| 5.0+ (incl. 5.3) | 11.4 |
| 4.4 – 4.5 | 9.6 |
| 3.11 – 4.3 | 9.5 |

**Paths since 5.1 (MDL-83424):** almost all code moved into `public/`. The
PHPUnit docs run the CLI scripts as `php public/admin/tool/phpunit/cli/...`
and `vendor/bin/phpunit` from the install root (the directory that contains
`public/`, i.e. `$CFG->root`; the page still calls it `$CFG->dirroot`). A
plugin's tests therefore live at `public/<plugintype>/<plugin>/tests/` in a git
checkout. The docs' `-c mod/forum` and `cohort/tests/lib_test.php` examples
were not updated for `public/` — they are reproduced as written below.

## Setup

PHPUnit is a Composer dependency, not part of the Moodle install:

```bash
cd /path/to/moodle
php composer.phar install   # add --prefer-source on connection problems
```

Add a dedicated test prefix and dataroot to `config.php` (examples in
`config-dist.php`):

```php
$CFG->phpunit_prefix = 'phpu_';
$CFG->phpunit_dataroot = '/home/example/phpu_moodledata';
```

Optionally use a separate database for tests (standard DB settings otherwise):

```php
$CFG->phpunit_dbtype    = 'pgsql';
$CFG->phpunit_dblibrary = 'native';
$CFG->phpunit_dbhost    = '127.0.0.1';
$CFG->phpunit_dbname    = 'mytestdb';
$CFG->phpunit_dbuser    = 'postgres';
$CFG->phpunit_dbpass    = 'some_password';
```

The docs' comment still lists `'oci'`, but Oracle is unsupported since 5.0.
Some tests need internet access; set `$CFG->proxyhost` etc. in `config.php`
if you are behind a proxy.

Initialise the test environment (php CLI must be on PATH; `en_AU` locale must
be installed):

```bash
cd /home/example/moodle
php public/admin/tool/phpunit/cli/init.php
```

Run `init.php` again after any upgrade or plugin (un)installation, or after
adding tests to a plugin — or use the faster `--upgrade` below.

## Maintaining the environment (since 5.3, MDL-88495)

`util.php` manages an initialised environment (`--help` lists everything).

After adding, removing or changing a plugin (including a version bump), PHPUnit
refuses to run ("Moodle PHPUnit environment was initialised for different
version"). Instead of a full re-init, upgrade the plugins in place:

```bash
php public/admin/tool/phpunit/cli/util.php --upgrade
```

This runs the standard non-core upgrade against the test DB (installs new
plugins, runs their upgrade steps, applies plugin default settings).

**Snapshots** capture an initialised environment (a `.zip` of DB structure +
data + `filedir`), stored under `<phpunit_dataroot>/snapshots/` and keyed by DB
type and Moodle version:

```bash
php public/admin/tool/phpunit/cli/util.php --snapshot            # pgsql-snapshot-<version>
php public/admin/tool/phpunit/cli/util.php --snapshot=mysnapshot
php public/admin/tool/phpunit/cli/util.php --list                # exact names for --restore
php public/admin/tool/phpunit/cli/util.php --drop                # DB must be empty first
php public/admin/tool/phpunit/cli/util.php --restore=pgsql-snapshot-2026072200
```

Pass the name without `.zip`. `--drop` preserves snapshots. CI flow: on vanilla
Moodle `init.php` + `--snapshot`, cache `<phpunit_dataroot>/snapshots/`; later
runs `--restore=<name>`, install the plugin under test, `--upgrade`, run tests.
Fall back to a fresh `init.php` when a new Moodle version invalidates the
snapshot.

## Running tests

From the install root:

```bash
vendor/bin/phpunit                                    # everything
vendor/bin/phpunit --filter metadata_registry_test    # one class
vendor/bin/phpunit --filter test_get_registry_metadata_count   # one method
vendor/bin/phpunit cohort/tests/lib_test.php          # one file
vendor/bin/phpunit --filter test_function_name path/to/file.php
vendor/bin/phpunit --testsuite mod_forum_testsuite    # one component
vendor/bin/phpunit --testsuite core_privacy_testsuite --filter test_component_is_compliant
vendor/bin/phpunit --group qtype_stack                # by @group
vendor/bin/phpunit --debug                            # print each test name
```

PHPUnit 11 re-added `--debug` (removed in 10). Per-component configs:

```bash
php public/admin/tool/phpunit/cli/util.php --buildcomponentconfigs
vendor/bin/phpunit -c mod/forum
```

Running by file path can execute tests a normal run would not find (e.g. a
misplaced file) — do at least one full or `--group` run. Tag every test class
with `@group <frankenstyle>` to run a plugin's tests alone. IDEs need the
absolute path to `phpunit.xml`.

Since 5.1 (MDL-86311) the default `phpunit.xml` enables `failOnDeprecation`
and `failOnWarning`: PHP warnings and deprecations fail tests again (as before
PHPUnit 10).

## Test files

Test files:

- MUST live in the component's `tests/` directory, lowercase, ending
  `_test.php`;
- MUST contain exactly one test class, named exactly like the file, extending
  a `_testcase` class (e.g. `\advanced_testcase`);
- SHOULD use the namespace of the unit under test (the `**/tests` namespace
  rules mirror `**/classes`; sub-directories mirror sub-namespaces).

```php
// mod/myplugin/tests/sample_test.php
namespace mod_myplugin;

/**
 * Tests for the sample class.
 *
 * @package    mod_myplugin
 * @copyright  2026 Kim Bloggs
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 * @covers     \mod_myplugin\sample
 */
final class sample_test extends \advanced_testcase {
    public function test_adding(): void {
        $this->assertEquals(3, 1 + 2);
    }
}
```

(`final` is not required by the docs; core marks some tests final — see
"Shared testcases" below.)

Prefer autoloading. Otherwise classes required to *define* the test class may
be included in the file; fixtures and code under test are loaded in
`setUpBeforeClass()`. Tests themselves are not autoloaded.

### Shared testcases

Custom testcase classes MUST be abstract, contain no tests, and be named with
the `_testcase` suffix (never `_test`). Core now ships autoloadable ones under
`<component>\tests\`:

| Replaces | Use |
|---|---|
| `\externallib_advanced_testcase` (5.1, MDL-86283) | `\core_external\tests\externallib_testcase` |
| `core_reportbuilder_testcase` + `require helpers.php` (5.0) | `\core_reportbuilder\tests\core_reportbuilder_testcase` |
| `\core\tests\route_testcase` (5.0) | `\core\tests\router\route_testcase` |
| extending `\mod_assign\privacy\provider_test` (now final, 5.0) | `\mod_assign\tests\provider_testcase` |
| extending `\mod_quiz\attempt_walkthrough_from_csv_test` (now final, 5.0) | `\mod_quiz\tests\attempt_walkthrough_testcase` (`$files` property -> static `::get_test_files()`) |

## Testcase classes

- `basic_testcase` — pure unit tests: no DB, filesystem or PHP-global changes.
- `advanced_testcase` — adds Moodle helpers (state reset, generators, sinks,
  user switching, clock mocking).

### Automatic state reset

DB and dataroot reset between tests. If a test modifies the DB or globals,
call `$this->resetAfterTest()`, otherwise: "Warning: unexpected database
modification, resetting DB state".

```php
public function test_deleting(): void {
    global $DB;

    $this->resetAfterTest(true);
    $DB->delete_records('user');
    $this->assertEmpty($DB->get_records('user'));
}
```

Rules:

- Call `resetAfterTest` immediately before the first data insertion on a given
  path, in the test or a helper it calls (multiple calls are fine).
- **Banned:** `resetAfterTest` or fixture creation in a shared `setUp()` —
  every test pays the reset cost, and testcase instances live for the whole
  run (memory bloat). Existing cases must be phased out.
- Avoid `resetAfterTest` where you can: write mockable code.

### Users

Only `guest` and `admin` exist and nobody is logged in:

```php
$this->setUser($user1);
$this->setAdminUser();
$this->setGuestUser();
$this->setUser(null);
```

## Data generators

From `advanced_testcase` test methods via `$this->getDataGenerator()`:

```php
$gen = $this->getDataGenerator();
$user = $gen->create_user(['email' => 'user1@example.com', 'username' => 'user1']);
$category = $gen->create_category(['name' => 'Some subcategory', 'parent' => $parentid]);
$course = $gen->create_course(['category' => $category->id]);
$page = $gen->create_module('page', ['course' => $course->id]);
$gen->get_plugin_generator('mod_page')->create_instance(['course' => $course->id]);
$gen->enrol_user($userid, $courseid);                      // self/manual enrol
$gen->enrol_user($userid, $courseid, $teacherroleid, 'manual');
$cohort = $gen->create_cohort();
$gen->create_scale(['name' => $name, 'scale' => $scale, 'courseid' => $courseid]);
$gen->create_role(['shortname' => $shortname, 'archetype' => $archetype]);
$gen->create_tag(['userid' => $userid, 'rawname' => $rawname]);
$gen->create_group(['courseid' => $courseid]);
$gen->create_group_member(['userid' => $userid, 'groupid' => $groupid]);
$gen->create_grouping(['courseid' => $courseid]);
$gen->create_grouping_group(['groupingid' => $groupingid, 'groupid' => $groupid]);
$gen->create_repository($type, $record, $options);
$gen->create_repository_type($type, $record, $options);
$gen->create_grade_category(['courseid' => $courseid]);
$gen->create_grade_item(['itemtype' => $itemtype, 'itemname' => $itemname]);
$gen->create_grade_outcome();
```

Plugin generators live in `tests/generator/lib.php`, extend
`\component_generator_base` (activities: `testing_module_generator` for
consistent method names) and are fetched with
`$this->getDataGenerator()->get_plugin_generator('my_componentname')`.

Generator changes 5.1–5.2 (UPGRADING): new `mod_choice` generator for choice
responses; `mod_data` entries generator can create approved entries; new
`mod_scorm` generator for attempts; `core_badges` `create_issued_badge` now
returns the issued badge object; `mod_subsection` generator accepts `summary`
(only for migration-tool tests, to be removed in 7.0).

## Data providers (PHPUnit 11 rules)

- Providers MUST be `public static` (non-static removed in PHPUnit 11).
- They only *define* data: no generators, no `setUser`, no DB calls, no
  objects with side effects. They run before any test; the DB is reset before
  each test.
- They must not be empty. Skip with `@requires` (e.g.
  `@requires extension Redis`) or add a "skip" parameter and skip inside the
  test.
- Keyed data sets must match the test's parameter names (mismatches, or mixed
  named/unnamed keys, raise a PHPUnit warning; `failOnWarning` is on since
  5.1).

```php
/**
 * Test foobar.
 *
 * @dataProvider foobar_provider
 * @param int $foo
 * @param int $bar
 */
public function test_foobar(int $foo, int $bar): void {
    // Perform the tests here.
}

/**
 * Data provider for {@see self::test_foobar()}.
 *
 * @return \Generator
 */
public static function foobar_provider(): \Generator {
    yield 'Same numbers' => ['foo' => 42, 'bar' => 42];
    yield 'Different numbers' => ['foo' => 21, 'bar' => 84];
}
```

**Note:** the "Writing PHPUnit tests" guide still shows a non-static provider;
that no longer works on 5.x. moodle-cs (phpcs) can help fix providers.

## PHPUnit 11 upgrade fixes (since 5.0)

- `withConsecutive`, `willReturnConsecutive`, `onConsecutive` removed from core
  (MDL-81308) — rewrite those mocks.
- `MockBuilder::setMethods()` removed: drop it, or use `onlyMethods()` /
  `addMethods()`.
- `assertEqualsCanonicalizing()` now compares array **keys** too: sort data
  consistently, or wrap values in `array_values()`.
- `assertTag`, `getName`, `isInIsolation` deprecated/removed upstream (Moodle
  polyfills some); remove the calls or polyfill in your own testcase.
- Deprecated in 9.6 (4.4) and removed in PHPUnit 10: `assertObjectHasAttribute()`
  (use `assertObjectHasProperty()`), `assertClassHasAttribute()` & co., the
  `expectDeprecation()`/`expectError()`/`expectNotice()`/`expectWarning()`
  families (no replacement), `getMockClass()`.
- One plugin branch for 4.x and 5.x tests is usually possible but may need
  changes; the docs recommend a separate plugin branch per Moodle series.

## Sinks and logstores

Messages (`message_send()` is deferred until transactions close, so disable
test-in-transaction):

```php
$this->preventResetByRollback();
$sink = $this->redirectMessages();
// ... code that sends messages.
$messages = $sink->get_messages();
$messages = $sink->get_messages_by_component('mod_forum');                     // Since 4.4.
$messages = $sink->get_messages_by_component_and_type('core', 'messagecontactrequests');
```

Emails (`email_to_user()`):

```php
unset_config('noemailever');
$sink = $this->redirectEmails();
// ... code that sends email.
$this->assertCount(1, $sink->get_messages());
```

Events written to a logstore:

```php
$this->preventResetByRollback();
set_config('enabled_stores', 'logstore_standard', 'tool_log');
set_config('buffersize', 0, 'logstore_standard');
get_log_manager(true);
```

Since 5.2 (MDL-87716) the helper classes are namespaced and autoloaded; old
names keep working without debugging until 6.0:
`\phpunit_coverage_info` -> `\core\test\phpunit\coverage_info`,
`\phpunit_event_mock` -> `\core\test\phpunit\event_mock`,
`\phpunit_event_sink` -> `\core\test\phpunit\event_sink`,
`\phpunit_message_sink` -> `\core\test\phpunit\message_sink`,
`\phpunit_phpmailer_sink` -> `\core\test\phpunit\phpmailer_sink`,
`\phpunit_util` -> `\core\test\phpunit\phpunit_util`,
`\testing_util` -> `\core\test\testing_util`.

## Mocking time (Clock API)

Code that uses `\core\clock` (via DI) instead of `time()` can be tested with
mocked time. Mock the clock **before** fetching the service under test; the DI
container resets after every test.

```php
$clock = $this->mock_clock_with_frozen();          // Or (?int $time).
$clock->set_to(12345678);
$clock->bump();                                    // +1 second.
$clock->bump(4);
$clock = $this->mock_clock_with_incrementing(12345678);   // +1s per call.
$service = \core\di::get(\mod_myplugin\local\service::class);
```

Since 5.1 `mod_assign` uses the clock, so mocked time now works there.

## Coverage

### coverage.php

Since 4.0 a default applies to every plugin (include folders `classes`,
`tests/generator`; files `externallib.php`, `lib.php`, `locallib.php`,
`renderer.php`, `rsslib.php`). Add `tests/coverage.php` only to cover more;
paths are relative to the component root and merged with the defaults:

```php
return new class extends \core\test\phpunit\coverage_info {
    /** @var array The list of folders relative to the plugin root to include */
    protected $includelistfolders = ['classes', 'externallib.php'];
    /** @var array The list of files relative to the plugin root to include */
    protected $includelistfiles = [];
    /** @var array The list of folders relative to the plugin root to exclude */
    protected $excludelistfolders = [];
    /** @var array The list of files relative to the plugin root to exclude */
    protected $excludelistfiles = [];
};
```

(The docs still extend `phpunit_coverage_info` — the pre-5.2 name, still valid
until 6.0; use it if the plugin must also run on 5.1 or earlier.)

### Declaring covered code

Every test class MUST declare what it covers, at class level, for whole
classes where possible.

- Annotations (work on 4.5 and 5.x): fully-qualified names,
  `@covers \mod_myplugin\example`; for `@covers ::some_method` add
  `@coversDefaultClass \mod_myplugin\example` on the class.
- Attributes (5.0+ only, normal namespace resolution):

  ```php
  namespace mod_myplugin;

  #[\PHPUnit\Framework\Attributes\CoversClass(example::class)]
  final class example_test extends \advanced_testcase {
  }
  ```

  Plugins that test on 4.5 *and* 5.x cannot use attributes. `CoversClass` is
  the only PHPUnit attribute the Moodle docs show; for others see the PHPUnit
  11.4 manual.

## Best practice and misc

- Respect encapsulation: test public APIs; for protected code use
  inheritance; reflection is a last resort; never widen core visibility just
  for tests.
- Use `::class` constants in assertions and coverage — they work for classes
  that do not exist and survive class aliasing across Moodle versions.
- Long tests (>10 s, or hitting external servers) run only when long tests are
  enabled.
- Large data sets: `\advanced_testcase::createXMLDataSet()`,
  `createCsvDataSet()`.
- `$CFG` from `config.php` is intentionally not available in tests: define
  constants in `phpunit.xml` or `config.php` (e.g.
  `define('TEST_EXTERNAL_FILES_HTTP_URL', 'http://localhost/moodle-exttests');`
  with a local clone of `moodlehq/moodle-exttests`; LDAP tests use
  `TEST_AUTH_LDAP_*`, `TEST_ENROL_LDAP_*`, `TEST_LDAPLIB_*`).
- Since 5.3 a core PHPUnit test scans all routes in the `route\api` namespace
  and fails any route lacking `#[scopeset]` or `#[unscoped_resource]`.
