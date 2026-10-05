# More Moodle 5.3 Subsystems

Compact reference for additional Moodle 5.3 core subsystems — Editor, Plagiarism, xAPI, Favourites, Time, Communication, AI, SMS, Badges, Question, Availability (and legacy conditional activities), Analytics, Check, and OpenTelemetry — from the consumer (plugin developer) perspective, per the Moodle 5.3 developer docs and UPGRADING notes.

## Editor API

Docs are a draft stub; core content is in `lib/editorlib.php`. Normally you do not use this API directly — include an `editor` element in a Moodle form and setup is automatic.

To attach the user's preferred editor to an existing HTML textarea:

```php
$editor = editors_get_preferred_editor(FORMAT_HTML);
$editor->use_editor('mytextareaid', [
    'context' => $context,
    'enable_filemanagement' => false,   // Disable the file management plugin.
    'autosave' => false,                // Disable autosave.
], [
    'return_types' => FILE_EXTERNAL,
]);
```

**Note:** The Atto editor was removed from core in 5.0 (it is available from the Plugins Database); TinyMCE (`editor_tiny`) is the core editor. The 5.3 Editor API page is stale and still documents Atto-specific `atto:toolbar` options — do not rely on them. The docs list no Tiny-specific `use_editor()` options.

## Plagiarism API

Docs are a draft stub with no code — an overview of the flow only. The Plagiarism API lets Moodle code send user-submitted content to plagiarism-prevention systems. Supported modules: Assignment, Forum, Workshop.

Flow: (1) when plagiarism tools are enabled, participating modules get extra settings; (2) a user submits content/files; (3) an event is triggered with user/module/submission details; (4) event handlers in the plagiarism plugin process it; (5) hooks display returned information to student and teacher.

Removed in 5.2 (MDL-87423, final deprecation): `plagiarism_get_file_results()` and `plagiarism_update_status()`.

## xAPI Subsystem (core_xapi)

Not a full LRS — a library so any plugin can handle xAPI statements ("Actor executes Verb on Object") and, since 4.2, states. Webservices: `core_xapi_statement_post`, `core_xapi_post_state`, `core_xapi_get_state`, `core_xapi_delete_state` (each takes a component frankenstyle name).

Key classes: `\core_xapi\handler` (extend it in your plugin as `\PLUGINNAME\xapi\handler`), `\core_xapi\iri`, `\core_xapi\local\statement`, `\core_xapi\local\state`, item classes in `\core_xapi\local\statement\` (`item_agent`, `item_group`, `item_verb`, `item_activity`, `item_definition`, `item_result`, `item_score`, `item_context`, `item_attachment`).

```php
use core_xapi\local\statement;
use core_xapi\local\statement\item_agent;
use core_xapi\local\statement\item_verb;
use core_xapi\local\statement\item_activity;

$statement = new statement();
$statement->set_actor(item_agent::create_from_user($USER));
$statement->set_verb(item_verb::create_from_id('bake'));           // Plain string or full IRI.
$statement->set_object(item_activity::create_from_id('cake'));
```

Group actors: `item_group::create_from_group(groups_get_group($groupid))`.

Plugin adaptation: extend `\core_xapi\handler` and override `statement_to_event(statement $statement): core\event\base` (must check permission and return a valid Moodle event) and optionally `supports_group_actors(): bool`. Every statement MUST map to a Moodle event. `statement` accessors: `get_user()`, `get_all_users()`, `get_group()`, `get_verb_id()`, `get_activity_id()`, `minify()`, `get_actor()/get_verb()/get_object()/get_context()/get_result()/get_timestamp()/get_attachments()`.

States: `new state(item_agent::create_from_user($USER), item_activity::create_from_id('cake'), $stateid, $statedata, $registration)`; handler overrides `validate_state(state $state): bool`. Storage defaults to the `xapi_states` table (customisable via `core_xapi\state_store`). Helpers in `core_xapi\api`: `remove_states_from_component(string $component)`, `execute_state_cleanup()`.

From JS: `ajax.call([{methodname: 'core_xapi_statement_post', args: {component: 'mod_h5p', requestjson: JSON.stringify(statement)}}])`. With an array of statements the response is an array of booleans (per-statement success).

## Favourites API (core_favourites, "Starred")

A favourite is identified by `component` (frankenstyle), `itemtype` (unique within the component), `itemid`, `contextid` (e.g. course context for courses, user context for conversations).

```php
$ufservice = \core_favourites\service_factory::get_service_for_user_context($usercontext);
$favourite = $ufservice->create_favourite('core_course', 'courses', $course->id, $coursecontext);
$ufservice->delete_favourite('core_course', 'courses', $course->id, $coursecontext);
```

Read methods: `count_favourites_by_type($component, $itemtype, \context $context = null): int`, `find_favourites_by_type($component, $itemtype, $limitfrom = 0, $limitnum = 0): array`, `favourite_exists($component, $itemtype, $itemid, \context $context): bool`, `get_favourite(...): favourite`.

Embed favourite info in your own SQL (LEFT JOIN; add `AND favouriteid IS NOT NULL` to keep only favourites):

```php
[$favsql, $favparams] = $ufservice->get_join_sql_by_type('core_course', 'courses', 'favalias', 'c.id');
$sql = "SELECT c.id, c.name, favalias.id AS favouriteid
          FROM {course} c
       $favsql
         WHERE c.category = :category";
