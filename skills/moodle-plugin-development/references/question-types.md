# Question Types, Question Bank and Quiz Access Rule Plugins (Moodle 5.3)

Reference for developing question type (`qtype_`), question bank (`qbank_`) and quiz access rule (`quizaccess_`) plugins, plus the question engine concepts their authors need, per Moodle 5.3 developer docs and UPGRADING notes.

## What changed since 4.5 (summary)

- **Shared question banks (5.0, MDL-71378).** Question categories no longer live in system, course category or course contexts. Questions now exist only at activity module context level; shared banks are provided by the new `mod_qbank` activity module. `question_make_default_categories()` no longer creates categories in `CONTEXT_SYSTEM`, `CONTEXT_COURSE` or `CONTEXT_COURSECAT` — use `question_get_default_category()`, which can optionally create one. `question_delete_course()` and `question_delete_course_category()` are deprecated with no replacement.
- **Permissions (5.1, MDL-85754).** `question_edit_contexts` only checks the provided context, not parent contexts, so permission overrides work normally per question bank.
- **Restore identity hashing (4.4.6 / 5.0, MDL-83541).** qtypes with their own tables or extra data must help restore match existing questions, or each restore duplicates questions. See "Restore code" below.
- **qbank filter conditions.** `build_query_from_filter()` and `get_condition_key()` are now abstract; `get_filter_class()` returns `null` by default (5.0, MDL-83859). New optional `filter_invalid_values()` (5.0), client-side `validation()` (5.0, MDL-83862), `restore_filtercondition()` (5.2, MDL-86524).
- **Bulk actions.** `get_bulk_actions()` accepts an optional qbank view (5.0); `get_bulk_action_classes()` (5.1); `get_action_icon()` must be overridden (5.3, MDL-73051).
- **Quiz access rules.** The legacy file `mod/quiz/accessrule/accessrulebase.php` was removed in 5.1 (MDL-76612): extend `mod_quiz\local\access_rule_base`. Rules can implement `mod_quiz\local\rule_overridable` for per-user/group overrides.
- Since 5.1 (MDL-83424) core code lives under `public/`; in a git checkout these plugin types are at `public/question/type/`, `public/question/bank/` and `public/mod/quiz/accessrule/`. Paths below are relative to `$CFG->dirroot`.

## Question subsystem overview

The question subsystem has two main subcomponents, each served by its own plugin types:

**The question engine** makes questions work when users attempt them. Main core user is `mod_quiz` (also `qbank_previewquestion`, and contrib plugins like `filter_embedquestion`). It orchestrates two plugin types:

- **Question types (`qtype_`)** define what the question *is* — multiple choice, drag-drop, shortanswer, numerical, or complex types like crossword or coderunner.
- **Question behaviours (`qbehaviour_`)** define how a student *interacts* with questions: `qbehaviour_deferredfeedback` (classic exam: graded and feedback given later), `qbehaviour_immediatefeedback` ('Check' button per question), `qbehaviour_interactive` ('Try again' after a wrong first try, using the feedback).

**The question bank** is the UI where teachers create, edit and manage questions. Core question bank code mainly ties together features supplied by plugins:

- **Question bank plugins (`qbank_`)** add features to the question bank (since 4.0, MDL-70329).
- **Question import/export plugins (`qformat_`)** provide import/export formats, used by `qbank_exportquestions`, `qbank_importquestions`, `qbank_exporttoxml`.
- **Question types (`qtype_`)** also integrate with the bank so their questions can be created and edited.

### Key question engine classes

The engine orchestrates qtype + behaviour, stores progress in the DB in bulk, and presents a uniform API to consumers like a quiz attempt:

1. `question_engine` — main entry point.
2. `question_attempt` — a user's attempt at one question; groups of attempts are held in `question_usage_by_activity` (even more important).
3. `question_state` — state of each question in a form consumers understand, without restricting behaviours.
4. `question_display_options` — how the consumer controls which parts (grade, feedback, ...) are visible; behaviours can modify them during rendering.
5. `core_question_renderer` — overall question layout, delegating details to the `qtype_` and `qbehaviour_` renderers.

