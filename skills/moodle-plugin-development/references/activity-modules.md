# Moodle 5.3 Activity Modules (mod plugins)

Compact reference for developing `mod_[modname]` activity module plugins against Moodle 5.3, distilled from the official developer docs and the 5.0–5.3 UPGRADING notes.

Activity modules are the primary delivery method for learning content. Plugin type is `mod`, frankenstyle name `mod_[modname]`, located in `/mod/[modname]/` relative to `$CFG->dirroot`. Since 5.1 (MDL-83424) the code lives under `public/`, so in a git checkout the path is `public/mod/[modname]/`. Replace `[modname]` with your module's name throughout.

Removed from core in 5.0: `mod_chat` and `mod_survey` (MDL-82457). Do not depend on them.

## Folder layout

Typical layout (example based on the core `folder` plugin; not every file is mandatory):

```
mod/[modname]/
├── backup/moodle2/
│   ├── backup_[modname]_activity_task.class.php
│   ├── backup_[modname]_stepslib.php
│   ├── restore_[modname]_activity_task.class.php
│   └── restore_[modname]_stepslib.php
├── classes/
│   ├── completion/custom_completion.php
│   ├── courseformat/overview.php   (Activities overview integration, 5.0+)
│   ├── event/                      (course_module_viewed.php, ...)
│   ├── output/courseformat/activitybadge.php (optional, 4.3+)
│   ├── privacy/provider.php
│   └── search/activity.php
├── db/
│   ├── access.php              (capability definitions - required)
│   ├── install.xml             (database schema - required)
│   ├── upgrade.php, events.php, services.php, install.php
├── lang/en/[modname].php       (language strings - required)
├── lib.php                     (required callbacks)
├── mod_form.php                (instance create/edit form - required)
├── index.php                   (instance list - required; redirect to overview in 5.0+)
├── view.php                    (activity view page - required)
├── pix/monologo.svg            (activity icon)
├── settings.php
├── tests/
└── version.php                 (required)
```

## version.php

Plugin metadata: version number, dependencies, minimum Moodle version, maturity. Must never contain `require_once` or any file inclusion — it must remain a plain data file with no side effects.

```php
defined('MOODLE_INTERNAL') || die();

$plugin->version = 2026100500;          // YYYYMMDDXX of this plugin release.
$plugin->requires = 2026100500;         // Moodle 5.3.0.
$plugin->supported = [503, 503];        // Moodle 5.3 branch.
$plugin->component = 'mod_[modname]';
$plugin->maturity = MATURITY_STABLE;
$plugin->release = '1.0';
```

**Note:** set `requires` to the `$version` in core `version.php` of the oldest release whose APIs you call (5.0+ once you ship `classes/courseformat/overview.php` or call `redirect_to_overview_page()`). Moodle 5.3.0 is `2026100500` (core `public/version.php` on `MOODLE_503_STABLE`); for other releases read `$version` from core `version.php` — the devdocs do not list build numbers.

## db/install.xml — required table and fields

Moodle requires a table whose name exactly matches the plugin name (plugin `certificate` → table `certificate`). Required fields:

| Field | Properties | Keys / Notes |
| --- | --- | --- |
| `id` | `INT(10)`, auto sequence | primary key |
| `course` | `INT(10)` | foreign key to `course` table |
| `name` | `CHAR(255)` | user-specified name of the activity instance |
| `timemodified` | `INT(10)` | timestamp of last modification |
| `intro` | `TEXT` | standard activity description (see `FEATURE_MOD_INTRO`) |
| `introformat` | `INT(4)` | format of the `intro` field |

Schema changes after release go in `db/upgrade.php` upgrade steps.

## db/access.php — capabilities

Two capabilities are **required**: `mod/[modname]:addinstance` (may create an instance) and `mod/[modname]:view` (may view an instance).

