# Moodle 5.3 Theme Plugins

Reference for building theme plugins (`theme_*`) against Moodle 5.3: structure, config.php, Boost inheritance, SCSS, layouts, overriding templates/renderers/images/fonts, and the 5.0–5.3 changes (Bootstrap 5, activity icon colours, dark mode, template relocations, Classic removal). The theme devdocs pages are unchanged since 4.5; the 5.x material comes from the developer updates and UPGRADING notes.

**Classic is gone (5.3, MDL-88351).** Boost is the only core theme to inherit from. On upgrade, if the Classic files are absent, compatible Classic settings (unaddable blocks, brand colour, raw initial/pre SCSS, background and login images) are migrated to Boost and Classic is uninstalled; course, category, cohort and user theme selections pointing to Classic are reset. Presets, the `navbardark` setting and block positions are not migrated (blocks from `side-post` show at the end of Boost's default region). To keep Classic, install it from the Moodle GitHub repository **before** upgrading. Sites forcing `$CFG->theme = 'classic';` in config.php must change that line. Child themes of Classic need Classic installed or must be re-based on Boost.

## File structure

Themes live in `/theme/<themename>` (since 5.1, MDL-83424, `public/theme/<themename>` in a git checkout):

```console
theme/example
|-- amd
|   └-- src
|-- classes
|   └-- output
|-- fonts
|-- fonts_core
|-- fonts_plugins
|   └-- plugintype
|        └-- pluginname
|-- lang
|   └-- en
|       └-- theme_example.php
|-- layout
|-- pix
|   └-- favicon.ico
|   └-- screenshot.png
|-- pix_plugins
|   └-- plugintype
|        └-- pluginname
|-- style
|-- scss
|-- templates
|-- config.php
|-- settings.php
└-- version.php
```

Required language strings in `lang/en/theme_<themename>.php`:

```php
$string['pluginname'] = 'Boost';
$string['choosereadme'] = 'Boost is a modern, highly-customisable theme. ...';
$string['configtitle'] = 'Boost';
```

- `pluginname` — name shown in the Theme selector (without it the theme shows as `[pluginname]`).
- `choosereadme` — text beneath the theme info screenshot.
- `configtitle` — settings title. Add further strings for settings form controls.

`version.php` must set `$plugin->component = 'theme_<themename>';`. The selector `screenshot` (in `pix/`) should be about 500 x 400 px.

## config.php

All theme options go on the `$THEME` object. Boost's own config (abridged):

```php
defined('MOODLE_INTERNAL') || die();

require_once(__DIR__ . '/lib.php');

$THEME->name = 'boost';
$THEME->sheets = [];
$THEME->editor_sheets = [];
$THEME->editor_scss = ['editor'];
$THEME->usefallback = true;
$THEME->scss = function($theme) {
    return theme_boost_get_main_scss_content($theme);
};

$THEME->layouts = [ /* see Layouts below */ ];

$THEME->parents = [];
$THEME->enable_dock = false;
$THEME->extrascsscallback = 'theme_boost_get_extra_scss';
$THEME->prescsscallback = 'theme_boost_get_pre_scss';
$THEME->precompiledcsscallback = 'theme_boost_get_precompiled_css';
$THEME->yuicssmodules = [];
$THEME->rendererfactory = 'theme_overridden_renderer_factory';
$THEME->requiredblocks = '';
$THEME->addblockposition = BLOCK_ADDBLOCK_POSITION_FLATNAV;
$THEME->iconsystem = \core\output\icon_system::FONTAWESOME;
$THEME->haseditswitch = true;
$THEME->usescourseindex = true;
$THEME->activityheaderconfig = ['notitle' => true];
```

Key settings:

- `$THEME->name` — theme name, normally the directory name.
- `$THEME->sheets` — CSS sheet names (no directory, no extension) in `{theme}/style/*.css`. Not needed with SCSS.
- `$THEME->editor_sheets` — stylesheets included inside text editors (Tiny).
- `$THEME->parents` — themes to inherit from. If the parent itself has a parent, list the grandparent too:

```php
$THEME->parents = ['boost'];
```

  Since 5.1 (MDL-79319) layouts inherit through grandparent > parent > child (child wins), so a child can use a layout declared by its parent without redeclaring it.
- `$THEME->scss` — name of a SCSS file in `scss/` compiled on the fly, or a closure returning SCSS (then `@import` paths are relative to the `scss` folder of this theme or any parent).
- `$THEME->prescsscallback` — function returning SCSS injected at the *beginning* (variables from settings).
- `$THEME->extrascsscallback` — function returning SCSS injected at the *end*.
- `$THEME->rendererfactory` — almost all themes must set `theme_overridden_renderer_factory`, or they cannot customise core renderers.
- `$THEME->layouts` — page-type-to-layout-file map (below).
- `$THEME->activityheaderconfig` — activity header options. Since 5.0 (MDL-75610) a layout's own `notitle` option takes priority over this top-level value (Boost sets `notitle` true globally but false in `secure`).
- `$THEME->csstreepostprocessor` (e.g. `theme_boost\autoprefixer`) / `$THEME->csspostprocess`.
- `$THEME->parents_exclude_sheets` / `$THEME->parents_exclude_javascripts` / `$THEME->plugins_exclude_sheets`.
- `$THEME->hidefromselector`, `$THEME->undeletableblocktypes`, `$THEME->addblockposition` (`BLOCK_ADDBLOCK_POSITION_FLATNAV`, `_DEFAULT`, `_CUSTOM`).
- `$THEME->yuicssmodules` — legacy; set empty. `$THEME->javascripts` / `$THEME->javascripts_footer` — deprecated; use AMD modules in `amd/src`.

A theme extending another theme needs no layouts (or layout files) if it makes no layout changes.

## SCSS and styles

Set `$THEME->scss = 'filename';` (a file in `scss/`); Moodle compiles it on first load, or on every load with `themedesignermode`. On-the-fly compilation exists so settings can change SCSS variables via `prescsscallback`.

Plain CSS alternative: sheets in `theme/<themename>/style/*.css` listed in `$THEME->sheets` (current theme after parents). Plugins ship their own `styles.css`; since 5.0 mod_chat/mod_survey styles left Boost (and those modules left core).

Boost builds SCSS via `theme_boost_get_main_scss_content($theme)` plus `theme_boost_get_pre_scss` / `theme_boost_get_extra_scss`; a Boost child reuses this pattern with its own callbacks in `lib.php`.

Every page body gets an ID and path classes, e.g. `/mod/forum/view.php`:

```html
<body id="page-mod-forum-view"
    class="format-weeks path-mod path-mod-forum dir-ltr lang-en
        pagelayout-incourse course-11 context-616 cmid-202 category-1 jsenabled">
```

```css
#page-mod-forum-view .forumpost { border: 1px solid blue; }
.path-mod-forum .forumpost { border: 1px solid blue; }
```

Other body classes: `jsenabled`, `dir-rtl`/`dir-ltr`, `lang-<code>`, `notloggedin`, `format-<courseformat>`, `pagelayout-<layout>`.

### Bootstrap 5 (since 5.0, MDL-75669)

Boost ships Bootstrap 5.3. Update theme SCSS and templates:

- Data attributes are namespaced: `data-bs-toggle`, `data-bs-target` (not `data-toggle`, `data-target`).
- `.dropdown-menu-start` / `-end` replace `-left` / `-right`; `.form-select`, `.form-check`, `.form-check.form-switch`, `.form-range` replace the `.custom-*` controls; `.input-group-append/-prepend` are gone.
- Badges: `.text-bg-*` (or `.bg-*` plus a text colour) instead of `.badge-*`; `.rounded-pill` instead of `.badge-pill`.
- BS4 compatibility layer, deprecated on arrival and removed in 6.0: the `bs4-compat` SCSS (`.media`, `.badge-success`, `.form-inline`, `.mr-1`/`.pl-2`, `.sr-only`, `.font-weight-bold`, `.custom-radio`...), jQuery Bootstrap plugins, and the `theme_boost/bs4-compat` JS module (`BS4compat.init()` rewrites old data attributes in static markup only).
- 5.3 (MDL-88766): AMD code must not import `theme_boost/bootstrap/*`. Use `import {Tooltip} from 'bootstrap';` (5.3+), or `import {Tooltip} from 'theme_boost/index';` for code that must also run on 5.2 and earlier (supported until 7.0). `util`/`dom` helpers load directly: `bootstrap/dom/event-handler`.
- 5.1 (MDL-86548): tables hit by Bootstrap reboot styles can opt out with the `.table-reboot` class.

See the devdocs guide `guides/bs5migration/index.md` for the full class mapping.

### Activity icon colours (since 5.0, MDL-83725)

Icon background colours per activity purpose are plain SCSS variables. Override them before the Boost SCSS loads (e.g. from `prescsscallback` or your pre SCSS):

```scss
$activity-icon-administration-bg: #5915b1;
$activity-icon-assessment-bg: #17857f;
$activity-icon-collaboration-bg: #cf7a34;
$activity-icon-communication-bg: #c70827;
$activity-icon-content-bg: #5915b1;
$activity-icon-interactivecontent-bg: #c70827;
```

Remove any old `$activity-icon-*-filter` variables; they are no longer used.

### Dark colour mode (5.3, experimental, MDL-68037)

Boost supports light/dark modes on the Bootstrap 5.3 colour modes API; the mode is written to `data-bs-theme` on `<html>`. It is off until an admin enables `enablecolourmodes` on Boost's Experimental settings tab. Entry point: `theme_boost\colour_mode`. For Boost children:

- Boost styles now use `var(--#{$prefix}gray-*)`, `var(--#{$prefix}white)`, `var(--#{$prefix}black)`, `var(--#{$prefix}body-bg)`, `var(--#{$prefix}body-color)` instead of the SCSS variables.
- `$card-bg`, `$card-border-color`, `$state-*-bg`, `$state-*-border` and the YUI `$dialogue-*` variables default to custom properties; a preset overriding them should use colours the dark mode can re-point, or override the custom property.
- The dark palette is `scss/moodle/dark.scss`, emitted through Bootstrap's color-mode mixin; it must stay the last import.
- A child theme rendering its own navbar must output `theme_boost\colour_mode::render_menu()` to show the switcher.
- The choice is a user preference mirrored to a `theme_boost_colourmode` cookie (`light`, `dark` or `auto`).
- Use `--bs-*` / `--mds-*` custom properties or `bg-body*` / `text-body*` utilities rather than literal colours; scope mode-specific rules with `[data-bs-theme="dark"]`.

### Fonts (5.3)

Boost's default UI typeface is now self-hosted Noto Sans (`theme/boost/fonts/`, declared in `theme/boost/scss/moodle/fonts.scss`; MDL-88412) with cyrillic subsets and Noto Sans JP scoped to `:lang(ja)` (MDL-89024). `$font-family-sans-serif` is set from the `$mds-font-family-base` token; child themes overriding `$font-family-sans-serif` are unaffected.

## Layouts

Defined in `$THEME->layouts`:

```php
$THEME->layouts = [
    'standard' => [
        'theme' => 'boost',            // Optional: take the layout file from another theme.
        'file' => 'columns2.php',      // Required: file in layout/.
        'regions' => ['side-pre'],     // Required: block regions.
        'defaultregion' => 'side-pre', // Required if regions is non-empty.
        'options' => [],               // Optional: available as $PAGE->layout_options.
    ],
];
```

Boost maps nearly all layout types to `drawers.php`; `login` uses `login.php`, `popup`/`frametop`/`print` use `columns1.php`, `embedded`/`redirect` use `embedded.php`, `maintenance` uses `maintenance.php`, `secure` uses `secure.php`. Common `options`: `langmenu`, `nonavbar`, `nofooter`, `nocoursefooter`, `activityheader` (`notitle`, `nocompletion`, `nodescription`), `noactivityheader`, and since 5.2 `activityinfoinheader` (MDL-87662, enabled by default: manual completion button and activity dates render in the activity header; themes setting it false must override `activity_header` / `activity_info` templates themselves).

Layout types: `base`, `standard`, `course`, `coursecategory`, `incourse`, `frontpage`, `admin`, `mycourses`, `mydashboard`, `mypublic`, `login`, `popup`, `frametop`, `embedded`, `maintenance`, `print`, `redirect`, `report`, `secure`.

```php
$hasnavbar = (empty($PAGE->layout_options['nonavbar']) && $PAGE->has_navbar());
$hasfooter = (empty($PAGE->layout_options['nofooter']));
```

### Layout files

Gather data into a context and render a mustache template:

```php
// theme/boost/layout/columns1.php
$bodyattributes = $OUTPUT->body_attributes([]);

$templatecontext = [
    'sitename' => format_string($SITE->shortname, true,
        ['context' => context_course::instance(SITEID), 'escape' => false]),
    'output' => $OUTPUT,
    'bodyattributes' => $bodyattributes,
];

echo $OUTPUT->render_from_template('theme_boost/columns1', $templatecontext);
```

Minimal layout template — treat every `output.*` call as required:

```handlebars
{{{ output.doctype }}}
<html {{{ output.htmlattributes }}}>
<head>
    <title>{{{ output.page_title }}}</title>
    <link rel="shortcut icon" href="{{{ output.favicon }}}" />
    {{{ output.standard_head_html }}}
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>
<body {{{ bodyattributes }}}>
<div id="page-wrapper">
    {{{ output.standard_top_of_body_html }}}
    <div id="page" class="container-fluid">
        <div id="page-content" class="row">
            <div id="region-main-box">
                <section id="region-main">
                    {{{ output.course_content_header }}}
                    {{{ output.main_content }}}
                    {{{ output.course_content_footer }}}
                </section>
            </div>
        </div>
    </div>
</div>
{{{ output.standard_end_of_body_html }}}
</body>
</html>
{{#js}}
require(['theme_boost/loader']);
{{/js}}
```

**Note:** the devdocs layout example still uses `col-xs-12`, `card-block` and `hidden-print`, which are not Bootstrap 5 classes; they are dropped above. Pick Bootstrap 5 equivalents for your grid.

Block regions: for each region declared in config.php, render its HTML in the layout:

```php
$blockshtml = $OUTPUT->blocks('side-pre');
$hasblocks = strpos($blockshtml, 'data-block=') !== false;
$templatecontext += ['sidepreblocks' => $blockshtml, 'hasblocks' => $hasblocks];
```

```handlebars
{{#hasblocks}}
<section data-region="blocks-column">
    {{{ sidepreblocks }}}
</section>
{{/hasblocks}}
```

All themes must include at least the divs (same ids, e.g. `page-header`) present in Boost.

## Overriding templates

Copy a template to `theme/<themename>/templates/<componentname>/<templatename>.mustache` and edit it; purge caches unless theme designer mode is on.

| Original template | Component | Theme override path |
| --- | --- | --- |
| `blocks/myoverview/templates/view-summary.mustache` | `block_myoverview` | `theme/mytheme/templates/block_myoverview/view-summary.mustache` |
| `group/templates/group_details.mustache` | `core_group` | `theme/mytheme/templates/core_group/group_details.mustache` |
| `lib/templates/modal.mustache` | `core` | `theme/mytheme/templates/core/modal.mustache` |
| `theme/boost/templates/navbar.mustache` | `theme_boost` | `theme/mytheme/templates/theme_boost/navbar.mustache` |

Since 5.2 (MDL-77894), code that appends `!` to a template name bypasses theme overrides.

### Template relocations and markup changes since 4.5

Re-check overrides of these after upgrading:

| Change | Version |
| --- | --- |
| `core_course/activitychooser` → `core_courseformat/activitychooser`; `core_course/activitychooserbutton` → `core_courseformat/local/content/activitychooserbutton`; `core_course/local/activitychooser/*` → `core_courseformat/local/activitychooser/*`. `core_course_renderer::course_activitychooser` no longer used. Chooser buttons need `data-section-id` and `data-returnsectionid`. | 5.1 (MDL-85597, MDL-80295) |
| `theme_boost/flat_navigation` and `theme_boost/nav-drawer` removed. | 5.2 (MDL-87425) |
| `core_courseformat/local/content`: collapse/expand-all toggle moved here, target `[data-toggle="toggleall"]`; course index subsection aria attributes moved to `courseindex/cm`. | 5.3 (MDL-89496, MDL-88949) |
| `theme_boost/drawer`: `drawerheadercontent` block removed, replaced by `drawercontrols` (course index has one collapse/expand-all button). | 5.3 (MDL-89050) |
| Boost's `core/loginform` template moved to core. | 5.3 (MDL-89196) |
| Primary navigation rendered by React `core/nav/PrimaryNav` via new `core/primarymoremenu` (replaces `core/moremenu`); items are `a.mds-nav-pill` (`.mds-nav-pill--selected`, `aria-current="page"`), no `.moremenu` wrapper with JS. Secondary navigation uses `core/nav/Nav`. Styles targeting `.primary-navigation .moremenu` / `.nav-link` need updating. | 5.3 (MDL-89294) |
| Edit mode switch uses the design-system Switch; review `core/editswitch` and `primarynavigation.scss` overrides. | 5.3 (MDL-89814) |
| `core/modal` title is `<h2 class="modal-title fs-5">` (was `<h5>`); apply the same pattern if you override its `header` block. | 5.3 (MDL-75699) |
| Grade action bars use `core/navigation_action_bar` and `core/action_bar` (moved from `core_grades` / `gradereport_grader`). | 5.3 (MDL-81096) |

React components (5.3, MDL-88509): run `./scripts/swizzle.mjs` for an interactive wizard that ejects or wraps a core React component into your theme.

## Overriding renderers

Set `$THEME->rendererfactory = 'theme_overridden_renderer_factory';`. Plugins expose `plugin_renderer_base` subclasses with `render_<renderable>` methods; a theme provides its own version in `classes/output/` to change the data or template. With `templatable`/`named_templatable` outputs, overriding the template alone is often enough. Notes: `core_renderer::sr_text()` is deprecated, use `visually_hidden_text()` (5.0); themes can override the enrolment index via the course renderer `enrolment_options()` method (5.0, MDL-84143); `block_timeline`'s renderer was removed in 5.3.

## Overriding images and fonts

Theme images go in `pix/` (subdirectories allowed), referenced without extension:

```css
.divone { background-image: url([[pix:theme|imageone]]); }
.divtwo { background-image: url([[pix:theme|subdir/imagetwo]]); }
```

In PHP use `$OUTPUT->image_url('imageone', 'theme')` (returns a URL). The devdocs example still shows `pix_url()`, which is deprecated since 3.3 and emits a debugging notice (verified in MOODLE_503_STABLE source); for icons use `pix_icon` or the Mustache `{{#pix}}` helper instead.

Override core/plugin images by mirroring their path:

- `/pix/moodlelogo.png` → `/theme/<themename>/pix_core/moodlelogo.png`
- `/pix/i/user.svg` → `/theme/<themename>/pix_core/i/user.svg`
- `/mod/chat/pix/monologo.svg` → `/theme/<themename>/pix_plugins/mod/chat/monologo.svg` (devdocs example; mod_chat left core in 5.0, the pattern is the same for any plugin)

Deprecated icons (do not rely on overriding them): `core:e/insert_col_after`, `core:e/insert_col_before`, `core:e/split_cells`, `core:e/text_color`, `core:t/locktime`, `tool_policy/level` (5.1); `core:e/text_highlight`, `core:e/text_highlight_picker` (5.1, removal in 6.0); `core:t/blocks_drawer`, `core:t/blocks_drawer_rtl`, `core:t/index_drawer` (5.2).

Fonts: theme fonts go in `fonts/` (no subdirectories; lowercase alphanumeric + underscore; WOFF recommended); core font overrides in `fonts_core/`, plugin fonts in `fonts_plugins/plugintype_pluginname/`:

```css
@font-face {
    font-family: goodDogFont;
    src: url([[font:theme|good_dog.woff]]);
}
```

(`[[font:mod_book|3dumb.woff]]` references `/mod/book/fonts/3dumb.woff`.)

## Settings

A theme may ship a standard `settings.php`; define lang strings for each control. `configtitle` titles the settings page. Boost exposes SCSS-affecting settings and injects them via `prescsscallback` / `extrascsscallback`. Since 5.3 (MDL-89528), admin setting classes are renamed and autoloaded, old names still working (e.g. `\admin_setting_scsscode` → `\core_admin\setting\setting\scsscode`, `\admin_setting_configthemepreset` → `\core_admin\setting\setting\configthemepreset`).

## CSS caching

Outside theme designer mode, compiled CSS is cached at:

```
<moodledata>/localcache/theme/<global_theme_revision>/<theme_name>/css/all_<theme subrevision>.css
```

The path uses the global `themerev` config and per-theme `themesubrev`; incrementing either regenerates it. Build from the CLI:

```bash
php admin/cli/build_theme_css.php --themes boost
```
