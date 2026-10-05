# Privacy API (Moodle 5.3)

GDPR compliance for plugins: declare stored personal data, export it, and delete it via a `provider` class. Per the Moodle 5.3 developer docs (Privacy API, Subject Access Request FAQ, Privacy API utilities).

**Note:** The Privacy API pages are unchanged between the 4.5 and 5.3 docs, and the 5.0–5.3 UPGRADING notes contain no `core_privacy` or `tool_dataprivacy` API changes. The only related 5.x change is a test-class fix in `mod_assign` (see Testing below). Since 5.1 (MDL-83424) plugin code lives under `public/` in a git checkout (e.g. `public/mod/forum/classes/privacy/provider.php`).

## Personal data and architecture

- Personal data = information entered by the user (any record with a `userid`, timestamps and ids on it, any free-text field) **and** information stored about the user (e.g. ratings and comments a teacher makes on a student's submission).
- `core_privacy` has three parts: **metadata providers** (describe what is stored and why), **request providers** (act on export/delete requests), and the `\core_privacy\manager` that bridges components and the tools requesting data.
- Every plugin implements one metadata provider and zero, one or two request providers. Request providers: find contexts where a user has data; export a user's data from those contexts; delete a user's data from those contexts; find users with data in one context; delete data for several users in one context; delete all users' data in one context.

## Provider class location

Every plugin needs exactly one class:

- Class: `provider` in namespace `\your_pluginname\privacy`
- File: `path/to/your/plugin/classes/privacy/provider.php`
- Implements the relevant metadata interface plus zero, one, or two request interfaces.

## Plugins storing NO personal data: null_provider

Implement `\core_privacy\local\metadata\null_provider`. For plugins with no external links, no DB tables with user data (incl. IP addresses), and no user preferences (typical: themes, blocks, filters, editors). Note: a plugin that *causes* data to be stored elsewhere (e.g. via the comments subsystem) must NOT use the null provider.

```php
namespace block_calendar_month\privacy;

class provider implements
    // This plugin does not store any personal user data.
    \core_privacy\local\metadata\null_provider {

    /**
     * Get the language string identifier with the component's language
     * file to explain why this plugin stores no data.
     *
     * @return string
     */
    public static function get_reason(): string {
        return 'privacy:metadata';
    }
}
```

```php
// lang/en/block_calendar_month.php
$string['privacy:metadata'] = 'The Calendar block only displays existing calendar data.';
```

## Plugins storing personal data

Implement `\core_privacy\local\metadata\provider` (describes data) plus `\core_privacy\local\request\plugin\provider` and `\core_privacy\local\request\core_userlist_provider` (export/delete). All descriptions are lang string identifiers.

`get_metadata()` calls on the `collection`:

| Method | Use for |
| --- | --- |
| `add_database_table(table, [field => langstring], summary)` | own DB tables |
| `add_subsystem_link(subsystem, [], summary)` | data stored via a core subsystem: ratings (`core_rating`), tags, comments, questions, files (`core_files`) |
| `add_user_preference(name, summary)` | site-wide user preferences |
| `add_external_location_link(name, [field => langstring], summary)` | personal data sent to an external system (describe the type of destination) |

### get_metadata

```php
namespace mod_forum\privacy;
use core_privacy\local\metadata\collection;

class provider implements \core_privacy\local\metadata\provider {
    public static function get_metadata(collection $collection): collection {
        // Own database tables.
        $collection->add_database_table('forum_discussion_subs', [
            'userid' => 'privacy:metadata:forum_discussion_subs:userid',
            'discussionid' => 'privacy:metadata:forum_discussion_subs:discussionid',
            'preference' => 'privacy:metadata:forum_discussion_subs:preference',
        ], 'privacy:metadata:forum_discussion_subs');

        // Data stored in a core subsystem (files, ratings, comments, tags, questions...).
        $collection->add_subsystem_link('core_files', [], 'privacy:metadata:core_files');

        // Site-wide user preferences (general name only, without variable suffix).
        $collection->add_user_preference('tool_usertours_tour_completion_time',
            'privacy:metadata:preference:tool_usertours_tour_completion_time');

        // Data sent to an external service (describe the *type* of destination).
        $collection->add_external_location_link('lti_client', [
            'userid' => 'privacy:metadata:lti_client:userid',
            'fullname' => 'privacy:metadata:lti_client:fullname',
        ], 'privacy:metadata:lti_client');

        return $collection;
    }
}
```

### Request provider: required methods

`\core_privacy\local\request\plugin\provider` requires:

- `get_contexts_for_userid(int $userid): contextlist` — where the user has data
- `export_user_data(approved_contextlist $contextlist)` — export it
- `delete_data_for_all_users_in_context(\context $context)` — context expiry
- `delete_data_for_user(approved_contextlist $contextlist)` — right to be forgotten

`\core_privacy\local\request\core_userlist_provider` additionally requires:

- `get_users_in_context(userlist $userlist)` — users with data in one context (no subcontexts)
- `delete_data_for_users(approved_userlist $userlist)` — delete several users in one context

### get_contexts_for_userid

Add SQL returning context ids only (performance matters — this runs across the whole site):

```php
public static function get_contexts_for_userid(int $userid): contextlist {
    $contextlist = new \core_privacy\local\request\contextlist();

    $sql = "SELECT c.id
              FROM {context} c
        INNER JOIN {course_modules} cm ON cm.id = c.instanceid AND c.contextlevel = :contextlevel
        INNER JOIN {modules} m ON m.id = cm.module AND m.name = :modname
        INNER JOIN {forum} f ON f.id = cm.instance
         LEFT JOIN {forum_discussions} d ON d.forum = f.id
             WHERE d.userid = :discussionuserid";

    $contextlist->add_from_sql($sql, [
        'modname' => 'forum',
        'contextlevel' => CONTEXT_MODULE,
        'discussionuserid' => $userid,
    ]);
    return $contextlist;
}
```

Subsystem joins are available, e.g. ratings: `\core_rating\privacy\provider::get_sql_join('rat', 'mod_forum', 'post', 'p.id', $userid)` returns an object with `->join`, `->userwhere`, `->params` to splice into your SQL.

### get_users_in_context

```php
public static function get_users_in_context(userlist $userlist) {
    $context = $userlist->get_context();
    if (!$context instanceof \context_module) {
        return;
    }
    $params = ['instanceid' => $context->instanceid, 'modulename' => 'forum'];
    $sql = "SELECT d.userid
              FROM {course_modules} cm
              JOIN {modules} m ON m.id = cm.module AND m.name = :modulename
              JOIN {forum} f ON f.id = cm.instance
              JOIN {forum_discussions} d ON d.forum = f.id
             WHERE cm.id = :instanceid";
    $userlist->add_from_sql('userid', $sql, $params);
    // Repeat add_from_sql for each table holding user data. No return value.
}
```

### export_user_data

The `content_writer` (obtained via `writer::with_context()`) handles these data types: data (`stdClass` only, the main object), related data (e.g. ratings of a post), metadata (key/value pairs with a description), user preferences, files, and custom files (sparingly, e.g. a calendar feed).

`approved_contextlist` has the user record plus contexts (`get_user()`, `get_contexts()`, `get_contextids()`, or iterate). Export with the writer, always against a context; the `$subcontext` array represents nesting (forum > discussion > post):

```php
use core_privacy\local\request\writer;

writer::with_context($context)
    ->export_data($subcontext, $post)                                    // main data (stdClass)
    ->export_area_files($subcontext, 'mod_forum', 'post', $post->id)     // files
    ->export_metadata($subcontext, 'postread', (object) ['firstread' => $firstread],
        new \lang_string('privacy:export:post:postread'));

// Text fields supporting embedded files must be rewritten:
$post->message = writer::with_context($context)
    ->rewrite_pluginfile_urls($subcontext, 'mod_forum', 'post', $post->id, $post->message);
```

### User preferences

If any site-wide preferences exist, also implement `\core_privacy\local\request\user_preference_provider`:

```php
public static function export_user_preferences(int $userid) {
    $pref = get_user_preferences('markasreadonnotification', null, $userid);
    if (null !== $pref) {
        $desc = $pref ? get_string('markasreadonnotificationyes', 'mod_forum')
                      : get_string('markasreadonnotificationno', 'mod_forum');
        writer::export_user_preference('mod_forum', 'markasreadonnotification', $pref, $desc);
    }
}
```

Context-specific preferences are instead handled in the regular plugin provider (identified in `get_contexts_for_userid`, exported in `export_user_data`, deleted in both delete methods).

### Delete methods

**Note:** The upstream examples below use `mod_choice` and `mod_chat`; `mod_chat` was removed from core in 5.0, but the pattern is unchanged.

```php
public static function delete_data_for_all_users_in_context(\context $context) {
    global $DB;
    // Called for EVERY expiring context — check it belongs to this plugin.
    if ($context->contextlevel != CONTEXT_MODULE) {
        return;
    }
    $cm = get_coursemodule_from_id('choice', $context->instanceid);
    if (!$cm) {
        return;
    }
    $DB->delete_records('choice_answers', ['choiceid' => $cm->instance]);
}

public static function delete_data_for_user(approved_contextlist $contextlist) {
    global $DB;
    if (empty($contextlist->count())) {
        return;
    }
    $userid = $contextlist->get_user()->id;
    foreach ($contextlist->get_contexts() as $context) {
        $instanceid = $DB->get_field('course_modules', 'instance', ['id' => $context->instanceid], MUST_EXIST);
        $DB->delete_records('choice_answers', ['choiceid' => $instanceid, 'userid' => $userid]);
    }
}

public static function delete_data_for_users(approved_userlist $userlist) {
    global $DB;
    $context = $userlist->get_context();
    $cm = $DB->get_record('course_modules', ['id' => $context->instanceid]);
    $chat = $DB->get_record('chat', ['id' => $cm->instance]);
    [$userinsql, $userinparams] = $DB->get_in_or_equal($userlist->get_userids(), SQL_PARAMS_NAMED);
    $params = array_merge(['chatid' => $chat->id], $userinparams);
    $DB->delete_records_select('chat_messages', "chatid = :chatid AND userid {$userinsql}", $params);
}
```

## Subplugins and subsystem-called plugins

- A plugin *defining* subplugins declares an interface extending `\core_privacy\local\request\plugin\subplugin_provider` (e.g. `\mod_assign\privacy\assignsubmission_provider`) and collects subplugin data itself. Skip this if no data flows to subplugins (e.g. Atto).
- A subplugin implements its parent's interface, e.g. `class provider implements \core_privacy\local\metadata\provider, \mod_assign\privacy\assignsubmission_provider`.
- Plugins only called via a subsystem (e.g. plagiarism) implement that subsystem's provider, e.g. `\core_plagiarism\privacy\plugin_provider`.

## FAQs / edge cases

- Messages/notifications sent: no declaration or export needed (`core_messages` owns them once sent).
- Repository plugins: not responsible for the files fetched (they are a channel); declare any external system integration, and only store/declare data if you keep credentials, preferences or other user information.
- Session data: not declared (temporary; the export process cannot read it).
- Only sends data externally, stores nothing locally: declare via `add_external_location_link()` AND implement the request plugin provider with empty methods (required for compliance).
- Context-specific user preferences: implement `user_preference_provider` (site-wide ones) and `plugin\provider` (identify, export and delete the context-specific ones).
- Enrolment plugins (`enrol_xxx`): most implement the null provider because enrolment data is stored and managed by `core_enrol`.
- Stores nothing but writes to logs/another subsystem: implement the metadata provider (subsystem link) only; no request interfaces.
- Multi-branch support (pre-3.5.3): use `\core_privacy\local\legacy_polyfill` trait with underscore-prefixed methods (`_get_metadata`), and conditionally-defined interfaces for `core_userlist_provider`. Irrelevant for plugins targeting only 5.x.

## Testing

- Unit tests: extend `core_privacy\tests\provider_testcase` (not `advanced_testcase` — it does not reset the content_writer between tests).
- Changed in 5.0 (MDL-81520): `\mod_assign\privacy\provider_test` is now `final`. Community plugins (e.g. assign subplugins) whose tests extended it must extend `\mod_assign\tests\provider_testcase` instead.
- Development helper (Privacy API utilities page): a CLI script placed in the Moodle root that checks compliance of every component (or one, via `$CHECK_COMPONENT`). It instantiates `new \core_privacy\manager()` and calls `$manager->component_is_compliant($component)` and `\core_privacy\manager::get_provider_classname_for_component($component)`, reports null providers with their `get_reason()` string and counts metadata items via `new \core_privacy\local\metadata\collection($component)`. These scripts assist development only and do not replace unit tests. Since 5.1 the real `config.php` stays in the install root (`$CFG->root`) and `public/config.php` is a loader stub that requires `../config.php` (verified in MOODLE_503_STABLE source), so a script requiring `config.php` from its own directory works in either place.
