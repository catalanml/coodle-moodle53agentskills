# Access and Roles API (Moodle 5.3)

Scope: contexts, defining capabilities in `db/access.php`, checking capabilities (`has_capability`, `require_capability`, `require_login`), roles, permissions, and archetypes. Per the Moodle 5.3 developer docs (Access API and Roles pages) plus the 5.0–5.3 UPGRADING notes.

**Note:** The Access API and Roles pages are unchanged between the 4.5 and 5.3 docs; the 5.x changes below come from UPGRADING notes. Since 5.1 (MDL-83424) plugin code lives under `public/` in a git checkout, so `mod/folder/db/access.php` is `public/mod/folder/db/access.php` at repository level (paths relative to `$CFG->dirroot` are unchanged).

## Core model

Moodle uses role-based access control. Entities are represented by **contexts** arranged in a tree (the context tree). A **role** is a set of capability definitions; roles are defined at the System context (top of the tree) and can be overridden in contexts further down. User access is calculated from the combination of roles assigned to the user.

Always ask "Does the user have capability to do something?" — never "Does the user have a role somewhere?".

## Contexts

| Class | Constant | Level | Represents | Immediate contents |
| --- | --- | --- | --- | --- |
| `context_system` | `CONTEXT_SYSTEM` | 10 | The site as a whole (root, only one) | user, course category, module, block |
| `context_user` | `CONTEXT_USER` | 30 | An individual user | block |
| `context_coursecat` | `CONTEXT_COURSECAT` | 40 | A course category | course category, course, block |
| `context_course` | `CONTEXT_COURSE` | 50 | A single course | module, block |
| `context_module` | `CONTEXT_MODULE` | 70 | An activity | block |
| `context_block` | `CONTEXT_BLOCK` | 80 | A block | none |

### Fetching context instances

```php
$systemcontext = context_system::instance();
$usercontext = context_user::instance($user->id);
$categorycontext = context_coursecat::instance($category->id);
$coursecontext = context_course::instance($course->id);
$contextmodule = context_module::instance($cm->id);   // note: cm id, not module instance id
$contextblock = context_block::instance($this->instance->id);

// By context id:
$context = context::instance_by_id($contextid);
```

Notes: by default an exception is thrown if the context cannot be created; deleted users no longer have contexts. Context instances are created/deleted automatically by the system.

The Access API page still uses the legacy `context_*` class names. Other 5.3 doc pages (comment, output, routing) use the namespaced equivalents:

```php
$context = \core\context\system::instance();
$context = \core\context\course::instance($course->id);
$context = \core\context\module::instance($cm->id);
```

Removed in 5.2 (MDL-87427, from `lib/deprecatedlib.php`): `get_context_instance()` and `get_system_context()`. Use `context_xxx::instance()` / `\core\context\xxx::instance()` and `context::instance_by_id()`.