Since 5.0 (MDL-68806): `question_attempt_step`'s constructor accepts `question_attempt_step::TIMECREATED_ON_FIRST_RENDER` for `$timecreated`; the first `question_attempt::render()` then sets the first step's timecreated to the current time (supports quiz attempt pre-creation).

## Question type (qtype_) plugin structure

Question types live in `question/type/<name>/` and must provide:

1. `edit_..._form.php` — editing form so teachers can create and edit questions of this type.
2. `questiontype.php` (docs: `questiontypes.php`) — class handling loading/saving data from that form, plus:
   - related methods providing metadata about the question type;
   - import and export in any question formats the type wants to support.
3. `question.php` — the class representing one instance of the question while it is being attempted. It must:
   - start a new attempt (e.g. multiple choice randomly shuffles the choices here);
   - or, when continuing an existing attempt, re-initialise the question to the same state from DB data;
   - tell the question engine what submitted data this question type expects;
   - analyse submitted responses: has it changed? is it complete?
   - automatically grade the response to a 'fraction' (mark between 0 and 1) and a state (correct / partially correct / incorrect);
   - check access to files for the File API.
4. `renderer.php` — displays the key bits of the question type, for `core_question_renderer` to combine into the overall question display.
5. Backup and restore, plus all standard Moodle plugin parts (DB tables, lang strings, version.php, ...). Restore code needs special care to avoid question duplication (below).
6. Optionally, defaults for new questions (below).

Other qtype-relevant changes:

- Deprecated in 5.0: `question_type::generate_test()` (no replacement).
- Deprecated in 5.2: `get_next_version()` in questionlib.php — use `\core_question\versions::get_next_version()`. Never query the new `question_bank_entries.nextversion` column directly; the method increments it unless you pass `increment: false`, so call it inside a transaction committed only after the version number is used in a `question_versions` record (MDL-86798).
- Since 5.1 (MDL-85240): `qtype_multichoice` restricts its margin-bottom override for `fitem_id_answer_*` / `fitem_id_fraction_*` to its own edit form. qtypes that relied on it must add equivalent rules to their own `styles.css` (see `qtype_calculatedmulti`).

## Defaults for new questions

Question types can remember a teacher's preferred settings (as user preferences) and reuse them as defaults when creating a *new* question. Preferences are saved only when a new question is created and saved — never when editing an existing question (which may have been created by someone else).

**Which settings to save:** not name or question text (those uniquely define a question). Good candidates are presentation options, e.g. whether multiple-choice answers are numbered 'a, b, c', '1, 2, 3', or not at all. Search core for implementations of `save_defaults_for_new_questions` for consistent examples.

**In the form class** — replace hard-coded defaults with `get_default_value`:

```php
// Before:
$mform->setDefault('shuffleanswers', 1);

// After:
$mform->setDefault('shuffleanswers', $this->get_default_value('shuffleanswers', 1));
```

**In the question-type class** — override `save_defaults_for_new_questions`:

```php
public function save_defaults_for_new_questions(stdClass $fromform): void {
    parent::save_defaults_for_new_questions($fromform);
    $this->set_default_value('shuffleanswers', $fromform->shuffleanswers);
}
```

Save the same settings that the form fetches with `get_default_value`. Calling `parent` is required because core saves some settings applying to all question types.

**Privacy provider:** since this uses user preferences, your privacy provider must declare them — including the ones saved by core. Easiest approach: copy another question type, e.g. `question/type/match/classes/privacy/provider.php` in moodle core.

**Tests:** unit tests for the privacy provider (see `question/type/match/tests/privacy/provider_test.php`) and a Behat test showing saved settings are reused (many qtypes have a `tests/behat/add.feature` where coverage is easy to add).

