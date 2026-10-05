# Backup and Restore API (Moodle 5.3)

Quick reference for including plugin data in course backups (Backup API) and restoring it from backup files (Restore API), per the Moodle 5.3 subsystem docs plus the 5.0–5.3 UPGRADING notes (`core_backup`, `core_question`, `mod_subsection`).

**Note:** The Backup and Restore API pages are unchanged between the 4.5 and 5.3 docs. All 5.x changes below come from UPGRADING notes. Since 5.1 (MDL-83424) code lives under `public/` in a git checkout, so `mod/foobar/backup/moodle2/` is `public/mod/foobar/backup/moodle2/` at repository level; `$CFG->dirroot`-relative paths (as in the `require_once` calls below) are unchanged.

## Concepts

- A **backup plan** consists of **backup tasks**; each task consists of one or more **backup steps**. A plugin developer implements one backup task for the plugin's data.
- Most plugins use a single step: the one that creates the XML file with data from the plugin's tables. Some activities need more (e.g. Quiz uses three steps: quiz settings plus the questions used by the quiz).
- Restore mirrors this: a **restore plan** of **restore tasks**, each with one or more **restore steps**. Most plugins use a single restore step that parses the plugin XML file and puts the data into its tables.
- The backup code organisation differs subtly between plugin types (activity modules, blocks, themes, course reports).

## Backup API — activity modules

### File layout

All backup files live in `backup/moodle2/` inside the plugin folder, e.g. `mod/foobar/backup/moodle2/`:

- `backup_foobar_activity_task.class.php` — the activity task class
- `backup_foobar_stepslib.php` — all backup step classes
- `backup_foobar_settingslib.php` — optional; only if the module declares its own backup settings beyond the core-provided common activity settings. Most modules do not need this.

Excerpt of the example layout in the activity module docs (`mod_folder`), including the legacy 1.x converter and a restore test:

```console
mod/folder/
├── backup
│   ├── moodle1
│   │   └── lib.php
│   └── moodle2
│       ├── backup_folder_activity_task.class.php
│       ├── backup_folder_stepslib.php
│       ├── restore_folder_activity_task.class.php
│       └── restore_folder_stepslib.php
└── tests
    └── backup
        └── restore_date_test.php
```

### Backup task class

`backup_foobar_activity_task.class.php` must provide a single class `backup_foobar_activity_task` extending `backup_activity_task`. Three methods must be defined:

- `protected function define_my_settings()` — add settings defined in `backup_foobar_settingslib.php`. Most modules leave the body empty.
- `protected function define_my_steps()` — typically one or more `$this->add_step()` calls defining the task as a sequence of steps.
- `static public function encode_content_links($content)` — URLs referencing the activity instance (e.g. `http://my.moodle.site/mod/foobar/view.php?id=42`) are not valid after restore elsewhere, so the backup stores them encoded in a transportable form. Restore applies the reverse process, replacing encoded URLs with new ones valid for the target site.

### Backup structure step class

Steps added in `define_my_steps()` are implemented in `backup_foobar_stepslib.php`. Most plugins define a single class `backup_foobar_activity_structure_step` extending `backup_activity_structure_step`, implementing `protected function define_structure()`, which must:

1. Create `backup_nested_element` instances describing the plugin's data.
2. Connect them into a hierarchy with their `add_child()` method.
3. Set data sources on the elements via methods like `set_source_table()` or `set_source_sql()`.

The method must return the root `backup_nested_element` processed by `prepare_activity_structure()` (which wraps your structure in a common envelope).

### Activities that use questions (5.1+)

Since 5.1 (MDL-41924): question backup only includes questions actually used or owned by the backed-up activities. Activities using question references are handled automatically. Activities using *question set references* (e.g. random quiz questions) must call `backup_question_set_reference_trait::annotate_set_reference_bank_entries()` alongside `backup_question_set_reference_trait::add_question_set_references()` in their backup step; see `backup_quiz_activity_structure_step::define_structure()` for an example.

## Backup API — blocks

### File layout

Files live in `blocks/foobar/backup/moodle2/`:

- `backup_foobar_block_task.class.php` — required
- `backup_foobar_stepslib.php` — only if the block has its own DB tables to back up
- `backup_foobar_settingslib.php` — only if the block declares its own backup settings (rare)

### Block task class

Class `backup_foobar_block_task` extends `backup_block_task`. Five methods must be defined:

- `protected function define_my_settings()` — usually empty.
- `protected function define_my_steps()` — empty if the block has no own tables; otherwise `$this->add_step()` calls.
- `public function get_fileareas()` — returns the array of file area names within the block context.
- `public function get_configdata_encoded_attributes()` — blocks usually store data in the configuration tables (see `instance_config_save()`); this returns the config elements that must be processed before storage, typically ones holding links to embedded media. Most blocks return an empty array.
- `static public function encode_content_links($content)` — encode URLs referencing the block instance; most blocks return `$content` unmodified.

## Backup API — admin tools

Files live in `admin/tool/foobar/backup/moodle2/`. The file `backup_tool_foobar_plugin.class.php` provides a class extending `backup_tool_plugin`. Minimal example from the docs:

**Note:** The upstream page is inconsistent: its prose names the class `backup_tool_foobar_task`, while the example (and the file name) use `backup_tool_foobar_plugin`; the example also returns an undefined `$plugin`, and the docs do not show what the method should return. Check a core `backup_tool_plugin` subclass before relying on it.

