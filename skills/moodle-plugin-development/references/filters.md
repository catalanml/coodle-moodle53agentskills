# Moodle 5.3 Text Filter Plugins

Scope: structure, class conventions, filter() method, local configuration and dynamic-content handling for `filter_*` plugins, per Moodle 5.3 developer docs and UPGRADING notes.

Filters automatically transform content before output (e.g. TeX to images, media links to players, glossary auto-linking). They are applied to all content passed through `format_text()` (headers and content) and `format_string()` (headers only) from the Output API. Filters are one of the easiest plugin types to create.

## Changes since 4.5

- The 5.3 docs now title the main file section `classes/text_filter.php` (it was still headed `filter.php` in the 4.5 docs, although the 4.5 example already used the autoloaded class). The autoloaded class itself dates from 4.5 (MDL-82427); see "Supporting Moodle 4.4" below.
- Removed in 5.2 (MDL-85233): MimeTeX support in `filter_tex` and `filter_algebra`; they need LaTeX tools (`latex`, `dvips`, and `convert`/`dvisvgm`) and the `pathmimetex` setting is gone.
- Since 5.3 (MDL-87554): `filter_tex` and `filter_algebra` store rendered images with the File Storage API instead of `$CFG->dataroot/filter/{tex,algebra}/`, with a new `rendered_images` cache definition in each; an upgrade step migrates old images. Follow the same pattern for any filter that generates files: use the File API and MUC, not ad-hoc dataroot directories.
- Since 5.1 (MDL-83424) core code lives under `public/`; in a git checkout filters are at `public/filter/<name>/`. Paths below are relative to `$CFG->dirroot`.

## File structure

Filter plugins live in `/filter`. Minimal layout for `filter_pluginname`:

```console
 filter/pluginname/
 |-- lang
 |   `-- en
 |       `-- filter_pluginname.php
 |-- classes
 |   `-- text_filter.php
 `-- version.php
```

- `classes/text_filter.php` (required): the main filter class. Filters have only one mandatory class: `filter_pluginname\text_filter`, extending the core `\core_filters\text_filter` class (not a class in the legacy `filter.php`).
- `version.php` (required): standard plugin version file.
- `lang/en/filter_pluginname.php` (required): language strings; filters use `filtername` for their display name, e.g.

```php
$string['filtername'] = 'Activity names auto-linking';
```

Other common plugin files (`db/access.php`, `settings.php`, privacy provider, ...) work as for any plugin.

## The filter class and filter() method

```php
<?php
// This file is part of Moodle - https://moodle.org/
//
// Moodle is free software: you can redistribute it and/or modify
// it under the terms of the GNU General Public License as published by
// the Free Software Foundation, either version 3 of the License, or
// (at your option) any later version.
//
// Moodle is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU General Public License for more details.
//
// You should have received a copy of the GNU General Public License
// along with Moodle.  If not, see <https://www.gnu.org/licenses/>.

namespace filter_pluginname;

/**
 * Example text filter.
 *
 * @package    filter_pluginname
 * @copyright  2026 Your Name <you@example.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */
class text_filter extends \core_filters\text_filter {
    /**
     * Filter the given text.
     *
     * @param string $text The text to filter.
     * @param array $options Filter options.
     * @return string The modified text.
     */
    public function filter(string $text, array $options = []) {
        return $text;
    }
}
```

`filter()` receives the text and an options array, and must return the (possibly modified) text.

### Supporting Moodle 4.4 and earlier

Since 4.5 (MDL-82427) filters use class autoloading. 5.3 can be upgraded from 4.4, so a plugin codebase may still need to run on 4.4. In that case also keep a file in the old location that aliases the new class:

```php
// filter/pluginname/filter.php
class_alias(\filter_pluginname\text_filter::class, \filter_pluginname::class);
```

## Testing a filter

Enable the filter in the filters administration screen (set it "On"). Add content to a label resource; viewing the course should show the transformed text.

## Performance

Every active filter runs on every `format_text()` / `format_string()` call, so filters can cause big performance problems. Use caching if the filter hits the database or similar sources. If the filter depends on a special syntax or substring, do a cheap `strpos()`/`stripos()` check before any regex work:

```php
public function filter(string $text, array $options = []) {
    if (empty($text)) {
        return $text;
    }
    if (stripos($text, '</a>') === false) {
        // Performance shortcut - if there is no </a> tag, nothing can match.
        return $text;
    }
    // Complex operations on <a> links here.
    return $text;
}
```

The devdocs version of this example keeps an `!is_string($text)` guard and an untyped `$text` from older filters; with the typed `string $text` signature the type check is redundant.

## Local (per-context) configuration

Filters can be configured differently per context (e.g. the glossary filter linking a different glossary in Forum A vs Forum B). Provide a `filterlocalsettings.php` file defining a form that subclasses `\core_filters\form\local_settings_form`; besides standard formslib methods the docs say you must define a `save_changes` method (the docs example does not show one — check the base class for its signature).

```php
// filter/pluginname/filterlocalsettings.php
class pluginfile_filter_local_settings_form extends \core_filters\form\local_settings_form {
    protected function definition_inner(\MoodleQuickForm $mform) {
        $mform->addElement(
            'text',
            'word',
            get_string('word', 'filter_pluginname'),
            ['size' => 20]
        );
        $mform->setType('word', PARAM_NOTAGS);
    }
}
```

The class name above is copied from the devdocs example; the docs do not state a naming rule for it.

Local configuration is available in the filter class via `$this->localconfig`:

```php
namespace filter_pluginname;

class text_filter extends \core_filters\text_filter {
    public function filter(string $text, array $options = []) {
        $search = $this->localconfig['word'] ?? 'default';
        return str_replace($search, "Hello $search!", $text);
    }
}
```

## Filtering dynamic (AJAX-loaded) content

Content added after page load has already been PHP-filtered, but filters that need JS post-processing (e.g. MathJax) can listen for the content-updated event from the `core_filters/events` module and reprocess the affected DOM nodes:

```js
import {eventTypes} from 'core_filters/events';

/** @var {bool} Whether the listener has been registered */
let initialised = false;

/**
 * Initialise the dynamic content filter.
 *
 * @listens event:filterContentUpdated
 */
export const init = () => {
    if (initialised) {
        return;
    }
    initialised = true;
    document.addEventListener(eventTypes.filterContentUpdated, contentUpdatedHandler);
};

/**
 * Process newly added nodes.
 *
 * @param {Event} event The event.
 */
const contentUpdatedHandler = (event) => {
    const updatedContent = event.detail.nodes;
    updatedContent.forEach(content => {
        // Alter any updated content.
    });
};
```

**Note:** the devdocs `_examples/dynamic.js` guard reads `if (!firstLoad) { return; }` with `firstLoad` uninitialised, so it never registers the listener; the guard above is inverted to register once.
