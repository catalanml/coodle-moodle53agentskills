# Moodle 5.3: Events API and Task API

Compact reference for defining/triggering events, observing them via `db/events.php`, and creating scheduled and adhoc tasks. Task API distilled from the Moodle 5.3 developer docs; the Events API has no moodledev.io page (the 5.3 docs still link to the legacy `docs.moodle.org/dev/Events_API` wiki), so that section follows the wiki plus the 5.0–5.3 `UPGRADING.md` notes.

## Events API

The Events API lets you define events with payload data, fire them whenever you like, and define observers that react to them. It is the recommended form of inter-plugin communication and forms the basis of logging in Moodle.

### Defining an event class

Each event class extends `\core\event\base` and lives in its own autoloaded file matching the class name: `plugindir/classes/event/something_happened.php` (namespace `plugintype_pluginname\event`).

Naming convention: `\component\event\object_verb` — the object in singular form, ending with a past-tense verb (last underscore-separated word). Examples: `\core\event\course_completed`, `\mod_assign\event\submission_commented`. Common approved verbs: `created`, `updated`, `deleted`, `viewed`, `submitted`, `graded`, `completed`, `commented`, `downloaded`, `uploaded`, `assigned`, `unassigned`, `locked`, `unlocked`, `started`, `ended`, `attempted`.

Required initialisation — `init()` sets the *static* data (identical for every instance of the class):

```php
protected function init() {
    $this->data['crud'] = 'c'; // One of: c(reate), r(ead), u(pdate), d(elete).
    $this->data['edulevel'] = self::LEVEL_PARTICIPATING;
    $this->data['objecttable'] = 'table_name';
}
```

`edulevel` constants:

- `LEVEL_TEACHING` (1) — performed by someone (typically a teacher) and has a teaching value.
- `LEVEL_PARTICIPATING` (2) — performed by a user, related to their learning experience.
- `LEVEL_OTHER` (0) — any other action without educational value.

Descriptive methods an event class typically implements:

```php
public function get_name() { }           // Localised event name (get_string).
public function get_description() { }    // Non-localised human-readable description.
public function get_url() { }            // moodle_url relevant to the event.
```

Inherited accessors: `get_data()` (standardised event data array), `get_context()`.

### Static vs instance data

- **Static data**: set in `init()` (`crud`, `edulevel`, `objecttable`) — the same for all event instances of this class.
- **Variable (instance) data**: passed to `create()` per instance — context, objectid, `other`, etc. The `other` payload must be JSON-serialisable data only (no objects; avoid floating-point numbers, see MDL-46701).

### Triggering an event

Create an instance via the static `create()` factory, then call `trigger()`:

```php
$event = \mod_myplugin\event\something_happened::create([
    'context' => $context,
    'objectid' => $record->id,
    'other' => ['sample' => $value],
]);
$event->trigger();
```

Valid properties for `create()`:

| Property | Required | Notes |
|---|---|---|
| `context` / `contextid` | Yes | Context in which the event occurred |
| `objectid` | Optional | Must pair with an `objecttable` set in `init()` |
| `userid` | No | Defaults to the current user |
| `courseid` | No | For course-level contexts |
| `relateduserid` | No | The user the action relates to |
| `other` | No | Serialisable payload only |
| `anonymous` | No | Defaults to 0 |

Record snapshots let observers access records (including deleted ones) without extra DB queries:

```php
$event->add_record_snapshot('table_name', $record);
// In an observer:
$record = $event->get_record_snapshot($tablename, $id);
```

For backup/restore, map IDs stored in the event:

```php
public static function get_objectid_mapping() {
    return ['db' => 'assign_grades', 'restore' => 'grade'];
}

public static function get_other_mapping() {
    $othermapped = [];
    $othermapped['assignid'] = ['db' => 'assign', 'restore' => 'assign'];
    return $othermapped;
}
```

Use `\core\event\base::NOT_MAPPED` when mapping is impossible.

### Observers: db/events.php

A plugin listens for events by declaring observers in its `db/events.php` (a component file refreshed during upgrade):

