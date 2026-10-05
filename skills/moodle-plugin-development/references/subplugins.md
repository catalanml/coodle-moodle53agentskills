# Activity Subplugins, Tiny Testing, and Format Migration (Moodle 5.3)

Reference for declaring and deprecating subplugin types, the subplugin types hosted by activities (book, data, forum), Behat testing for TinyMCE plugins, and migrating pre-4.0 course formats to the current format API. Distilled from the Moodle 5.3 developer docs and the 5.0–5.3 UPGRADING notes.

## Defining subplugin types: db/subplugins.json

Any plugin that hosts subplugins declares its subplugin types in `db/subplugins.json`: key = subplugin type (also the namespace prefix of every subplugin), value = the directory the subplugins live in.

- Since 5.0 (MDL-83705): the `subplugintypes` object is the format to use. Its paths are **relative to the parent plugin's root**.
- Moodle 4.5 and earlier only read the legacy `plugintypes` object, whose paths are relative to the Moodle code root (`$CFG->dirroot`; the `public/` directory in a 5.1+ checkout — do not write `public/` in these paths).
- A plugin supporting both 5.0+ and 4.5-or-earlier must specify both objects. Both may co-exist, but the keys must match and the paths must point at the same directories.
- Since 5.0 (MDL-83703): `db/subplugins.php` is no longer supported. All subplugin metadata must be in `db/subplugins.json`.

```json
{
    "subplugintypes": {
        "quiz": "report",
        "quizaccess": "accessrule"
    },
    "plugintypes": {
        "quiz": "mod/quiz/report",
        "quizaccess": "mod/quiz/accessrule"
    }
}
```

(That is `mod/quiz/db/subplugins.json`; a 5.0+-only plugin can drop `plugintypes`.)

Each subplugin is a full plugin in its own directory under the declared path, with its own `version.php`, `lang/en/<type>_<name>.php`, `classes/`, and so on — see common-files.md. A subplugin's component is `<subplugintype>_<name>`, e.g. `quizaccess_seb`.

**Plugin-type name strings.** Since 5.1 (MDL-84948): `core_plugin_manager::plugintype_name()` and `plugintype_name_plural()` require plugin-type names to be defined as `type_<type>` and `type_<type>_plural` language strings. The UPGRADING note does not say in which component's lang file a subplugin type's strings must live; check how a core parent such as `mod_quiz` or `mod_assign` defines them in your target release before relying on it.

### Deprecating a subplugin type (5.0+)

Since 5.0 (MDL-79843): a parent plugin can retire one of its subplugin types. Unlike code deprecation, a deprecated (sub)plugin type is treated as **end-of-life immediately**: core stops including its plugins in core-plugin communication (hooks, callbacks, events, ...), and `\core\task\manager::queue_adhoc_task()` returns `false` for tasks of a deprecated component. Class autoloading and string resolution keep working during the deprecation window so migration scripts can run. The owning plugin **must** already have removed or replaced every reference to those subplugins before deprecating the type.

Three stages, each edited in the parent's `db/subplugins.json`:

1. **Deprecated** — move the type from `subplugintypes` to `deprecatedsubplugintypes` and bump the parent plugin's version. Installed subplugins stay but are end-of-life; admins uninstall or migrate them.
2. **Deleted** — move it from `deprecatedsubplugintypes` to `deletedsubplugintypes` and bump the version again. Site upgrade is **blocked** while any plugin of that type is still present.
3. **Final removal** — delete the entry (and any now-empty object). No version bump needed.

```json
{
    "subplugintypes": {
        "ltiservice": "service"
    },
    "deprecatedsubplugintypes": {
        "ltisource": "source"
    }
}
```

The docs' example is `mod/lti/db/subplugins.json`; at stage 2 the `ltisource` entry moves to `"deletedsubplugintypes"`. Limitation: only plugin types that do **not** themselves support subplugins can be deprecated. Core plugin types follow the same process in `lib/components.json` (see other-plugin-types.md).