```php
$capabilities = [
    'mod/[modname]:addinstance' => [
        'riskbitmask' => RISK_XSS,
        'captype' => 'write',
        'contextlevel' => CONTEXT_COURSE,
        'archetypes' => [
            'editingteacher' => CAP_ALLOW,
            'manager' => CAP_ALLOW,
        ],
        'clonepermissionsfrom' => 'moodle/course:manageactivities',
    ],
    'mod/[modname]:view' => [
        'captype' => 'read',
        'contextlevel' => CONTEXT_MODULE,
        'archetypes' => [
            'guest' => CAP_ALLOW,
            'student' => CAP_ALLOW,
            'teacher' => CAP_ALLOW,
            'editingteacher' => CAP_ALLOW,
            'manager' => CAP_ALLOW,
        ],
    ],
];
```

## lang/en/[modname].php

```php
$string['pluginname'] = 'The name of your activity';
$string['modulename'] = 'Your activity';
$string['modulename_summary'] = 'One-paragraph introduction shown in the activity chooser.';
$string['modulename_help'] = 'Detailed description: Key features, Ways to use it.';
$string['modulename_tip'] = 'Optional best-practice tips shown in the activity chooser.';
```

Since 5.2 (MDL-87117): the activity chooser displays two new optional strings when defined — `modulename_summary` (concise single-paragraph intro) and `modulename_tip` (supplemental tips). `modulename_help` holds the detailed description, structured with **Key features** and **Ways to use it** sections, as core modules now do.

If `FEATURE_QUICKCREATE` is supported, also provide a `quickcreatename` string used as the name of quick-created instances.

## lib.php — required callbacks

```php
function [modname]_add_instance($instancedata, $mform = null): int;   // Creation form submit; returns new instance id.
function [modname]_update_instance($instancedata, $mform): bool;      // Edit form submit.
function [modname]_delete_instance($id): bool;                        // Confirmed deletion; remove all instance data.
```

`lib.php` is one of the older parts of Moodle; functionality is gradually migrating to class-based calls.

### [modname]_supports — feature flags

Return `true`/`false`, `null` if unknown, or a string for the purpose features. Constants live in `lib/moodlelib.php`. Notable features:

- `FEATURE_GROUPS`, `FEATURE_GROUPINGS` — group/grouping support
- `FEATURE_MOD_INTRO` — uses the standard intro/introformat description
- `FEATURE_SHOW_DESCRIPTION` — description can show on the course page
- `FEATURE_COMPLETION`, `FEATURE_COMPLETION_TRACKS_VIEWS`, `FEATURE_COMPLETION_HAS_RULES` — completion support (see below)
- `FEATURE_GRADE_HAS_GRADE` — the activity provides a grade; also enables completion 'on grade'
- `FEATURE_BACKUP_MOODLE2` — implements Moodle 2 backup/restore
- `FEATURE_QUICKCREATE` — `[modname]_add_instance()` can create an instance with defaults, without the form (used by the `core_courseformat_new_module` webservice)
- `FEATURE_NO_VIEW_LINK` — no view page (like Label); removes the link from course page and navigation
- `FEATURE_CAN_DISPLAY` — whether the module type can be displayed on the course page; types without it (e.g. Question bank) are skipped by linear navigation (MDL-88003)
- `FEATURE_MOD_PURPOSE` — mandatory main purpose; sets the activity chooser category and the icon colour
- `FEATURE_MOD_OTHERPURPOSE` — since 5.1 (MDL-85598): optional secondary purpose, used as an extra activity chooser category
- `FEATURE_GROUPMEMBERSONLY` — deprecated in 5.3 and unsupported; remove it from plugin code

```php
function [modname]_supports(string $feature) {
    return match ($feature) {
        FEATURE_GROUPS => true,
        FEATURE_GROUPINGS => true,
        FEATURE_MOD_INTRO => true,
        FEATURE_COMPLETION_TRACKS_VIEWS => true,
        FEATURE_GRADE_HAS_GRADE => true,
        FEATURE_BACKUP_MOODLE2 => true,
        FEATURE_SHOW_DESCRIPTION => true,
        FEATURE_MOD_PURPOSE => MOD_PURPOSE_COLLABORATION,
        FEATURE_MOD_OTHERPURPOSE => MOD_PURPOSE_COMMUNICATION, // Optional, 5.1+.
        default => null,
    };
}
```

