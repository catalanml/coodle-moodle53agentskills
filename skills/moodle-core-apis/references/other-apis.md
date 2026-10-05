# Moodle 5.3 — Other Core APIs Reference

Compact reference for Moodle 5.3 core/subsystem APIs: strings, messaging, navigation, preferences, report builder, custom fields, grading, calendar, admin settings, tags, comments, locking, groups, user APIs — per the Moodle 5.3 developer docs and UPGRADING notes.

Moved elsewhere: Check, Availability and Analytics APIs → `more-subsystems.md`; Backup/restore → `backup-restore.md`.

## String API (get_string / lang files)

Each plugin defines strings in `lang/en/<frankenstyle>.php` (e.g. `auth/ldap/lang/en/auth_ldap.php`); every plugin must define `pluginname`. Lang files must never contain `require_once` or any file inclusion — plain key-value definitions only.

```php
// lang/en/local_myplugin.php
$string['pluginname'] = 'My plugin';
$string['newsectionname'] = 'New name for section {$a}';  // {$a} placeholder.

// Usage anywhere:
get_string('pluginname', 'local_myplugin');
get_string('newsectionname', 'format_topics', $sectionname);  // 3rd arg fills {$a}.
```

`{$a}` may be a scalar or an object/array (`{$a->name}`). In admin trees prefer `new lang_string(...)` (lazy). Conventions: `_desc` suffix for setting descriptions, `_help` for help strings. Since 5.2 activity modules may add optional `modulename_summary` and `modulename_tip` strings for the activity chooser.

## Message API

`message_send()` (in `lib/messagelib.php`, auto-loaded) sends a message. Register providers in `db/messages.php` (bump plugin version after changes):

```php
// mod/quiz/db/messages.php
defined('MOODLE_INTERNAL') || die();

$messageproviders = [
    'submission' => ['capability' => 'mod/quiz:emailnotifysubmission'], // Checked at system context.
    'posts' => [],                                                      // Visible to all users.
    'mynotification' => [
        'defaults' => [
            'pop-up' => MESSAGE_PERMITTED + MESSAGE_DEFAULT_ENABLED,
            'email' => MESSAGE_PERMITTED,
        ],
    ],
];
```

Lang string: `$string['messageprovider:submission'] = 'Notification of quiz submissions';`

```php
$message = new \core\message\message();
$message->component = 'mod_yourmodule';
$message->name = 'mynotification';            // Provider name from messages.php.
$message->userfrom = core_user::get_noreply_user();
$message->userto = $user;
$message->subject = 'message subject 1';
$message->fullmessage = 'message body';
$message->fullmessageformat = FORMAT_MARKDOWN;
$message->fullmessagehtml = '<p>message body</p>';
$message->smallmessage = 'small message';
$message->notification = 1;                   // 0 only for user-to-user messages.
$message->contexturl = new \core\url('/course/'); // Since 5.1 a \core\url is accepted, as well as a string.
$message->contexturlname = 'Course list';
$messageid = message_send($message);
```

Notifications live in `notifications`; user messages in `messages` + conversation tables. Only `\core\message\message` objects are accepted. Removed in 5.2: many legacy `\core_message\api` methods (`search_users()`, `get_contacts()`, `get_messages()`, `can_post_message()`, `is_user_blocked()`, `create_conversation_between_users()`, …) and `\core_message\helper::get_messages()/create_messages()`.

## Navigation API

Three structures on `$PAGE`: `$PAGE->navigation` (main tree), `$PAGE->settingsnav` (settings), `$PAGE->navbar` (breadcrumbs). Generated lazily after `$PAGE` context/course/cm/url are set (usually by `require_login()` / `$PAGE->set_url()`).

