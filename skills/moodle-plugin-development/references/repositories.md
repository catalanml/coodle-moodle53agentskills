# Repository plugins (Moodle 5.3)

Structure and API for `repository_*` plugins that bring external content into Moodle via the file picker, per Moodle 5.3 developer docs and UPGRADING notes. A full repository has two parts: administration (how admins and users configure repositories) and file picker integration (communication with the external service and the picker display).

## Changes since 4.5

The devdocs repository page is unchanged since 4.5. From UPGRADING:

- Removed in 5.0 (MDL-78706): the base `repository::get_file_size()` method — do not call or rely on it.
- Removed in 5.2 (MDL-87425): `repository_onedrive::can_import_skydrive_files()` and `repository_onedrive::import_skydrive_files()`.
- Since 5.1 (MDL-83424) core lives under `public/`: in a git checkout repositories are at `public/repository/<name>/` and the base class at `public/repository/lib.php`. Code still uses `$CFG->dirroot . '/repository/lib.php'`.

## File structure

Located in `/repository/pluginname/`:

```
repository/pluginname/
|-- db
|   `-- access.php
|-- lang
|   `-- en
|       `-- repository_pluginname.php
|-- lib.php          (required: main repository class)
|-- pix
|   `-- icon.png
`-- version.php
```

`lib.php` must define a class extending core `\repository`:

```php
defined('MOODLE_INTERNAL') || die();

require_once($CFG->dirroot . '/repository/lib.php');

class repository_pluginname extends repository {
    public function get_listing($encodedpath = '', $page = '') {
        return ['list' => []];
    }

    public function supported_returntypes() {
        return FILE_INTERNAL | FILE_REFERENCE;
    }

    public function default_returntype() {
        return FILE_INTERNAL;
    }