Purposes: `MOD_PURPOSE_ADMINISTRATION`, `MOD_PURPOSE_ASSESSMENT` (Assignment, Quiz, Workshop), `MOD_PURPOSE_COLLABORATION` (Database, Forum, Glossary, Wiki), `MOD_PURPOSE_COMMUNICATION` (Choice, Feedback...), `MOD_PURPOSE_INTERACTIVECONTENT` (H5P, Lesson, SCORM...), `MOD_PURPOSE_CONTENT` (Book, File, Page, URL...), `MOD_PURPOSE_OTHER`.

### Activity chooser callbacks

Implement `[modname]_get_course_content_items()` in lib.php to add items to the activity chooser. Each content item ID must be unique to your component, stable, and an integer (see `lti_get_course_content_items()`). For admin recommendations, also implement `[modname]_get_all_content_items()`.

Changes since 4.5:
- 5.1 (MDL-86337): all activity chooser code (templates, JS, output) moved from `core_course` to `core_courseformat`. The chooser now loads items per section via `core_courseformat_get_section_content_items`; `core_course_get_course_content_items` is deprecated.
- 5.1 (MDL-86036): `core_course\local\entity\content_item` has a `gradable` property. 5.3 (MDL-87373): items can be disabled with the `disabled` and `disabledreason` parameters.
- 5.1 (MDL-85597): if your plugin implements `[plugintype]_[pluginname]_custom_chooser_footer`, remove any `modal.setFooter(...)` call from the footer AMD module; the chooser footer now manages that. Render custom content with `course/templates/local/activitychooser/footer.mustache`.

## mod_form.php — instance create/edit form

Class must be named `mod_[modname]_mod_form` extending `moodleform_mod` (a current exception to class autoloading rules, MDL-74472). Excerpt:

```php
require_once($CFG->dirroot . '/course/moodleform_mod.php');

class mod_certificate_mod_form extends moodleform_mod {
    public function definition() {
        $mform = $this->_form;

        $mform->addElement('header', 'general', get_string('general', 'certificate'));
        $mform->addElement('text', 'name', get_string('certificatename', 'certificate'), ['size' => '64']);
        $mform->setType('name', PARAM_TEXT);
        $mform->addRule('name', null, 'required', null, 'client');

        // Standard course module elements (visibility, groups, completion, etc.).
        $this->standard_coursemodule_elements();
        $this->add_action_buttons();
    }

    public function validation($data, $files) {
        $errors = [];
        if (empty($data['name'])) {
            $errors['name'] = get_string('errornoname', 'certificate');
        }
        return $errors;
    }
}
```

Use `data_preprocessing(&$defaultvalues)` to set field defaults before display.

## view.php and index.php

`view.php` (required) is linked as `/mod/[modname]/view.php?id=CMID` (**course module id**):

```php
require(__DIR__ . '/../../config.php');

$id = required_param('id', PARAM_INT);
[$course, $cm] = get_course_and_cm_from_cmid($id, '[modname]');
$instance = $DB->get_record('[modname]', ['id' => $cm->instance], '*', MUST_EXIST);
```

Since 5.0 (MDL-83892), `cm_info::get_instance_record()` returns the activity table record without a direct `$DB` call and caches it per request.

`index.php` (required). Since 5.0 the course **Activities** page replaces per-module index pages; the 5.3 template simply redirects there:

```php
require_once('../../config.php');

$courseid = required_param('id', PARAM_INT);

\core_courseformat\activityoverviewbase::redirect_to_overview_page($courseid, '[modname]');
```

After redirecting, you may deprecate any method, output or renderer that only served the old index page.

## Activities overview page (since 5.0, MDL-83872)

The course "Activities" tab shows a table of all activities (name, section, completion, grade, due date...). Customise your columns with `mod_[modname]\courseformat\overview` in `classes/courseformat/overview.php`, extending `core_courseformat\activityoverviewbase`. An empty subclass already shows name, section and completion.