Since 5.1 (MDL-82159) the classes are namespaced and autoloaded — `\core\navigation\navigation_node`, `global_navigation`, `settings_navigation`, `navbar`, `navigation_node_collection`, `navigation_cache`, `breadcrumb_navigation_node`, … (old global names still work). `NAVIGATION_CACHE_NAME` / `NAVIGATION_SITE_ADMIN_CACHE_NAME` are deprecated for `\core\navigation\navigation_node::CACHE_NAME` / `::SITE_ADMIN_CACHE_NAME`. `flat_navigation*` was removed in 5.2.

```php
use core\navigation\navigation_node;

$previewnode = $PAGE->navigation->add(get_string('preview'),
    new moodle_url('/a/link.php'), navigation_node::TYPE_CONTAINER);
$thingnode = $previewnode->add(get_string('thingname'), new moodle_url('/a/link.php'));
$thingnode->make_active();

$coursenode = $PAGE->navigation->find($courseid, navigation_node::TYPE_COURSE);

$PAGE->navbar->ignore_active();
$PAGE->navbar->add(get_string('preview'), new moodle_url('/a/link.php'));
```

Plugin callbacks in `lib.php` (called automatically):

```php
function {modulename}_extend_navigation($node, $course, $module, $cm);
function {modulename}_extend_settings_navigation($settings, $node);
function local_{pluginname}_extend_navigation(global_navigation $nav);
function local_{pluginname}_extend_settings_navigation(settings_navigation $nav, context $context);
function <component>_extend_navigation_course(navigation_node $parentnode, stdClass $course, context_course $context);
function <component>_extend_navigation_user_settings(navigation_node $parentnode, stdClass $user, context_user $context, stdClass $course, context_course $coursecontext);
function <component>_extend_navigation_category_settings(navigation_node $parentnode, context_coursecat $context);
function <component>_extend_navigation_frontpage(navigation_node $parentnode, stdClass $course, context_course $context);
function <component>_extend_navigation_user(navigation_node $parentnode, stdClass $user, context_user $context, stdClass $course, context_course $coursecontext);
```

Insert before a sibling: `$parent->create(...)` then `$parent->add_node($node, 'siblingkey')`. If the active page isn't matched: `navigation_node::override_active_url(new moodle_url(...))`. Since 5.2 sections with visible restrictions appear in the tree. Since 5.3 the primary/secondary nav bars render via React (`core/nav/Nav`, `.mds-nav-pill` instead of `.nav-link`) — re-check theme CSS/JS targeting the old markup.

## Preference API

User preferences, stored in DB (session for guests). Operate on the current user by default; pass a user id/object as last arg. Don't store defaults — apply them in code.

```php
set_user_preference('foo_nameformat', 'long');
get_user_preferences('foo_nameformat');           // 'long'.
get_user_preferences('foo_x', 'default');         // With default.
set_user_preferences(['foo_displaynum' => '100', 'foo_nameformat' => 'short']);
unset_user_preference('foo_showavatars');
```

Since 5.0 `{user_preferences}.value` is TEXT: conditions on it in WHERE/JOIN need `$DB->sql_compare_text()`. From JavaScript use the `core_user/repository` module — `M.util.set_user_preference`, `user_preference_allow_ajax_update()` and `lib/ajax/setuserpref.php` were removed in 5.0.

## Report Builder API

Entities (reusable column/filter collections) extend `\core_reportbuilder\local\entities\base` and implement `get_default_tables()` (abstract since 5.0; `get_default_table_aliases()` support is gone), `get_default_entity_title()`, `initialise()`. Plugin entities go in e.g. `\mod_assign\reportbuilder\local\entities\...` (or `reportbuilder\entities`); core ones in `\core_reportbuilder\local\entities\`.

```php
// Column (in entity initialise()):
$columns[] = (new column('starttime', new lang_string('task_starttime', 'admin'), $this->get_entity_name()))
    ->add_joins($this->get_joins())          // Redundant since 5.3 (entity joins auto-prepended).
    ->set_type(column::TYPE_TIMESTAMP)
    ->add_field("{$tablealias}.timestart")
    ->set_is_sortable(true)                  // Redundant since 5.3 (sortable defaults to true).
    ->add_callback([format::class, 'userdate']);