### Activity-hosted subplugin types covered here

| Subplugin type | Component prefix | Location | Purpose | Since |
| --- | --- | --- | --- | --- |
| Book tools | `booktool` | `/mod/book/tool` | Extra tools for the Book activity | 2.1+ |
| Database fields | `datafield` | `/mod/data/field` | Custom field data types for the Database activity | 1.6+ |
| Database presets | `datapreset` | `/mod/data/preset` | Pre-defined templates for the Database activity (legacy) | 1.6+ |
| Forum reports | `forumreport` | `/mod/forum/report` | Display various reports in the forum activity | 3.8+ |

Locations are relative to `$CFG->dirroot`; in a 5.1+ git checkout they are under `public/` (e.g. `public/mod/data/field`). Assignment subplugins (`assignsubmission`, `assignfeedback`) are covered in other-plugin-types.md.

## Book tool plugins (booktool_)

The `mod_book` activity can be extended using the `booktool` subplugin type, located in `/mod/book/tool`. The official 5.3 docs still mark this plugin type as undocumented; use the core subplugins (`booktool_print`, `booktool_exportimscp`, `booktool_importhtml` in the Moodle source) as reference implementations.

Related `mod_book` changes: `book_get_nav_classes()` was final-deprecated in 5.0 and removed in 5.2; the `\mod_book\event\course_module_instance_list_viewed` event is deprecated since 5.1 — use `\core\event\course_resources_list_viewed`.

## Database field types (datafield_)

The Database activity ships predefined field types (text, date, URL, ...) and supports custom ones — e.g. discipline-specific ("Protein PDB code" rendering a 3D viewer), institution-specific ("library reference number" auto-linked to library services), or module-specific ("wiki page" drop-down of wiki page names).

### File structure

Plugins live in `/mod/data/field/<pluginname>`. Example layout for `datafield_number`:

```
mod/data/field/number
├── classes
│   └── privacy
│       └── provider.php
├── field.class.php
├── lang
│   └── en
│       └── datafield_number.php
├── mod.html
└── version.php
```

### field.class.php (required)

The field, its behaviours, and its properties are defined in a class named `data_field_[pluginname]` in `field.class.php`, extending `data_field_base`. **This class is not autoloaded by Moodle** — it must live in `field.class.php`, not `classes/`.

Key base-class methods to override:

- `display_add_field($recordid = 0)` — HTML shown when a user adds/edits a record.
- `display_browse_field($recordid, $template)` — HTML for displaying a record.
- `update_content($recordid, $value, $name = '')` — store the data a user entered for a record.
- `get_sort_sql($fieldname)` — SQL for how this field sorts.
- `get_content_value($value)` — when the stored value differs from what is presented to the user.

### mod.html (required)

Form definition for adding/editing the field configuration. Note: this is one of the older parts of Moodle and does not follow current form best practice (raw HTML file, not a moodleform).

### mod_data API changes that affect field plugins (5.0–5.3)

- Removed in 5.2 from `mod/data/deprecatedlib.php` (final-deprecated in 5.0): `data_print_template()` (use `mod_data\manager::get_template()` and `mod_data\template::parse_entries()`), `data_view()` (use `mod_data\manager::set_module_viewed()`), `data_get_available_presets()` (use `manager::get_available_presets()`), plus the other `data_preset*` helpers (replaced by methods of the `preset` class, e.g. `preset::save()`, `preset::export()`, `preset::can_manage()`).
- Removed in 5.0: classes `data_preset_importer`, `data_preset_existing_importer`, `data_preset_upload_importer`, `data_import_preset_zip_form`.
- Deprecated in 5.0: unused capabilities `mod/data:comment` and `mod/data:managecomments`.
- Added in 5.1: `mod_data\manager` methods `get_approval_requested()`, `get_all_entries()`, `filter_entries_by_user()`, `filter_entries_by_approval()`, `get_comments()`; the entries generator can create approved entries.

