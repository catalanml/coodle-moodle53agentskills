# Hooks, DI, Clock and Deprecation APIs (Moodle 5.3)

PSR-14 hooks (replacement for lib.php one-to-many callbacks), the PSR-11 DI container, the PSR-20 clock, and the deprecation attribute/helper — per the Moodle 5.3 developer docs and UPGRADING notes.

## Hooks API (since 4.3)

Replaces lib.php callbacks implemented via `get_plugins_with_function()`, `plugin_callback()`, or `component_callback()`. PSR-14 mapping: Event = Hook, Listener = Hook callback, Dispatcher/Listener Provider = Hook manager.

Naming rules (4.4+): `before`/`after` only as prefixes; subsystem hooks live in that subsystem's namespace (e.g. `\core_course\hook\before_course_visibility_changed`). Hooks placed in any `*\hook\*` namespace are auto-discovered for the admin "Hooks overview" page; otherwise you must provide a `\core\hook\discovery_agent` implementation in class `[component]\hooks`.

### Defining a hook class

Any final PHP class, normally in `some_component\hook\*`. Describe it with attributes (or implement `\core\hook\described_hook` with `get_hook_description()` / `get_hook_tags()`). Use constructor property promotion with readonly properties for data:

```php
// mod/activity/classes/hook/installation_finished.php
namespace mod_activity\hook;

#[\core\attribute\label('Hook dispatched at the very end of installation of mod_activity plugin.')]
#[\core\attribute\tags('installation')]
final class installation_finished {
    public function __construct(
        public readonly string $version,
    ) {
    }
}
```

### Dispatching

```php
$hook = new \mod_activity\hook\installation_finished('1.0');
\core\di::get(\core\hook\manager::class)->dispatch($hook);
```

(Pre-4.4 style, harder to test: `\core\hook\manager::get_instance()->dispatch($hook);`)

### Registering callbacks: db/hooks.php

```php
// local/stuff/classes/local/hook_callbacks.php
namespace local_stuff\local;

use mod_activity\hook\installation_finished;

class hook_callbacks {
    public static function activity_installation_finished(installation_finished $hook): void {
        if (during_initial_install()) {
            return;
        }
        if (!get_config('local_stuff', 'version')) {
            return;
        }
        // Do something...
    }
}
```

```php
// local/stuff/db/hooks.php
$callbacks = [
    [
        'hook' => mod_activity\hook\installation_finished::class,
        'callback' => [\local_stuff\local\hook_callbacks::class, 'activity_installation_finished'],
        'priority' => 500,
    ],
];
```

Execution order: callbacks run by priority in **descending** order. Default priority is `100`; higher numbers run earlier, lower numbers (down to 0) later. If a hook needs a specific ordering, its description should document which priorities to use.

Notes:

- Array callback notation requires 4.4+; for 4.3 compatibility use string notation `'some\class\name::static_method'`.
- Hooks may be dispatched at any time, **including during site install and upgrade** (e.g. `before_http_headers`), when the database may not exist. Guard with `during_initial_install()`, `get_config('your_pluginname', 'version')`, or `isset($CFG->upgraderunning)` — failing to do so can break the web install/upgrade page.
- The legacy lib.php callback system did NOT call callbacks during install/upgrade, so callbacks ported to hooks may need guards they never had before.
- Registrations are cached: bump the plugin version or purge caches after editing `db/hooks.php`. Since 5.2 (MDL-87107) the hook manager caches in localcache instead of MUC.
- Admins can disable/override callbacks in config.php:

```php
$CFG->hooks_callback_overrides = [
    \mod_activity\hook\installation_finished::class => [
        'test_otherplugin\\hook_callbacks::activity_installation_finished' => ['disabled' => true],
    ],
];
```

### Stoppable hooks

Implement `Psr\EventDispatcher\StoppableEventInterface` (`isPropagationStopped(): bool`); a callback calls e.g. `$hook->stop()` and later callbacks are skipped.

### Migrating a legacy lib.php callback

1. Create the hook class with `#[\core\attribute\hook\replaces_callbacks('after_config')]` (or implement `core\hook\deprecated_callback_replacement` with `get_deprecated_plugin_callbacks(): array`).
2. At the emit site, pass `$migratedtohook = true` as the 4th argument to `get_plugins_with_function('after_config', 'lib.php', true, true)`, then dispatch the hook immediately after the legacy loop.
3. The hook manager emits deprecation debugging for plugins still using the lib.php callback. A plugin may ship BOTH the legacy callback and the hook callback (for multi-branch support); the legacy one is auto-ignored when a hook callback exists.

