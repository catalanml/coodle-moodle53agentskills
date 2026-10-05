# Local plugins (Moodle 5.3)

Reference for `local` plugins: when to use them, how they differ from standard plugin types, and common extension patterns (navigation callbacks, admin settings). The Moodle 5.3 devdocs page for this type is unchanged since 4.5; 5.x changes that affect it come from the UPGRADING notes and are marked inline.

## Purpose: when to choose local vs other types

The recommended way to add functionality to Moodle is a standard plugin type (activity, block, authentication, enrol, ...). The `local` plugin type is for things that do not fit any standard type. Typical cases:

- event consumers communicating with external systems
- custom definitions of web services and external functions
- applications that extend Moodle at the system level (for example hub server, amos server)
- custom admin settings
- extending the navigation block with custom menus
- new database tables used in core hacks (**strongly discouraged**)
- new capability definitions used in core hacks (**strongly discouraged**)

## Differences from standard plugins

- Always executed **last** during install and upgrade (guaranteed by their order in `get_plugin_types()`).
- Expected to use event handlers — event subscriptions are intended for core-to-plugin communication, making local plugins the ideal candidate. Hook callbacks (`db/hooks.php`) are the other main integration point (see common-files.md).
- Can add admin settings to **any** settings page; they are loaded last when constructing the admin tree to enable this.
- Do not need any UI (other plugin types are usually visible somewhere in the interface).

## File structure

Local plugins live in `/local/pluginname/` relative to `$CFG->dirroot`. Since 5.1 (MDL-83424) that is `public/local/pluginname/` inside a Moodle git checkout, because almost all code moved into the `public/` directory; code paths (`$CFG->dirroot . '/local/...'`, `new moodle_url('/local/...')`) are unchanged.

They support the standard plugin files used by other plugin types (`version.php`, `lang/en/local_pluginname.php`, `lib.php`, `settings.php`, `db/`, `classes/`, optional `composer.json` with `"type": "moodle-local"` since 5.2, etc.). No files beyond the common ones are specific to this type.

## Examples

The examples use `local_myplugin`; replace `myplugin` with your plugin name.

### Adding an element to the settings menu

Define `local_[pluginname]_extend_settings_navigation` in the plugin's `lib.php`. It is called when Moodle builds the settings block.

```php
/**
 * Add a link to the course administration menu.
 *
 * @param settings_navigation $settingsnav The settings navigation object.
 * @param context $context The context of the current page.
 */
function local_myplugin_extend_settings_navigation(settings_navigation $settingsnav, context $context) {
    global $PAGE;

    // Only add this settings item on non-site course pages.
    if (!$PAGE->course || $PAGE->course->id == SITEID) {
        return;
    }

    // Only let users with the appropriate capability see this settings item.
    if (!has_capability('moodle/backup:backupcourse', context_course::instance($PAGE->course->id))) {
        return;
    }

    if ($settingnode = $settingsnav->find('courseadmin', navigation_node::TYPE_COURSE)) {
        $strfoo = get_string('foo', 'local_myplugin');
        $url = new moodle_url('/local/myplugin/foo.php', ['id' => $PAGE->course->id]);
        $foonode = navigation_node::create(
            $strfoo,
            $url,
            navigation_node::NODETYPE_LEAF,
            'myplugin',
            'myplugin',
            new pix_icon('t/addcontact', $strfoo)
        );
        if ($PAGE->url->compare($url, URL_MATCH_BASE)) {
            $foonode->make_active();
        }
        $settingnode->add_node($foonode);
    }
}
```

### Modifying the navigation menu

`local_[pluginname]_extend_navigation()` in `lib.php` can modify existing navigation components. Example removing the "Site Home" link:

```php
/**
 * Remove the Site home node from the global navigation.
 *
 * @param global_navigation $navigation The global navigation object.
 */
function local_myplugin_extend_navigation(global_navigation $navigation) {
    if ($home = $navigation->find('home', global_navigation::TYPE_SETTING)) {
        $home->remove();
    }
}
```

**Note:** since 5.3 the primary and secondary navigation bars are rendered by React components (`core/nav/PrimaryNav`, `core/nav/Nav`) using `.mds-nav-pill` markup instead of `.nav-link` / `.moremenu`. The PHP navigation callbacks above still feed the navigation tree, but CSS/JS in a local plugin that targeted the old navbar markup must be re-checked.

### Adding site-wide settings

In `settings.php`. Note: unlike most plugin types, `$settings` is **not** pre-defined for local plugins (it is null), so create the settings page yourself and add it to the `localplugins` category:

```php
defined('MOODLE_INTERNAL') || die();

// Ensure the configurations for this site are set.
if ($hassiteconfig) {
    // In a local plugin $settings is not defined as standard, so create the page.
    $settings = new admin_settingpage('local_myplugin', new lang_string('pluginname', 'local_myplugin'));

    $ADMIN->add('localplugins', $settings);

    // Add a setting field to the settings for this page.
    $settings->add(new admin_setting_configtext(
        // The reference used to read the configuration.
        'local_myplugin/apikey',
        // The friendly title for the config, which will be displayed.
        new lang_string('apikey', 'local_myplugin'),
        // Helper text for this config field.
        new lang_string('apikey_desc', 'local_myplugin'),
        // The default value.
        '',
        // The type of parameter this config is.
        PARAM_TEXT
    ));
}
```

Read the value back with `get_config('local_myplugin', 'apikey')`.

Since 5.3 (MDL-81935): these classes also have autoloadable names — `admin_settingpage` is `\core_admin\setting\settingpage\settingpage`, `admin_setting_configtext` is `\core_admin\setting\setting\configtext`, `admin_category` is `\core_admin\setting\tree\category`, `admin_externalpage` is `\core_admin\setting\tree\externalpage`. The UPGRADING note states the existing class names are unaffected, so the example above works on 5.3 and on older releases; use the new names only if the plugin is 5.3+ only.

### Adding a standalone admin page

Prefer admin settings wherever possible (they are searchable, manageable from `admin/cli/cfg.php`, cached, can be forced in `config.php`, and logged in the config log). For screens that do not fit the settings pattern, register an `admin_externalpage` (5.3 name `\core_admin\setting\tree\externalpage`) pointing at a script inside the plugin:

```php
if ($hassiteconfig) {
    $ADMIN->add('localplugins', new admin_externalpage(
        'local_myplugin_report',
        get_string('report', 'local_myplugin'),
        new moodle_url('/local/myplugin/report.php')
    ));
}
```

Near the start of that page script call `admin_externalpage_setup('local_myplugin_report');`, then output with `echo $OUTPUT->header();` / `echo $OUTPUT->footer();` as usual. Since 5.1 (MDL-85518), custom subclasses of `admin_settings` or `admin_externalpage` are encouraged to set the new `searchmatchtype` property using the types in `admin_search`.

## Checklist

- `version.php` with `$plugin->component = 'local_myplugin';` and `$plugin->supported` including `503` (see common-files.md).
- `lang/en/local_myplugin.php` defining `pluginname` plus every string used above.
- Event observers in `db/events.php`, hook callbacks in `db/hooks.php`, web services in `db/services.php` — bump the version after editing any of them.
- `classes/privacy/provider.php` (null provider if no personal data is stored).
