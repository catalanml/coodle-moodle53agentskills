# Enrolment plugins and API (Moodle 5.3)

Structure and API for `enrol_*` plugins and the core enrolment subsystem (enrollib), per Moodle 5.3 developer docs and UPGRADING notes. The devdocs enrolment pages are unchanged since 4.5; the 5.0–5.3 changes below come from UPGRADING.

## Changes since 4.5

- Since 5.0 (MDL-84142): plugins implementing `enrol_page_hook()` are encouraged to build the enrolment page HTML with the renderable `\core_enrol\output\enrol_page`, and to display forms in a modal dialogue. See `enrol_self` as the example. Deprecated in 5.0: `enrol_self_enrol_form` (use `enrol_self\form\enrol_form`) and `enrol_guest_enrol_form` (use `enrol_guest\form\enrol_form`).
- Since 5.0 (MDL-84139): new `enrol_plugin::get_instance_name_for_management_page()` can display extra details next to the instance name on the enrolment methods page.
- Since 5.0 (MDL-84143): themes can override the course enrolment index page by overriding the course renderer method `enrolment_options`.
- Since 5.0 (MDL-83432): the `after_user_enrolled` hook has a `roleid` property (role assigned during enrolment, if any); `enrol_plugin::send_course_welcome_message_to_user()` accepts a `$roleid` parameter to populate the `courserole` placeholder.
- Removed: `enrol_plugin::update_communication()` (final removal 5.0, MDL-80491; file cleanup 5.2); `enrol_self_plugin::get_welcome_email_contact()` — use `enrol_plugin::get_welcome_message_contact()` (5.0, MDL-81185); `enrol_mnet` and `mnetservice_enrol` removed from core (5.0).
- Web services: `core_enrol_get_enrolled_users` now returns basic user info even without permission to view full details, matching the web UI (5.2, MDL-87242); several `core_enrol_*` user functions add an optional `initials` field (5.2); `enrol_manual` `unenrol_users()` lost its unused `roleid` parameter (5.2, MDL-51152).
- Deprecated in 5.3 (MDL-89439): the manual enrol instance `->enrol_cohort(...)` method (unused, no group support).
- Since 5.1 (MDL-83424) core lives under `public/`: in a git checkout enrol plugins are at `public/enrol/<name>/` and enrollib at `public/lib/enrollib.php`. Paths below are relative to `$CFG->dirroot`.

## Concepts

- Enrolment data lives in tables `enrol` (instances per course) and `user_enrolments` (user ↔ instance), plus optional plugin tables. Enrolments and role assignments are separate: roles live in `role_assignments`, where `component` is the enrol plugin name and `itemid` the instance id.
- Enrolment states: `ENROL_USER_ACTIVE`, `ENROL_USER_SUSPENDED`. Prefer suspending over full unenrolment — unenrolment irreversibly purges participation data (grades, group membership).
- A user has an **active enrolment** when: a `user_enrolments` record exists, enrolment has started, is not past `timeend`, has active status, the `enrol` instance is active, and the plugin is enabled.
- Plugin kinds: manual (enrol_manual; enrol_flatfile automates enrol/unenrol actions), fully automatic sync (enrol_ldap, enrol_database, enrol_category — configured at system level, sync via scheduled tasks; enrol_cohort is similar but may need per-course instance configuration), and interactive (enrol_self, enrol_fee — override `show_enrolme_link()`, `enrol_page_hook()`, and implement instance add/edit; since 5.0 render the page via `\core_enrol\output\enrol_page` with forms in a modal).

## File structure

```
enrol/pluginname/
|-- db
|   `-- access.php
|-- lang
|   `-- en
|       `-- enrol_pluginname.php
|-- lib.php          (required)
`-- version.php
```

`lib.php` must contain the plugin class extending `enrol_plugin` (defined at the end of `lib/enrollib.php`):

```php
class enrol_pluginname_plugin extends enrol_plugin {
    // Override enrol_plugin methods to define the workflow.
}
```