// Filter:
$filters[] = (new filter(course_selector::class, 'courseselector', new lang_string('courses'),
    $this->get_entity_name(), "{$tablealias}.id"))->add_joins($this->get_joins());
```

Filter classes live in `reportbuilder/classes/local/filters/` (text, date, number, boolean_select, select, course_selector, duration, tags, autocomplete, category). Always use `$this->get_table_alias('tablename')` in SQL fragments.

System report — extend `\core_reportbuilder\system_report`:

```php
protected function initialise(): void {
    $entitymain = new task_log();
    $entitymainalias = $entitymain->get_table_alias('task_log');
    $this->set_main_table('task_log', $entitymainalias);   // Alias mandatory since 5.3.
    $this->add_entity($entitymain);
    $this->add_base_fields("{$entitymainalias}.id");      // Needed for action :placeholders.

    $entityuser = new user();
    $entityuseralias = $entityuser->get_table_alias('user');
    $this->add_entity($entityuser->add_join(
        "LEFT JOIN {user} {$entityuseralias} ON {$entityuseralias}.id = {$entitymainalias}.userid"));

    $this->add_columns_from_entities(['task_log:name', 'user:fullname', 'task_log:starttime']);
    $this->add_filters_from_entities(['task_log:name', 'task_log:timestart']);
    $this->add_action((new action(
        new moodle_url('/admin/tasklogs.php', ['logid' => ':id']),
        new pix_icon('t/download', ''), [], new lang_string('downloadtasklog', 'report_tasklogs'))));
    $this->set_downloadable(true);
    $this->set_initial_sort_column('task_log:starttime', SORT_DESC);
}
```

Custom report data source — extend `\core_reportbuilder\datasource` in the `reportbuilder\datasource` namespace; same `initialise()` pattern plus `get_name()`, `get_default_columns()`, `get_default_filters()`, `get_default_conditions()`; optionally `get_default_column_sorting()`, `get_default_condition_values()`, `$this->add_all_from_entities()`. Test with `\core_reportbuilder\tests\core_reportbuilder_testcase` (autoloaded since 5.0 — no `require_once` of `reportbuilder/tests/helpers.php`) and its stress helpers (`datasource_stress_test_columns()` etc.).

Changes since 4.5 (from UPGRADING; the 5.3 page still shows the older boilerplate):

- 5.0: `set_report_action()` / `set_report_info_container()` on reports; `report::get_report_row_count()` (report tables no longer populate `countsql`); `get_deprecated_tables()` on entities; column `set_help_icon()`; `date` aggregation; `select` filter with ≤2 options drops the operator field; custom sort fields must be among the column's selected fields.
- 5.1: custom schedule types extend `core_reportbuilder\local\schedules\base` as `<component>\reportbuilder\schedule\<type>`; column `get_effective_type()`.
- 5.2: the base entity has a default `initialise()` that calls `get_available_columns()`, `get_available_filters()`, `get_available_conditions()` — entities can drop that boilerplate; `add_all_from_entities()` honours the given order; `add_[columns|filters|conditions]_from_entity()` accept entity instances; text filter supports `*`/`?` wildcards.
- 5.3: `set_main_table_sql()` for complex main-table SQL (`get_main_table()` deprecated → `get_main_table_sql()`); `prepend_join[s]()` on the join trait; column `add_header_attributes(...)`; `add_fields()` accepts string or array; `week`/`month`/`year` aggregations (extend abstract `datebase`); columns are sortable by default — call `->set_is_sortable(false)` where needed.

## Custom Fields API

Components define handler `<PLUGIN>\customfield\<AREA>_handler` extending `\core_customfield\handler` (in `classes/customfield/`). Get one via `HANDLERCLASS::create($itemid)` or `\core_customfield\handler::get_handler('core_course', 'course')`.

Handler caching (5.3 behaviour): `create()` returns a **new instance** by default. Handlers that need per-request caching opt in with a class constant:

```php
namespace local_myplugin\customfield;