Removed in 5.1 (MDL-80327): `\core\hook\manager::is_deprecated_plugin_callback()` — use `\core\hook\manager::get_hooks_deprecating_plugin_callback()`.

### Core hooks added in 5.x (from UPGRADING notes)

- `\core_course\hook\before_course_viewed` (5.0) — replaces the deprecated `externservercourse.php` override of the course view page.
- `after_role_switched` (5.0) — dispatched when a user switches role in a course (namespace not stated in the notes; check the Hooks overview page).
- `\core_ai\hook\after_ai_action_settings_form_hook` (5.0) — AI providers add action-settings form elements.
- `\core_group\hook\after_group_membership_added` / `\core_group\hook\after_group_membership_removed` (5.2).
- `before_email_to_user` (5.3, MDL-69724) — emitted by `email_to_user()`; subscribers can modify content, add headers, or add block reasons (any reason stops the email).
- `\core_user\hook\extend_user_menu` (changed 5.3) — call `add_menu_item()` with a `\core_user\output\user_action_menu\base` item; `add_navitem()` is deprecated.

### Testing hooks

```php
\core\di::set(
    \core\hook\manager::class,
    \core\hook\manager::phpunit_get_instance([
        'test_plugin1' => __DIR__ . '/fixtures/.../hooks_fixture.php', // A $callbacks array file.
    ]),
);
```

## Dependency Injection (since 4.4)

PSR-11 container (PHP-DI) accessed via `\core\di`. Most classes resolve by class name with no configuration.

```php
// Outside classes / legacy code:
$client = \core\di::get(\core\http_client::class);
$manager = \core\di::get(thing_manager::class);

// Inside classes: constructor injection (promoted readonly properties recommended).
class thing_manager {
    public function __construct(
        protected readonly \moodle_database $db,
    ) {
    }

    public function get_things(): array {
        return $this->db->get_records('example_things');
    }
}
```

`\core\di::get()` always returns the same **shared** instance for a given entry.

### Building a new instance each time: `\core\di::make()` (since 5.3, MDL-89528)

`make()` resolves the entry again on every call — for a class, a new instance each time (factory behaviour). Use it to replace `new $classname()` in legacy entry points and factories.

```php
$thing1 = \core\di::make(my_thing::class);
$thing2 = \core\di::make(my_thing::class); // A different instance.

// Force specific constructor arguments; the rest are resolved by the container.
$renderer = \core\di::make(\core\output\core_renderer::class, [
    'page' => new \moodle_page(),
    'target' => \RENDERER_TARGET_CLI,
]);
```

### Attribute-based injection (since 5.3, MDL-89528)

Inject a dependency directly onto a property with PHP-DI's `#[\DI\Attribute\Inject]`:

```php
class example_class {
    #[\DI\Attribute\Inject]
    private \core\formatting $formatter;
}

$example1 = \core\di::get(example_class::class);  // Shared instance; property populated.
$example2 = \core\di::make(example_class::class); // New instance; property populated.
```

- The class and injected properties must NOT be `readonly` — they are populated after construction.
- Populated only when the instance is built through the container (`get()` or `make()`), not with `new`.
- Recommended for routed controllers and for classes whose constructor signature cannot easily change. Otherwise prefer constructor injection: it keeps dependencies explicit and the class usable without the container (e.g. in unit tests).

### Rules and testing

- Do not inject `\Psr\Container\ContainerInterface` itself; always go through `\core\di`.
- `\core\di::reset_container()` exists but is for unit-test-like situations only (the container is auto-reset between unit tests; resetting an in-use container is dangerous).
- Mock in tests with `\core\di::set(\core\http_client::class, $mockclient);` — set BEFORE fetching the service that uses it.
- Since 5.3 (MDL-88580) password and auth-plugin helpers are DI-resolvable classes: `\core\authentication\password` (`validate()`, `hash()`, `update()`, …) and `\core\authentication` (`plugin_exists()`, `is_enabled()`, `get_plugin()`, …); the global functions remain as wrappers.

