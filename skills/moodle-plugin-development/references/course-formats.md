# Moodle 5.3 Course Format Plugins

Scope: structure, `core_courseformat\base` class, output classes/templates, section and activity rendering, format options, course editor integration and linear navigation for `format_*` plugins, per the Moodle 5.3 developer docs and the 5.0–5.3 UPGRADING notes.

Course formats determine the layout of `/course/view.php` in view and editing mode, build the course navigation tree (navigation block, course index, breadcrumb), organize content in sections, and can add options to the course edit form and content to the header/footer of any course page.

**Pre-4.0 formats are deprecated (5.0, MDL-83527).** If your format class does not override `supports_components()` or returns `false`, it still uses the 3.11 YUI editing libraries. Those are deprecated since 5.0 and stop working in 6.0. Migrate by following the devdocs "Migrating 3.11 formats" guide (`apis/plugintypes/format/migration.md`).

## File structure

Files live in `course/format/pluginname` (since 5.1, MDL-83424, under `public/course/format/pluginname` in a git checkout):

```console
 course/format/pluginname/
 |-- amd/src/mutations.js           (optional: frontend for custom state actions)
 |-- classes
 |   |-- courseformat/stateactions.php  (optional: custom state actions)
 |   `-- output
 |       |-- courseformat/          (overridden output classes)
 |       `-- renderer.php
 |-- db/access.php
 |-- lang/en/format_pluginname.php
 |-- templates/local/...
 |-- format.php
 |-- lib.php
 |-- settings.php                   (optional: e.g. enablelinearnav)
 `-- version.php
```

- `format.php` (required): renders the course; included from `course/view.php`.
- `lib.php` (required): class `format_pluginname` extending `core_courseformat\base` (no namespace, stays in `lib.php` for compatibility).
- `classes/output/renderer.php` (required): class `format_pluginname\output\renderer` extending `core_courseformat\output\section_renderer`.
- `classes/output/courseformat/`: output classes placed here automatically override the core ones.
- Format names cannot exceed 21 characters. Scaffold with `tool_pluginskel`, or copy the topics/weeks format.

## format.php

```php
defined('MOODLE_INTERNAL') || die();

require_once($CFG->libdir . '/filelib.php');
require_once($CFG->libdir . '/completionlib.php');

// Retrieve course format option fields and add them to the $course object.
$format = core_courseformat\base::instance($course);
$course = $format->get_course();
$context = context_course::instance($course->id);

// Make sure section 0 is created.
course_create_sections_if_missing($course, 0);

$renderer = $format->get_renderer($PAGE);

if (!empty($displaysection)) {
    $format->set_sectionnum($displaysection);
}
// Output course content.
$outputclass = $format->get_output_classname('content');
$widget = new $outputclass($format);
echo $renderer->render($widget);
```

**Note:** the 5.3 devdocs example still calls `set_section_number()`. That method and `get_section_number()` were final-deprecated in 5.0 (MDL-80116), and `get_section_number()` was removed in 5.2 (MDL-87425). Use `set_sectionnum()` / `get_sectionnum()`.

Output classes implement `named_templatable`, so pass them straight to `$renderer->render()`.

## Format base class (lib.php)

```php
class format_pluginname extends core_courseformat\base {
    public function uses_sections() {
        return true;
    }
    public function uses_indentation(): bool {
        return false;
    }
    public function uses_course_index() {
        return true;
    }
    public static function uses_linear_navigation(): bool {
        return true; // 5.3+: opt in to the Previous/Next footer.
    }
    public function supports_ajax() {
        $ajaxsupport = new stdClass();
        $ajaxsupport->capable = true;
        return $ajaxsupport;
    }
    public function supports_components() {
        return true;
    }
    public function can_delete_section($section) {
        return true;
    }
    public function supports_news() {
        return true;
    }
    public function get_section_name($section) {
        $section = $this->get_section($section);
        if ((string)$section->name !== '') {
            return format_string($section->name, true,
                ['context' => context_course::instance($this->courseid)]);
        }
        return $this->get_default_section_name($section);
    }
}

