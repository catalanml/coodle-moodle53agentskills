# Moodle 5.3 Output API and Mustache Templates

Compact reference for page output ($PAGE/$OUTPUT), renderers/renderables (Mustache and React), Mustache templates, human-readable dates, output formatting functions, and html_writer, distilled from the Moodle 5.3 developer docs and the 5.0–5.3 upgrade notes.

## Page setup and output journey

`$PAGE` is a global `moodle_page` instance tracking page state: context, URL, layout, title, headings. `$PAGE->requires` is a `\core\output\requirements\page_requirements_manager` for queuing JS/CSS in the correct place. `$OUTPUT` is the generic core renderer; plugins should use a plugin-specific renderer obtained via `$PAGE->get_renderer()`.

Typical page (admin tool example; in a 5.1+ git checkout the file lives under `public/admin/tool/demo/`, see note below):

```php
<?php
require(__DIR__ . '/../../../config.php');
require_once($CFG->libdir . '/adminlib.php');

admin_externalpage_setup('tooldemo'); // Admin pages: setup + permission checks.

$title = get_string('pluginname', 'tool_demo');
$PAGE->set_url(new \core\url('/admin/tool/demo/index.php'));
$PAGE->set_title($title);
$PAGE->set_heading($title);

$output = $PAGE->get_renderer('tool_demo');

echo $output->header();
echo $output->heading($title);

$renderable = new \tool_demo\output\index_page('Some text');
echo $output->render($renderable);

echo $output->footer();
```

Notes:
- Do all logic (DB queries, params, access checks) before creating the renderable; pass results as data.
- `footer()` also prints everything queued in the requirements manager. All JavaScript goes in the footer for performance — never inline in content or the head.
- Non-admin pages set context/URL themselves, e.g. `$PAGE->set_context(\core\context\system::instance()); $PAGE->set_url('/');`.
- Since 5.1 (MDL-83424) code sits in a `public/` directory of the repository; URLs and `$CFG->dirroot`-relative paths are unchanged.
- Since 5.3: `$PAGE->set_supplementary_content()` / `get_supplementary_content()` inject an `action_link` into the sticky footer; `$PAGE->set_show_navigation_footer(false)` suppresses the sticky navigation footer; `set_has_sticky_footer()` / `has_sticky_footer()` track sticky-footer presence.

### Renderers via dependency injection (since 5.0, MDL-83888)

In DI-built classes avoid the `$PAGE` global; inject `\core\output\renderer_helper` (a wrapper around `$PAGE`, which cannot be injected directly because it can change during the request):

```php
class my_di_example {
    public function __construct(
        /** @var \core\output\renderer_helper $rendererhelper the renderer helper */
        protected readonly \core\output\renderer_helper $rendererhelper,
    ) {
    }

    public function do_something(): void {
        /** @var \tool_demo\output\renderer $renderer */
        $renderer = $this->rendererhelper->get_renderer('tool_demo');
        $core = $this->rendererhelper->get_core_renderer();
    }
}
```

## Renderables

A renderable (`\core\output\renderable`, no methods) holds all data needed to display something. When you call `$renderer->render($widget)` the rendering priority is:

1. A matching `render_<classname>()` method on the renderer.
2. A class implementing `\core\output\react_component_renderable` (5.3).
3. A class implementing `\core\output\named_templatable`.
4. A class implementing `\core\output\templatable` (template name guessed from the class name).

Docs recommendation: use `react_component_renderable` for new React UI and `named_templatable` for Mustache; plain `templatable` is preserved but not recommended.

### named_templatable (preferred for Mustache)

```php
namespace tool_demo\output;

use core\output\named_templatable;
use core\output\renderable;
use core\output\renderer_base;

class index_page implements named_templatable, renderable {
    public function __construct(
        /** @var string $sometext Text passed to the template. */
        protected string $sometext,
    ) {
    }

    #[\Override]
    public function get_template_name(renderer_base $renderer): string {
        return 'tool_demo/index_page';
    }

    #[\Override]
    public function export_for_template(renderer_base $output): \stdClass {
        return (object) [
            'sometext' => $this->sometext,
        ];
    }
}
```

- `export_for_template()` must return only simple types (`array`, `stdClass`, `bool`, `int`, `float`, `string`, or `Stringable`).
- With plain `templatable` the template is guessed: class `tool_demo\output\mywidget` → `tool_demo/mywidget`. You can also render explicitly: `$renderer->render_from_template('tool_demo/x', $widget->export_for_template($renderer))`.