```php
namespace mod_[modname]\courseformat;

use core\output\action_link;
use core\output\local\properties\button;
use core\output\local\properties\text_align;
use core\output\pix_icon;
use core\url;
use core_courseformat\activityoverviewbase;
use core_courseformat\local\overview\overviewitem;

class overview extends activityoverviewbase {
    #[\Override]
    public function get_extra_overview_items(): array {
        // Indexed by item shortname; return null items to skip them.
        return ['submitted' => $this->get_extra_submitted_overview()];
    }

    #[\Override]
    public function get_due_date_overview(): ?overviewitem {
        $duedate = 0; // Your plugin logic to get the due date.
        if (empty($duedate)) {
            // If every activity returns a null value, the column is not displayed.
            return new overviewitem(name: get_string('duedate'), value: null, content: '-');
        }
        return new overviewitem(name: get_string('duedate'), value: $duedate, content: userdate($duedate));
    }

    #[\Override]
    public function get_actions_overview(): ?overviewitem {
        if (!has_capability('mod/[modname]:viewreports', $this->context)) {
            return null;
        }
        $content = new action_link(
            url: new url('/mod/[modname]/report.php', ['id' => $this->cm->id]),
            text: get_string('viewreport', 'mod_[modname]'),
            attributes: ['class' => button::SECONDARY_OUTLINE->classes()],
        );
        return new overviewitem(
            name: get_string('reports', 'mod_[modname]'),
            value: null,
            content: $content,
            textalign: text_align::CENTER,
        );
    }

    private function get_extra_submitted_overview(): ?overviewitem {
        if (!has_capability('mod/[modname]:submit', $this->context)) {
            return null;
        }
        $submitted = false; // Plugin logic here.
        $content = $submitted
            ? new pix_icon('i/checkedcircle', alt: get_string('submitted', 'mod_[modname]'))
            : '-';
        return new overviewitem(name: get_string('submitted', 'mod_[modname]'), value: $submitted, content: $content);
    }
}
```

`core_courseformat\local\overview\overviewitem`:
- Mandatory: `name` (column header), `value` (`int|string|bool|null`, used for filtering, output as a data attribute), `content` (`string|renderable|null`; null renders `-`). Prefer renderables.
- Optional: `textalign` (`core\output\local\properties\text_align` enum), `alertcount` + `alertlabel` (pending actions; also put them in `content`), `extradata` (`stdClass`, exported to web services/app), `key` (normally auto-assigned).

Fixed columns: activity name (with section and visibility badge) and completion are automatic; due date (`get_due_date_overview()`) and main action (`get_actions_overview()`, always last column) are empty unless overridden.

Grades: one grade item is shown automatically. With several grade items (e.g. workshop), override `protected function get_grade_item_names(array $items): array` returning names indexed by grade item id (only named items are shown; return `[]` to hide grades). Do not override `get_grades_overviews()`.

Dependency injection: the class is built through DI. The constructor must accept `cm_info $cm` and call `parent::__construct($cm)`; it can take extra dependencies such as `\moodle_database $db` or `\core\clock $clock` (then use `$this->db` / `$this->clock->time()` instead of globals).

Later additions:
- 5.1 (MDL-85509): data exposed through `core_courseformat_get_overview_information`. For app compatibility, plain-string content should use only basic tags (`<a>`, `<span>`, `<strong>`, `<em>`) and no CSS classes; renderable content should implement the `exportable` interface; extra app data goes in `extradata`. `core_courseformat\local\overview\overviewfactory::activity_has_overview_integration()` tells whether a module integrates.
- 5.1 (MDL-85852): `needs_filtering_by_groups()`, `get_groups_for_filtering()` and `has_error()` on `activityoverviewbase` for separate-groups handling.
- 5.1: output helpers `core_courseformat\output\local\overview\overviewaction` (action link with badge, MDL-85981) and `...\overview\overviewdialog` (title, description, label/value list, MDL-83896).
- 5.2 (MDL-86660): `$cm` on `activityoverviewbase` is public.