$courses = $DB->get_records_sql($sql, ['category' => 3] + $favparams);
```

## Time API

All times are stored as Unix timestamps. Distinguish System Time (server, tasks) from User Time (per-user timezone display). Use `\core\clock` (PSR-20, via DI — preferred over `time()` / `new DateTime()`; see `hooks-di.md`) and `core_date` (DateTimeZone handling).

```php
$clock = \core\di::get(\core\clock::class);
$now = $clock->now();          // \DateTimeImmutable (modifiers return NEW objects).
$ts = $clock->time();          // Current Unix timestamp.
$tomorrow = $clock->now()->modify('+1 day')->getTimestamp();

echo userdate($ts, get_string('strftimedaydate', 'core_langconfig'));
```

Display functions (user time): `userdate` (`strftime()`-style formats), `usergetmidnight`, `usertimezone`, `make_timestamp`. System time helpers: `format_time`, `dst_offset_on`, `find_day_in_month`, `days_in_month`, `dayofweek`. Legacy, avoid: `usergetdate`, `usertime`. Use `MINSECS`/`HOURSECS` constants only for UI option lists — never date arithmetic (no DST/leap handling).

Since 5.0 (MDL-83873), human-readable renderers `core_calendar\output\humandate` and `core_calendar\output\humantimeperiod` ("Today"/"Tomorrow", near-deadline styling) replace `calendar_format_event_time()` etc.:

```php
$renderer = $PAGE->get_renderer('core', 'output');
echo $renderer->render(\core_calendar\output\humandate::create_from_timestamp($ts));
echo $renderer->render(\core_calendar\output\humantimeperiod::create_from_timestamp($ts, $ts + HOURSECS));
```

## Communication API (core_communication)

Experimental — enable `enablecommunicationsubsystem` under Site administration > Development > Experimental. Manages provider rooms (e.g. Matrix) and members. All actions are queued as ad-hoc tasks, except member deletion when the user is being deleted. Use only the public API `\core_communication\api`; core actions reach it via hooks (`communication\hook_listener`, registrations in `lib/db/hooks.php`).

```php
$communication = \core_communication\api::load_by_instance(
    context: $context,
    component: 'core_course',
    instancetype: 'coursecommunication',
    instanceid: $course->id,
    provider: 'communication_matrix',   // Optional; defaults to the enabled provider.
);
$communication->create_and_configure_room(
    communicationroomname: $course->fullname,
    avatar: $courseimage ?: null,
    instance: $course,
);
```

Other methods (all queue tasks): `update_room(?int $active, ?string $communicationroomname, ?\stored_file $avatar, ?\stdClass $instance)`, `delete_room()`, `configure_room_and_membership_by_provider(...)`, `add_members_to_room(array $userids, bool $queue = true)`, `update_room_membership(...)`, `remove_members_from_room(...)`, `remove_all_members_from_room()`, `show_communication_room_status_notification()`, `form_definition(\MoodleQuickForm $mform, ...)` plus `set_data()`.

## AI Subsystem (core_ai, since 4.5)

Components: **Placements** (UI plugins), **Actions** (`generate_text`, `generate_image`, `summarise_text` in `\core_ai\aiactions`; 5.3 UPGRADING also references an `explain_text` action table), **Providers** (bridges to external AI), **Pre-defined Models** (used by providers, not part of the subsystem itself), and the **Manager** (`\core_ai\manager`). Placements and providers never talk directly — everything goes via the Manager.

Consumer entry point: instantiate an action (constructor takes a `contextid` plus action-specific args) and call `\core_ai\manager::process_action(base $action): responses\response_base`. The manager picks a provider, logs the call, and returns a `response_<action_name>` object; read it with `$response->get_response_data()` (array).

Custom actions extend `\core_ai\aiactions\base`, implement `__construct()` (with `contextid`) and `store(response_base $response): int` (persist to table `ai_action_<action_name>`). Responses implement `set_response_data(array)` / `get_response_data(): array`.

AI User Policy — placements must gate usage on acceptance: `\core_ai\manager::get_user_policy_status(int $userid): bool`, `\core_ai\manager::user_policy_accepted(int $userid, int $contextid): bool`. Webservices: `core_ai_get_policy_status`, `core_ai_set_policy_status`.

Changes since 4.5:

- 5.0: providers with pre-defined models extend `\core_ai\aimodel\base`; hook `\core_ai\hook\after_ai_action_settings_form_hook` adds action-settings fields; `\core_ai\form\action_settings_form` adds Save/Cancel itself — remove `$this->add_action_buttons();` from provider forms.
- 5.1: errors via `core_ai\error\factory::create($errorcode, $reason, $errorsource)` (extend `core_ai\error\base`, see `core_ai\error\ratelimit`); manager helpers `get_enabled_actions_in_course_module()`, `is_ai_tools_enabled_in_course()`, `is_action_enabled_in_context()`, `get_ai_fields_from_course_module()`; `aimodel\base::has_model_settings()` now derives from `get_model_settings()`.
- 5.3: `prompttokens`/`completiontokens` moved from the action child tables to `ai_action_register` (written by `core_ai\manager::store_action_result()`) — update direct queries; `ai_action_register` gains `courseid`; new `report_aiusage` plugin.

## SMS Subsystem (core_sms, since 4.5)

`\core_sms\manager` (fetch via DI) sends messages, fetches them, checks status, and lists gateway instances (plugin type `smsgateway`).

```php
$message = \core\di::get(\core_sms\manager::class)->send(
    recipientnumber: '+61987654321',
    content: 'This is the content of the message',
    component: 'mod_example',
    messagetype: 'demonstrationmessage',
    recipientuserid: $user->id,
    issensitive: false,   // True = content never stored in the log; cannot be async.
    async: false,
);
```

Message length: up to 480 UTF-8 characters per the docs. Since 5.0 (MDL-84342) over-length messages are **truncated** before sending by `\core_sms\gateway::truncate_message()` (gateways may change the default maximum or override the method); the `MESSAGE_OVER_SIZE` status is gone. **Note:** the 5.3 page still also says over-length messages are "immediately rejected" — that sentence is stale.

Always set accurate `component` and `messagetype` for reporting. Fetch and check status (`\core_sms\message`, status is a PHP enum):

```php
$manager = \core\di::get(\core_sms\manager::class);
$message = $manager->get_message(['id' => $id]);
$messages = $manager->get_messages(['recipientuserid' => $userid]);
$message->is_failed(); $message->is_in_progress(); $message->is_sent(); $message->description();
```

Gateways: `$manager->get_gateway_records()` (optionally filtered), `$manager->get_enabled_gateway_instances()`. Hooks `before_gateway_deleted` / `before_gateway_disabled` are dispatched before a gateway is removed/disabled — listen and call `$hook->stop_propagation()` if your component still uses `$hook->gateway->id`.

## Badges API (core_badges; reorganised in 5.2)

Manages site/course badges and their export to Open Badges compliant "backpacks". Moodle supports **Open Badges 2.0 and 2.1** (same schema) and the Badgr-style backpack API; Open Badges 3.0 support is planned. Since 5.2 (MDL-82147 / MDL-85621) JSON exporters are organised per OB version:

```
local/backpack/ob/v2p0/
├── assertion_exporter.php
├── badge_exporter.php
├── issuer_exporter.php
├── recipient_exporter.php
└── revoked_assertion_exporter.php
```

JSON is served from `badges/json/`, selecting the version by URL: `/badges/json/assertion.php?b=<assertionhash>&obversion=v2p0`. The docs give the exporter folder only as `local/backpack/ob/v2p0/` (relative to the badges component).

Class moves (old names keep working):

- 5.0: `badges/lib/bakerlib.php` → `core_badges\png_metadata_handler`; image author fields (`imageauthorname/email/url`) removed.
- 5.1: `core_badges_observer` → `core_badges\event\observer`; `core_badges\backpack_api` gains Canvas Credentials helpers (`get_providers()`, `get_regions()`, `get_region_url()`, …).
- 5.2: `\badge_award_selector_base` → `\core_badges\award_selector_base`; `\badge_potential_users_selector` → `\core_badges\potential_award_selector`; `\badge_existing_users_selector` → `\core_badges\existing_award_selector`; `process_manual_award()` / `process_manual_revoke()` → `\core_badges\award_manager::process_manual_award()` / `::process_manual_revoke()`. Deprecated: `core_badges_assertion` → `\core_badges\achievement_credential`; `badges_get_default_issuer()` (no replacement). The `create_issued_badge` generator now returns the issued badge.
- 5.3: `badge_message_from_template()` accepts an extra `$user` argument to interpolate user-name placeholders.

## Question Subsystem

Two subcomponents:

**Question engine** — runs questions when users attempt them (main consumer: mod_quiz). Plugin types: question types (`qtype_`) and behaviours (`qbehaviour_`, e.g. `qbehaviour_deferredfeedback`, `qbehaviour_immediatefeedback`, `qbehaviour_interactive`). Key classes: `question_engine` (entry point), `question_attempt` (one user's attempt at one question), `question_usage_by_activity` (groups related attempts), `question_state`, `question_display_options`, `core_question_renderer`.

**Question bank** — plugin types `qbank_` (bank features), `qformat_` (import/export), plus `qtype_` editing forms.

5.x changes that affect plugin code:

- Since 5.0, course and category contexts no longer hold question categories; since 5.1 questions exist only at the activity-module context (shared banks) and `question_edit_contexts` checks only the given context. `question_make_default_categories()` is deprecated — use `question_get_default_category()`.
- 5.0: qtypes defining their own tables with ID fields must implement `restore_qtype_*_plugin::define_excluded_identity_hash_fields()` (restore de-duplication hashing).
- 5.1: activities using question *set* references must also call `backup_question_set_reference_trait::annotate_set_reference_bank_entries()` in their backup step.
- 5.2: use `\core_question\versions::get_next_version()` (inside a transaction) — never read the new `question_bank_entries.nextversion` column directly; `get_next_version()` in questionlib.php is deprecated. qbank filter conditions using DB IDs override `restore_filtercondition()`.
- 5.3: bulk action classes (`bulk_action_base`) must implement `get_action_icon()`.

## Availability API (conditional access)

Controls access to activities/sections (the Completion API tracks *progress*). Normally automatic: `require_login` enforces it (since 5.2 it redirects users to a restricted page when restrictions are visible) and forms include the controls. Gated by `$CFG->enableavailability`; custom conditions are `availability_` plugins.

```php
$modinfo = get_fast_modinfo($course);           // Optionally pass a userid.
$cm = $modinfo->get_cm($cmid);
if ($cm->uservisible) {
    // User can access the activity.
} else if ($cm->availableinfo) {
    // Shown greyed-out with an explanation (HTML).
}
// Sections: same properties on section_info. An activity check already includes its section.