### React component renderables (since 5.3, MDL-89296)

```php
namespace tool_demo\output;

class my_react_widget implements
    \core\output\react_component_renderable,
    \core\output\renderable
{
    #[\Override]
    public function get_react_component_name(): string {
        return 'core/example'; // Path inside @moodle/lms; Moodle adds the '@moodle/lms/' prefix.
    }

    #[\Override]
    public function get_react_component_props(\core\output\renderer_base $renderer): \stdClass {
        return (object) [
            'filter' => get_user_preference('tool_demo/widget_filter', 'all'),
            'limit' => get_user_preference('tool_demo/widget_result_limit', 25),
        ];
    }
}

// echo $PAGE->get_renderer('tool_demo')->render(new my_react_widget());
```

`render()` emits a mount placeholder via `html_writer::react_component()`. Keep props to initial configuration and preferences only — the component fetches its own data (no Mustache-style pre-loaded strings or data). For the `{{#react}}` Mustache helper (5.2), autoinit, TypeScript sources, build and theming see `react.md`.

## Renderers

```php
namespace tool_demo\output;

class renderer extends \core\output\plugin_renderer_base {
    // Optional: a render_<classname>() method wins over automatic routing.
    public function render_index_page(index_page $page): string {
        return $this->render_from_template('tool_demo/index_page', $page->export_for_template($this));
    }
}
```

The renderer class may be empty. You do NOT need a `render_*` method when using `react_component_renderable`, `named_templatable`, or `templatable` with a same-named template. Themes override renderer methods (e.g. to manipulate data) or templates to customise output.

## Reusing output classes in web services (since 5.1, MDL-85509)

`export_for_template()` data is tied to the template and is not a stable API. To reuse an output class in a web service, also implement `\core\output\externable` and write an exporter (`\core\external\exporter`):

```php
class myname implements \core\output\externable, \core\output\named_templatable, \core\output\renderable {
    // ... constructor, export_for_template(), get_template_name() ...

    #[\Override]
    public function get_exporter(?\core\context $context = null): myname_exporter {
        $context = $context ?? \core\context\system::instance();
        return new myname_exporter($this, ['context' => $context]);
    }

    #[\Override]
    public static function get_read_structure(
        int $required = VALUE_REQUIRED,
        mixed $default = null,
    ): \core_external\external_single_structure {
        return myname_exporter::get_read_structure($required, $default);
    }

    #[\Override]
    public static function read_properties_definition(): array {
        return myname_exporter::read_properties_definition();
    }
}
```

The exporter (`mod_x\external\myname_exporter extends \core\external\exporter`) implements `define_properties()`, `define_related()` (usually `['context' => 'context']`), `define_other_properties()` and `get_other_values(\renderer_base $output)`. In the external function: `$renderer = \core\di::get(\core\output\renderer_helper::class)->get_core_renderer(); return (new myname($cm))->get_exporter($context)->export($renderer);` and `execute_returns()` returns `myname::get_read_structure()`. Any class implementing `externable` must have a PHPUnit test verifying the structure and values of the exported data.

## Rendering templates from PHP

```php
$data = [
    'name' => 'Lorem ipsum',
    'description' => format_text($description, FORMAT_HTML),
];
echo $OUTPUT->render_from_template($templatename, $data);
```

Do not name context variables after helpers (`str`, `js`, `pix`, `react`, ...) — they will silently fail.

## Template files

- Location: `templates/*.mustache` in a component. Identified as `frankenstyle_component/filename` (no extension): `mod_lesson/timer` → `mod/lesson/templates/timer.mustache`. Sub-directories allowed (first level must follow namespace level-2 naming rules).
- Every template starts with a GPL comment plus a documentation comment:

```handlebars
{{!
    @template plugintype_pluginname/template_name

    Template purpose and description.

    Classes required for JS:
    * none

    Data attributes required for JS:
    * none

    Context variables required for this template:
    * none

    Example context (json):
    {
    }
}}
```

The "Example context (json)" is used by the Template library to preview the template.

## Mustache syntax