## Restore code: avoiding duplicate questions

Since 4.4.6 (MDL-83541; in 5.0 UPGRADING under core_question "Fixed"). When a question is restored, a SHA1 hash of its data is compared with a hash of each question in the target category. On a match the existing question is reused (references in the backup point to it); otherwise a new question is created. Both sides are converted to the `$questiondata` structure (as returned by `get_question_options()`), stripped of inconsistent data such as DB IDs, flattened, concatenated and hashed.

**Impact if you do nothing:** every restore (and every quiz duplication using a shared bank) creates another copy of the question.

**Check your plugin:** `mod/quiz/tests/backup/repeated_restore_test.php` runs against every installed qtype and fails if your plugin lacks the required information. Your plugin is skipped unless `tests/helper.php` defines a test question with a `form_data` method (so `create_question()` can generate it). Include a hint in the test question if the type supports multiple tries. qtypes not using the standard `question_answers` table may still have some tests skipped; consider your own test that edited custom data produces a different hash. 5.1 (MDL-85556) fixed `test_restore_course_with_same_stamp_questions` passing incorrectly for qtypes using answers — re-run it, and review any copy of it in your own tests.

**What to change**, in `backup/moodle2/restore_qtype_<name>_plugin.class.php` (class `restore_qtype_<name>_plugin`):

1. If `define_question_plugin_structure()` only calls `$this->add_question_*()` methods, those fields (and their IDs) are handled automatically. If it adds extra paths to `$paths[]`, override `convert_backup_to_questiondata()`: start with `parent::convert_backup_to_questiondata($backupdata)`, then add the plugin-specific paths from `$backupdata` at the right points, so the result matches `qtype_<name>::get_question_options()`.
2. If `db/install.xml` defines extra tables, exclude their primary and foreign key (ID) fields from the hash with `define_excluded_identity_hash_fields()`.
3. If `get_question_options()` attaches data not present in the backup (or vice versa), override `remove_excluded_question_data()` to strip it, then call the parent.

```php
// question/type/example/backup/moodle2/restore_qtype_example_plugin.class.php

public static function convert_backup_to_questiondata(array $backupdata): stdClass {
    $questiondata = parent::convert_backup_to_questiondata($backupdata);
    // Copy plugin-specific paths from $backupdata into $questiondata->options here.
    return $questiondata;
}

protected function define_excluded_identity_hash_fields(): array {
    return [
        '/options/extradata/id',
        '/options/extradata/questionid',
    ];
}

public static function remove_excluded_question_data(stdClass $questiondata, array $excludefields = []): stdClass {
    if (isset($questiondata->options->pluginconfig)) {
        unset($questiondata->options->pluginconfig);
    }
    return parent::remove_excluded_question_data($questiondata, $excludefields);
}
```

**Note:** the docs show only the first line of `convert_backup_to_questiondata()`; the parent in `backup/moodle2/restore_qtype_plugin.class.php` is `public static function convert_backup_to_questiondata(array $backupdata): \stdClass` (verified in MOODLE_503_STABLE source) — the override must be static too. The exact exclusion paths depend on where `get_question_options()` and `convert_backup_to_questiondata()` place the records (see the PHPDoc of `restore_qtype_plugin::define_excluded_identity_hash_fields()`).

**Debugging:** breakpoint the `return` of `restore_questions_parser_processor::generate_question_identity_hash()`. Most tests hit it 3 times per question (first read/restore, second read, hashing the restored question); compare `$questiondata` in the last two.

**Core examples:** `qtype_calculated` / `restore_qtype_match_plugin` (`convert_backup_to_questiondata()`), `qtype_truefalse` (`define_excluded_identity_hash_fields()` for `trueanswer`/`falseanswer`), `qtype_multianswer` (`remove_excluded_question_data()` removing `subquestions`).

## Question bank (qbank_) plugins

