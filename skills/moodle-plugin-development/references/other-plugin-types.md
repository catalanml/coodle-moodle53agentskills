# Moodle 5.3 Plugin Types Reference

Breadth reference: the Moodle 5.3 plugin type catalogue, the 5.0+ plugin-type deprecation process, plus quick-start notes for less common types. Depth for major types (mod, block, qtype, etc.) lives in sibling reference files. Distilled from the Moodle 5.3 developer docs and the 5.0–5.3 UPGRADING notes.

## Naming rules

Machine names must match `/^[a-z](?:[a-z0-9_](?!__))*[a-z0-9]+$/` — start with a lowercase letter; lowercase letters, digits, underscores only; no hyphens; end with a letter or digit. Exception: activity modules (`mod`) may not contain underscores. Validation: `core_component::is_valid_plugin_name()` (since 5.2 it also rejects `qtype_random`). Invalid names are silently ignored.

## All plugin types (Moodle 5.3)

Paths are relative to `$CFG->dirroot`. Since 5.1 (MDL-83424) that is the `public/` directory of a git checkout, so `/mod` is `public/mod` in the repository.

**Note:** the type table on the 5.3 devdocs "Plugin types" page is unchanged since 4.5 and still lists Atto and omits the AI/SMS/communication/tiny types; the table below follows the UPGRADING notes and the per-type pages.

| Plugin type | Frankenstyle prefix | Path | Purpose |
| --- | --- | --- | --- |
| Activity modules | mod | /mod | Course activities (Forum, Quiz, Assignment) |
| Antivirus plugins | antivirus | /lib/antivirus | Virus-scan uploaded files via third-party tools (e.g. ClamAV) |
| Assignment submission | assignsubmission | /mod/assign/submission | Different forms of assignment submissions |
| Assignment feedback | assignfeedback | /mod/assign/feedback | Different forms of assignment feedback |
| Book tools | booktool | /mod/book/tool | Tools for the Book activity |
| Custom fields | customfield | /customfield/field | Custom field types (course and other custom fields) |
| Database fields | datafield | /mod/data/field | Data types for the Database activity |
| Database presets | datapreset | /mod/data/preset | Pre-defined templates for the Database activity (legacy) |
| LTI sources | ltisource | /mod/lti/source | Custom handlers to process LTI messages |
| File converters | fileconverter | /files/converter | Conversion between file types (e.g. .doc to PDF) |
| LTI services | ltiservice | /mod/lti/service | Implement IMS LTI services |
| ML backends | mlbackend | /lib/mlbackend | Prediction processors for the Analytics API |
| Forum reports | forumreport | /mod/forum/report | Reports in the forum activity |
| Quiz reports | quiz | /mod/quiz/report | Analyse quiz results / extend quiz behaviour |
| Quiz access rules | quizaccess | /mod/quiz/accessrule | Conditions on when/where quizzes can be attempted |
| SCORM reports | scormreport | /mod/scorm/report | Analysis of SCORM attempts |
| Workshop grading strategies | workshopform | /mod/workshop/form | Grading form type and submission grade calculation |
| Workshop allocation | workshopallocation | /mod/workshop/allocation | How submissions are assigned for assessment |
| Workshop evaluation | workshopeval | /mod/workshop/eval | Grading-grade calculation for assessments |
| Blocks | block | /blocks | Movable information displays/tools on pages |
| Question types | qtype | /question/type | Question types (multiple-choice, drag-and-drop, ...) |
| Question behaviours | qbehaviour | /question/behaviour | How students interact with questions during attempts |
| Question formats | qformat | /question/format | Import/export question definitions |
| Text filters | filter | /filter | Automatically convert/highlight posted text |
| Editors | editor | /lib/editor | Alternative text editors |
| TinyMCE plugins | tiny | /lib/editor/tiny/plugins | Subplugins for the TinyMCE editor |
| Enrolment plugins | enrol | /enrol | Control who is enrolled in courses |
| Authentication plugins | auth | /auth | External authentication sources |
| Admin tools | tool | /admin/tool | Site administration/maintenance utilities |
| Log stores | logstore | /admin/tool/log/store | Event log storage back-ends |
| Availability conditions | availability | /availability/condition | Restrict access to activities and sections |
| Calendar types | calendartype | /calendar/type | How dates are displayed |
| Messaging consumers | message | /message/output | Targets for messages/notifications (email, sms, ...) |
| Course formats | format | /course/format | Course layout of activities and blocks |
| Data formats | dataformat | /dataformat | Data export/download formats |
| User profile fields | profilefield | /user/profile/field | New data types for user profiles |
| Reports | report | /report | Site-wide data views for admins and teachers |
| Gradebook export | gradeexport | /grade/export | Export grades in various formats |
| Gradebook import | gradeimport | /grade/import | Import grades in various formats |
| Gradebook reports | gradereport | /grade/report | Display/edit grades in various layouts |
| Advanced grading | gradingform | /grade/grading/form | Grading interfaces (e.g. rubrics) |
| Webservice protocols | webservice | /webservice | Web service protocols (SOAP, REST, ...) |
| Repository plugins | repository | /repository | Connect to external file sources |
| Portfolio plugins | portfolio | /portfolio | External portfolio destinations |
| Search engines | search | /search/engine | Search engine back-ends for global search |
| Media players | media | /media/player | Pluggable media players |
| Plagiarism plugins | plagiarism | /plagiarism | External services to process submitted content |
| Cache stores | cachestore | /cache/stores | Cache storage back-ends |
| Cache locks | cachelock | /cache/locks | Cache lock implementations |
| Themes | theme | /theme | Change look via HTML/CSS |
| Local plugins | local | /local | Generic local customisations |
| Content bank types | contenttype | /contentbank/contenttype | Content types for the content bank |
| H5P libraries | h5plib | /h5p/h5plib | Versions of the H5P integration library |
| Question bank plugins | qbank | /question/bank | Extend question bank functionality |
| AI providers | aiprovider | /ai/provider | Interface between AI subsystem and external AI services |
| AI placements | aiplacement | /ai/placement | UX/UI for AI-backed functionality |
| Communication providers | communication | /communication/provider | Communication rooms (chat) for courses/instances |
| SMS gateways | smsgateway | /sms/gateway | SMS sending back-ends (4.5+) |

