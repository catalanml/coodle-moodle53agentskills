# Block plugins (Moodle 5.3)

Reference for developing block plugins against Moodle 5.3: file structure, block class API, instance and global configuration, capabilities, page-type restrictions, and what changed in 5.0–5.3. The block devdocs pages are unchanged since 4.5; the 5.x changes below come from the UPGRADING notes.

Block plugins show supplemental information and features within different parts of Moodle. They live in the `/blocks` directory (relative to `$CFG->dirroot`; since 5.1, MDL-83424, `public/blocks` in a git checkout), one subdirectory per plugin.

## File structure

```console
blocks/pluginname/
|-- db
|   `-- access.php
|-- lang
|   `-- en
|       `-- block_pluginname.php
|-- pix
|   `-- icon.png
|-- block_pluginname.php
|-- edit_form.php (optional)
`-- version.php
```

Required files: `block_pluginname.php` (block definition class), `db/access.php` (capabilities), `lang/en/block_pluginname.php`, `version.php`. `edit_form.php` is only needed if the block has per-instance configuration.

The easiest way to scaffold a new block is [tool_pluginskel](https://moodle.org/plugins/tool_pluginskel).

## Block class: block_pluginname.php

All blocks must provide a main class extending one of two base classes:

- `block_base` - default base class for content blocks.
- `block_list` - for blocks that display a list of items.

```php
class block_pluginname extends block_base {

    public function init() {
        $this->title = get_string('pluginname', 'block_pluginname');
    }

    public function get_content() {
        global $OUTPUT;

        if ($this->content !== null) {
            return $this->content;
        }

        $this->content = new stdClass();
        $this->content->footer = '';

        // Add logic here to define your template data or any other content.
        $data = ['YOUR DATA GOES HERE'];

        $this->content->text = $OUTPUT->render_from_template('block_yourplugin/content', $data);

        return $this->content;
    }

    public function applicable_formats() {
        return [
            'admin' => false,
            'site-index' => true,
            'course-view' => true,
            'mod' => false,
            'my' => true,
        ];
    }
}
```

### Class attributes

- `$this->config` - block instance configuration. Empty object by default; if the block has `edit_form.php`, holds the form data.
- `$this->content` - the actual displayed content; NULL or a stdClass whose members depend on the base class.
- `$this->page` - the page the block is displayed on.
- `$this->context` - the context the block is displayed in.
- `$this->title` - the block title.

### init()

Essential for all blocks; called before the block is displayed to give values to class member variables. It runs **before** `$this->config` is set — if a config value is needed for global attributes like the title, use `specialization()` instead.

### specialization()

Called right after an instance is loaded. Used to customize title and other attributes depending on page type, context, configuration, etc.

```php
public function specialization() {
    if (isset($this->config->title)) {
        $this->title = format_string($this->config->title, true, ['context' => $this->context]);
    } else {
        $this->title = get_string('newhtmlblock', 'block_html');
    }
}
```

### get_content()

Produces the on-screen content. It can be called several times during page rendering, so return early if `$this->content` is already set (see example above).

For a `block_list` block, populate `items`, `icons` and `footer` instead:

```php
class block_pluginname extends block_list {

    public function get_content() {
        global $OUTPUT;
        if ($this->content !== null) {
            return $this->content;
        }

        $this->content = (object) [
            'items'  => [],
            'icons'  => [],
            'footer' => 'Footer here...',
        ];

        $this->content->items[] = 'An item of pluginname block';
        $this->content->icons[] = $OUTPUT->pix_icon('i/course', get_string('course'));

        return $this->content;
    }
}
```

If the content is empty (`text` and `footer` for block_base; empty `items` array for block_list), Moodle's `is_empty()` check hides the block entirely.

### Other overridable methods

```php
// Allow more than one instance per page (default: one). Admins can still
// disallow this per-block in site administration.
public function instance_allow_multiple() {
    return true;
}

// Hide the standard block header (ignored in edit mode).
public function hide_header() {
    return true;
}