```php
require_once($CFG->dirroot . '/backup/moodle2/backup_tool_plugin.class.php');

class backup_tool_foobar_plugin extends backup_tool_plugin {

    protected function define_course_plugin_structure() {
        $this->step->log('Yay, backup!', backup::LOG_DEBUG);
        return $plugin;
    }

}
```

Note the step-level logging call: `$this->step->log($message, backup::LOG_DEBUG)`.

## Backup API — themes and reports

The 5.3 docs only link out to legacy wiki pages ("Backup 2.0 theme data", "Backup 2.0 course report data") without inline content.

## Restore API — activity modules

### File layout

Restore files live in the same `backup/moodle2/` folder as the backup files, e.g. `mod/foobar/backup/moodle2/`:

- `restore_foobar_activity_task.class.php` — the activity task class
- `restore_foobar_stepslib.php` — all restore step classes

The official activity-module restore section ends here (the 5.3 page is still marked "(to be continued)").

Restore notes from UPGRADING:

- Since 5.2 (MDL-86524): when restoring a question set reference, ID mapping inside its `filtercondition` is delegated to qbank plugins. A qbank plugin whose filter condition stores database IDs must override `restore_filtercondition()` in its `condition` class and replace the IDs with mapped values; see `qbank_managecategories\category_condition`.
- Changed in 5.2 (MDL-87621): restoring `mod_subsection` ignores subsection descriptions (legacy summaries are no longer restored).
- 5.2: `qtype_random` was removed from core; `core\component::is_valid_plugin_name` blocks new plugins of that name so old backups containing random questions can still be restored.

## Restore API — admin tools

Files live in `admin/tool/foobar/backup/moodle2/`. The class extends `restore_tool_plugin`. Minimal example from the docs. **Note:** the upstream prose is garbled here: it says file `backup_tool_foobar_plugin.class.php` provides class `restore_tool_foobar_task`, while the example class is `restore_tool_foobar_plugin`; the docs give no correct restore file name.

```php
require_once($CFG->dirroot . '/backup/moodle2/restore_tool_plugin.class.php');

class restore_tool_foobar_plugin extends restore_tool_plugin {

    protected function define_course_plugin_structure() {
        $paths = [];
        $this->step->log('Yay, restore!', backup::LOG_DEBUG);
        return $paths;
    }

}
```

`define_course_plugin_structure()` returns an array of paths (element paths the step will process); logging is again via `$this->step->log()`.

## Question type plugins: restore matching (since 4.4.6 / 5.0)

Since MDL-83541, restored questions are matched against existing ones in the target category by a SHA1 hash of their `questiondata` structure (ID fields excluded) to avoid duplicates. Question type plugins that back up extra data must override `restore_qtype_plugin::convert_backup_to_questiondata()`, `define_excluded_identity_hash_fields()` and/or `remove_excluded_question_data()`; check with `mod/quiz/tests/backup/repeated_restore_test.php`. Changed in 5.2 (MDL-85975): `restore_qtype_plugin::unset_excluded_fields()` now returns the modified structure, so overrides of `remove_excluded_question_data()` that skip the parent must use the return value.

Full details: the "Question type plugin restore code" docs page (`docs/apis/plugintypes/qtype/restore.md`), covered in `moodle-plugin-development/references/question-types.md`.

## Core backup changes in 5.x

- Added in 5.0 (MDL-83479): hooks in the restore/copy process to (1) define extra course-restore settings, (2) add extra fields to the copy-course form, (3) extend the required fields list in `copy_helper::process_formdata()`, (4) interact with the copy task before it runs. The UPGRADING note does not give the hook class names; find them in the site's hooks list (see `hooks-di.md`). Related: `base_task::add_setting()` is now public so hook callbacks can add settings, and restore settings are included in the data of the `course_restored` event.
- Removed in 5.0 (MDL-66873): all `MODE_HUB` related code.
- Removed in 5.2 (MDL-87425): `\base_controller::set_copy()` and `\base_controller::get_copy()` (`backup/controller/base_controller.class.php`).
- 5.3: backup admin settings classes renamed (old names still work), e.g. `\admin_setting_special_backupdays` is now `\core_admin\setting\setting\special_backupdays` and `\admin_setting_special_backup_auto_destination` is now `\core_admin\setting\setting\special_backup_auto_destination`.

## Restoring from Moodle 1.x backups

Supported via the separate Backup conversion API (`backup/moodle1/lib.php` in the plugin; legacy wiki page, not documented in the 5.3 docs tree).

## Coverage note (doc-limited)

The official 5.3 pages for this subsystem are brief quick-references and defer detail to the legacy wiki ("Backup 2.0", "Backup 2.0 for developers", "Restore 2.0 for developers"). The following commonly needed topics are **not covered** by the 5.3 docs and are therefore omitted here rather than reconstructed from memory:

- File and id annotations in backup steps, and user-info (with/without user data) handling
- Restore structure step internals: path definitions, per-path processing methods, applying the activity instance, old-to-new id mapping, post-execution work and restoring related files
- Restore task methods for decoding content links and for restore log rules
- Backup/restore settings classes (settingslib) beyond the note that they exist and are rarely needed
- Backup logging beyond the single `$this->step->log()` call shown above

For those, read existing core implementations under `mod/*/backup/moodle2/` (e.g. `mod/quiz/backup/moodle2/`, `mod/folder/backup/moodle2/`) or the legacy developer wiki tutorials referenced by the docs ("Backup 2.0 for developers", "Restore 2.0 for developers").