Since Moodle 4.0 (MDL-70329). Located in `question/bank/<name>/`. They can extend the question bank with:

- Table columns
- Action menu items
- Bulk actions
- Navigation node (tabs)
- Question preview additions (via callback)
- Question filters (since 4.3, MDL-72321)

The starting point for most features is a class `classes/plugin_feature.php` (class `qbank_<name>\plugin_feature`, extending `core_question\local\bank\plugin_features_base`) declaring which features the plugin adds. The devdocs page is a draft ("until more documentation is written"); core `qbank_*` plugins are the reference implementations to copy from.

**Bulk actions:**

- Since 5.0 (MDL-79281): `plugin_features_base::get_bulk_actions()` can receive the qbank view object; the parameter is nullable, so existing plugins need not change.
- Since 5.1 (MDL-84548): `bulk_action_base::get_bulk_action_classes()` lets a bulk action add CSS classes to its menu entry (default `''`).
- Since 5.3 (MDL-73051): `bulk_action_base` defines `get_action_icon()`; bulk action classes should override it to return an icon identifier (the base `public function get_action_icon(): string` returns the placeholder `'i/empty'` (verified in MOODLE_503_STABLE source)), shown on the action's button in the sticky footer. See `\qbank_bulkmove\bulk_move_action`. The base class is `\core_question\local\bank\bulk_action_base` (verified in MOODLE_503_STABLE source; UPGRADING misspells it `\core\question\local\bank\bulk_action_base`).
- Deprecated in 5.0: `qbank_bulkmove/helper::get_displaydata` and `qbank_bulkmove\output\renderer::render_bulk_move_form` (replaced by a modal + `core_question_external\move_questions`, and `qbank_bulkmove\output\bulk_move`).

**Removed APIs to check for:** many `core_question\local\bank\view` methods (`print_choose_category_message()`, `display_advanced_search_form()`, `init_search_conditions()`, `get_current_category()`, `display_options_form()`, `start_table()`, `end_table()`), `action_column_base`, `menu_action_column_base` and `menuable_action` (removed 5.0, MDL-78090); `view::display_top_pagnation()` / `display_bottom_pagination()` (removed 5.1); `core_question_bank_renderer::render_category_condition*()` (removed 5.2). Pagination is now AJAX-based (5.1, MDL-78091).

## Question bank filters

Filters apply when viewing the question bank, are included in the URL (shareable filtered views), and define criteria for adding random questions to a quiz.

A filter condition has two parts:

- a backend **condition** PHP class (name, options, how it applies to the question bank query), extending `core_question\local\bank\condition`;
- a frontend **filter** JavaScript class (UI, passing selected values back), by default `core/datafilter/filtertype` — overridable.

### Minimal implementation

```php
// question/bank/myplugin/classes/myfilter_condition.php
namespace qbank_myplugin;

use core_question\local\bank\condition;

class myfilter_condition extends condition {
    public function get_name(): string {
        return get_string('myfilter_name', 'qbank_myplugin');   // Label shown in the filter UI.
    }

    public function get_condition_key(): string {
        return 'myfilter';   // Unique machine-readable ID used in filter parameters.
    }

    public function build_query_from_filter(array $filter): array {
        $andor = ' AND ';
        $equal = '=';
        if ($filter['jointype'] === self::JOINTYPE_ANY) {
            $andor = ' OR ';
        } else if ($filter['jointype'] === self::JOINTYPE_NONE) {
            $equal = '!=';
        }
        $conditions = [];
        $params = [];
        // In real life we'd probably use $DB->get_in_or_equal here.
        foreach ($filter['values'] as $key => $value) {
            $conditions[] = 'q.fieldname ' . $equal . ' :myfilter' . $key;
            $params['myfilter' . $key] = $value;
        }
        return [
            '(' . implode($andor, $conditions) . ')',
            $params,
        ];
    }
}
```