// Required for the inplace section name editor.
function format_pluginname_inplace_editable($itemtype, $itemid, $newvalue) {
    global $DB, $CFG;
    require_once($CFG->dirroot . '/course/lib.php');
    if ($itemtype === 'sectionname' || $itemtype === 'sectionnamenl') {
        $section = $DB->get_record_sql(
            'SELECT s.* FROM {course_sections} s JOIN {course} c ON s.course = c.id WHERE s.id = ? AND c.format = ?',
            [$itemid, 'pluginname'], MUST_EXIST);
        $format = core_courseformat\base::instance($section->course);
        return $format->inplace_editable_update_section_name($section, $itemtype, $newvalue);
    }
}
```

`uses_linear_navigation()` is `public static function uses_linear_navigation(): bool` in `core_courseformat\base` (verified in MOODLE_503_STABLE source).

### Key overridable methods on `core_courseformat\base`

Sections: `uses_sections()`, `uses_linear_navigation()` (5.3), `get_default_section_name()`, `get_section_name()` (use only fields cached in `section_info`), `get_view_url()`, `is_section_current()`, `get_section_highlighted_name()`, `set_sectionnum()` / `get_sectionnum()`, `get_course_display()` (`COURSE_DISPLAY_SINGLEPAGE` / `COURSE_DISPLAY_MULTIPAGE`), `get_last_section_number()`, `page_title()`. Section 0 always exists; call `rebuild_course_cache()` after changing sections. Always define `$string['sectionname']`.

Course features: `ajax_section_move()`, `uses_course_index()` (course index drawer), `uses_indentation()`, `supports_components()` (reactive course editor; mandatory from 6.0), `supports_news()`, `get_default_blocks()`, `extend_course_navigation()`.

Helpers (usually not overridden): `get_course()`, `get_section()`, `get_sections()`, `get_modinfo()`, `get_renderer()`, `get_output_classname()`, `show_editor()` (defaults to `moodle/course:manageactivities`).

Changes since 4.5:
- 5.0: `get_generic_section_name()` (MDL-82349) — how the format names sections; use it instead of `get_string('sectionname', ...)`. `get_update_url()` replaces deprecated `get_non_ajax_cm_action_url()` (MDL-82767). `invalidate_all_session_caches()` resets the course editor cache for all users (MDL-83185).
- 5.1: `get_max_sections()` and the `maxsections` setting are deprecated, removed in 6.0 (MDL-84291); courses have unlimited sections. Add your own format setting if you need a limit (format_weeks has one). The `numsections` option (add/remove sections one by one) is deprecated too (MDL-85284). `get_num_sections_data()` in the addsection output no longer uses `$maxsections`; overrides should default it to `0`.
- 5.1: implement the new core_courseformat interface `main_activity_interface` (MDL-85433) if the format shows a single activity on the course page (the notes do not give its namespace).
- 5.2: `set_show_restrictions_expanded()` / `get_show_restrictions_expanded()` (MDL-87929). Subsections always render inline and delegated sections have no description (MDL-87276).
- 5.3: `get_return_section()` is deprecated, use `get_page_section()`; the `get_view_url()` option `sr` is deprecated, use `pagesectionid` (MDL-86284).
- Course-module and section operations moved to action classes (5.2): `core_courseformat\local\cmactions` (`move_before`, `move_end_section`, `delete`, `set_groupmode`, `duplicate`) and `core_courseformat\local\sectionactions` (`set_marker`, `remove_all_markers`, `set_visibility`, `move_after`, `move_at`). They replace `moveto_module()`, `course_delete_module()`, `set_coursemodule_groupmode()`, `duplicate_module()`, `course_set_marker()`, `set_section_visible()` and `move_section_to()`.

### Course format options

Stored in `course_format_options`, per course or per section. Option names must not clash with `course` / `course_sections` columns or completion/conditional form fields. Options are backed up as extra `course` fields — never store course/section IDs in them (section numbers are safe).

Methods: `course_format_options()` (can merge in linear-navigation defaults, below), `section_format_options()`, `get_format_options()`, `create_edit_form_elements()`, `edit_form_validation()`, `update_format_options()`, `update_course_format_options()`, `update_section_format_options()`, `editsection_form()`, `get_default_course_enddate()`, `delete_format_data()`. Since 5.3 (MDL-88669) an option definition can set an `inline_help` flag to show its help text as static text beneath the element.

## Renderer (classes/output/renderer.php)

```php
namespace format_pluginname\output;