Since 5.2 (MDL-87264): routes can accept `core\router\parameters\query_course` and `core\router\parameters\query_coursemodule` (verified in MOODLE_503_STABLE source); UPGRADING writes `parameter\` query parameters, which are converted automatically into the course/course-module objects and their contexts (routes are covered in `webservices-advanced.md`).

## Defining capabilities in db/access.php

Capability names are `plugintype/pluginname:capabilityname`. Define them in the plugin's `db/access.php`:

```php
// mod/folder/db/access.php
$capabilities = [
    'mod/folder:managefiles' => [
        'riskbitmask' => RISK_SPAM,
        'captype' => 'write',
        'contextlevel' => CONTEXT_MODULE,
        'archetypes' => [
            'editingteacher' => CAP_ALLOW,
        ],
    ],
];
```

| Key | Meaning |
| --- | --- |
| `riskbitmask` | associated risks (see Risk bitmask below) |
| `captype` | `read` or `write`; the system prevents all write capabilities for guest and not-logged-in users |
| `contextlevel` | typical context level where checked; may be checked at lower levels too (e.g. `moodle/site:accessallgroups` checked with `CONTEXT_MODULE`) |
| `archetypes` | defaults per role archetype, applied on install/upgrade/role reset; use only `CAP_ALLOW` here |
| `clonepermissionsfrom` | copy per-role permissions from an existing capability, e.g. `'clonepermissionsfrom' => 'moodle/quiz:attempt'` |

After any change to `db/access.php`, bump the plugin version number so upgrade scripts run.

Name each capability in the plugin language file (string key `pluginname:capabilityname`):

```php
// mod/folder/lang/en/folder.php
$string['folder:managefiles'] = 'Manage files in folder module';
```

### Risk bitmask values

- `RISK_SPAM` — user can add visible content, send messages to other users
- `RISK_PERSONAL` — access to private personal information (e.g. backups with user details, hidden email)
- `RISK_XSS` — user can submit content that is not cleaned (HTML with active content, unprotected files)
- `RISK_CONFIG` — user can change global configuration; actions missing sanity checks
- `RISK_MANAGETRUST` — manage trust bitmasks of other users
- `RISK_DATALOSS` — can destroy large amounts of unrecoverable information

Default expectation: Guest role has only risk-free capabilities; Student may add SPAM; Teacher may add PERSONAL and XSS; admins have everything.

Example combining risks, several archetypes and `clonepermissionsfrom` (from the Roles page):

```php
$capabilities = [
    'tool/brickfield:viewcoursetools' => [
        'riskbitmask' => RISK_PERSONAL,
        'captype' => 'read',
        'contextlevel' => CONTEXT_COURSE,
        'archetypes' => [
            'teacher' => CAP_ALLOW,
            'editingteacher' => CAP_ALLOW,
            'manager' => CAP_ALLOW,
        ],
        'clonepermissionsfrom' => 'coursereport/participation:view',
    ],
];
```

### Deprecating a capability

Follow the normal deprecation policy timeline. Remove it from `$capabilities` and add it to `$deprecatedcapabilities` in the same file. Optional keys: `replacement` (checks route to the replacement capability) and `message`. Checking a deprecated capability always emits a `DEBUG_DEVELOPER` debugging message.

```php
// mod/folder/db/access.php
$deprecatedcapabilities = [
    'mod/folder:managefiles' => [
        'replacement' => 'mod/folder:newmanagefiles',
        'message' => 'This was replaced with another capability',
    ],
];
```

## Checking capabilities

### has_capability()

```php
function has_capability(
    string $capability,
    context $context,
    object $user = null,
    bool $doanything = true
): bool;
```

```php
$context = context_module::instance($cm->id);
if (has_capability('mod/folder:managefiles', $context)) {
    // Do or display something.
}
```

Defaults to the current user; returns true for admins by default (`$doanything` — not recommended to pass false).

### require_capability()

Same check, but throws an access control exception on failure:

```php
function require_capability($capability, context $context, $userid = null, $doanything = true, $errormessage = 'nopermissions', $stringfile = '')
```

### require_login() and friends

```php
function require_login($courseorid = null, $autologinguest = true, $cm = null, $setwantsurltome = true, $preventredirect = false)
function require_course_login($courseorid, $autologinguest = true, $cm = null, $setwantsurltome = true, $preventredirect = false)
function isguestuser($user = null)
function isloggedin()
function is_siteadmin($user_or_id = null)
function is_guest(context $context, $user = null)
function is_viewing(context $context, $user = null, $withcapability = '')
```

Every plugin page script must call `require_login()` or `require_course_login()` after setting `$PAGE->url`. `require_login()`:

- verifies the user is logged in before accessing any course/activity
- verifies access to hidden courses and activities
- verifies availability restrictions for the given activity (`$cm`). Changed in 5.2 (MDL-87283): when the activity's restrictions are visible to the user, `require_login()` now redirects users to a restricted page (the 5.3 Access API page does not mention this yet)
- verifies the user is enrolled, has `moodle/course:view`, or has temporary guest access from an enrol plugin
- logs course access

`require_course_login()` is only for activities allowing read access to frontpage content without login (viewing resource files, glossary entries, etc.).

`isguestuser()`/`isloggedin()`/`is_siteadmin()` are usually unnecessary — write/risky capabilities are automatically denied to special accounts by `has_capability()`. Using `is_siteadmin()` in activity modules is strongly discouraged; use capabilities and enrolment status.

To access course data, one of these must be true for the user:

- `is_enrolled()` — active record in user_enrolments
- `is_viewing()` — has `moodle/course:view` (access without being a participant)
- `is_guest()` — temporary guest access granted by an enrolment plugin

`get_users_by_capability()` returns users holding a capability; it ignores enrolment status and should only be used above the course context:

```php
function get_users_by_capability(context $context, $capability, $fields = '', $sort = '', $limitfrom = '', $limitnum = '',
    $groups = '', $exceptions = '', $doanything_ignored = null, $view_ignored = null, $useviewallgroups = false)