```php
// question/bank/myplugin/classes/plugin_feature.php
namespace qbank_myplugin;

use core_question\local\bank\plugin_features_base;
use core_question\local\bank\view;

class plugin_feature extends plugin_features_base {
    public function get_question_filters(?view $qbank = null): array {
        return [
            new myfilter_condition($qbank),
        ];
    }
}
```

`build_query_from_filter()` returns an SQL `WHERE` fragment and params. `$filter` contains `'values'` (selected values) and `'jointype'` (`JOINTYPE_ANY`, `JOINTYPE_ALL`, or `JOINTYPE_NONE`). Conditions from all filters are combined in `core_question\local\bank\view::build_query()`. Since 5.0 (MDL-83859) `build_query_from_filter()` and `get_condition_key()` are declared abstract. Validate the data it receives and throw exceptions on invalid values — client-side validation alone is not enough.

**Note:** the devdocs example calls `get_string('myfilter_name', 'myplugin')` and extends `core_question\local\bank\plugin_features_base` without a leading backslash inside a namespace; both are fixed above.

### Optional condition overrides

| Method | Purpose |
|---|---|
| `get_initial_values(): array` | Pre-defined selectable values, an array of `['value' => ..., 'title' => ...]` entries, searchable in the autocomplete field. |
| `allow_custom(): bool` | Return `false` to restrict terms to those from `get_initial_values()`. |
| `get_join_list(): array` | Restrict join types, e.g. `[datafilter::JOINTYPE_ANY, datafilter::JOINTYPE_NONE]` when `JOINTYPE_ALL` makes no sense. |
| `allow_multiple(): bool` | Return `false` to allow only a single value at a time. |
| `allow_empty(): bool` | Return `false` to make selecting a value compulsory when the condition is added. |
| `is_required(): bool` | Return `true` to always display the condition. |
| `get_filter_class()` | Namespaced path to a custom JS filter class, e.g. `'qbank_myplugin/datafilter/filtertype/myfilter'`. Since 5.0 no longer abstract; returns `null` (use `core/datafilter/filtertype`) by default, so drop overrides that only return `null`. |
| `get_filteroptions(): object` | Extra options for how values are applied (e.g. include child categories). Read current options in the constructor and use them in `build_query_from_filter()`. |
| `filter_invalid_values($filterconditions)` | Since 5.0 (MDL-83784): remove anything invalid or unwanted from the filterconditions array. |
| `restore_filtercondition()` | Since 5.2 (MDL-86524): during restore of a `question_set_reference` (e.g. random quiz questions), ID mapping in the filtercondition is delegated to qbank plugins. If your condition stores database IDs, override this to replace them with mapped values. See `qbank_managecategories\category_condition`. The docs do not give its signature; copy it from that class. |

### Custom JavaScript filter class

Default (`core/datafilter/filtertype`) provides a single autocomplete for numeric IDs with text labels. To customise, either reuse another core type from `/lib/amd/src/datafilter/filtertypes`, or create `amd/src/datafilter/filtertypes/myfilter.js` exporting a default class extending `core/datafilter/filtertype`:

- Override `get values()` for textual (non-`parseInt`) values — see `qbank_viewquestiontype/datafilter/filtertypes/type`.
- Override `addValueSelector()` for a different value-selection UI.
- For filter options: override the constructor to add the options UI and `get filterOptions()` to return current values — see `qbank_managecategories/datafilter/filtertypes/categories` (JS) and `qbank_managecategories\category_condition` (PHP).

### Client-side validation

Since 5.0 (MDL-83862). Filters support standard browser form validation: the simplest approach is to set validation attributes on the inputs in the Mustache template used by `addValueSelector()`. For more advanced checks, define a `validation()` method in the JS filter class; it is called when "Apply filters" is clicked. On failure, report errors with the standard `setCustomValidity()` and `reportValidity()` on your input elements and return `false`. Example: `core/datafilter/filtertypes/datetime`.