// Filter a user list to those who can see the activity:
$info = new \core_availability\info_module($cm);
$filtered = $info->filter_user_list($users);
```

Set restrictions programmatically (JSON in `course_modules.availability`):

```php
$restriction = \core_availability\tree::get_root_json([
    \availability_group\condition::get_json($group->id),
]);
// Or a date range:
$dates = [
    \availability_date\condition::get_json(">=", $start),
    \availability_date\condition::get_json("<", $end),
];
$restriction = \core_availability\tree::get_root_json($dates, \core_availability\tree::OP_AND, [true, true]);

$DB->set_field('course_modules', 'availability', json_encode($restriction), ['id' => $cmid]);
rebuild_course_cache($course->id, true);
```

**Legacy Conditional activities API** (`condition_info` in `lib/conditionlib.php`) — the `apis/core/conditionalactivities` page documents this pre-2.7 API, replaced by the Availability API since 2.7. Read-only knowledge for old code (`is_available()`, `get_full_information()`, `add_completion_condition()`, …); never use it in new code. Removed in 5.2: `availability/amd/src/availability_more.js` and `availability/renderer.php`.

## Analytics API

Prediction models = target + indicators (+ analysis interval). Since 5.0 analytics is **disabled by default on new installs**. Key classes: `\core_analytics\model` (evaluate/train/predict; not extended); analysers extend `\core_analytics\local\analyser\by_course` or site-wide (implement `get_all_samples()`); targets extend `\core_analytics\local\target\base`; indicators extend `\core_analytics\local\indicator\base` (values in -1..1); analysis intervals extend `\core_analytics\local\time_splitting\base`. Declare default models in the plugin's `db/analytics.php` (not db/install.php).

```php
$target = \core_analytics\manager::get_target('\mod_yours\analytics\target\spammer_users');
$indicator1 = \core_analytics\manager::get_indicator('\mod_yours\analytics\indicator\posts_straight_after_new_account_created');
$indicators = [$indicator1->get_id() => $indicator1];
$model = \core_analytics\model::create($target, $indicators,
    '\core\analytics\time_splitting\single_range', '\mlbackend_python\processor');
