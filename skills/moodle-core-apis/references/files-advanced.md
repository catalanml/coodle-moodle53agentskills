# Files: Advanced APIs (Browsing, Conversion, Internals) + DML Extras

Advanced/supplemental Moodle 5.3 APIs: file browsing, document conversion, file storage internals for debugging, DML driver query logging, database schema references, and privacy API test CLI scripts. Distilled from the Moodle 5.3 developer docs (all these pages are unchanged since 4.5) plus the 5.0–5.3 `UPGRADING.md` notes for `core_files` and `core`.

## File Browser API

Supplemental API to fetch information about files stored via the File API. Entry point is `get_file_browser()`, which returns a `file_browser`; `get_file_info()` returns a `file_info` object (or null) supporting navigation such as `get_parent()` and `get_visible_name()`.

Example — build a breadcrumb trail of a file's parent folders:

```php
public function get_file_breadcrumbs(\stored_file $file): ?array {
    $browser = get_file_browser();

    $fileinfo = $browser->get_file_info(
        \context::instance_by_id($file->get_contextid()),
        $file->get_component(),
        $file->get_filearea(),
        $file->get_itemid(),
        $file->get_filepath(),
        $file->get_filename()
    );

    if ($fileinfo) {
        $level = $fileinfo->get_parent();
        while ($level) {
            $path[] = ['name' => $level->get_visible_name()];
            $level = $level->get_parent();
        }
        return array_reverse($path);
    }
    return null;
}
```

## Document Converter API

Converts stored files to other formats (canonical example: `assignfeedback_editpdf` converting submissions to PDF for annotation). Core ships two converter plugins: Unoconv and Google Drive. The API tries multiple plugins with automatic fallback, and is designed to be called **asynchronously**.

Conversions are managed by `\core_files\converter`; a single conversion is a `\core_files\conversion`. Always go through the converter API.

```php
// Start (or fetch an existing matching) conversion to PDF.
$converter = new \core_files\converter();
$conversion = $converter->start_conversion($file, 'pdf');

// Force a fresh conversion (rarely needed).
$conversion = $converter->start_conversion($file, 'pdf', true);

// start_conversion() polls automatically, but you can poll explicitly:
$converter->poll_conversion($conversion);
```

Check status — four states:

```php
switch ($conversion->get_status()) {
    case \core_files\conversion::STATUS_COMPLETE:
        // The file is ready.
        $file = $conversion->get_destfile(); // stored_file result.
        break;
    case \core_files\conversion::STATUS_PENDING:      // Not started yet.
    case \core_files\conversion::STATUS_IN_PROGRESS:  // Show spinner.
        break;
    case \core_files\conversion::STATUS_FAILED:       // Permanent failure.
        break;
}
```

To provide a new conversion backend, write a `fileconverter` plugin (see plugin type docs).

## File Storage Internals (debugging)

Layered design:

- **File system** — low-level content storage, no access control. Files are stored/retrieved by `contenthash` = SHA1 of the file content (plus a file-length check as a cheap collision guard). Identical content is stored once no matter how many uses.
- **File storage** — metadata, via the `\file_storage` class: `$fs = get_file_storage();`. Works with the File System API and `\stored_file`.
- **File serving** — `draftfile.php` (draft area), `pluginfile.php` (plugin content), `tokenpluginfile.php` (token access for not-logged-in contexts, e.g. links in email). The plugin is responsible for access control and XSS protection (force download for untrusted user-submitted files). Each plugin should only use the File Storage API for its **own** files.

Default on-disk layout: `$CFG->dataroot` `moodledata/filedir/<aa>/<bb>/<contenthash>` where `aa`/`bb` are the first two byte-pairs of the hash. Example: contenthash `081371cb102fa559e81993fddc230c79205232ce` lives at `moodledata/filedir/08/13/081371cb...`. Alternative file systems (e.g. S3) can replace the default.

Corruption check — the filename must equal the file's `sha1sum`:

```console
$ cd /moodlepath/moodledata/filedir/1d/df/
$ sha1sum 1ddf5b375fcb74929cdd7efda4f47efc61414edf
1ddf5b375fcb74929cdd7efda4f47efc61414edf  1ddf5b375fcb74929cdd7efda4f47efc61414edf   # OK: matches
```

The `files` DB table holds one row per **usage** of a file (same image in a profile and a forum post = two rows, one disk copy). Rows with filename `.` are directory entries, auto-created. (Table is named `files` plural because `file` is an SQL reserved word.)

Editor/form integration: `filepicker` and `filemanager` form field types; HTML editor files go to a draft area in the `user` component during editing and move to the real area on form submit.

### core_files changes in 5.0–5.3