### Context-sensitive configuration

The condition constructor receives the current `$qbank` view object; extract data from it for use in other methods. Example: the tag condition (`qbank_tagquestion\tag_condition`) uses the current page context to control which tags the filter offers.

## Quiz access rule (quizaccess_) subplugins

Quiz access rules add conditions a user must meet to attempt a quiz. Core rules: `quizaccess_delaybetweenattempts`, `quizaccess_ipaddress`, `quizaccess_numattempts`, `quizaccess_offlineattempts`, `quizaccess_openclosedate`, `quizaccess_password`, `quizaccess_seb`, `quizaccess_securewindow`, `quizaccess_timelimit`.

Rules are subplugins of `mod_quiz`, located in `mod/quiz/accessrule/<name>/`. Keep all files inside your plugin folder. Since 5.0 (MDL-83705) `mod/quiz/db/subplugins.json` declares the type with the new `subplugintypes` object (`"quizaccess": "accessrule"`, relative to the plugin root) alongside the legacy `plugintypes` entry (`"quizaccess": "mod/quiz/accessrule"`); this only matters if you write your own subplugin-defining module.

```console
mod/quiz/accessrule/delaybetweenattempts
├── classes
│   └── privacy
│       └── provider.php
├── lang
│   └── en
│       └── quizaccess_delaybetweenattempts.php
├── tests
│   └── rule_test.php
├── rule.php
└── version.php
```

### rule.php (required)

Defines class `quizaccess_<name>` extending `mod_quiz\local\access_rule_base`. Every method is optional to override; which ones depends on the rule's behaviour. Full PHPDoc lives in `mod/quiz/classes/local/access_rule_base.php`. Do not use the legacy base file `mod/quiz/accessrule/accessrulebase.php` — removed in 5.1.

```php
// mod/quiz/accessrule/pluginname/rule.php (GPL boilerplate and method PHPDoc omitted for brevity).
use mod_quiz\local\access_rule_base;
use mod_quiz\quiz_settings;

/**
 * Example quiz access rule.
 *
 * @package    quizaccess_pluginname
 * @copyright  2026 Your Name <you@example.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */
class quizaccess_pluginname extends access_rule_base {
    // Return an instance of this rule if it applies to the quiz, otherwise null.
    public static function make(quiz_settings $quizobj, $timenow, $canignoretimelimits) {
        if (empty($quizobj->get_quiz()->pluginnameenabled)) {
            return null;
        }
        return new self($quizobj, $timenow);
    }

    // False to allow access, or a string explaining why access is blocked.
    public function prevent_access() {
        return false;
    }

    // Text shown on the quiz view page.
    public function description() {
        return get_string('ruledescription', 'quizaccess_pluginname');
    }

    // Add rule settings to the quiz settings form.
    public static function add_settings_form_fields(mod_quiz_mod_form $quizform, MoodleQuickForm $mform) {
        $mform->addElement('selectyesno', 'pluginnameenabled',
            get_string('pluginnameenabled', 'quizaccess_pluginname'));
    }

    // Save rule settings when the quiz form is saved.
    public static function save_settings($quiz) {
        global $DB;
        if (empty($quiz->pluginnameenabled)) {
            $DB->delete_records('quizaccess_pluginname', ['quiz' => $quiz->id]);
        } else if (!$DB->record_exists('quizaccess_pluginname', ['quiz' => $quiz->id])) {
            $DB->insert_record('quizaccess_pluginname', ['quiz' => $quiz->id, 'enabled' => 1]);
        }
    }

    // Delete rule settings when the quiz is deleted.
    public static function delete_settings($quiz) {
        global $DB;
        $DB->delete_records('quizaccess_pluginname', ['quiz' => $quiz->id]);
    }

    // SQL fragments merged into the quiz settings query.
    public static function get_settings_sql($quizid) {
        return [
            'pluginname.enabled AS pluginnameenabled',
            'LEFT JOIN {quizaccess_pluginname} pluginname ON pluginname.quiz = quiz.id',
            [],
        ];
    }
}
```