// Inject extra HTML attributes into the block wrapper.
public function html_attributes() {
    $attributes = parent::html_attributes();
    $attributes['class'] .= ' block_' . $this->name();
    return $attributes;
}
```

## Instance configuration: edit_form.php

Add an `edit_form.php` with a class extending `block_edit_form`; settings then appear in `$this->config`.

```php
class block_pluginname_edit_form extends block_edit_form {
    protected function specific_definition($mform) {
        // Section header title according to language file.
        $mform->addElement('header', 'config_header', get_string('blocksettings', 'block'));
        // A sample string variable with a default value.
        $mform->addElement('text', 'config_text', get_string('blockstring', 'block_pluginname'));
        $mform->setDefault('config_text', 'default value');
        $mform->setType('config_text', PARAM_TEXT);
    }
}
```

Caveats:

- All field names must start with `config_`, otherwise they are not saved and not available via `$this->config`.
- `$this->config` is available in all block methods **except init()**; use `specialization()`.
- Do not use the `checkbox` form element (once set it stays set); use `advcheckbox`.

### instance_config_save() (optional)

By default all `config_*` settings are base64-encoded into the `configdata` field of the `block_instances` table and decoded into `$this->config` on load. To store data elsewhere or alter it before saving:

```php
public function instance_config_save($data, $nolongerused = false) {
    // Example of add new data.
    $data->somenewattribute = 'Some new value';

    // Example of alter the current data.
    $data->text = 'Some new text';

    // Call the parent method to store the data inside block_instances.configdata.
    return parent::instance_config_save($data, $nolongerused);
}
```

## Global settings: settings.php + has_config()

Site-wide settings go in `blocks/pluginname/settings.php` (standard admin settings file). Unlike other plugin types, blocks must also override `has_config()`:

```php
public function has_config() {
    return true;
}
```

Global settings are not part of the block instance; read them with:

```php
$settingvalue = get_config('block_pluginname', 'settingname');
```

## Capabilities: db/access.php

Blocks require the `addinstance` and `myaddinstance` capabilities:

```php
$capabilities = [
    'block/pluginname:myaddinstance' => [
        'captype' => 'write',
        'contextlevel' => CONTEXT_SYSTEM,
        'archetypes' => [
            'user' => CAP_ALLOW,
        ],
        'clonepermissionsfrom' => 'moodle/my:manageblocks',
    ],
    'block/pluginname:addinstance' => [
        'riskbitmask' => RISK_SPAM | RISK_XSS,
        'captype' => 'write',
        'contextlevel' => CONTEXT_BLOCK,
        'archetypes' => [
            'editingteacher' => CAP_ALLOW,
            'manager' => CAP_ALLOW,
        ],
        'clonepermissionsfrom' => 'moodle/site:manageblocks',
    ],
];
```

Language file must include the capability strings:

```php
$string['pluginname'] = 'Pluginname';
$string['pluginname:addinstance'] = 'Add a new pluginname block';
$string['pluginname:myaddinstance'] = 'Add a new pluginname block to the My Moodle page';
```

## Limiting the block to specific page types

`applicable_formats()` returns an associative array mapping page format keys to booleans. Conventions:

- `all` - catch-all; `['all' => true]` allows any page.
- `site-index` - Moodle frontpage.
- `course-view` - course page, any format.
- `course-view-FORMATNAME` - course page with a given format (e.g. `course-view-weeks`).
- `mod` - any activity page.
- `mod-MODNAME-view` - a specific activity page (e.g. `mod-forum-view`).
- `my` - the Dashboard.
- `admin` - any administration page.

## Templates and styles in 5.x

- **Bootstrap 5 (5.0, MDL-75669).** Block templates must use Bootstrap 5 markup: namespaced data attributes (`data-bs-toggle`, `data-bs-target` instead of `data-toggle`, `data-target`), `.dropdown-menu-end` / `-start` instead of `-right` / `-left`, `.form-select` / `.form-check` instead of `.custom-select` / `.custom-check`, `.text-bg-*` or `.bg-*` instead of `.badge-*`. A BS4 compatibility layer (`theme_boost/bs4-compat` JS module and SCSS helpers such as `.mr-1`, `.sr-only`, `.badge-success`) is deprecated on arrival and stays until 6.0.
- **Bootstrap JS imports (5.3, MDL-88766).** AMD modules must not import `theme_boost/bootstrap/*` submodules. Use `import {Tooltip} from 'bootstrap';` (5.3+), or `import {Tooltip} from 'theme_boost/index';` when the code must also run on 5.2 and earlier (supported until 7.0).
- **Dark colour mode (5.3, MDL-68037).** Boost has an experimental dark mode (`theme_boost | enablecolourmodes`), so a block's `styles.css` must not assume a light page. Take colours from Bootstrap custom properties (`--bs-body-bg`, `--bs-body-color`, `--bs-secondary-bg`, `--bs-tertiary-bg`, `--bs-border-color`, `--bs-emphasis-color`, `--bs-link-color`) or design-system tokens (`--mds-bg-surface-*`, `--mds-text-*`, `--mds-border-*`), or utility classes (`bg-body`, `bg-body-secondary`, `bg-body-tertiary`, `text-body`, `text-body-secondary`, `text-bg-*`, `border`) rather than `bg-white` / `text-dark`. Scope mode-specific colours with `[data-bs-theme="dark"]`. Render SVG icons through the icon API so they inherit `currentColor`.
- **Modal headings (5.3).** `core/modal` titles are now `<h2 class="modal-title fs-5">`; headings inside modal content should start at `<h3>`.

### React-rendered content (5.3)

Block content can be rendered by a React component through the `\core\output\react_component_renderable` interface (MDL-89296); `block_timeline` now does this (MDL-88287):

```php
final class my_renderable implements \core\output\react_component_renderable {
    #[\Override]
    public function get_react_component_name(): string {
        return 'block_pluginname/Main';
    }

    #[\Override]
    public function get_react_component_props(
        \core\output\renderer_base $renderer,
    ): \stdClass {
        return (object) [];
    }
}

// In get_content():
$this->content->text = $OUTPUT->render(new my_renderable());
```

`\core\output\html_writer::react_component()` renders a component directly (pass props unserialised). How to build the React/ESM side is not covered by the block devdocs.

## Behaviour changes in 5.0–5.3

- 5.1 (MDL-85433): subcontext visibility is turned on by default when adding blocks.
- 5.3 (MDL-89289): uninstalling a block no longer deletes its instances synchronously. `\core\plugininfo\block::uninstall_cleanup()` queues `\core\task\delete_block_instances_task`, which deletes instances, contexts, positions, preferences and search entries in batches. Leftover instances are not displayed once the block record is gone.
- 5.3 (MDL-88287): `block_timeline\output\main`, `block_timeline\output\renderer` and the block's external services were removed without deprecation; themes overriding that renderer must be updated.
- Core blocks removed: `block_mnet_hosts` (5.0), `block_section_links` (5.1), `block_activity_modules` (5.2). The "Main menu" block (`block_site_main_menu`) is now "Additional activities" (5.1) and lost its activity selector (5.0), as did `block_social_activities`.
- 5.2: `block_html` treats the Dashboard (`my-index`) as trusted in web services, so `get_content_for_external()` keeps embedded HTML there.