class myarea_handler extends \core_customfield\handler {
    /** @var bool Cache handler instances returned by create(). */
    protected const CACHE_HANDLER_INSTANCES = true;
}
```

History: 5.2 (MDL-88176) made the base `create()` always cache and told handlers to remove their own caching; 5.3 reverted that default. The 5.3 Custom fields page and the 5.2 dev update still describe the 5.2 always-cached behaviour — stale.

Instance edit form integration:

```php
$handler->instance_form_definition($mform);
$handler->instance_form_before_set_data();
$handler->instance_form_definition_after_data();
$handler->instance_form_validation();
$handler->instance_form_save($data);   // After save; $data must have id.
$handler->delete_instance();           // On instance deletion.
```

Retrieving values:

```php
$handler = \core_customfield\handler::get_handler('core_course', 'course');
foreach ($handler->get_instance_data($courseid) as $data) {
    if (empty($data->get_value())) {
        continue;
    }
    $shortname = $data->get_field()->get('shortname');
    $value = $data->get_value();
}
```

Also: `export_instance_data()`, `display_custom_fields_data()`, `get_fields()`, `get_categories_with_fields()`. Field type plugins live in `/customfield/field/` with `field_controller` and `data_controller` classes. 5.x: `\core_customfield\external\field_data_exporter` (5.0); `api::get_instance_fields_data()` / `api::get_instances_fields_data()` take `component`, `area`, `itemid` and categories can be **shared** (5.1); `api::is_shortname_unique(...)` and WS `core_customfield_convert_category` (5.2).

## Advanced Grading API

Entities: grading **area** (managed by `grading_manager`), form **definition** (e.g. rubric criteria, table `grading_definitions`), form **instance** (one per evaluation, `INSTANCE_STATUS_ACTIVE` / `INSTANCE_STATUS_NEEDUPDATE`).

```php
function [activityname]_supports(string $feature): ?bool {
    switch ($feature) {
        case FEATURE_ADVANCED_GRADING:
            return true;
    }
}
// Also define [activityname]_grading_areas_list().

if ($controller = get_grading_manager(...)->get_active_controller()) {
    $controller->set_grade_range(...);
    $gradinginstance = $controller->get_or_create_instance(...);
    $mform->addElement('grading', 'ELEMENTNAME', '...', ['gradinginstance' => $gradinginstance]);
    $grade = $gradinginstance->submit_and_get_grade($data->ELEMENTNAME, ...);  // On submit.
    // Display: $controller->render_grade(...);  Preview: $controller->render_preview($PAGE);
}
```

The gradebook API (`grade_update()`, grade items) is not in the 5.3 devdocs apis tree. Gradebook notes from UPGRADING: `grade_regrade_final_grades(..., async: true)` queues `\core_course\task\regrade_final_grades` (5.0); `grade_item::is_gradable()` (5.1); scale-less outcomes linked to modules via `grade_outcome::add_outcome_to_module()`, `remove_outcome_from_module()`, `get_outcomes_in_module()`, `get_used_outcomes_in_course()` (5.3).

## Calendar API

`require_once($CFG->dirroot . '/calendar/lib.php');`

```php
$event = new stdClass();
$event->eventtype = SCORM_EVENT_TYPE_OPEN;          // Your own string identifier.
$event->type = CALENDAR_EVENT_TYPE_STANDARD;        // Or CALENDAR_EVENT_TYPE_ACTION (+ $event->timesort).
$event->name = get_string('calendarstart', 'scorm', $scorm->name);
$event->description = format_module_intro('scorm', $scorm, $cmid, false);
$event->format = FORMAT_HTML;
$event->courseid = $scorm->course;
$event->groupid = 0;
$event->userid = 0;
$event->modulename = 'scorm';
$event->instance = $scorm->id;
$event->timestart = $scorm->timeopen;
$event->visible = instance_is_visible('scorm', $scorm);
$event->timeduration = 0;
calendar_event::create($event);