```php
$observers = [
    [
        'eventname' => '\core\event\course_module_created',
        'callback'  => '\plugintype_pluginname\event\observer\course_module_created::store',
        'priority'  => 1000,
    ],
];
```

Full set of observer keys:

```php
$observers = [
    [
        'eventname'   => '\core\event\sample_executed',
        'callback'    => 'core_event_sample_observer::observe_one',
    ],
    [
        'eventname'   => '\core\event\sample_executed',
        'callback'    => 'core_event_sample_observer::external_observer',
        'priority'    => 200,
        'internal'    => false,
    ],
    [
        'eventname'   => '\*',                       // Observe all events.
        'callback'    => 'core_event_sample_observer::observe_all',
        'includefile' => null,                        // Optional, relative to dirroot.
        'internal'    => true,
        'priority'    => 9999,
    ],
];
```

- `eventname`: fully-qualified event class name, or `\*` for all events.
- `callback`: PHP callable (usually a static method on an autoloaded observer class).
- `includefile`: optional file to require, relative to dirroot (default none).
- `priority`: optional, default 0; higher priority observers are notified first.
- `internal`: optional, default true. `false` = external observer, only called after a successful database transaction commit (buffered during transactions).

Observer behaviour and rules:

- An observer receives the event data but cannot modify it or prevent the action.
- Observer definitions are cached — purge caches after adding or changing observers.
- Events triggered inside an observer are queued and processed after all current observers complete.
- Exceptions thrown by an observer are captured and logged; notification of remaining observers continues.
- Do not subscribe to another plugin's events without declaring a dependency on that plugin (component communication policy). Events must not be observed by core or core subsystems; event subscriptions are intended for communication from core to plugins (local plugins are the ideal candidate).

### Event changes in 5.0–5.3 (UPGRADING notes)

- **Removed in 5.0** (MDL-84593): `\core\event\course_module_instances_list_viewed` (final deprecation).
- **Deprecated in 5.1**: `\mod_book\event\course_module_instance_list_viewed`, and the same event in `mod_folder`, `mod_imscp`, `mod_page`, `mod_resource` and `mod_url`. Use `\core\event\course_resources_list_viewed` instead. Update observers listening for the old names.
- **Removed in 5.2** (MDL-87427): the file `public/lib/classes/event/content_viewed.php`, i.e. `\core\event\content_viewed` no longer exists; do not extend it.
- **Testing classes renamed in 5.2** (autoloaded; old names kept without debugging until 6.0): `\phpunit_event_sink` is now `\core\test\phpunit\event_sink`, `\phpunit_event_mock` is now `\core\test\phpunit\event_mock`.

## Task API

Tasks are background activities in two forms:

- **Scheduled tasks** — run regularly on a schedule the administrator can change, same configuration each time.
- **Adhoc tasks** — queued by developers, can have multiple concurrent instances, each with its own custom data and the ability to run as a different user.

Any operation taking more than a few seconds is a candidate. Tasks only run as often as cron runs; run cron at least once per minute. Legacy cron (`cron.php` / `modname_cron()`) was removed in Moodle 4.3 — everything must be a task. Blocking tasks were removed in Moodle 4.4; `\core\task\task_base::set_blocking()` / `is_blocking()` were finally deprecated in 5.0 and removed in 5.2 (MDL-87426), so any remaining call is a fatal error.

Also removed in 5.2 (MDL-87423, from `lib/deprecatedlib.php`): `cron_run()`, `cron_run_scheduled_tasks()`, `cron_run_adhoc_tasks()`, `cron_run_inner_scheduled_task()`, `cron_run_inner_adhoc_task()`, `cron_set_process_title()`, `cron_trace_time_and_memory()`, `cron_prepare_core_renderer()` and `cron_setup_user()`. `\core\task\manager::ensure_adhoc_task_qos()` was removed too.

### Dependency injection in task classes (since 5.3, MDL-89528)

Task instances are now constructed through the DI container, so both scheduled and adhoc task classes can use constructor injection:

```php
namespace mod_example\task;

class discussion_formatter extends \core\task\adhoc_task {
    public function __construct(
        private \core\formatting $formatter,
    ) {
    }

    public function execute() {
        // Use $this->formatter.
    }
}
```

In tests, replace a dependency with `\core\di::set()` before the manager builds the task:

```php
$clock = $this->createMock(\core\clock::class);
$clock->method('time')->willReturn(1234567890);
\core\di::set(\core\clock::class, $clock);

// The manager creates the task via the container, which supplies the mock clock.
$task = \core\task\manager::adhoc_task_from_record((object) [
    'classname' => \mod_example\task\do_something::class,
]);
$task->execute();
```

### Scheduled tasks

To create one: (1) a class extending `\core\task\scheduled_task`, (2) an entry in the plugin's `db/tasks.php`, (3) bump the plugin version number.

Class goes in `classes/task/` of the plugin:

```php
namespace mod_example\task;

class do_something extends \core\task\scheduled_task {

    /** Return the task's name as shown in admin screens. */
    public function get_name() {
        return get_string('dosomething', 'mod_example');
    }

    /** Execute the task. */
    public function execute() {
        // Call your own API.
    }
}
```

To run even when the plugin is disabled (rare, e.g. enrol cleanup), override `get_run_if_component_disabled()` to return `true`.

### db/tasks.php

```php
$tasks = [
    [
        'classname' => 'mod_example\task\do_something', // Required: FQCN of the task.

        // Date/time fields (all optional strings; default '*'):
        // month, day, dayofweek, hour, minute.
        'month' => '*',
        'day' => '*',
        'hour' => 'R',    // R = fixed random value chosen at install/upgrade.
        'minute' => 'R',

        // Optional: default the task to disabled until an admin enables it.
        'disabled' => 1,
    ],
];
```

Field format: `<fieldlist> := <range>(/<step>)(,<fieldlist>)` where `<range>` is `*`, an int, `min-max`, or `R`. Multiple values allowed, e.g. `'minute' => '5, 35'`. Defaults when unspecified: every month/day/day-of-week/hour/minute. If either `day` or `dayofweek` is `*`, the other field is used; otherwise the soonest value wins.

### Adhoc tasks

Create a class extending `\core\task\adhoc_task` in `classes/task/` (no `db/tasks.php` entry), then queue it:

```php
namespace mod_example\task;

class do_something extends \core\task\adhoc_task {
    public function execute() {
        // Call your own API.
    }
}
```

**Custom data** — any JSON-serialisable content, set with `set_custom_data(mixed $content)` and read with `get_custom_data(): mixed`. Recommended pattern: only call `set_custom_data()` from a factory method on the task class:

```php
class do_something extends \core\task\adhoc_task {

    public static function instance(int $id, string $status): self {
        $task = new self();
        $task->set_custom_data((object) [
            'id' => $id,
            'status' => $status,
        ]);
        return $task;
    }

    public function execute() {
        $data = $this->get_custom_data();
        mtrace($data->id);
        mtrace($data->status);
    }
}
```

**Queueing**:

```php
$taskid = \core\task\manager::queue_adhoc_task($task);
```

Since 5.3 (MDL-86422) `queue_adhoc_task()` returns the ID of the newly queued task, or of the matching existing task when duplicate checking is on (including a matching task that is already running). It returns `false` if the task's component is deprecated or the task could not be queued due to a DML error. Before 5.3 a found duplicate returned `false`.

- **Run as a specific user** (tasks otherwise run as the CLI admin user): `$task->set_userid($someuser->id);` — also best done in the factory method.
- **Future run time**: `$task->set_next_run_time($futuretime);` (unix timestamp) before queueing. Tasks are not guaranteed to run at that time, but will not run before it.
- **Prevent duplicates**: `$taskid = \core\task\manager::queue_adhoc_task($task, true);` reuses an existing queued task matching all of `classname`, `component`, `customdata`, `userid` and returns its ID. Since 5.3, if the matching task has exhausted its retries it is reactivated with the supplied task's retry count, its failure delay is cleared, and it is scheduled to run immediately.
- **Reschedule instead**: same duplicate detection, but queue the task or update the next run time of the matching task:

  ```php
  $task->set_next_run_time($futuretime);
  \core\task\manager::reschedule_or_queue_adhoc_task($task);
  ```