$model->enable();  // Models start disabled.
```

Removed: chat/survey analytics (5.0); `\core_analytics\manager::add_builtin_models()` (5.2).

## Check API

Runtime checks (security/status/performance/environment), reports at e.g. `/report/status/index.php`, CLI `php admin/cli/checks.php`. Result states: N/A, OK, INFO, UNKNOWN, WARNING, ERROR, CRITICAL.

```php
// mod/myplugin/classes/check/foobar.php
namespace mod_myplugin\check;

use core\check\check;
use core\check\result;

class foobar extends check {
    public function get_action_link(): ?\action_link {
        return new \action_link(new \moodle_url('/mod/myplugin/dosomething.php'),
            get_string('sitepolicies', 'admin'));
    }

    public function get_result(): result {
        if (some_check()) {
            return new result(result::ERROR, get_string('check_foobar_error', 'mod_myplugin'), $details);
        }
        return new result(result::OK, get_string('check_foobar_ok', 'mod_myplugin'), $details);
    }
}
```

Register via a lib.php callback named for the report type: `function mod_myplugin_security_checks(): array { return [new \mod_myplugin\check\foobar()]; }` (also `_status_checks`, `_performance_checks`). Multiple instances need a unique `get_id()`. Defer expensive details to a custom `result::get_details()`. Reuse in settings: `$page->add(new admin_setting_check('antivirus/checkantivirus', new \core\check\environment\antivirus()));`

Since 5.2 (MDL-87648) a check can return several results: implement `get_results(): array` returning `\core\check\result` objects. (The 5.3 Check API page still documents only `get_result()`.)

## OpenTelemetry (since 5.2)

Telemetry support lives **outside Moodle core**, in the Composer package `moodlehq/moodle-package-otel`. You also need the PHP OpenTelemetry extension (`opentelemetry-php-instrumentation`), a transport (HTTP included, or `open-telemetry/transport-grpc`), an exporter (e.g. `open-telemetry/exporter-otlp`), a Collector, and a backend (SigNoz, Jaeger, Zipkin, Prometheus…).

Included: a root span for every web request and CLI command, plus instrumentations `moodlelms.cronlistener` (scheduled/adhoc tasks), `moodlelms.eventlistener` (events), `moodlelms.externalapilistener` (web service calls).

Configure via php.ini or environment variables. Disable all Moodle instrumentation (other OTel integrations keep working) or specific ones:

```ini
OTEL_PHP_DISABLED_INSTRUMENTATIONS=moodlelms
OTEL_PHP_DISABLED_INSTRUMENTATIONS=moodlelms.cronlistener,moodlelms.eventlistener
```

Custom instrumentation: either a standard OTel PHP instrumentation package, or a Moodle-specific Composer package that:

- declares package type `moodle-package-otelhook`;
- defines listeners implementing `Moodlehq\MoodlePackageOtel\Instrumentation\MoodleListenerInterface` (optionally using `Moodlehq\MoodlePackageOtel\Instrumentation\MoodleListenerTrait`);
- defines a `\Namespace\Instrumentation\ListenersDescriber` class implementing `Moodlehq\MoodlePackageOtel\Instrumentation\ListenersDescriberInterface` that lists those listeners.