$event = calendar_event::load($eventid);
$event->update($data);
$event->delete($repeats);
```

Overrides: `$event->priority = null` (normal), `CALENDAR_EVENT_USER_OVERRIDE_PRIORITY` (user override, wins), positive int for group overrides (lower = higher priority).

Action events (dashboard/timeline) — module callbacks:

```php
function mod_xyz_core_calendar_is_event_visible(calendar_event $event): bool;
function mod_xyz_core_calendar_provide_event_action(calendar_event $event, \core_calendar\action_factory $factory) {
    return $factory->create_instance(get_string('enter', 'scorm'),
        new \moodle_url('/mod/scorm/view.php', ['id' => $cm->id]),
        1 /* itemcount */, $actionable /* bool */);   // Return null to hide from timeline.
}
function mod_xyz_core_calendar_event_action_shows_item_count(calendar_event $event, $itemcount = 0): bool;
```

Drag & drop: implement `mod_xyz_core_calendar_event_timestart_updated(\calendar_event $event, \stdClass $instance)` and `mod_xyz_core_calendar_get_valid_event_timestart_range()` returning `[$mindate, $maxdate]` (each `[timestamp, 'error string']` or null; `false` blocks dragging). Non-module plugins: `$event->component = 'tool_yourtool'`, `$event->modulename = ''`; site events use `courseid = SITEID`.

5.0 deprecations: `calendar_day_representation()`, `calendar_time_representation()`, `calendar_format_event_time()` → `core_calendar\output\humandate` / `humantimeperiod` (see Time API in `more-subsystems.md`); `calendar_add_month()` / `calendar_sub_month()` → `\core_calendar\type_factory::get_calendar_instance()->get_next_month()` / `->get_prev_month()`; `calendar_get_courselink()`, `calendar_events_by_day()` deprecated. Removed: `calendar_top_controls()`, `calendar_get_link_previous/next()`, `calendar_add_event_metadata()`, `core_calendar_renderer::event()`.

## Admin Settings

Tree of `admin_root` → `admin_category` → `admin_settingpage` / `admin_externalpage` → `admin_setting` subclasses. Library in `lib/adminlib.php`; plugin settings go in the plugin's `settings.php`.

```php
// local plugin settings.php
defined('MOODLE_INTERNAL') || die();

if ($hassiteconfig) {   // Shorthand for moodle/site:config check.
    $settingspage = new admin_settingpage('managelocalhelloworld', new lang_string('manage', 'local_helloworld'));
    if ($ADMIN->fulltree) {   // Only build settings when needed.
        $settingspage->add(new admin_setting_configcheckbox(
            'local_helloworld/showinnavigation',   // Slash => config_plugins, read via get_config().
            new lang_string('showinnavigation', 'local_helloworld'),
            new lang_string('showinnavigation_desc', 'local_helloworld'),
            1));
    }
    $ADMIN->add('localplugins', $settingspage);
}
```

`admin_setting_configtext` constructor: `($name, $visiblename, $description, $defaultsetting, $paramtype = PARAM_RAW, $size = null)`. Names without a slash go to global `config` / `$CFG` — discouraged for plugins. Activity defaults with flags: `$setting->set_advanced_flag_options(admin_setting_flag::ENABLED, false)`, `$setting->set_locked_flag_options(...)`, then `$this->apply_admin_defaults();` at the end of the mod_form constructor. Post-update callback: `$setting->set_updatedcallback('theme_reset_all_caches');`. External pages: `new admin_externalpage('name', $title, $url)`; the page calls `admin_externalpage_setup($pagename)`. Prefer settings over external pages (searchable, CLI `admin/cli/cfg.php`, forceable in config.php, logged).

Since 5.3 the admin classes are namespaced and autoloaded (old names "currently unaffected", so the example above stays valid and is the choice for multi-version plugins):

- `\admin_setting_<name>` → `\core_admin\setting\setting\<name>` (e.g. `\core_admin\setting\setting\configtext`, `configcheckbox`, `heading`, `flag`, `check`)
- `\admin_setting` → `\core_admin\setting`; `\admin_settingpage` → `\core_admin\setting\settingpage\settingpage`
- `\admin_root` / `\admin_category` / `\admin_externalpage` → `\core_admin\setting\tree\root` / `category` / `externalpage`

Since 5.1, settings and external pages have a `searchmatchtype` property (types from `admin_search`) that plugins extending them are encouraged to set.

## Tag API

Main classes: `core_tag_tag`, `core_tag_collection`, `core_tag_area`. A **tag area** = component + itemtype (itemtype must be a DB table name). Define in `db/tag.php` (bump version after changes):

```php
$tagareas = [[
    'itemtype' => 'wiki_pages',
    'component' => 'mod_wiki',
    'callback' => 'mod_wiki_get_tagged_pages',
    'callbackfile' => '/mod/wiki/locallib.php',
]];
// Lang string: $string['tagarea_wiki_pages'] = 'Wiki pages';
```

```php
$mform->addElement('tags', 'tags', get_string('tags'),
    ['itemtype' => 'wiki_pages', 'component' => 'mod_wiki']);   // Auto-hides if area disabled.