- `\core\task\manager::task_is_scheduled()` is deprecated in 5.3 (MDL-86422); call `\core\task\manager::get_queued_adhoc_task_record()` directly instead.
- If a task carries a subset of data also processed by other instances of the same task type, store that data in a DB table, not in custom data.

### Failures and retries

If a task (scheduled or adhoc) fails, just throw an exception — the task system catches it, logs it, and retries after 1 minute, backing off on repeated failures up to a maximum of 24 hours between attempts.

If you catch exceptions yourself (e.g. so one failing course does not block processing the others in a loop), you become responsible for logging details (`mtrace_exception` helps) and for deciding whether the overall run should end by throwing, so a partial failure is recorded as a failure.

Adhoc retry controls (since 4.4):

```php
$task->set_attempts_available(2);   // Default 12 retries if not set.
\core\task\manager::queue_adhoc_task($task);
$task->get_attempts_available();    // Remaining attempts.

// Disable retrying entirely by overriding in the task class:
public function retry_until_success(): bool {
    return false;
}
```

**Soft retry without failing** (since 5.3, MDL-79763 — UPGRADING only, not yet in the devdocs pages): call `set_soft_retry_delay()` from inside an adhoc task's `execute()` to ask for the task to be retried later via `\core\task\manager::adhoc_task_delayed()` **without** marking it as failed. Pass `null` for automatic exponential backoff (based on time elapsed since the task first started, capped at 24 hours) or a positive integer for an explicit delay in seconds. Signatures: `public function set_soft_retry_delay(?int $softretrydelay = null): void` and `public function get_soft_retry_delay(): ?int` on `\core\task\adhoc_task` (verified in MOODLE_503_STABLE source); the number of remaining attempts is still decremented, so a task cannot soft-retry forever. Companion: `is_adhoc_task_delayed()`.

```php
public function execute() {
    // The work cannot be done yet (for example an external resource is not ready).
    $this->set_soft_retry_delay(null); // Or e.g. 300 for five minutes.
}
```

Since 5.1, `adhoc_task_failed()` and `scheduled_task_failed()` accept a new optional parameter to skip log finalisation when called from a separate task.

### Other task mechanics

- **Static caches**: after a task that performed many changes, call `\core\task\manager::clear_static_caches()` at the end of the task; the task runner exits and the next task starts in a fresh process with cleared static caches. Particularly relevant to enrolment-related tasks.
- **Security**: background tasks run as the cron user. For access checks, do them as the user they relate to: pass that user's ID to `require_capability()`, or for adhoc tasks use `set_userid()` when queueing. The 5.3 task page still recommends `cron_setup_user()` for switching user in scheduled tasks, but that function was removed in 5.2 (MDL-87423); the docs do not name its replacement, so prefer passing the user ID.
- **Output**: safe to use the Output API in cron tasks since 3.5. Use `mtrace()` liberally inside `execute()` for debugging/log visibility.
- **CLI debugging**: running a scheduled task from the command line can hide errors behind a misleading lock message; pass `--showdebugging` for details.
- **Scheduled next run time**: since 5.3, `\core\task\manager::set_scheduled_task_nextruntime()` returns a bool; `false` means the scheduled task is already running and the time was not updated.
- **Progress bars for tasks** (5.0, MDL-81714): `\core\task\stored_progress_task_trait` gained `initialise_stored_progress()`, which calls `store_pending()` so the progress bar shows a "pending" state while the task is queued. Display it with the new `\core\output\task_indicator` component (see "Task indicator" in the component library).
- **Admin tools**: task log viewer (adhoc + scheduled run logs), scheduled task manager (edit schedules), and "Tasks running now".