Configure complex definitions (rare, legacy) via the `\core\hook\di_configuration` hook: register a callback in `db/hooks.php`, then in the listener call `$hook->add_definition(id: complex_client::class, definition: function (\moodle_database $db): complex_client { ... });`.

## Clock (\core\clock, since 4.4)

PSR-20 clock; use instead of `time()` so tests can mock time.

```php
$clock = \core\di::get(\core\clock::class);
$clock->now();   // \DateTimeImmutable
$clock->time();  // Unix timestamp (Moodle convenience method)

// Preferred: constructor injection.
class post {
    public function __construct(
        protected readonly \core\clock $clock,
        protected readonly \moodle_database $db,
    ) {
    }
}
```

Unit testing helpers on the standard testcase (mock BEFORE fetching your service; DI resets after each test):

```php
$clock = $this->mock_clock_with_incrementing();        // +1s per call; optional start time arg.
$clock = $this->mock_clock_with_frozen();              // Fixed time; optional time arg.
$clock->set_to(12345678);                              // Frozen clock: set time.
$clock->bump(4);                                       // Frozen clock: advance 4s (default 1).
```

Custom clocks: any class implementing `\core\clock`, injected via `\core\di::set(\core\clock::class, $clock)`.

## Deprecation API (since 4.4)

`#[\core\attribute\deprecated]` attribute + `\core\deprecation` helper. Complements (does not replace) the `@deprecated` phpdoc annotation. Applies to classes/traits/interfaces/enums, enum cases, global functions, class constants/properties/methods.

```php
/** @deprecated since 4.3 */
#[\core\attribute\deprecated('random_bytes', since: '4.3')]
function random_bytes_emulate($length) {
    \core\deprecation::emit_deprecation_if_present(__FUNCTION__); // Debugging notice.
    return random_bytes($length);
}

// Final deprecation -> emit_deprecation_if_present throws an exception.
#[\core\attribute\deprecated(replacement: 'Events API', since: '2.3', final: true)]
function add_to_log() {
    \core\deprecation::emit_deprecation_if_present(__FUNCTION__);
}

// Full argument set:
#[\core\attribute\deprecated(
    replacement: '\core\example::do_something',
    since: '4.3',
    reason: 'No longer required',
    mdl: 'MDL-12345',
)]
public function do_something(): void {}
```

Since 5.1 (MDL-84531) the `$replacement` constructor parameter defaults to null and can be omitted (e.g. `#[\core\attribute\deprecated(since: '5.1', reason: 'Removed')]`).

Since 5.1 (MDL-85897) use `\core\deprecation::emit_deprecation()` where a deprecation is *known* to occur — it emits the notice, and raises debugging if no deprecation attribute is found (unlike `emit_deprecation_if_present()`, which silently does nothing):

```php
#[\core\attribute\deprecated(replacement: 'new_method', since: '5.1')]
public function deprecated_method(): void {
    \core\deprecation::emit_deprecation([self::class, __FUNCTION__]);
}
```

Inspection helpers:

```php
\core\deprecation::is_deprecated(\core\param::INTEGER);            // Enum case.
\core\deprecation::is_deprecated(\core\task\manager::class);       // Class name.
\core\deprecation::is_deprecated(new \moodle_url('/example/'));    // Instance.
\core\deprecation::is_deprecated([\moodle_url::class, 'out']);     // Method.
\core\deprecation::emit_deprecation_if_present($value);            // Notice/exception if deprecated.
```

Related attribute: since 5.2 (MDL-87799) `\core\attribute\description` (formerly `\core_sms\description`) represents a language string in code attributes.

### JavaScript deprecations: core/deprecated (since 5.2, MDL-87867)

Same semantics as the PHP attribute; respects the `developerdebug` setting.

```javascript
import emitDeprecation from 'core/deprecated';

const deprecatedMethod = () => {
    emitDeprecation('core/example::deprecatedMethod', {
        mdl: 'MDL-12345',
        since: '5.0',
        replacement: 'myNewMethod',
        reason: 'To support some new technology',
        // final: true,  // After the deprecation period: console Error instead of warning.
    });
};
```

An `emit` option can suppress the UI message for non-final deprecations. Items can be silenced via the `jsdeprecationignorelist` config setting (array of first-argument strings).
