# File API

Compact reference for the Moodle 5.3 File API: file areas, `file_storage`/`stored_file`, saving/retrieving files, the `before_file_created` hook, the `PLUGIN_pluginfile()` serving callback, draft areas for forms, and pluginfile URL generation. Distilled from the Moodle 5.3 developer docs (the File API page is unchanged since 4.5) plus the 5.0–5.3 `UPGRADING.md` notes for `core_files`.

**What changed since 4.5:** `file_clear_draft_area()` added in 5.0; `\file_storage::content_exists()` and `\file_storage::try_content_recovery()` removed in 5.2 (see files-advanced.md); since 5.1 the serving scripts (`pluginfile.php`, `draftfile.php`, `tokenpluginfile.php`) and `lib/filestorage/` live under `public/` in a git checkout. Plugin code and URLs are unaffected.

## File areas

Files are stored in _file areas_, uniquely identified by:

- `contextid`
- full component name (Frankenstyle: `course`, `mod_forum`, `block_html`, ...)
- file area type (e.g. `intro`, `post`)
- `itemid` — `0` if there is only one instance of the area per context; otherwise the id of the related record (e.g. forum post id).

File areas are not registered anywhere; they exist implicitly in the `files` table. Each plugin/subsystem must only access its **own** file areas — access another component's files through that component's API. Area names: prefer singular common names (`intro`, `post`, `attachment`, `description`).

Internals: metadata lives in the `files` DB table (one row per usage; rows with filename `.` are directories); content is stored once per SHA1 `contenthash` in `$CFG->dataroot/filedir/ab/cd/<contenthash>`. The low-level API is `\file_storage`, obtained with `get_file_storage()`.

## Creating files

Every create call needs a `fileinfo` record:

```php
$fileinfo = [
    'contextid' => $context->id,   // ID of the context.
    'component' => 'mod_mymodule', // Your component name.
    'filearea'  => 'myarea',       // Usually = table name.
    'itemid'    => 0,              // Usually = ID of row in table.
    'filepath'  => '/',            // Any path beginning and ending in /.
    'filename'  => 'myfile.txt',   // Any filename.
];
```

```php
$fs = get_file_storage();

$fs->create_file_from_pathname($fileinfo, $requestdir . '/helloworld.txt'); // From a file on disk.
$fs->create_file_from_url($fileinfo, 'https://example.com/helloworld.txt'); // From a URL.
$fs->create_file_from_string($fileinfo, 'hello world');                     // From a string.
$fs->create_file_from_storedfile($fileinfo, $existingfile);                 // From another stored_file.
```

### Before file creation hook (since 4.5, MDL-83245)

The `\core_files\hook\before_file_created` hook lets a plugin inspect or modify content before it is stored. Register a callback in `db/hooks.php`:

```php
$callbacks = [
    [
        'hook' => \core_files\hook\before_file_created::class,
        'callback' => [\local_myplugin\local\hook_callbacks::class, 'before_file_created'],
    ],
];
```

In the callback:

- `get_filerecord()` returns the file metadata (`stdClass|null`).
- Content supplied as a file: `has_filepath()` / `get_filepath()`; content supplied as a string: `has_filecontent()` / `get_filecontent()`.
- Supply replacement content with `update_filepath()` or `update_filecontent()`.
- Throw an exception to reject the file (for example when enforcing a quota).

## Retrieving and using files

```php
$fs = get_file_storage();

// One file (returns stored_file or false).
$file = $fs->get_file($context->id, 'mod_myplugin', $filearea, $itemid, $filepath, $filename);

// All files in an area (array of stored_file, includes directory entries).
$files = $fs->get_area_files($contextid, 'mod_myplugin', 'post', $post->id);

// Content.
$contents = $file->get_content();

// Read-only PHP file handle — you cannot write to it, and must fclose() it.
$fh = $file->get_content_file_handle();
while ($line = fgets($fh)) {
    // ...
}
fclose($fh);

// Copy out to disk (direct filedir access is not allowed).
$requestdir = make_request_directory();
$file->copy_content_to("{$requestdir}/helloworld.txt");

// Delete.
$file->delete();
```

Move/rename within the same context+filearea+itemid:

```php
$file->rename('/newpath/', '/newname.txt');
```

Move to a different context/filearea/itemid — create from the old file, then delete the original:

```php
$fs = get_file_storage();
$filerecord = [
    'contextid'    => $file->get_contextid(),
    'component'    => $file->get_component(),
    'filearea'     => 'newfilearea',
    'itemid'       => 0,
    'filepath'     => '/newpath/',
    'filename'     => $file->get_filename(),
    'timecreated'  => time(),
    'timemodified' => time(),
];
$fs->create_file_from_storedfile($filerecord, $file);
$file->delete();
```

Document conversion (requires `unoconv` installed and configured on the site, so not available everywhere): `$convertedfile = $fs->get_converted_document($file, 'pdf');`. For new code prefer the asynchronous `\core_files\converter` API (files-advanced.md).

## Serving files

Serving has two halves — URL generation and a serving callback — and you must implement both.

### URL generation

```php
$url = moodle_url::make_pluginfile_url(
    $file->get_contextid(),
    $file->get_component(),
    $file->get_filearea(),
    $file->get_itemid(),
    $file->get_filepath(),
    $file->get_filename(),
    false                     // forcedownload.
);
```