core_tag_tag::is_enabled('mod_wiki', 'wiki_pages');
core_tag_tag::set_item_tags('mod_wiki', 'wiki_pages', $pageid, $modulecontext, $data->tags);
$data->tags = core_tag_tag::get_item_tags_array('mod_wiki', 'wiki_pages', $pageid);
echo $OUTPUT->tag_list(core_tag_tag::get_item_tags('mod_wiki', 'wiki_pages', $page->id), null, 'wiki-tags');
core_tag_tag::remove_all_item_tags($component, $itemtype, $itemid);
core_tag_tag::delete_instances($component, $itemtype = null, $contextid = null);
core_tag_tag::add_item_tag($component, $itemtype, $itemid, $context, $rawname);  // E.g. during restore.
```

Use `core_tag_tag::get_items_tags()` for many item ids. The search callback should use `core_tag_index_builder` and return `core_tag\output\tagindex`. Since 5.0 `core_renderer::tag_list()` has a `displaylink` parameter and `core_tag\taglist` a `displaylink` property (default true; false renders tag names as plain text).

## Comment API (core_comment)

Since 5.1 the API is `\core_comment\manager` (was `\comment`) and `\core_comment\comment_exception` (was `\comment_exception`). `comment/locallib.php` and `comment_manager` are deprecated; `comment/lib.php` is empty and will be removed in 6.0 — do not include it.

Constructor takes one args object: `context` (`\core\context`), `component`, optional `itemid`, `area`, `cm` (`cm_info`), `course`; rendering options `client_id`, `autostart`, `showcount`, `displaycancel`, `notoggle`, `linktext`.

```php
$args = (object) [
    'context' => \core\context\module::instance($cm->id),
    'component' => 'mod_data',
    'area' => 'database_entry',
    'itemid' => $entry->id,
    'cm' => $cm,
];
$comment = new \core_comment\manager($args);
echo $comment->output();                    // Comment area HTML.

$count = $comment->count();
foreach ($comment->get_comments() as $c) {  // get_comments(int $page, int $sortdirection = DESC).
    echo $c->content;                       // Page size: $CFG->commentsperpage (default 15).
}

$new = $comment->add('Nice entry', FORMAT_MOODLE);
$comment->delete($new);                     // int|stdClass.
\core_comment\manager::delete_comments($args); // All comments for context/itemid/commentarea.
```

Option setters: `set_view_permission()`, `set_post_permission()`, `set_notoggle()`, `set_autostart()`, `set_displaycancel()`, `set_displaytotalcount()` (all `bool`).

Plugins using comments must validate the area in a `lib.php` callback `PLUGINNAME_comment_display`:

```php
/**
 * Validate comment data before displaying comments.
 *
 * @param stdClass $comments
 * @param stdClass $args
 * @return stdClass
 */