- `{{name}}` — variable, HTML-escaped, looked up in current then parent contexts.
- `{{{description}}}` — raw unescaped HTML. Only for values already processed by `format_text()` or similar.
- Sections `{{#key}}...{{/key}}` — hidden when false/empty list; repeated per item for a non-empty list; a callable is invoked (e.g. `{{#output.should_display_navbar_logo}}...{{/output.should_display_navbar_logo}}`).
- Inverted sections `{{^key}}...{{/key}}`; comments `{{! ... }}`; partials `{{> core/loading }}` (share the parent's context).
- Blocks: parent defines `{{$name}}default{{/name}}`; a child extends it:

```handlebars
{{< tool_demo/section }}
    {{$heading}} Latest news {{/heading}}
    {{$content}} Today I learned how to use blocks in Mustache! {{/content}}
{{/ tool_demo/section}}
```

- `{{uniqid}}` — page-unique ID prefix; the only acceptable way to use IDs (for one-to-one JS/template links): `<div id="{{uniqid}}-somethingspecific">`.

## Mustache helpers

- `{{#str}} helloworld, mod_greeting {{/str}}` — get_string. Optional third param is `$a`: `{{#str}} backto, core, {{name}} {{/str}}`; JSON for non-scalar `$a`: `{{#str}} counteditems, core, { "count": "42", "items": "courses" } {{/str}}`. Help icons use the `{{>core/help_icon}}` partial.
- `{{#quote}}` — escape variables inside `{{#str}}` JSON: `{ "count": {{count}}, "items": {{#quote}} {{itemname}} {{/quote}} }`.
- `{{#pix}} t/edit, core, Edit this section {{/pix}}` — icon: identifier, component, alt text.
- `{{#userdate}} {{time}}, {{#str}} strftimedate, core_langconfig {{/str}} {{/userdate}}` — format a timestamp in the user's timezone.
- `{{#shortentext}} 15, {{{description}}} {{/shortentext}}` — shorten text, word- and HTML-preserving.
- `{{#js}}...{{/js}}` — JS run when the template is added to the page (footer in PHP rendering; returned separately in JS rendering). Keep logic in AMD modules. Still supported for enhancing Mustache markup but discouraged for new React-based UI.
- `{{#react}} {"component": "@moodle/lms/...", "props": {...}} {{/react}}` — since 5.2 (MDL-87765), React mount point; see `react.md`.

```handlebars
{{#js}}
require(['theme_boost/form-display-errors'], function(module) {
    module.enhance({{#quote}}{{element.id}}{{/quote}});
});
{{/js}}
```

## Rendering templates from JavaScript

```javascript
import {exception as displayException} from 'core/notification';
import Templates from 'core/templates';

Templates.renderForPromise('block_looneytunes/profile', {name: 'Tweety bird', intelligence: 2})
.then(({html, js}) => {
    Templates.appendNodeContents('.block_looneytunes .content', html, js);
    return;
})
.catch((error) => displayException(error));
```

`core/templates` loads and caches templates via AJAX, fetches missing strings in one request, and splits JS from HTML. After inserting HTML yourself, run `Templates.runTemplateJS(js)`. Other insertion helper: `Templates.replaceNodeContents(node, html, js)`.

## Context data rules and gotchas

- PHP arrays must be sequentially indexed from 0 to iterate; fix with `array_values($myarray)`.
- No portable "non-empty array" test (`{{#users.0}}` PHP only, `{{#users.length}}` JS only). Provide a boolean like `hasusers`.
- Squash significant whitespace between tags with comments:

```handlebars
<a href="blah">{{!
}}{{icon}}{{!
}}{{name}}</a>
```

- `{{globals.config.wwwroot}}`, `{{globals.config.sesskey}}` (same properties as `get_config_for_javascript()`) are available ONLY in client-side JS rendering.
- Since 5.2 (UPGRADING, MDL-87066) a `clean_string()` helper prevents double escaping in Mustache templates; the devdocs do not document its signature.

## Overriding templates in a theme

Copy the template into `theme/<yourtheme>/templates/<component>/<name>.mustache` (repository paths shown, 5.1+):

| Original | Component | Theme override |
| --- | --- | --- |
| `public/blocks/myoverview/templates/view-summary.mustache` | `block_myoverview` | `public/theme/mytheme/templates/block_myoverview/view-summary.mustache` |
| `public/lib/templates/modal.mustache` | `core` | `public/theme/mytheme/templates/core/modal.mustache` |
| `public/theme/boost/templates/navbar.mustache` | `theme_boost` | `public/theme/mytheme/templates/theme_boost/navbar.mustache` |

Templates are cached: purge caches (or use theme designer mode). Since 5.2 (MDL-77894), appending an exclamation mark to a template name ignores theme overrides (UPGRADING only; no devdocs example). Removed in 5.2: `theme_boost/flat_navigation`, `theme_boost/nav-drawer`. In 5.3 the primary/secondary navigation is a React component (`core/nav/PrimaryNav`, `core/nav/Nav`; `a.mds-nav-pill` instead of `.nav-link`/`.moremenu`) — overrides targeting the old markup need review.

## Template coding style

- GPL header and documentation comment in every template (override may omit docs if parent's still apply).
- Use data-attributes for JS hooks — not classes (styling only) or IDs (must be unique per page).
- Prefer framework (Bootstrap 5, since 5.0) classes over custom CSS: `visually-hidden` (not `sr-only`), `ms-*`/`me-*`, `float-start`/`float-end`, `data-bs-toggle`/`data-bs-target`. A BS4 compatibility layer exists until 6.0.
- Re-use core templates; wrap each template in one node classed after it (e.g. `<div class="core_user_header">`); lower-case hyphenated class names.
- Since 5.3 Boost has an experimental dark colour mode: take colours from Bootstrap custom properties (`--bs-body-bg`, `--bs-body-color`, `--bs-border-color`, ...) or utilities (`bg-body`, `text-body`, `bg-body-secondary`) rather than `bg-white`/`text-dark`; scope genuinely different colours with `[data-bs-theme="dark"]`; render icons through the icon API so they inherit `currentColor`.
- Accessible headings: `core/modal` titles are `<h2 class="modal-title fs-5">` since 5.3 (start headings inside modal content at `<h3>`); `headinglevel` context exists on `core/select_menu` (5.0) and `core/notification_base` (5.3), and as a `$displayoptions` key of `core_renderer::confirm()` (5.2).

### Reusable core templates

Put customization points in `{{$block}}`s, keep JS coupled to semantic `data-*`/ARIA attributes (not DOM nesting or CSS classes), and split repeated items into partials so callers override small blocks (e.g. `{{$tablist}}`, `{{$tabcontent}}` wrapping `{{#tabs}}{{> mod_example/tab_header_item}}{{/tabs}}`) instead of copying core HTML. Keep the example context valid JSON and document every context field, class and data attribute.

## String formatting functions

- `s($var, $strip=false)` / `p($var, $strip=false)` — escape `< > " ' &`; `s()` returns, `p()` prints. For form values and plain user text.
- `format_text($text, $format = FORMAT_MOODLE, $options)` — html/plain/markdown/moodle text. Cleans, applies filters, caches. Formats: `FORMAT_MOODLE`, `FORMAT_HTML`, `FORMAT_PLAIN`, `FORMAT_MARKDOWN`. Key options: `noclean` (never true for user content), `trusted`, `filter`, `context` (always pass the right one), `para`, `newlines`, `overflowdiv`, `allowid`, `blanktarget`.
- `format_string($text, $striplinks = true, ['context' => $context, 'escape' => true, 'filter' => true])` — short non-HTML strings (activity names). No cleaning: input must be cleaned with `xxx_param()`.

Apply filters only at output time (e.g. language filters depend on the viewing user).

## Human-readable dates (since 5.0, MDL-83873)

`core_calendar\output\humandate` (single timestamp) and `core_calendar\output\humantimeperiod` (range) show "Today/Yesterday/Tomorrow", respect the user's 12/24h preference, apply alert styling near the date, and drop the redundant end date for same-day periods.

```php
use core_calendar\output\humandate;
use core_calendar\output\humantimeperiod;

$renderer = $PAGE->get_renderer('core', 'output');
$timestamp = \core\di::get(\core\clock::class)->time();

echo $renderer->render(humandate::create_from_timestamp($timestamp));
echo $renderer->render(humandate::create_from_timestamp(
    timestamp: $timestamp,
    link: new \core\url('/calendar/view.php', ['view' => 'day', 'time' => $timestamp]),
));
echo $renderer->render(humandate::create_from_timestamp(timestamp: $timestamp, timeonly: true));

echo $renderer->render(humantimeperiod::create_from_timestamp($timestamp, $timestamp + HOURSECS));
```

`calendar_format_event_time()`, `calendar_day_representation()` and `calendar_time_representation()` are deprecated since 5.0 — use these classes.

## Simple element rendering (renderer methods)

```php
$output->container(string $contents, ?string $classes = null, ?string $id = null): string // <div>
$output->paragraph(string $contents, ?string $classes = null, ?string $id = null): string // <p>
$output->visually_hidden_text(string $contents): string // <span class="visually-hidden">
```

`classes` is a space/comma-separated string, not an array. `sr_text()` is deprecated since 5.0 (MDL-81825) — use `visually_hidden_text()`.

## html_writer essentials

`\core\output\html_writer` outputs basic HTML, typically inside renderer methods. Prefer templates.

```php
html_writer::div('kermit', 'frog', ['id' => 'tophat']); // <div class="frog" id="tophat">kermit</div>
html_writer::start_span('zombie') . 'BRAINS' . html_writer::end_span();
html_writer::tag($tagname, $contents, $attributes = null);
html_writer::start_tag($tagname, $attributes = null);
html_writer::end_tag($tagname);
html_writer::empty_tag($tagname, $attributes = null);
html_writer::nonempty_tag($tagname, $content, $attributes = null);
html_writer::attributes($attributesarray);
```

Since 5.3 (MDL-89296): `html_writer::react_component(string $modulename, array|string|\stdClass|\JsonSerializable $props)` outputs `<div data-react-component="..." data-react-props="...">`. Unlike the renderable, `$modulename` must include the full `@moodle/lms/` prefix (e.g. `'@moodle/lms/core/example'`), and `$props` must NOT be pre-JSON-encoded.

## Progress bars and task indicator

Standard (renders while page loads; needs `define('NO_OUTPUT_BUFFERING', true);` outside CLI):

```php
$pb = new \core\output\progress_bar('bar', 500);
$pb->create();
$pb->update($current, $total, 'message');
```

Stored (DB-backed, live AJAX updates; usable in CLI or tasks): `$progress = new \core\output\stored_progress_bar('example-cli-bar'); $progress->start(); $progress->update($current, $total, 'message');` (or `update_full()` to set % directly).

In tasks: `use \core\task\stored_progress_task_trait;`. Since 5.0 (MDL-81714) call `$task->initialise_stored_progress()` when queuing (wraps `stored_progress_bar::store_pending()`; shows a "pending" bar), then in `execute()` `$this->start_stored_progress()` (sets the start time) and `$this->progress->update(...)`. `\core\output\task_indicator` (5.0) displays the bar and message for such a task — see the "Task indicator" page of the component library.

## Inplace editable

Class `\core\output\inplace_editable` (templatable), template/JS module `core/inplace_editable`, web service `core_update_inplace_editable`.

1. Callback `<component>_inplace_editable($itemtype, $itemid, $newvalue)` in `lib.php`: `\core_external\external_api::validate_context(...)`, `require_capability(...)`, `clean_param($newvalue, ...)`, update the record, return a new `\core\output\inplace_editable(component, itemtype, itemid, editable, displayvalue, value, edithint, editlabel)`. The web service only checks login — all capability checks happen in the callback.
2. Where displayed, build the same object (`has_capability()` for `$editable`) and `echo $OUTPUT->render($tmpl);`.
3. Variants: `$tmpl->set_type_select($optionsmenu)` or `$tmpl->set_type_toggle([0, 1])`.

Events: `core/local/inplace_editable/events` (`elementUpdated`, `elementUpdateFailed`).

## Other output changes 5.0–5.3

- Added: `core\output\local\collapsable_section` / `core/local/collapsable_section` template (5.0); `core\output\local\properties\iconsize` enum (5.0); `core_course\output\activity_icon` (5.0); `toggle.mustache` `extraattributes` replaces `dataattributes` (5.1); `\core\output\local\dropdown\dialog` `$definition['autoclose']` (5.1); `core\output\action_menu\subpanel` `url` param and `core\output\submenu` renderable (5.3); `flexible_table::set_columnheadersattributes(...)` for header attributes (5.3). Constructor details: see the component library.
- Removed: boolean `primary` 4th param of `\core\output\single_button` (5.0, now a string); `core_renderer::activity_information()`, `htmllize_file_tree()`, `renderer_base::should_display_main_logo()`, `action_menu::set_alignment()`/`set_constraint()`/`do_not_enhance()` (5.0–5.2); `core_output_load_fontawesome_icon_map` WS (use `core_output_load_fontawesome_icon_system_map`).
- Deprecated: `core/checkbox-toggleall-master*`/`-slave` templates → `-toggler*`/`-target` (5.1); `core/external_content_banner` (5.3).