Retired or retiring types:

- **Atto plugins** (`atto`, `/lib/editor/atto/plugins`) — the Atto editor and all its standard subplugins were removed from core in 5.0 (MDL-83282); Atto remains available from the Plugins Database. Build editor features as `tiny` plugins.
- **MNet services** (`mnetservice`, `/mnet/service`) — plugin type deprecated in 5.0 (MDL-84311); `mnetservice_enrol` was removed from core. Do not create new ones.
- **Course reports** (`coursereport`, `/course/report`) — legacy, up to 2.1; use `report`.

To list the exact types known to a site, use `core_plugin_manager::instance()->get_plugin_types()`; `plugintype_name()` / `plugintype_name_plural()` read the `type_<type>` / `type_<type>_plural` lang strings (required since 5.1, MDL-84948).

## Plugin type deprecation (5.0+)

Since 5.0 (MDL-79843): core plugin types (in `lib/components.json`) and subplugin types (in a plugin's `db/subplugins.json`, see subplugins.md) can be deprecated. A deprecated type is **end-of-life immediately**: core excludes its plugins from core-plugin communication (hooks, callbacks, events, ...); only class autoloading and string resolution keep working, to support migration scripts. Only types that do not support subplugins can be deprecated.

Three stages, each a core version bump except the last:

1. Move the type from `plugintypes` to `deprecatedplugintypes` — installed plugins may stay but are end-of-life.
2. Move it to `deletedplugintypes` — site upgrade is **blocked** while any plugin of that type is present; admins must remove them first.
3. Remove the entry (and any empty object) from the config file.

```json
{
    "plugintypes": {
        "mod": "mod",
        "antivirus": "lib/antivirus"
    },
    "deprecatedplugintypes": {
        "aiplacement": "ai/placement"
    }
}
```

(Shape of `lib/components.json` from the docs' illustration — `aiplacement` is not actually deprecated.) For plugin authors: if your plugin is of a deprecated type, migrate or plan uninstall before the "deleted" stage reaches your users' release.

## AI providers (`aiprovider_*`) — /ai/provider

Rewritten for 5.0+; the 4.5 `settings.php` model no longer applies to providers.

- Provider class `classes/provider.php` extends `\core_ai\provider`. Golden rule: Providers and Placements never know about each other; all communication goes through the Manager (`core_ai\manager`). Providers are a thin wrapper: no data storage, no UI beyond configuration.
- **Provider instances:** an admin can create many instances of one provider plugin, each with its own configuration (e.g. a cheap model for summarising, a stronger one for generation). Instance config is available as `$this->config`.
- Required methods:
  - `get_action_list(): array` (static) — supported action classes, e.g. `\core_ai\aiactions\generate_text::class`, `generate_image::class`, `summarise_text::class`.
  - `is_provider_configured(): bool` — e.g. `return !empty($this->config['apikey']) && !empty($this->config['endpoint']);`. Defaults to `false`; UI only appears when `true`.
- For each supported action, implement `process_<action>` (e.g. `process_generate_image`) extending `\core_ai\process_base`, with a `process()` method that calls the external API and returns an Action Response object. Recommended: a shared `abstract_processor` extending `process_base`.
- **Instance settings form via hook:** `classes/hook_listener.php` with a static callback for `\core_ai\hook\after_ai_provider_form_hook`; return early unless `$hook->plugin === 'aiprovider_<name>'`, then add elements to `$hook->mform`. Register it in `db/hooks.php`:

```php
$callbacks = [
    [
        'hook' => \core_ai\hook\after_ai_provider_form_hook::class,
        'callback' => \aiprovider_openai\hook_listener::class . '::set_form_definition_for_aiprovider_openai',
    ],
];
```

- **Per-action settings:** override static `get_action_settings(string $action, array $customdata = []): action_settings_form|bool` returning a form in `classes/form/` that extends `core_ai\form\action_settings_form` and implements `definition()` (reads `$this->_customdata['actionconfig']['settings']`, `actionname`, `action`, `providerid`). Since 5.0 the base form adds Save/Cancel itself — remove `$this->add_action_buttons();`.
- **Pre-defined models (5.0, MDL-82980):** classes in `classes/aimodel/` extending `\core_ai\aimodel\base` with `get_model_name()` and `get_model_display_name()`; per-model settings via `add_model_settings(MoodleQuickForm $mform)`. Since 5.1 (MDL-84779) `has_model_settings()` is derived from the new `get_model_settings()`. Discover models with `\core_component::get_component_classes_in_namespace('aiprovider_x', 'aimodel')`, offer them in a `modeltemplate` select (plus a `custom` option), and add model settings through a `\core_ai\hook\after_ai_action_settings_form_hook` listener.
- **Rate limiting:** implemented by default in `\core_ai\provider`; override `is_request_allowed()` only to change the behaviour.
- **Errors (5.1, MDL-83147):** build error objects with `core_ai\error\factory::create($errorcode, $reason, $errorsource)`; add new error types by extending `core_ai\error\base` (example: `core_ai\error\ratelimit`).
- 5.3 (MDL-89123): `prompttokens` / `completiontokens` now live on `ai_action_register` and are written by `core_ai\manager::store_action_result()`; do not query them on the action child tables. 5.3 also adds a `courseid` column to `ai_action_register` and the `report_aiusage` plugin.
- Core examples: `aiprovider_openai`, `aiprovider_azureai`, `aiprovider_gemini` (5.3 adds the `gemini31flashimage` model).

## AI placements (`aiplacement_*`) — /ai/placement

- Placement class extends `\core_ai\placement`; must implement `get_action_list(): array`.
- Placements own access control: define capabilities in `db/access.php` (e.g. `aiplacement/editor:generate_image`); Providers never check access.
- Workflow: instantiate an action (`new \core_ai\aiactions\generate_image(contextid: $contextid, userid: $USER->id, prompttext: $prompttext, ...)`), then `$manager = \core\di::get(\core_ai\manager::class); $response = $manager->process_action($action);` — response is a `responses\response_base` instance; the placement checks success/failure.
- Typical files: `classes/placement.php`, `classes/external/*` (web services), `db/access.php`, `db/services.php`, `settings.php` using `core_ai\admin\admin_settingspage_provider`.
- 5.1 manager helpers (MDL-85738): `get_enabled_actions_in_course_module()`, `is_ai_tools_enabled_in_course()`, `is_action_enabled_in_context()`, `get_ai_fields_from_course_module()`. 5.2 (MDL-87658) adds an advisory page-level `moodle_page` property saying whether AI UI is preferred on a page (the note does not name it).
- Core examples: `aiplacement_courseassist`, `aiplacement_editor`.

## Antivirus (`antivirus_*`) — /lib/antivirus

- `classes/scanner.php`: class `scanner` in the plugin namespace, extends `\core\antivirus\scanner`.
- Compulsory methods: `is_configured(): bool`; `scan_file($file, $filename, $deleteinfected): void` — must throw `\core\antivirus\scanner_exception` if a virus is found.
- Other files: `version.php`, `settings.php` (e.g. `admin_setting_configexecutable` for the scanner path), `lang/en/antivirus_<name>.php`. Test with doubles for the external scanner (see `antivirus_clamav` unit tests).

## Availability conditions (`availability_*`) — /availability/condition

- `classes/condition.php`: extends `\core_availability\condition`. Key methods: `__construct($structure)`; `save()` (plain object, e.g. `(object)['type' => 'name', 'allow' => $this->allow]`); `is_available($not, \core_availability\info $info, $grabthelot, $userid)` (honour `$not`); `get_description($full, $not, \core_availability\info $info)`; `get_debug_string()`. "Permanent" conditions (like groups) also implement `is_applied_to_user_lists()` and `filter_user_list()`.
- `classes/frontend.php`: extends `\core_availability\frontend`; optional `get_javascript_strings()`, `get_javascript_init_params($course, $cm, $section)`, `allow_add($course, $cm, $section)`.
- The editing-form UI is still a YUI module (`yui/src/form/`) registering `M.availability_<name>.form` from `Y.Object(M.core_availability.plugin)` with `initInner`, `getNode(json)`, `fillValue(value, node)`, optional `fillErrors(errors, node)`; call `M.core_availability.form.update()` on change. Depends on `moodle-core_availability-form`.
- 5.2 removed `availability/amd/src/availability_more.js` and `availability/renderer.php` from core. Starting example: `availability_grouping`.

## Communication providers (`communication_*`) — /communication/provider

- Single required class `communication_feature` in `classes/communication_feature.php`, implementing: `\core_communication\communication_provider` (mandatory; `load_for_instance()` factory from a `core_communication\processor`), `user_provider` (`create_members()`), `room_chat_provider` (`create_chat_room()`, `update_chat_room()`, `delete_chat_room()` returning bool, `generate_room_url()`), `room_user_provider` (`add_members_to_room()`, `update_room_membership()`, `remove_members_from_room()`), `synchronise_provider` (`synchronise_room_members()`, run by `core_communication\task\synchronise_providers_task`).
- Changes are applied asynchronously via scheduled/ad-hoc tasks. Real example: `communication_matrix`. Docs still marked draft.

## Custom fields (`customfield_*`) — /customfield/field

- `classes/field_controller.php`: `field_controller` extending `\core_customfield\field_controller`; minimum `TYPE` constant (= plugin name) and `config_form_definition()`; optional `config_form_validation($formdata, $formfiles)`. Config element names must be `configdata[configname]`.
- `classes/data_controller.php`: `data_controller` extending `\core_customfield\data_controller`; minimum `datafield(): string` and `instance_form_definition()`. `datafield()` picks the column: `intvalue`, `decvalue`, `shortcharvalue` (255, indexed), `charvalue` (1333), `value` (text).
- Minimal files: the two controllers, `classes/privacy/provider.php`, `lang/en/customfield_<name>.php`, `version.php`. Core example: `customfield_checkbox`.
- 5.x: shared custom field categories (5.1 adds `component`/`area`/`itemid` to `customfield_data` and `shared` to `customfield_category`; `api::get_instance_fields_data()` takes them). 5.0 adds `\core_customfield\external\field_data_exporter`. Handler caching: 5.2 cached all handlers, 5.3 (MDL-89238) reverted to a new instance per `create()` unless the handler declares `protected const CACHE_HANDLER_INSTANCES = true;`.

## File converters (`fileconverter_*`) — /files/converter

- `classes/converter.php`: class `converter` implementing `\core_files\converter_interface`: `are_requirements_met()`; `start_document_conversion()` / `poll_conversion_status()` (set `\core_files\conversion::STATUS_IN_PROGRESS`, `STATUS_FAILED` or `STATUS_COMPLETE` and store the result with `store_destfile_from_string()` / `store_destfile_from_path()`); `supports($from, $to)`; `get_supported_conversion()`.
- Files: `classes/converter.php`, `settings.php`, `lang/`, `version.php`. Core example: `fileconverter_unoconv`.

## Log stores (`logstore_*`) — /admin/tool/log/store

- Custom event log storage back-ends. The 5.3 docs page is still a stub — use core `logstore_standard` / `logstore_database` as reference.

## TinyMCE plugins (`tiny_*`) — /lib/editor/tiny/plugins

- TinyMCE is the standard editor (since 4.1; Atto was removed from core in 5.0). JS in `amd/src/` (build with `grunt`):
  - `plugin.js` — entrypoint; default export is a `Promise` that awaits `getTinyMCE()` (from `editor_tiny/loader`), `getPluginMetadata(component, pluginName)` and setup, then calls `tinyMCE.PluginManager.add(pluginName, (editor) => {...})` and `resolve([pluginName, Configuration])`. All async work must finish before `PluginManager.add` — it is synchronous.
  - `commands.js` — `getSetup()` returning a per-editor setup function that calls `editor.ui.registry.addIcon/addButton/addToggleButton/addMenuItem/addToggleMenuItem`.
  - `options.js` — register/parse per-instance options; `common.js` — shared constants.
- `classes/plugininfo.php`: `plugininfo extends \editor_tiny\plugin`; optionally implements `\editor_tiny\plugin_with_configuration` with `get_plugin_configuration_for_context(context $context, array $options, array $fpoptions, ?\editor_tiny\editor $editor = null): array`.
- Since 5.0 (MDL-84353): new external function `editor_tiny_get_configuration`; subplugins provide configuration to it by implementing the `plugin_with_configuration_for_external` interface and/or overriding `is_enabled_for_external`.
- Since 5.3 (MDL-68037): the editor follows the page colour mode (skin chosen at setup; mode attribute repeated on the content iframe root so `content_css` can react). Style editor content and dialogs with mode-aware colours; toolbar icons load as isolated `<image>` documents, so `currentColor` cannot reach them — monochrome dark icons are lightened by a filter on `svg[data-buttonsource="moodle"]`. 5.3 also enables the bundled Accordion and List Styles (advlist) plugins by default.
- Generate a skeleton with `tool_pluginskel` (`php admin/tool/pluginskel/cli/generate.php tiny_example.yml`), then `grunt`. Behat testing: see subplugins.md.

## SMS gateways (`smsgateway_*`) — /sms/gateway (since 4.5)

- `classes/gateway.php`: class `gateway` extends `\core_sms\gateway`: `send(message $message): message` (return `$message->with(status: $status)`), `get_send_priority(message $message): int`. Since 5.0, `\core_sms\gateway::truncate_message()` truncates content to the gateway's length limit.
- `classes/hook_listener.php` + `db/hooks.php`: listen to `after_sms_gateway_form_hook` to add gateway config fields (guard with `if ($hook->plugin !== 'smsgateway_<name>') return;`).
- Used by e.g. MFA SMS verification. Production example: `smsgateway_aws`.

## Assignment subplugins (`assignsubmission_*`, `assignfeedback_*`)

- `mod_assign` hosts submission (`/mod/assign/submission`) and feedback (`/mod/assign/feedback`) subplugins — `public/mod/assign/...` in a 5.1+ checkout. Main class lives in `locallib.php`.
- Naming/table limits: table names max 53 chars; plugin name max 36 chars for `assignsubmission_`, 38 for `assignfeedback_`.
- Both extend `assign_plugin` (`$this->assignment`, `get_config()`/`set_config()`). Since 5.0 (MDL-83440) override `assign_plugin::settings_validation` to validate the assignment settings form on save.
- Submission plugins extend `assign_submission_plugin`: `get_name()` (required), `get_settings(MoodleQuickForm $mform)` / `save_settings()`, `get_form_elements()`, `save(stdClass $submission, stdClass $data)`, `is_empty()`, `submission_is_empty()`, `get_files()`, `get_file_areas()`, `view_summary()`, `view()`, `copy_submission()`, `format_for_log()`, `delete_instance()`, legacy `can_upgrade()`/`upgrade_settings()`/`upgrade()`. `settings.php` should include a `default` setting. Reference: `assignsubmission_onlinetext`.
- Since 5.0 (MDL-84387): `submission_summary_for_messages(stdClass $submission): array` returns `[$text, $html]` summarising the submission for confirmation messages; render via templates so themes can override, and remember other plugins' summaries appear alongside:

```php
#[\Override]
public function submission_summary_for_messages(stdClass $submission): array {
    global $PAGE;

    $onlinetextsubmission = $this->get_onlinetext_submission($submission->id);
    if (!$onlinetextsubmission || !$onlinetextsubmission->onlinetext) {
        return ['', ''];
    }

    $renderer = $PAGE->get_renderer('mod_assign');
    $templatecontext = ['wordcount' => count_words($onlinetextsubmission->onlinetext)];
    return [
        // Mustache strips trailing whitespace, but a newline is wanted at the end.
        $renderer->render_from_template('assignsubmission_onlinetext/notification_text', $templatecontext) . "\n",
        $renderer->render_from_template('assignsubmission_onlinetext/notification_html', $templatecontext),
    ];
}
```

- Feedback plugins extend `assign_feedback_plugin`: `get_name()`, `get_settings()` / `save_settings()`, `get_form_elements_for_user($grade, MoodleQuickForm $mform, stdClass $data, $userid)`, `is_feedback_modified(stdClass $grade, stdClass $data)`, `save(stdClass $grade, stdClass $data)`, `view_summary()`, `view()`, `is_empty()`, `get_file_areas()`, `delete_instance()`, gradebook `format_for_gradebook()` / `text_for_gradebook()`, quick grading `supports_quickgrading()` / `get_quickgrading_html($userid, $grade)`, batch operations `get_grading_batch_operations()` / `grading_batch_operation($action, $users)`. Since 5.3 (MDL-88688) `get_grading_batch_operation_details()` may return a `'confirmationyes'` key for the confirm button. Reference: `assignfeedback_file`.
- Other 5.x changes: `mod/assign/renderer.php` and `ASSIGN_ATTEMPT_REOPEN_METHOD_NONE` removed (5.2); privacy tests extend `\mod_assign\tests\provider_testcase` (5.0); `mod_assign` uses the core clock (5.1); overrides moved to an `override_manager` class and `mod_assign\event\marker_updated` is no longer triggered — observe `marker_added` / `marker_removed` (5.3).

## Machine learning backends (`mlbackend_*`) — /lib/mlbackend

- Process Analytics API datasets for training, prediction and evaluation; communication with Moodle is file-based. Main class `classes/processor.php`:
  - **Predictor** (required): `is_ready()`, `clear_model($uniqueid, $modelversionoutputdir)`, `delete_output_dir($modeloutputdir, $uniqueid)`
  - **Classifier**: `train_classification($uniqueid, \stored_file $dataset, $outputdir)`, `classify(...)`, `evaluate_classification($uniqueid, $maxdeviation, $niterations, \stored_file $dataset, $outputdir)`
  - **Regressor** (not implemented by core): `train_regression()`, `estimate()`, `evaluate_regression()`
- Since 5.0 (MDL-84107): `mlbackend_php` was removed from core (available at github.com/moodlehq/moodle-mlbackend_php); `mlbackend_python` (requires the `moodlemlbackend` pip package) is the only core backend and the default `predictionsprocessor`.