## Database presets (datapreset_)

The `datapreset` subplugin type (`/mod/data/preset`) packages pre-defined Database activity configurations for sharing. **It is legacy and no longer recommended** — presets are now shared through the Database activity UI instead (see the user docs at https://docs.moodle.org/en/Database_presets). The 5.3 docs page is tagged `deprecated`, but the type has not been moved to `deprecatedsubplugintypes`.

## Forum subplugins (forumreport_)

The `mod_forum` activity can be extended using the `forumreport` subplugin type (`/mod/forum/report`, since 3.8), used to display reports within a forum — core's `forumreport_summary` is the reference implementation. The official 5.3 docs still mark this plugin type as undocumented; no other forum subplugin type is documented.

Related `mod_forum` changes: the forum report entity's `get_context_joins()` is deprecated since 5.2 (use `get_course_modules_joins(...)`); `forum_tp_get_untracked_forums()` is deprecated since 5.1; `forum_print_discussion_header()` was removed in 5.2; 5.3 adds `discussion::get_discussion_navigation_buttons()` and the `mod_forum_set_read_state` web service.

## Testing TinyMCE plugins (tiny_) with Behat

When a test merely *uses* a text editor (rather than testing the editor or one of its plugins), avoid editor-specific steps. For actual TinyMCE tests (paths below are relative to `$CFG->dirroot`, i.e. under `public/` in a 5.1+ checkout):

**Note:** since 5.0 (MDL-83848) behat/gherkin 4.11 requires backslashes in feature files to be escaped.

### Required tags

The feature/scenario **must** have one or more of `@editor_tiny` or `@tiny_[pluginname]`. These tags make Moodle set TinyMCE as the site default editor for the duration of the test regardless of other installed editors.

### Generic editor steps (prefer these)

Defined mostly in `lib/tests/behat/behat_forms.php`. Setting content:

```gherkin
Given I set the field "Description" to "<p>Some value which includes valid HTML</p>"
Given I set the field "Description" to multiline:
"""
<p>Some value which includes valid HTML</p>
"""
Given I set the field "Description" in the "Create event" "dialogue" to "<p>...</p>"
Given I set the following fields to these values:
  | Name        | Some name here |
  | Description | <p>...</p>     |
```

Checking content:

```gherkin
Then the field "Description" matches value "<p>...</p>"
Then the field "Description" does not match value "<p>...</p>"
Then the field "Description" in the "Create event" "dialogue" matches value "<p>...</p>"
Then the following fields match these values:
  | Description | <p>...</p> |
```

### TinyMCE-specific steps

Defined in `lib/editor/tiny/tests/behat/behat_editor_tiny.php`. TinyMCE renders its editor UI inside its own iframe; WebDriver commands are scoped to the current window/frame, so interacting with editor internals may require switching context — most Tiny-specific steps already handle the iframe.

Toolbar (always expand first — the toolbar auto-collapses to fit the page; the step is safe if already expanded):

```gherkin
Given I expand all toolbars for the "Description" TinyMCE editor
When I click on the "Bold" button for the "Description" TinyMCE editor
Then the "Bold" button of the "Description" TinyMCE editor has state "true"
```

Menu bar:

```gherkin
I click on the "Insert" menu item for the "Description" TinyMCE editor
I click on the "Insert > Image" menu item for the "Description" TinyMCE editor
```

Selecting content (performed inside the editor's iframe — some standard Behat locators will not work there because they expect elements in the page body):

```gherkin
I select the "p" element in position "2" of the "Description" TinyMCE editor
I select the ".example_placeholder" "css_element" in the "Description" TinyMCE editor
```

### Writing your own steps

Include your plugin name or "TinyMCE editor" in the step wording so it is not misused, and check the feature/scenario carries your frankenstyle tag:

```php
if (!$this->has_tag('tiny_myplugin')) {
    throw new DriverException(
        'This step must have the @tiny_myplugin tag on either the Feature, or the Scenario.'
    );
}
```

## Migrating a 3.11 course format to the 4.x/5.x format API

The 4.0 course editor reimplemented the course rendering webservices, AMD modules, and internals. Since 5.0 (MDL-83527): every course format still depending on the 3.11 libraries is **deprecated and will stop working in Moodle 6.0**; all course-editing YUI modules are deprecated and formats not using components must migrate before 6.0. Your format is affected if its format class in `lib.php` does not override `supports_components()` or returns `false` from it. See course-formats.md for the current API; this section covers the deltas a pre-4.0 format must absorb. (The 5.3 migration page is unchanged since 4.5; the 5.0 corrections below come from UPGRADING.)

### Changes summary

- Course formats are now part of the `core_courseformat` subsystem.
- `format_base` is now `core_courseformat\base` — extending it is **mandatory**. For compatibility the base class stays un-namespaced in your plugin's `lib.php`.
- A **renderer class is now mandatory**, in most cases extending `core_courseformat\output\section_renderer` (the old `format_section_renderer_base`). It extends `core_course_renderer` directly, so `$this->courserenderer` is deprecated.
- All renderer methods using `html_writer` are deprecated (full list in `course/upgrade.txt`). UI is rendered by **output classes + mustache templates**, optionally with reactive AMD components.
- Elements are located by **data attributes**, not CSS classes.
- `core_course/actions` AMD logic moved to `core_courseformat` modules: `local/courseeditor` (reactive state), `local/courseeditor/mutations` (backend calls), `local/content/actions` (action clicks/modals), `content/*` (UI components). Section/activity edit webservices are replaced by `core_courseformat_update_course` (returns state data, not HTML); initial state loads via `core_courseformat_get_state`. Show/hide actions were not yet migrated.
- Override `core_courseformat\base::uses_course_index()` to enable the course index (Boost-based themes only; Classic themes never show the drawer).
- Override `core_courseformat\base::supports_components()` to opt in to the reactive course editor instead of the legacy YUI one.
- Optional hook `core_courseformat\base::delete_format_data()` cleans plugin data on course deletion.
- Use `$format->show_editor()` (optionally passing an array of capabilities; defaults to `moodle/course:manageactivities`) instead of `$PAGE->user_is_editing() && has_capability(...)`.
- 5.0 changes to absorb at the same time:
  - `get_section_number()` / `set_section_number()` are final-deprecated (MDL-80116); use `set_sectionnum(?int $sectionnum): void` and `get_sectionnum(): ?int` (verified in MOODLE_503_STABLE source); the UPGRADING note misspells the getter `get_sectionum`.
  - Section and activity control menus are output classes returning `action_menu_link` objects, not arrays: extend `core_courseformat\output\local\content\cm\controlmenu` (replaces `course_get_cm_edit_actions()`) and `core_courseformat\output\local\content\section\controlmenu`.
  - `core_courseformat_create_module` is deprecated — use `core_courseformat_new_module` (takes the section id); links with `data-action="addModule"` become `data-action="newModule"` plus a `data-sectionid` attribute.
  - `core_courseformat\base::get_non_ajax_cm_action_url()` is deprecated — use `get_update_url()`; direct edit URLs in `course/view.php` / `course/mod.php` (indent, duplicate, hide, show, stealth, delete, groupmode, marker) are replaced by `course/format/update.php`.
  - Reactive components are initialised with a selector (`'#id'`) rather than a bare element id (MDL-83339).
- 5.1: `maxsections` and `core_courseformat\base::get_max_sections()` are deprecated (removal in 6.0); the activity chooser moved to `core_courseformat`.

### Step 1: base class and renderer

In `lib.php`, change `extends format_base` to `extends core_courseformat\base`. If the plugin has a renderer, change `extends format_section_renderer_base` to `extends core_courseformat\output\section_renderer`; if not, create `course/format/pluginname/classes/output/renderer.php`:

```php
namespace format_pluginname\output;

use core_courseformat\output\section_renderer;

class renderer extends section_renderer {
    // Your renderer methods go here.
}
```

### Step 2: new format.php workflow

The format base instance is the single exchange parameter for all output classes (replacing `print_single_section_page` / `print_multiple_section_page` and their long parameter lists):

```php
$format = core_courseformat\base::instance($course);
$course = $format->get_course();
$context = context_course::instance($course->id);

course_create_sections_if_missing($course, 0);

$renderer = $format->get_renderer($PAGE);

if (!empty($displaysection)) {
    $format->set_sectionnum($displaysection);
}

$outputclass = $format->get_output_classname('content');
$widget = new $outputclass($format);
echo $renderer->render($widget);
```

(The 5.3 docs still show `set_section_number()` here; it is final-deprecated since 5.0.) Notes: get the renderer via `$format->get_renderer()`; resolve output classes via `$format->get_output_classname()` so plugin overrides are found; `render()` works directly (no `render_from_template`) because output classes implement the `named_templatable` interface.

### Step 3: override output classes

Each UI component has its own small output class. To override one, place a class at `format_pluginname\output\courseformat\...` mirroring the core path — e.g. `core_courseformat\output\local\content\section\controlmenu` is overridden by `format_pluginname\output\courseformat\content\section\controlmenu` (real example: `course/format/topics/classes/output/courseformat/content/section/controlmenu.php`). `get_output_classname()` returns the plugin class when present, the core one otherwise.

### Step 4: mustache structure

Mustache files cannot be extended or overridden directly — the plugin must provide a minimum template structure and output-class overrides pointing at its own templates. Minimum of 3 templates to render a course:

- `local/content` — the full course template
- `local/content/section` — a section (used for ajax section refresh)
- `local/content/section/cmitem` — an activity item (used for ajax activity refresh)

### Step 5: data attributes for the reactive editor

Replace CSS-class-based selectors with data attributes:

| Concept | 3.11 selector | 4.x data attributes |
| --- | --- | --- |
| Section | `li.section#section-{NUM}` | `data-for="section"` `data-id={SECTION.ID}` `data-number={SECTION.NUM}` |
| Section header | `#sectionid-{ID}-title` | `data-for="section_title"` `data-id={SECTION.ID}` `data-number={SECTION.NUM}` |
| Activity (cm item) | `li.activity#module-{CM.ID}` | `data-for="cmitem"` `data-id={CM.ID}` |
| Course sections list | `.course-content>ul` | `data-for="course_sectionlist"`; section's cm list: `data-for="cmlist"` |
| Course module action link | `a.cm-edit-action` | `data-action={ACTIONNAME}` `data-id={CM.ID}` |
| Section action link | `.section_action_menu` | `data-action={ACTIONNAME}` `data-id={SECTION.ID}` |
| Section info | `.section_availability` | `data-for="sectioninfo"` |

### Step 6: enable and verify components

```php
class format_pluginname extends core_courseformat\base {
    public function uses_course_index() {
        return true;
    }

    public function supports_components() {
        return true;
    }
}
```

To verify: enable developer debug, open a course as admin, find "Reactive instances" in the debug footer, click "CourseEditorXXXX" (XXXX = course id), toggle "Highlight OFF" to ON — all sections, section headers, and activities should get thick blue borders. Missing borders on inner elements mean missing data attributes; no CourseEditor button usually means an overridden `content.mustache` dropped the JS initialization. To restore it, add `id="{{uniqid}}-course-format"` to the course content div and append:

```handlebars
{{#js}}
require(['core_courseformat/local/content'], function(component) {
    component.init('#{{uniqid}}-course-format', {}, {{sectionreturn}});
});
{{/js}}
```

The docs' snippet passes the bare id `'{{uniqid}}-course-format'`; since 5.0 (MDL-83339) bare-id initialisation of reactive components is deprecated in favour of a `querySelector` selector, hence the `#`.