```

The Access API page points to the Enrolment API for `is_enrolled()` and related functions (see `moodle-plugin-development/references/enrolment.md`).

## Roles, permissions, archetypes

A **permission** is the level at which a capability applies to a role:

- `CAP_INHERIT` — no value set; inherited from a more general context
- `CAP_ALLOW`
- `CAP_PREVENT`
- `CAP_PROHIBIT` — cannot be overridden; prohibit always wins across all the user's roles

Setting a different value for the same capability in a more specific context is an **override**. Roles are assigned to users in a context (e.g. Teacher in a particular course).

Capability conflicts: a user with several roles (e.g. Teacher and Student in the same course) can get different values for the same capability from each role, at site, category or course level. `CAP_PROHIBIT` always wins: a user whose site-wide role prohibits forum posting cannot post even where another role (e.g. a facilitator role in one forum) allows it.

Special automatic roles: not-logged-in users get `$CFG->notloggedinroleid`; the guest account gets `$CFG->guestroleid`; all authenticated users get `$CFG->defaultuserroleid`, plus `$CFG->defaultfrontpageroleid` in the frontpage context. None of these can be assigned manually.

### Role archetypes

Archetypes act as templates for roles; a capability's `archetypes` defaults apply to every role based on that archetype:

- **manager** — system-level, manages courses without enrolment
- **coursecreator** — system-level, creates new courses
- **editingteacher** — course-level, grades students and manages the course
- **teacher** — course-level, grades students but cannot add/edit activities
- **student** — course-level participant, completes activities
- **guest** — non-authenticated access; not supposed to change anything
- **user** — assigned to every authenticated user
- **frontpage** — all authenticated users on the site home page

```php
$archetypes = get_role_archetypes();
```

Since 5.0 (MDL-83542): a hook `after_role_switched` is dispatched when a user switches to another role in a course (the UPGRADING note does not give the full class name; see `hooks-di.md` for how to find and register hook callbacks).

### Rarely needed role functions

Prefer per-user capability checks. In rare cases:

```php
// Roles holding a capability in a context:
[$roleids] = get_roles_with_cap_in_context($context, 'moodle/course:manageactivities');

// Custom enrolment plugin development only:
$ras = get_user_roles($context, $user, $checkparentcontexts);
role_assign($roleid, $userid, $contextid, $component, $enrolmentpluginid);
```

## Related 5.x changes

- 5.1 (MDL-85754): `question_edit_contexts` now checks permissions only in the provided context, not in all parent contexts. Questions now live only in activity-module contexts (question banks), so normal inheritance and overrides apply per bank.
- 5.3: admin setting classes renamed to autoloaded `core_admin` classes (old names still work); e.g. `\admin_setting_users_with_capability` is now `\core_admin\setting\setting\users_with_capability` and `\admin_setting_pickroles` is now `\core_admin\setting\setting\pickroles`.
- 5.0 (MDL-84267): example of a core capability deprecation: `mod/data:comment` and `mod/data:managecomments` were deprecated as unused.