- **Removed in 5.2** (MDL-87426) from `public/lib/filestorage/file_storage.php`: `\file_storage::content_exists()` and `\file_storage::try_content_recovery()`. Do not call them; there is no documented replacement.
- **Added in 5.0** (MDL-81437): adhoc task `core_files\task\asynchronous_mimetype_upgrade_task` upgrades file mimetypes asynchronously during core upgrades; upgradelib provides `upgrade_create_async_mimetype_upgrade_task` to create it. The note describes it for core upgrades; it does not document use from plugin upgrade steps.
- **Added in 5.0** (MDL-72050): `file_clear_draft_area()` deletes the files in a draft area.
- **Added in 5.2** (MDL-87822): `curl::removeopt()` removes an option previously set with `setopt()`; passing `'CURLOPT_USERPWD' => false` in the `$options` of `curl::put()` removes that option from the request.

## DML Drivers

Native drivers in 5.3: MySQLi, MariaDB, PostgreSQL and Microsoft SQL Server. The 5.3 drivers page still lists Oracle, but Oracle support was removed in 5.0 (MDL-83172). Minimum versions for 5.3: PostgreSQL 17, MySQL 8.4, MariaDB 11.4, SQL Server 2019.

Read replicas (5.0, MDL-71257): `want_read_slave()` / `perf_get_reads_slave()` in `lib/dml/moodle_database.php` are deprecated in favour of `replica`-named versions, and `moodle_read_slave_trait` in favour of `moodle_read_replica_trait` (functionally identical; `master` becomes `primary`). Custom drivers or code calling these must switch to the new names.

Query logging to a DB table can be enabled in `config.php`:

```php
$CFG->dboptions = [
    'dbpersist' => 0,
    //'logall'   => true,  // Log all queries - developers only, high load.
    'logslow'  => 5,       // Log queries slower than N seconds (float ok).
    'logerrors'  => true,  // Log all error queries.
];
```

## Database Schema

No official generated ER diagram for 5.3 (the schema page is unchanged). Community-maintained web views exist at examulator.com (latest published: Moodle 4.0 at `http://www.examulator.com/er/4.0`). The authoritative schema source is the XMLDB definitions (`db/install.xml` files) in the codebase.

## Privacy API Test Scripts (CLI)

Development-only helper scripts (not a substitute for unit tests). The docs say to place them in the root of your Moodle development installation and run them via CLI; they `require` `config.php` from the same directory. Since 5.1 the real `config.php` stays in the install root and `public/config.php` is a loader stub that requires `../config.php` (verified in MOODLE_503_STABLE source), so either directory works. All follow the same pattern: `define('CLI_SCRIPT', true); require(__DIR__ . '/config.php');`, then impersonate a user:

```php
$user = \core_user::get_user(2); // Or get_user_by_username(...).
\core\session\manager::init_empty_session();
\core\session\manager::set_user($user);
```

**Compliance check** — iterate components from `\core_privacy\manager` (its `get_component_list()` / `component_implements()` methods are private; the script accesses them via `ReflectionClass` + `setAccessible(true)`), call `$manager->component_is_compliant($component)`, then for compliant plugins inspect: `null_provider` (`$classname::get_reason()`), `metadata\provider` (`$classname::get_metadata(new \core_privacy\local\metadata\collection($component))` — zero items is an error), `user_preference_provider` (each preference must be described in metadata), and `core_user_data_provider` (run a test export via `get_contexts_for_userid()` + `approved_contextlist` + `export_user_data()`).

**Test export** — build an approved list from every component's contexts, export, and unpack:

```php
$manager = new \core_privacy\manager();
$approvedlist = new \core_privacy\local\request\contextlist_collection($user->id);
foreach ($manager->get_contexts_for_userid($user->id) as $contextlist) {
    $approvedlist->add_contextlist(new \core_privacy\local\request\approved_contextlist(
        $user, $contextlist->get_component(), $contextlist->get_contextids()
    ));
}
$exportedcontent = $manager->export_user_data($approvedlist);
$exportpath = make_unique_writable_directory(make_temp_directory('privacy'), true);
get_file_packer()->extract_to_pathname($exportedcontent, $exportpath);
```

Note: the export script also needs `$PAGE = new moodle_page(); $OUTPUT = new core_renderer($PAGE, RENDERER_TARGET_GENERAL);` and `require_once("$CFG->libdir/clilib.php");` for `cli_get_params()` (`--username=` / `--userid=`).

**Test delete** — same approved-list construction, then:

```php
$trace = new text_progress_trace();
$contextlists = $manager->get_contexts_for_userid($user->id, $trace);
// ... build $approvedlist as above ...
$manager->delete_data_for_user($approvedlist, $trace);
```