function mod_myplugin_comment_display(stdClass $comments, stdClass $args): stdClass {
    if ($args->commentarea != 'entry_comments') {
        throw new \core_comment\comment_exception('invalidcommentarea');
    }
    if ($args->itemid != 0) {
        throw new \core_comment\comment_exception('invalidcommentitemid');
    }
    return $comments;
}
```

## Lock API

Cluster-safe mutual exclusion (correct, not fast — avoid hot paths). Always release, even on failure.

```php
$locktype = 'mod_assign_download_submissions';  // Prefix with component.
$resource = 'user:' . $USER->id;                // Unique resource key.
$lockfactory = \core\lock\lock_config::get_lock_factory($locktype);
if ($lock = $lockfactory->get_lock($resource, 5 /* timeout secs */)) {
    // ... exclusive work ...
    $lock->release();
} else {
    throw new moodle_exception('locktimeout');
}
```

Backend configurable via `$CFG->lock_factory` (custom implementations implement `\core\lock\lock_factory`).

## Groups API

Functions in `lib/grouplib.php`, prefixed `groups_`. Modes: `NOGROUPS`, `SEPARATEGROUPS`, `VISIBLEGROUPS`. `moodle/site:accessallgroups` bypasses separation. Visibility constants: `GROUPS_VISIBILITY_ALL|MEMBERS|OWN|NONE`; `moodle/course:viewhiddengroups` sees all.

```php
$cm = get_coursemodule_from_id('forum', $id, 0, false, MUST_EXIST);
$currentgroupid = groups_get_activity_group($cm);
$currentgroupname = groups_get_group_name($currentgroupid);
if (groups_get_activity_groupmode($cm)) {
    $groups = groups_get_activity_allowed_groups($cm);
}
groups_print_activity_menu($cm, $url);   // Group selection dropdown.

[$visibilitywhere, $visibilityparams] = \core_group\visibility::sql_group_visibility_where($userid);
```

`groups_get_all_groups()` with `$participationonly = true` returns only participation groups. Since 5.2 `groups_print_activity_menu()` and `groups_get_activity_group()` also take `$participationonly` (default true); pass false only for teacher-facing filtering — never in menus shown to students. Use core functions (`groups_get_members()` etc.) — they respect visibility. Hooks: `\core_group\hook\after_group_membership_added` / `_removed` (5.2). `FEATURE_GROUPMEMBERSONLY` is deprecated (5.3) — remove it from plugins.

## User-related APIs

- `core_user::get_profile_picture($user, $context, $options)` — options include `size`, `link`, `courseid`, `includefullname`.
- `core_user::get_profile_url($user, $context)`, `core_user::get_fullname($user, $context, $options)` (4.3+), `\core_user::get_dummy_fullname(...)` (5.1, configured name fields only).
- Field validation/cleaning: `core_user::validate($data)`, `core_user::clean_data($data)`, `core_user::clean_field($value, $field)`, `core_user::get_property_type($field)` — use these instead of `clean_param()` for user fields.
- User selector widget: subclass `user_selector_base` (`user/selector/lib.php`). Always sort user lists with `users_order_by_sql()`.
- Since 5.3 (MDL-82650) `user/lib.php` functions are deprecated (DEBUG_DEVELOPER notice) in favour of static methods on `\core\user`: `user_create_user()` → `\core\user::create_user()`, `user_update_user()` → `update_user()`, `user_delete_user()` → `delete_user()`, `user_get_users_by_id()` → `get_users_by_id()`, `user_get_user_details()` → `get_user_details()`, `user_can_view_profile()` → `can_view_profile()`, `user_get_default_fields()` → `get_default_fields()`, etc. (same name minus the `user_` prefix); `user_update_device_public_key()` → `\core_user\devicekey::update_device_public_key()`.
- 5.2: the MoodleNet profile field is now a custom profile field (core `moodlenetprofile` column being removed).