### db/access.php capabilities

Define depending on workflow: `enrol/xxx:enrol` (when `allow_enrol()` returns true), `enrol/xxx:unenrol` (with `allow_unenrol()`/`allow_unenrol_user()`), `enrol/xxx:manage` (with `allow_manage()`), `enrol/xxx:unenrolself` (self-unenrolment), `enrol/xxx:config` (editable instance properties; sync plugins usually skip it).

```php
$capabilities = [
    'enrol/pluginname:enrol' => [
        'captype' => 'write',
        'contextlevel' => CONTEXT_COURSE,
        'archetypes' => ['manager' => CAP_ALLOW, 'editingteacher' => CAP_ALLOW],
    ],
    'enrol/pluginname:manage' => [ /* same shape */ ],
    'enrol/pluginname:unenrol' => [ /* same shape */ ],
    'enrol/pluginname:unenrolself' => [
        'captype' => 'write',
        'contextlevel' => CONTEXT_COURSE,
        'archetypes' => [],
    ],
];
```

### Language strings

`lang/en/enrol_pluginname.php` defines `pluginname`, `pluginname_desc`, `privacy:metadata` (or the full privacy metadata strings) and one string per capability, e.g. for enrol_fee:

```php
$string['fee:config'] = 'Configure enrolment on payment enrol instances';
$string['fee:manage'] = 'Manage enrolled users';
$string['fee:unenrol'] = 'Unenrol users from course';
$string['fee:unenrolself'] = 'Unenrol self from course';
$string['pluginname'] = 'Enrolment on payment';
$string['privacy:metadata'] = 'The enrolment on payment enrolment plugin does not store any personal data.';
```

## enrol_plugin workflow methods (override in your plugin)

- `allow_enrol(stdClass $instance): bool` — plugin compatible with manual enrolments; manual enrol UI shows if true AND user has `enrol/pluginname:enrol`.
- `allow_unenrol(stdClass $instance): bool` — allow other code to unenrol everybody from an instance (course reset, manual unenrol); paired with `enrol/pluginname:unenrol` capability.
- `allow_unenrol_user(stdClass $instance, stdClass $userenrolment): bool` — allow unenrolling a specific user; defaults to `allow_unenrol()`. Sync plugins commonly allow only suspended enrolments:

```php
public function allow_unenrol_user(stdClass $instance, stdClass $userenrolment) {
    return $userenrolment->status == ENROL_USER_SUSPENDED;
}
```

- `allow_manage(stdClass $instance): bool` — allow manual edits of user enrolments; return false for sync plugins (edits would be reverted). Paired with `enrol/pluginname:manage`.
- `roles_protected(): bool` — default true; return false to let users remove roles assigned by this plugin.
- `enrol_user(stdClass $instance, $userid, $roleid = null, $timestart = 0, $timeend = 0, $status = null, $recovergrades = null)` — enrolment hook; add validation / alter dates or role, throw exceptions on failure, then call `parent::enrol_user(...)`.
- `get_user_enrolment_actions(course_enrolment_manager $manager, $userenrolment): array` — extend the default edit/unenrol actions; append `new user_enrolment_action($icon, $title, $url)` to `parent::get_user_enrolment_actions(...)`, gated by `allow_manage()` + capability checks.
- `find_instance(array $enrolmentdata, int $courseid): ?stdClass` — uniquely identify an instance from CSV course upload data (implemented by enrol_manual, enrol_self, enrol_guest, enrol_cohort); e.g. enrol_cohort matches on `$i->enrol == 'cohort' && $i->customint1 == $cohortid && $i->roleid == $roleid` over `enrol_get_instances($courseid, false)`.
- `is_csv_upload_supported()` — default false; override to support CSV course upload.

## Standard editing UI (participants page add/edit instance)

Implement:

```php
public function use_standard_editing_ui() {
    return true;
}

public function can_add_instance($courseid) {
    global $DB;
    $context = context_course::instance($courseid, MUST_EXIST);
    if (!has_capability('moodle/course:enrolconfig', $context)
            || !has_capability('enrol/pluginname:config', $context)) {
        return false;
    }
    // Example: only one instance per course.
    if ($DB->record_exists('enrol', ['courseid' => $courseid, 'enrol' => 'pluginname'])) {
        return false;
    }
    return true;
}

public function edit_instance_form($instance, MoodleQuickForm $mform, $context) {
    $mform->addElement('text', 'customtext1', get_string('extraname', 'enrol_pluginname'));
}

public function edit_instance_validation($data, $files, $instance, $context) {
    $errors = [];
    // return 'element_name' => 'error_description' entries.
    return $errors;
}

public function add_instance($course, ?array $fields = null) {
    $instanceid = parent::add_instance($course, $fields);
    // Insert rows into plugin tables if needed.
    return $instanceid;
}
```

The devdocs example writes `array $fields = null`; the explicit nullable `?array` shown here is what the coding style recommends and remains compatible with the parent.

## Expiration, sync, welcome email

- Sync plugins usually offer an "External unenrol action" setting and can provide scheduled tasks to synchronise enrolments.
- Plugins setting `timeend` in `user_enrolments` should handle expiry via `enrol_plugin::process_expirations()` and `enrol_plugin::send_expiry_notifications()`.
- Welcome email options (constants in enrollib.php, form options via `enrol_send_welcome_email_options()`):

```php
define('ENROL_DO_NOT_SEND_EMAIL', 0);
define('ENROL_SEND_EMAIL_FROM_COURSE_CONTACT', 1);
define('ENROL_SEND_EMAIL_FROM_KEY_HOLDER', 2);
define('ENROL_SEND_EMAIL_FROM_NOREPLY', 3);
```

Typical pattern: store the option in an instance field (e.g. `customint4`), and in `enrol_user()` after `parent::enrol_user(...)`:

```php
if ($instance->customint4 != ENROL_DO_NOT_SEND_EMAIL) {
    $this->email_welcome_message($instance, core_user::get_user($userid));
}
```

## Enrol subsystem API (core functions)

```php
is_enrolled(context $context, stdClass $user = null, string $withcapability = '', bool $onlyactive = false)
// True for students/teachers, false for admins. E.g. is_enrolled($context, $USER, 'mod/choice:choose')

get_enrolled_users(context $context, string $withcapability = '', int $groupid = 0,
    string $userfields = 'u.*', ?string $orderby = null, int $limitfrom = 0,
    int $limitnum = 0, bool $onlyactive = false)
// E.g. get_enrolled_users($modcontext, 'mod/assign:submit');

count_enrolled_users(context $context, string $withcapability = '', int $groupid = 0, bool $onlyactive = false)

get_enrolled_sql(context $context, string $withcapability = '', int $groupid = 0,
    bool $onlyactive = false, bool $onlysuspended, int $enrolid = 0)
// For joining against enrolled users in custom SQL.
```

### Enrolling/unenrolling programmatically

Get the instance record, then the plugin object via `enrol_get_plugin()`:

```php
$instance = $DB->get_record('enrol', ['courseid' => $course->id, 'enrol' => 'manual']);
$enrolplugin = enrol_get_plugin($instance->enrol);
$enrolplugin->enrol_user($instance, $user->id, $role->id, $timestart, $timeend, ENROL_USER_ACTIVE);
$enrolplugin->unenrol_user($instance, $user->id);
$enrolplugin->update_user_enrol($instance, $user->id, $timestart, $timeend, ENROL_USER_SUSPENDED);
```

### Instance management

```php
$enrolplugin->add_default_instance($course->id);       // new instance with defaults, returns id
$enrolplugin->add_custom_instance($course, $settings); // course object + settings array, returns id
$enrolplugin->delete_instance($instance);              // removes instance and its user enrolments
```