use core_courseformat\base as format_base;
use core_courseformat\output\section_renderer;

class renderer extends section_renderer {
    // Required to enable the inplace section title editor.
    public function section_title($section, $course) {
        return $this->render(format_base::instance($course)->inplace_editable_render_section_name($section));
    }
    public function section_title_without_link($section, $course) {
        return $this->render(format_base::instance($course)->inplace_editable_render_section_name($section, false));
    }
}
```

Renderer methods used to refresh page fragments: `render_content` (optional), `course_section_updated` (one section; default `core_courseformat\output\local\content\section`), `course_section_updated_cm_item` (full cm list item; default `...\content\section\cmitem`).

Since 5.1 (MDL-80295, MDL-86337): `course_section_add_cm_control()` and `course_activitychooser()` are deprecated; call `section_add_cm_controls()`. The UPGRADING notes also refer to it as `section_renderer::add_cm_controls`.

## Output classes and templates

Per-element output classes live in `course/format/classes/output/local/`, templates in `course/format/templates/`. `get_output_classname('content\\section')` resolves to `format_pluginname\output\courseformat\content\section` if it exists, else core.

Recommended minimum overrides — `content`, `content\section`, `content\section\cmitem` — each overriding at least `get_template_name()`:

```php
namespace format_pluginname\output\courseformat;

use core_courseformat\output\local\content as content_base;