**Note:** the devdocs only show the default `['', '', []]`; the `get_settings_sql()` contract above comes from the PHPDoc of `mod/quiz/classes/local/access_rule_base.php` (MOODLE_503_STABLE): `quiz` is the table alias for `{quiz}`; return `[fields, joins, params]` where fields are aliased to start with the plugin name, joins should normally be `LEFT JOIN`s, and params use named placeholders prefixed with the plugin name. `save_settings($quiz)` is called from `quiz_after_add_or_update()` with the quiz form data (including `$quiz->id`); `delete_settings($quiz)` from `quiz_delete_instance()`. The table and column names are this example's own. The constructor `__construct($quizobj, $timenow)` sets `$this->quizobj`, `$this->quiz` (`$quizobj->get_quiz()`) and `$this->timenow`.

**Overridable `access_rule_base` methods** (default behaviour in brackets):

| Method | Purpose |
|---|---|
| `static make(quiz_settings $quizobj, $timenow, $canignoretimelimits)` | Factory; `null` means rule does not apply [`null`]. |
| `prevent_new_attempt($numprevattempts, $lastattempt)` | Block starting another attempt: return `false` to allow, or a string message explaining why not [`false`]. |
| `prevent_access()` | Block starting or continuing an attempt now: return `false` to allow, or a string message explaining why not [`false`]. |
| `is_preflight_check_required($attemptid)` | Show a check form before attempting [`false`]. |
| `add_preflight_check_form_fields(preflight_check_form $quizform, MoodleQuickForm $mform, $attemptid)` | Fields for that form. |
| `validate_preflight_check($data, $files, $errors, $attemptid)` | Return `$errors`. |
| `notify_preflight_check_passed($attemptid)` / `current_attempt_finished()` | Lifecycle notifications. |
| `description()` | Text shown on the quiz view page [`''`]. |
| `is_finished($numprevattempts, $lastattempt)` | No more attempts possible [`false`]. |
| `end_time($attempt)` / `time_left_display($attempt, $timenow)` | Attempt deadline / countdown [`false`]. |
| `attempt_must_be_in_popup()` / `get_popup_options()` / `setup_attempt_page($page)` | Attempt page presentation. |
| `get_superceded_rules()` | Names of rules this rule replaces [`[]`]. |
| `static add_settings_form_fields(mod_quiz_mod_form $quizform, MoodleQuickForm $mform)` | Quiz settings form fields. |
| `static validate_settings_form_fields(array $errors, array $data, $files, mod_quiz_mod_form $quizform)` | Return `$errors`. |
| `static get_browser_security_choices()` | Options for the browser security setting [`[]`]. |
| `static save_settings($quiz)` / `static delete_settings($quiz)` | Persist/remove rule settings. |
| `static get_settings_sql($quizid)` | `[selects, joins, params]` [`['', '', []]`]. |
| `static get_extra_settings($quizid)` | Extra settings not loadable by SQL [`[]`]. |

### Overridable rules: rule_overridable

Optional. Implement `mod_quiz\local\rule_overridable` so your rule's settings can be overridden per user/group on the quiz overrides page (PHPDoc in `mod/quiz/classes/local/rule_overridable.php`). All interface methods must be implemented:

```php
use mod_quiz\form\edit_override_form;
use mod_quiz\local\access_rule_base;
use mod_quiz\local\rule_overridable;

class quizaccess_pluginname extends access_rule_base implements rule_overridable {
    public static function add_override_form_fields(edit_override_form $quizform, MoodleQuickForm $mform): void {
        $mform->addElement('select', 'plgnm_setting1',
            get_string('plgnm_setting1', 'quizaccess_pluginname'), ['A', 'B', 'C']);
        $mform->addElement('select', 'plgnm_setting2',
            get_string('plgnm_setting2', 'quizaccess_pluginname'), ['1', '2', '3']);
    }

    public static function get_override_form_section_header(): array {
        return ['name' => 'pluginname', 'title' => get_string('pluginname', 'quizaccess_pluginname')];
    }

    public static function get_override_form_section_expand(edit_override_form $quizform): bool {
        // Typically true if the quiz has existing rule settings.
        global $DB;
        return $DB->record_exists('quizaccess_pluginname', ['quiz' => $quizform->get_quiz()->id]);
    }

    public static function validate_override_form_fields(array $errors, array $data, array $files,
            edit_override_form $quizform): array {
        return $errors;
    }

    public static function save_override_settings(array $override): void {
        global $DB;
        $record = (object) [
            'overrideid' => $override['overrideid'],
            'setting1' => $override['plgnm_setting1'],
            'setting2' => $override['plgnm_setting2'],
        ];
        $table = 'quizaccess_pluginname_overrides';
        if ($id = $DB->get_field($table, 'id', ['overrideid' => $override['overrideid']])) {
            $record->id = $id;
            $DB->update_record($table, $record);
        } else {
            $DB->insert_record($table, $record);
        }
    }

    public static function delete_override_settings($quizid, $overrides): void {
        global $DB;
        $ids = array_column($overrides, 'id');
        [$insql, $inparams] = $DB->get_in_or_equal($ids);
        $DB->delete_records_select('quizaccess_pluginname_overrides', "id $insql", $inparams);
    }

    public static function get_override_setting_keys(): array {
        return ['plgnm_setting1', 'plgnm_setting2'];
    }

    public static function get_override_required_setting_keys(): array {
        return ['plgnm_setting1'];
    }

    public static function get_override_settings_sql($overridetablename): array {
        return [
            'plgnm.setting1 plgnm_setting1, plgnm.setting2 plgnm_setting2',
            "LEFT JOIN {quizaccess_pluginname_overrides} plgnm ON plgnm.overrideid = {$overridetablename}.id",
            [],
        ];
    }

    public static function add_override_table_fields($override, $fields, $values, $context): array {
        if (!empty($override->plgnm_setting1)) {
            $fields[] = get_string('pluginname', 'quizaccess_pluginname');
            $values[] = "{$override->plgnm_setting1}, {$override->plgnm_setting2}";
        }
        return [$fields, $values];
    }
}
```

**Gotcha:** form element names, `get_override_setting_keys()`, `save_override_settings()` keys and the `get_override_settings_sql()` aliases must all match (the upstream example mixes `plgn_` and `plgnm_`; use one prefix).

Related mod_quiz changes: quiz attempts gained `NOT_STARTED` and `SUBMITTED` states and pre-creation (5.0, MDL-68806; `quiz_attempt::process_finish()` deprecated in favour of `process_submit()` + `process_grade_submission()`); quiz overrides are cached via `\mod_quiz\local\quiz_overrides_cache_manager` (5.2, MDL-86493); `override_manager::get_effective_open_close_times()` is a deprecated wrapper for `get_effective_times()` (5.3); a `duedate` field was added to `quiz` and `quiz_overrides` (5.3, MDL-82521).

## Core examples to copy from

- `question/type/match` — defaults-for-new-questions, privacy provider, Behat patterns, `restore_qtype_match_plugin`.
- `question/type/truefalse`, `question/type/multianswer`, `question/type/calculated` — restore identity-hash overrides.
- `question/bank/managecategories` — filter options (PHP condition + JS filter class), `restore_filtercondition()`.
- `question/bank/tagquestion` — context-sensitive filter condition.
- `question/bank/viewquestiontype` — custom JS filter type with textual values.
- `question/bank/bulkmove` — bulk action with `get_action_icon()`.
- `mod/quiz/accessrule/delaybetweenattempts`, `password`, `seb` — access rules.