Passing `null` for `itemid` omits it from the URL entirely — your serving callback must then not expect it in `$args`. Serving scripts: `pluginfile.php` (normal), `tokenpluginfile.php` (no login, e.g. links in email), `draftfile.php` (user draft areas).

### The pluginfile callback (lib.php)

The script authenticates the user, parses the URL, and calls `[component]_pluginfile()` in your plugin's `lib.php`. The plugin is responsible for access control (and forcing download of untrusted user-submitted files for XSS safety).

```php
// mod/myplugin/lib.php
/**
 * @return bool false if the file not found, just send the file otherwise and do not return anything
 */
function mod_myplugin_pluginfile(
    $course,
    $cm,
    $context,
    string $filearea,
    array $args,          // [itemid, ...path, filename]
    bool $forcedownload,
    array $options = []
): bool {
    global $DB;

    // Check the contextlevel is as expected - for a block this becomes CONTEXT_BLOCK, etc.
    if ($context->contextlevel != CONTEXT_MODULE) {
        return false;
    }

    // Make sure the filearea is one of those used by the plugin.
    if ($filearea !== 'expectedfilearea' && $filearea !== 'anotherexpectedfilearea') {
        return false;
    }

    // Non course-module plugins leave out the 'cm' part.
    require_login($course, true, $cm);

    if (!has_capability('mod/myplugin:view', $context)) {
        return false;
    }

    // $args is [itemid, path...]; use the itemid to verify the record belongs to this context.
    $itemid = array_shift($args);
    if ($filearea === 'expectedfilearea') {
        $post = $DB->get_record('myplugin_posts', ['id' => $itemid]);
        if ($post->myplugin !== $context->instanceid) {
            return false;
        }
        // Additional checks: groups, hidden records, etc. Return false to deny.
    }
    // If your URLs have no itemid, use $itemid = null; instead of array_shift.

    $filename = array_pop($args);
    if (empty($args)) {
        $filepath = '/';
    } else {
        $filepath = '/' . implode('/', $args) . '/';
    }

    $fs = get_file_storage();
    $file = $fs->get_file($context->id, 'mod_myplugin', $filearea, $itemid, $filepath, $filename);
    if (!$file) {
        return false;
    }

    // Cache lifetime of 1 day, no filtering.
    send_stored_file($file, DAY_SECS, 0, $forcedownload, $options);
}
```

## Draft areas and forms

Form uploads (`filepicker`, `filemanager`, `editor` elements) land in a per-user _draft_ file area in the `user` component; you move them into your real area on submission.

### filemanager: load existing files, then save

```php
// Before displaying the form: copy existing files into a draft area.
$draftitemid = file_get_submitted_draft_itemid('attachments');

file_prepare_draft_area(
    $draftitemid,          // Target draft area.
    $context->id,          // Source: contextid / component / filearea / itemid.
    'mod_glossary',
    'attachment',
    $entry->id,
    ['subdirs' => 0, 'maxbytes' => $maxbytes, 'maxfiles' => 50]
);

$entry->attachments = $draftitemid;
$mform->set_data($entry);
```

```php
// On submission: move draft files into the real area.
if ($data = $mform->get_data()) {
    // ... store or update $entry first.
    file_save_draft_area_files(
        $data->attachments,    // Draft area itemid from the form.
        $context->id,
        'mod_glossary',
        'attachment',
        $entry->id,
        ['subdirs' => 0, 'maxbytes' => $maxbytes, 'maxfiles' => 50]
    );
}
```

### Editor content with embedded files

Editor HTML references draft files by full URL; in storage the URL prefix is replaced by the `@@PLUGINFILE@@` placeholder. Simple pattern — DB fields `textfield`, `textfieldformat` (and optionally `textfieldtrust`), plus options:

```php
$textfieldoptions = [
    'trusttext' => true,
    'subdirs' => true,
    'maxfiles' => $maxfiles,
    'maxbytes' => $maxbytes,
    'context' => $context,
];

$mform->addElement('editor', 'textfield_editor', get_string('fieldname', 'somemodule'), null, $textfieldoptions);

// Before display: copy files to draft area, rewrite placeholders to draft URLs.
$data = file_prepare_standard_editor($data, 'textfield', $textfieldoptions,
    $context, 'mod_somemodule', 'somearea', $data->id);

// After submission: move files to real area, rewrite URLs back to @@PLUGINFILE@@.
$data = file_postupdate_standard_editor($data, 'textfield', $textfieldoptions,
    $context, 'mod_somemodule', 'somearea', $data->id);
```

### Clearing a draft area (since 5.0, MDL-72050)

`file_clear_draft_area()` deletes the files in a draft area. The UPGRADING note gives no signature; check its definition in core before use.

### Low-level helpers

Low level: `file_prepare_draft_area()` (with a `$text` argument) returns the text with links made absolute; `file_save_draft_area_files()` (with the text as its last argument) merges files back and returns the text with links relativised. When displaying stored content, pass it through `file_rewrite_pluginfile_urls()` to turn `@@PLUGINFILE@@` back into servable URLs, and serve via your pluginfile callback. Real-world examples: `mod/glossary/edit.php`, `mod/glossary/comment.php`.