## Activity completion

With no changes an activity supports only manual completion. Everything else requires feature flags in `[modname]_supports`.

**Completion on view:** return `true` for `FEATURE_COMPLETION_TRACKS_VIEWS`, then call this when the user successfully views the activity (before printing the header):

```php
$completion = new completion_info($course);
$completion->set_module_viewed($cm);
```

**Completion on grade:** return `true` for `FEATURE_GRADE_HAS_GRADE`; nothing else is needed.

**Custom completion rules:** return `true` for `FEATURE_COMPLETION_HAS_RULES`, then:

1. Store rule settings in your main table (e.g. forum's `completionposts`) and add them to the backup `backup_nested_element` field list; restore must default them when absent.
2. Expose them in `[modname]_get_coursemodule_info()` via `$result->customdata['customcompletionrules']` only when `$coursemodule->completion == COMPLETION_TRACKING_AUTOMATIC`.
3. In `mod_form.php` implement `add_completion_rules()` (returns top-level element names; **since 4.3, MDL-78528, suffix every element name with `$this->get_suffix()`**), `completion_rule_enabled($data)`, and zero unticked values in `get_data()` only inside `!empty($data->completionunlocked)`.

```php
public function add_completion_rules() {
    $mform = $this->_form;
    $suffix = $this->get_suffix();
    $group = [
        $mform->createElement('checkbox', 'completionpostsenabled' . $suffix, ' ',
            get_string('completionposts', 'forum')),
        $mform->createElement('text', 'completionposts' . $suffix, ' ', ['size' => 3]),
    ];
    $mform->setType('completionposts' . $suffix, PARAM_INT);
    $mform->addGroup($group, 'completionpostsgroup' . $suffix,
        get_string('completionpostsgroup', 'forum'), [' '], false);
    $mform->disabledIf('completionposts' . $suffix, 'completionpostsenabled' . $suffix, 'notchecked');
    return ['completionpostsgroup' . $suffix];
}
```

4. State class `mod_[modname]\completion\custom_completion` extending `\core_completion\activity_custom_completion`, implementing `get_state(string $rule): int` (`COMPLETION_COMPLETE` / `COMPLETION_INCOMPLETE` for `$this->cm`, `$this->userid`), `get_defined_custom_rules(): array`, `get_custom_rule_descriptions(): array`, and `get_sort_order(): array` (including core rules such as `completionview`, `completionusegrade`).

   Do NOT implement the pre-3.11 `[modname]_get_completion_state()` / `mod_[modname]_get_completion_active_rule_descriptions()` callbacks — removed in 4.3 (MDL-71196). The activity-completion devdocs page still shows them; it is stale.

5. Notify the completion system when a rule's underlying value may change, passing `COMPLETION_COMPLETE`, `COMPLETION_INCOMPLETE` or `COMPLETION_UNKNOWN` (slow), plus `$userid` when acting on another user. In tasks, `require_once($CFG->dirroot . '/lib/completionlib.php');` first.

```php
$completion = new completion_info($course);
if ($completion->is_enabled($cm) && $forum->completionposts) {
    $completion->update_state($cm, COMPLETION_COMPLETE);
}
```

Since 5.0 (MDL-83542) a core_completion hook `after_cm_completion_updated` fires when an activity completion is updated. Since 5.2 (MDL-87662) the manual completion button and activity dates render in the activity header (theme layout option `activityinfoinheader`).

## Course page display (lib.php callbacks)

- `[modname]_get_coursemodule_info($cm)` — returns `cached_cm_info`; cached in course modinfo, so DB queries are fine. Settable: `name`, `icon`/`iconcomponent`, `content`, `customdata` (no complex objects), `extraclasses`, `onclick`. No renderers here (MDL-41074); render from `customdata` in `_cm_info_view()`.
- `[modname]_cm_info_dynamic(cm_info $cm)` — runs on every course page load; **no DB queries**. Can affect access/navigation (`$cm->set_user_visible(false)`) and, since 5.2, the linear navigation URL (below).
- `[modname]_cm_info_view(cm_info $cm)` — course view page only; limited DB queries OK (`$cm->set_after_link(...)`). Cannot change access flags or the navigation URL (throws `coding_exception`).

Always obtain `$cm` from `get_fast_modinfo($course)` (`$modinfo->get_cm($cmid)`). Since 5.1 (MDL-86155) the modinfo classes are autoloaded as `\core_course\cm_info`, `\core_course\cached_cm_info`, `\core_course\section_info` and `\core_course\modinfo`; the old global names still work. 5.1 also adds `modinfo::get_instance_of()` (one cm by module name and instance id) and `modinfo::sort_cm_array()`.

## Linear navigation (5.2/5.3)

Course formats that opt in (5.3, see course-formats.md) show a sticky "Previous / Next" footer. Activities take part automatically; to change their destination URL use these `cm_info` methods (5.2, MDL-87984) from `[modname]_cm_info_dynamic()` only:

| Method | Effect |
| --- | --- |
| `get_navigation_url(): ?url` | Navigation URL; falls back to `get_url()` (also `$cm->navigationurl`) |
| `set_navigation_url(?url $navigationurl): void` | Override; `null` removes the activity from the Previous/Next flow |
| `reset_navigation_url(): void` | Drop the override |

```php
function [modname]_cm_info_dynamic(cm_info $cm) {
    // Like mod_resource / mod_url: keep linear navigation inside Moodle even for popup display.
    $cm->set_navigation_url(new \core\url($cm->get_navigation_url(), ['forceview' => 1]));
}
```

Activities are also skipped when they have no URL, are not visible on the course page for the user, are stealth without `moodle/course:viewhiddenactivities`, or lack `FEATURE_CAN_DISPLAY`. Use `\core_courseformat\local\linearnavigationsettings::is_linear_navigation_enabled($course)` to adapt your own page controls. Since 5.3 (MDL-88601) any page can add a link to the sticky footer:

```php
$PAGE->set_supplementary_content(new action_link(
    new moodle_url('/mod/forum/view.php', ['id' => $cm->id]),
    get_string('gotoalldiscussions', 'mod_forum'),
));
```

## Events

Event classes live in `classes/event/` (e.g. `course_module_viewed`). `db/events.php` defines observers; do not subscribe to another plugin's events without declaring a dependency on it.

```php
$observers = [
    [
        'eventname' => '\core\event\course_module_created',
        'callback' => '\mod_[modname]\event\observer\course_module_created::store',
        'priority' => 1000,
    ],
];
```

## Backup and restore

If the plugin stores data, implement backup/restore/duplication under `backup/moodle2/` (`backup_[modname]_activity_task.class.php`, `backup_[modname]_stepslib.php` and the matching `restore_*` files) and declare `FEATURE_BACKUP_MOODLE2`. List every instance field (including custom completion fields) in the `backup_nested_element`.

## Icon and badge

Provide `pix/monologo.svg` — every core module ships only this file (no `icon.svg`/`icon.png`) (verified in MOODLE_503_STABLE source); the devdocs mod pages still show `pix/icon.*`. It is used on the course page and in the activity chooser; override per instance via `cached_cm_info::$icon`/`$iconcomponent`. The icon background colour comes from `FEATURE_MOD_PURPOSE`; since 5.0 themes set it through `$activity-icon-*-bg` SCSS variables and core renders icons with `core_course\output\activity_icon` (MDL-84555). For a badge next to the activity name see `mod_[modname]\output\courseformat\activitybadge` in course-formats.md.

## Activity sub-plugins (examples from core)

- `mod_book` — `booktool` sub-plugins.
- `mod_data` — database **field types** and **presets**.
- `mod_forum` — `forumreport` sub-plugins.

Since 5.0, `db/subplugins.json` accepts a `subplugintypes` object with paths relative to the plugin root (see subplugins.md).