    public function search($search, $page = 0) {
        return ['nologin' => true, 'list' => []];
    }
}
```

`db/access.php` capability example:

```php
$capabilities = [
    'repository/pluginname:view' => [
        'captype' => 'read',
        'contextlevel' => CONTEXT_MODULE,
        'archetypes' => [
            'coursecreator' => CAP_ALLOW,
            'editingteacher' => CAP_ALLOW,
            'manager' => CAP_ALLOW,
        ],
    ],
];
```

Language file must define `pluginname`, `configplugin`, `pluginname_help`:

```php
$string['pluginname'] = 'Example repository';
$string['configplugin'] = 'Configuration for Example repository';
$string['pluginname_help'] = 'A repository description';
```

## Course and user instances

A system-wide instance is created when the repository is enabled. To also allow course- and user-level instances:

1. Define `$string['enablecourseinstances']` and `$string['enableuserinstances']` in the lang file (see `repository_filesystem`).
2. Return at least one name from `get_instance_option_names()`; do not add those fields in `type_config_form()`.
3. Alternatively (several core repositories, e.g. `repository_wikimedia`), create the original instance in `db/install.php` by constructing a `repository_type` with the options array as its second constructor parameter.

## Return types: supported_returntypes(): int

Return any combination (bitwise OR) of:

- `FILE_INTERNAL` — file is copied into the Moodle file system.
- `FILE_EXTERNAL` — file stays in the external repository, served from there (only type mod_url accepts; never shown as a picker option elsewhere).
- `FILE_REFERENCE` — file stays external but may be cached/synchronised ("alias/shortcut" option in the picker).
- `FILE_CONTROLLED_LINK` — file stays external; ownership transferred to the OAuth 2 system account which grants access.

```php
function supported_returntypes() {
    return FILE_INTERNAL | FILE_EXTERNAL | FILE_REFERENCE | FILE_CONTROLLED_LINK;
}
```

The actual type used depends on the user's choice in the file picker (link → FILE_EXTERNAL, shortcut/alias → FILE_REFERENCE). Plugins embedding a picker can narrow the set (e.g. assignsubmission_file disallows FILE_EXTERNAL and FILE_REFERENCE).

## supported_filetypes()

Optional; hides the repository when it cannot supply the requested type. Return `'*'`, an array of mimetypes (`['image/gif', 'image/jpeg', 'image/png']`), or file groups (`['web_image']`) — see `core_filetypes`.

## Site-level (type) settings — all static, all optional

- `get_type_option_names(): array` — setting names shared by all instances: `return array_merge(parent::get_type_option_names(), ['rootpath']);`
- `type_config_form($mform, $classname = 'repository')` — add form elements; call `parent::type_config_form($mform)` first; read stored values with `get_config('repository_pluginname', 'rootpath')`.
- `type_form_validation($mform, $data, $errors)` — push `'fieldname' => 'message'` into `$errors` and return it.
- `plugin_init()` — called once when the admin adds the plugin (e.g. create a default instance via `repository::static_function`).

## Instance settings — all static, all optional

- `get_instance_option_names(): array` — per-instance setting names (e.g. `['fs_path']`). Empty array (the parent default) means a single site-wide repository in the picker. Returning at least one name is also required to support course/user instances (with `enablecourseinstances` / `enableuserinstances` lang strings).
- `instance_config_form($mform)` — add per-instance form fields (a name text box always exists).
- `instance_form_validation($mform, $data, $errors)` — same pattern as type validation.

**Worked pattern (devdocs "Flickr Public" example):** a per-account setting goes in instance options (`get_instance_option_names()` returns `['email_address']`, `instance_config_form()` adds the text box); a value shared by every instance goes in type options (`get_type_option_names()` returns `['api_key']`, `type_config_form()` adds it); `plugin_init()` uses `repository::static_function` to create a default instance the admin cannot delete.

Read/write settings from instance code with `$this->get_option('settingname')` and `$this->set_option(['settingname' => 'value'])` (not usable from static methods).

## Core methods

### __construct($repositoryid, $context = SYSCONTEXTID, $options = [], $readonly = 0)

Do initialisation (login via `optional_param`, load options). Always call `parent::__construct(...)`; it sets `$this->id`, `$this->context`, `$this->instance`, `$this->readonly`, `$this->options` (includes `'ajax'` bool and `'mimetypes'`), `$this->name`, `$this->returntypes`.

### get_listing($path = '', $page = '') — mandatory

Returns an array describing the current folder:

```php
public function get_listing($encodedpath = '', $page = '') {
    return [
        'path' => [                      // breadcrumb navigation
            ['name' => 'root', 'path' => '/'],
            ['name' => 'subfolder', 'path' => '/subfolder'],
        ],
        'manage' => 'http://webmgr.moodle.com',
        'list' => [
            ['title' => 'filename1', 'date' => '1340002147', 'size' => '10451213',
             'source' => 'http://www.moodle.com/dl.rar'],
            ['title' => 'folder', 'date' => '1340002147', 'size' => '0', 'children' => []],
        ],
    ];
}
```

Top-level keys (all optional except `list`): `path`, `dynload` (bool — fetch subfolders via ajax; return `children` as empty array), `page`/`pages` (pagination; `pages = -1` if unknown but next page exists; do not mix pagination with subfolders), `manage`, `help`, `nologin`, `norefresh`, `logouttext`, `nosearch`, `issearchresult`, `upload` (`['label' => ..., 'id' => ...]` to show an upload form), `object` (`['type' => 'text/html', 'src' => url]` to embed an external chooser instead of a list).

File entries in `list`: `title`, `shorttitle`, `date`/`datemodified`/`datecreated` (UNIX timestamps), `size` (bytes), `source` (plugin-specific unique path/id — passed back on download), `url`, `thumbnail`, `thumbnail_width`, `thumbnail_height`, `realthumbnail`, `icon`, `realicon`, `author`, `license`, `image_width`, `image_height`. Folder entries: `title`, `path`, dates, `thumbnail`, `icon`, and `children` (presence marks it a folder; empty array with dynload, nested list otherwise) — no `source`/`url`.

### Login (optional)

- `print_login()` — check `$this->options['ajax']`: if true return `['login' => [$field, ...]]` where each field is an object with `label`, `id`, `type` (`text`/`password`/`popup`), `name` (popup type uses `url` instead); if false echo a full HTML form. Add `$ret['allowcaching'] = true;` for static forms. Retrieve submitted values in `__construct` via `optional_param()`.
- `check_login(): bool` — whether the user is logged in (default true), e.g. `return !empty($SESSION->{$this->sessname});`
- `logout()` — clean up session/connection, then `return $this->print_login();`

### Transferring files (optional; defaults usually fine)

- `get_file_reference($source)` — parse/clean user-supplied source (use `clean_param()`); result feeds get_file/get_link/get_file_by_reference and is stored for aliases.
- `get_file($url, $filename = '')` — for FILE_INTERNAL/FILE_REFERENCE, download the file locally:

```php
public function get_file($url, $filename = '') {
    $path = $this->prepare_file($filename); // unique temp filename
    $curlobject = new curl();
    $result = $curlobject->download_one($url, null, ['filepath' => $path, 'timeout' => self::GETFILE_TIMEOUT]);
    if ($result !== true) {
        throw new moodle_exception('errorwhiledownload', 'repository', '', $result);
    }
    return ['path' => $path, 'url' => $url];
}
```

- `get_link($url)` — for FILE_EXTERNAL, convert the reference into a browser-usable URL (often just returns `$url`).
- `get_file_source_info($source)` — value stored in `files.source`; include the full URL or repository name, e.g. `'Dropbox: /filename.jpg'`.

### Search (optional)

- `print_search()` — return the search form HTML; default renders `repository_default_searchform()`. Custom forms need fields: `s` (text), hidden `repo_id`, `ctx_id`, `sesskey`.
- `search($search_text, $page = 0)` — return `['list' => ...]` (same layout as get_listing's list); extra form params via `optional_param()`. Example: `return ['dynload' => true, 'list' => $gdocs->get_file_list($search_text)];`
- `global_search()` — return true to be included in all-repository search (no UI currently).

### FILE_REFERENCE (alias/shortcut) support

Requires FILE_REFERENCE in `supported_returntypes()`. Files are synchronised when the UI needs the file size.

- `get_reference_file_lifetime($ref)` — seconds before checking for changes; default `DAYSECS`.
- `sync_individual_file(stored_file $storedfile)` — whether to sync now once lifetime passed (default true).
- `get_reference_details($reference, $filestatus = 0)` — human-readable origin, e.g. `$this->get_name() . ': ' . $details->filename`; `$filestatus` 666 means source missing (`return get_string('lostsource', 'repository', '');`).
- `get_file_by_reference($reference)` — return an object with `filepath`, `handle`, or `content` for images (so the thumbnail updates), just `filesize` for non-images (avoid downloading), or `null` for missing files. Called often.
- `send_file($storedfile, $lifetime = 86400, $filter = 0, $forcedownload = false, ?array $options = null)` (devdocs writes `array $options = null`; the explicit `?array` is the coding-style form) — serve the file: either `header('Location: ' . $url);` to redirect, or download/cache (`$this->import_external_file_contents()`) then `send_stored_file()`. Use `$storedfile->get_reference()`; call `send_file_not_found()` if missing.