class content extends content_base {
    public function get_template_name(\renderer_base $renderer): string {
        return 'format_pluginname/local/content';
    }
}
```

Equivalents: `format_pluginname\output\courseformat\content\section` extends `core_courseformat\output\local\content\section` returning `'format_pluginname/local/content/section'`; `...\content\section\cmitem` returning `'format_pluginname/local/content/section/cmitem'`.

### Action menus (5.0, MDL-83527)

- `core_courseformat\output\local\content\cm\controlmenu` fully replaces `course_get_cm_edit_actions()` (deprecated). Extend it as `format_pluginname\output\courseformat\content\cm\controlmenu` to add activity menu options.
- `core_courseformat\output\local\content\section\controlmenu` and `...\content\cm\delegatedcontrolmenu` now build `action_menu_link` objects. Returning arrays of menu items is deprecated; return `action_menu_link` objects.
- format_topics' section controlmenu deprecates `get_highlight_control()` (use `get_section_highlight_item()`) and `get_course_url()` (use `$this->format->get_update_url()`). See `course/format/topics/classes/output/courseformat/content/section/controlmenu.php`.

### Activity chooser (5.1, MDL-86337)

All activity chooser code moved from `core_course` to `core_courseformat`, so formats can override it like any other output. `core_course\output\activitychooserbutton` is now `core_courseformat\output\local\activitychooserbutton` and requires a `section_info` (passing a section number is deprecated). The button must carry `data-section-id` and `data-returnsectionid`.

| Old template | New template |
| --- | --- |
| `core_course/activitychooser` | `core_courseformat/activitychooser` |
| `core_course/activitychooserbutton` | `core_courseformat/local/content/activitychooserbutton` |
| `core_course/local/activitychooser/*` | `core_courseformat/local/activitychooser/*` |

### Minimum mustache structure

Core templates include sub-templates via blocks (`{{$ path }}{{> path }}{{/ path }}`) that children can replace or wrap:

```handlebars
{{! templates/local/content.mustache }}
{{< core_courseformat/local/content }}
    {{$ core_courseformat/local/content/section }}
        {{> format_pluginname/local/content/section }}
    {{/ core_courseformat/local/content/section }}
{{/ core_courseformat/local/content }}
```

```handlebars
{{! templates/local/content/section.mustache }}
{{< core_courseformat/local/content/section }}
    {{$ core_courseformat/local/content/section/cmitem }}
        {{> format_pluginname/local/content/section/cmitem }}
    {{/ core_courseformat/local/content/section/cmitem }}
{{/ core_courseformat/local/content/section }}
```

```handlebars
{{! templates/local/content/section/cmitem.mustache }}
{{< core_courseformat/local/content/section/cmitem }}
    {{! Add any cm item blocks override here. }}
{{/ core_courseformat/local/content/section/cmitem }}
```

Strategies: (1) small tweaks — override blocks in the 3 main templates; (2) big UI change — mirror the intermediate template chain with matching output classes; (3) fully custom — keep the data attributes below and implement `course_section_updated` / `course_section_updated_cm_item`.

Template changes that affect copies of core templates:
- 5.0 (MDL-82679): collapse IDs are `#collapsesectionid{{id}}` (header) and `#coursecontentcollapseid{{id}}` (content), keyed by section id.
- 5.3 (MDL-89496): the collapse/expand-all toggle moved into `core_courseformat/local/content`; target it with `[data-toggle="toggleall"]` (no `collapsesections` id). The `collapsemenu` in `...\content\section` is deprecated; formats overriding `local/content` should render the toggle above the section list.
- 5.3 (MDL-88949): for subsections, the course index treeitem semantics (`aria-owns`, `aria-labelledby`, `aria-expanded`) move to the delegating cm in `core_courseformat/local/courseindex/cm`; update both `courseindex/cm` and `courseindex/section` overrides.

## Course editor integration

The editor is reactive: state loaded via `core_courseformat_get_state`, mutations via `core_courseformat_update_course`, components registered on `core_courseformat/local/courseeditor`. Enable it with `supports_components()` returning `true`.

The editor locates elements by data attributes — keep them in any altered template:

| Concept | Required data attributes |
|---------|------------------------|
| Section | `data-for="section"`, `data-id={SECTION.ID}`, `data-number={SECTION.NUM}` |
| Section header | `data-for="section_title"`, `data-id={SECTION.ID}`, `data-number={SECTION.NUM}` |
| Course module item | `data-for="cmitem"`, `data-id={CM.ID}` |
| Course sections list | `data-for="course_sectionlist"` |
| Section course modules list | `data-for="cmlist"` |
| Course module action link | `data-action={ACTIONNAME}`, `data-id={CM.ID}` |
| Section action link | `data-action={ACTIONNAME}`, `data-id={SECTION.ID}` |
| Section info | `data-for="sectioninfo"` |

If a custom content.mustache breaks editor initialisation, add `id="{{uniqid}}-course-format"` to the content div and init `core_courseformat/local/content` with `component.init('{{uniqid}}-course-format', {}, {{sectionreturn}})` in a `{{#js}}` block. Debug via the footer "Reactive instances" > Highlight ON.

Creating activities (5.0, MDL-83469): `core_courseformat_new_module` replaces deprecated `core_courseformat_create_module` (uses section **id**, not number). Links with `data-action="addModule"` must become `data-action="newModule"` with a `data-sectionid` attribute; the `addModule` mutation is deprecated in favour of `newModule`.

Other 5.0 changes: all course-editing YUI modules deprecated (MDL-82341); the no-JS move UI is gone from action menus (MDL-83562); `core_course_edit_module` / `core_course_edit_section` web services deprecated in favour of `core_courseformat_update_course` (MDL-82342); `externservercourse.php` deprecated in favour of the `\core_course\hook\before_course_viewed` hook (MDL-83764).

### Format-specific state actions

Extend `core_courseformat\stateactions` in `classes/courseformat/stateactions.php`. Every action receives `stateupdates $updates`, `stdClass $course`, `int[] $ids` (cm or section ids), optional `?int $targetsectionid`, optional `?int $targetcmid`. Reference: `format_topics\courseformat\stateactions` (`section_highlight`, `section_unhighlight`). Each action needs a frontend mutation (see `course/format/topics/amd/src/mutations.js`), dispatched from components with `this.reactive.dispatch`.

Non-AJAX execution (5.0, MDL-82767): any state action can run through `course/format/update.php` with `sesskey`, `action`, `courseid`, `ids[]` or `id`, optional `targetsectionid` / `targetcmid`, and `returnurl`. This replaces the old edit GET actions on `course/view.php` / `course/mod.php` (indent, duplicate, hide, show, stealth, delete, groupmode, marker), and non-AJAX deletions now always use the recycle bin. For a confirmation step, create `format_pluginname\output\courseformat\courseupdate` extending `core_courseformat\output\local\courseupdate` with an `ACTIONNAME_confirmation_dialog()` method:

```php
namespace format_pluginname\output\courseformat;

use core\output\renderer_base;
use core\output\single_button;
use core\url;
use core_courseformat\output\local\courseupdate as courseupdate_base;
use stdClass;

class courseupdate extends courseupdate_base {
    public function section_dosomething_confirmation_dialog(
        renderer_base $output,
        stdClass $course,
        array $ids = [],
        ?int $targetsectionid = null,
        ?int $targetcmid = null,
    ): string {
        return $output->confirm(
            message: get_string('sectiondosomething', 'format_pluginname'),
            cancel: $this->returnurl,
            continue: new url($this->actionurl, ['confirm' => 1]),
            displayoptions: ['type' => single_button::BUTTON_DANGER],
        );
    }
}
```

**Note:** the devdocs example declares `namespace format_yourplugin` and `class courseupdate extends courseupdate` (a name clash); the namespace and alias above follow the prose of the same page.

## Linear navigation (since 5.3, MDL-84921)

A sticky "Previous / Next" footer that walks through sections and activities. It is **off** for third-party formats until the format opts in with `uses_linear_navigation()` returning `true` (returning `false` disables it completely).

Course-level setting, like format_topics / format_weeks:

```php
public function course_format_options($foreditform = false) {
    static $courseformatoptions = false;
    if ($courseformatoptions === false) {
        $courseformatoptions = array_merge(
            \core_courseformat\local\linearnavigationsettings::get_course_format_options_default(self::get_format()),
            [], // Your own options.
        );
    }
    if ($foreditform) {
        $courseformatoptions = array_merge_recursive(
            [], // Your own edit-form options.
            \core_courseformat\local\linearnavigationsettings::get_course_format_options_edit_form(self::get_format()),
        );
    }
    return $courseformatoptions;
}
```

Site-level default in `settings.php`:

```php
$settings->add(new admin_setting_configselect(
    'format_pluginname/enablelinearnav',
    new lang_string('linearnavigationsettings', 'core_courseformat'),
    new lang_string('linearnavigationsettings_help', 'core_courseformat'),
    1,
    [1 => get_string('yes'), 0 => get_string('no')],
));
```

Page control:
- `$PAGE->set_show_navigation_footer(false);` suppresses the footer on a page (MDL-87575).
- `\core_courseformat\local\linearnavigationsettings::show_navigation_footer($PAGE)` — will the footer show here.
- `\core_courseformat\local\linearnavigationsettings::is_linear_navigation_enabled($course)` — format + course option only, ignoring page state.
- `$PAGE->set_supplementary_content(action_link $link)` / `get_supplementary_content()` (MDL-88601): adds a link to the sticky footer even when the format has no linear navigation; reset by `reset_theme_and_output()`. `moodle_page::set_has_sticky_footer()` / `has_sticky_footer()` track footer presence.

Previous/Next links (5.2, MDL-87467) point to routes `/cms/{cm}/previous` and `/cms/{cm}/next` handled by `\core_course\route\controller\course_navigation`, which picks the destination at click time with the user's access rules (so the routing system must be configured). Build them with the router, never hard-coded paths:

```php
$nexturl = \core\router\util::get_path_for_callable(
    [\core_course\route\controller\course_navigation::class, 'cm_next_element'],
    ['cm' => $cm->id],
);
```

Activities can override or drop their destination with `cm_info::set_navigation_url()` (see activity-modules.md). Behat steps: `the course linear navigation should be visible` / `should not be visible`.

## Activity badge (since 4.3)

Activity modules (not formats) can show a plain-text badge near the activity name by implementing `mod_PLUGINNAME\output\courseformat\activitybadge` extending `core_courseformat\output\activitybadge`, overriding `update_content()` to set `content`, `style` (one of `self::STYLES`: none/dark/danger/warning/info), `url`, `elementid`, `extraattributes`. Rendered by `core_courseformat/local/content/cm/activitybadge`, which formats can override.

## Core formats in 5.x

- format_social is disabled by default since 5.1 (MDL-85660); admins must re-enable it.
- format_topics ("Custom sections") no longer asks for initial sections on course creation; format_weeks has a "Maximum initial number of weeks" setting replacing max sections (5.1).
