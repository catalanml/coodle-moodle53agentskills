# JavaScript in Moodle (5.3)

ES2015+ modules in `amd/src`, built with grunt, loaded via RequireJS; core helpers for AJAX, strings, templates, and modals. Moodle 5.3 also supports ESM/TypeScript/React components (see "Frontend direction" and `react.md`).

## Frontend direction (5.2+ / 5.3)

Per the 5.3 frontend overview, Moodle now supports **ECMAScript modules, React and TypeScript** for component-based UI, alongside the existing AMD system:

- Source in `<component>/js/esm/src/` (TypeScript, `.ts`/`.tsx`), compiled into `js/esm/build/` — never edit compiled files. Modules are resolved through the browser import map as `@moodle/lms/<component>/<path>` (e.g. `@moodle/lms/mod_book/viewer`).
- A React module's **default export** is a function component; templates only decide *where* it appears (`{{#react}}` helper, since 5.2), React decides the UI.
- Pass minimal props (IDs, simple config); the component fetches the rest. Put web service calls in service modules (e.g. `services/courses.ts`), not inline in components.
- Use the design system and tokens; avoid inline styles and hard-coded colours so themes can restyle.
- `{{#js}}` + `require()` remains supported for enhancing Mustache markup and legacy code, but is discouraged for new React-based components. Prefer ESM-native APIs where available, use compatibility helpers for AMD, and avoid introducing new AMD-based patterns.
- The custom reactive library (`core/reactive`) remains supported for existing code but should not be used for new features — use React.

Full contract (`{{#react}}`, `react_autoinit`, `grunt react`, import map, Jest testing, theming, swizzle): see `react.md`. PHP side (`react_component_renderable`, `html_writer::react_component()`): see `output-templates.md`.

## Module format and naming (AMD/ESM in `amd/src`)

- All new JS must be ES module (ESM) format; Babel transpiles it to AMD and RequireJS loads it. AMD `define()` syntax is legacy but interoperable.
- Source: `<componentdir>/amd/src/modulename.js` -> module name `component_name/modulename` (Frankenstyle component, no `.js` extension). In a 5.1+ git checkout components live under `public/` (e.g. `public/mod/forum/amd/src/`).
- Built output: `<componentdir>/amd/build/modulename.min.js` — commit the build files too.
- Subdirectories: first directory must be a Moodle API name or `local`. Conventions: `local/<feature>/selectors.js` for CSS selectors, `local/<feature>/repository.js` for web service calls. Examples: `mod_forum/discussion`, `core_user/local/participants/selectors`.
- Only exported functions are callable from outside. ESM named exports and default export cannot be mixed in one file (default wins if both used).
- Suffix a file `-lazy.js` to make it load on first use instead of on every page (require it with the `-lazy` name).

```js
// mod/example/amd/src/helloworld.js
export const init = () => {
    window.console.log('Hello, world!');
};
```

## Build tooling (grunt)

```bash
npx grunt amd        # build all AMD/ESM modules
npx grunt watch      # rebuild changed files during development (recommended)
npx grunt watch -f   # since 5.2: --force keeps building even when lint errors exist
```

Development mode: set `$CFG->cachejs = false;` — Moodle still serves minified JS but sends source maps. If `npm-shrinkwrap.json` changed after installing extras: `git checkout npm-shrinkwrap.json && npm ci`. React/TypeScript sources use the separate `grunt react` task (see `react.md`).

## Including modules

From PHP (`js_call_amd` on the page requirements manager):

```php
$PAGE->requires->js_call_amd('mod_forum/discussion', 'init');
$PAGE->requires->js_call_amd('mod_forum/discussion', 'init', [$course->id]);
// Multi-dimensional array -> destructure in JS:
$PAGE->requires->js_call_amd('mod_forum/discussion', 'init', [[
    'courseid' => $course->id,
    'categoryid' => $course->category,
]]);
```

```js
export const init = ({courseid, categoryid}) => { /* ... */ };
```

Params must be `json_encode`-able; > 1Kb triggers a developer warning. Prefer data attributes in the DOM or AJAX over large params.

From a Mustache template (must be inside `{{#js}}`; keep inline JS minimal — it is not transpiled/minified):

```handlebars
<div id="mod_forum-discussion-wrapper-{{uniqid}}">...</div>

{{#js}}
require(['mod_forum/discussion'], function(Discussion) {
    Discussion.init(document.querySelector("#mod_forum-discussion-wrapper-{{uniqid}}"));
});
{{/js}}
```

`{{uniqid}}` is one unique string per rendered template — combine it with other identifiers.

Dynamic import from JS (advanced): `import(`local_examples/local/types/type_${modname}`).then(...)`. Since 5.3, `core/import` lets AMD code do a native ESM dynamic import without Babel rewriting it, and `core/component` provides `appendToDom` / `prependToDom` to mount React components (picked up by `react_autoinit`); signatures are covered in `react.md`, not here.

## DOM event conventions

One delegated listener; selectors in a private (non-exported) `Selectors` object using namespaced `data-*` attributes (not classes); match with `e.target.closest()`:

```js
import Selectors from './local/helloworld/selectors';

const registerEventListeners = () => {
    document.addEventListener('click', e => {
        if (e.target.closest(Selectors.actions.showAlertButton)) {
            window.alert("Thank you for clicking on the button");
        }
    });
};

export const init = () => {
    registerEventListeners();
};
```

```js
// local/helloworld/selectors.js
export default {
    actions: {
        showAlertButton: '[data-action="mod_example/helloworld-update_button"]',
    },
};
```

## AJAX: core/ajax

Call Moodle web service functions directly. The function must have `'ajax' => true` in `db/services.php` (bump the plugin version after changing service definitions). Convention: wrap all calls in a `repository.js` module.

```js
// mod/assign/amd/src/repository.js
import {call as fetchMany} from 'core/ajax';

export const submitGradingForm = (
    assignmentid,
    userid,
    data,
) => fetchMany([{
    methodname: 'mod_assign_submit_grading_form',
    args: {
        assignmentid,
        userid,
        jsonformdata: JSON.stringify(data),
    },
}])[0];
```

```js
import {submitGradingForm} from './repository';

export const doSomething = async() => {
    const response = await submitGradingForm(assignmentId, userId, data);
    window.console.log(response);
};
```

`fetchMany([...requests])` chains multiple WS calls in one HTTP request and returns one promise per request. Considerations:

- WS text output must go through `external_format_text` / `external_format_string` (the 5.3 output docs use `\core_external\util::format_string()`) with the most specific context.
- After adding dynamic content, notify filters via `M.core.event.FILTER_CONTENT_UPDATED`.
- Session-less calls (rare, 100% public data only): `'loginrequired' => false` in `db/services.php` and pass `false` as the 3rd argument to `call`.

## Strings: core/str

Strings are fetched from Moodle and cached in LocalStorage.

- `getString` / `getStrings` — native Promises (preferred, 4.3+)
- `get_string` / `get_strings` — jQuery Promises (legacy)

```js
import {getString} from 'core/str';

getString('close', 'core')
.then((closeString) => {
    window.console.log(closeString);
    return closeString;
})
.catch();
```

Prefetch assets for perceived performance:

```js
import Prefetch from 'core/prefetch';
Prefetch.prefetchString('discussion', 'mod_forum');
Prefetch.prefetchStrings('core', ['yes', 'no', 'maybe']);
Prefetch.prefetchTemplate('core/toast');
Prefetch.prefetchTemplates(['core/toast', 'core/modal']);
```

## Templates: core/templates

`Templates.render(templateName, context)` returns a promise; usable anywhere a string/HTML/Promise is accepted (e.g. modal bodies). See `output-templates.md` for `renderForPromise()` and node insertion helpers.

```js
import Templates from 'core/templates';

const bodyPromise = Templates.render('mod_example/example_modal_content', {id: 42});
```

## Modals: core/modal

Call static `create()` on the modal class (since 4.3, MDL-78324). **Removed in 5.2:** the AMD modules `core/modal_factory` and `core/modal_registry` — it is no longer possible or necessary to register modal types. (The 5.3 devdocs modal page still shows `ModalFactory` examples; ignore them.)

```js
import Modal from 'core/modal';

export const init = async() => {
    const modal = await Modal.create({
        title: 'Test title',
        body: '<p>Example body content</p>',
        footer: 'An example footer content',
        show: true,
        removeOnClose: true,
    });
};
```

Standard types: Delete/Cancel, Save/Cancel (`core/modal_save_cancel`), Cancel. Title/body/footer accept strings or Promises (getString, Templates.render).

```js
import ModalSaveCancel from 'core/modal_save_cancel';
import ModalEvents from 'core/modal_events';
import {getString} from 'core/str';

export const init = async() => {
    const modal = await ModalSaveCancel.create({
        title: 'test title',
        body: getString('confirmchange', 'mod_example'),
    });

    modal.getRoot().on(ModalEvents.save, (e) => {
        // e.g. validate before saving.
    });
};
```

Custom modal type: extend `core/modal` with `TYPE` and `TEMPLATE` static properties; the template extends the `core/modal` core template overriding `{{$title}}`, `{{$body}}`, `{{$footer}}` blocks. Override `configure(modalConfig)` (calling `super.configure(modalConfig)`) for default options or custom arguments.

```js
import Modal from 'core/modal';

export default class MyModal extends Modal {
    static TYPE = "mod_example/my_modal";
    static TEMPLATE = "mod_example/my_modal";
}

// Usage: const modal = await MyModal.create({}); modal.show();
```

Since 5.3 the `core/modal` title is an `<h2 class="modal-title fs-5">` (was `<h5>`): start headings inside modal content at `<h3>`, and use the same pattern if you override the `header` block. Since 5.3, `core/imagedetails/modal` collects alt text / decorative flag / size for an image being embedded.

## Bootstrap 5 (since 5.0, MDL-75669)

- Bootstrap plugins are vanilla JS; data attributes are namespaced: `data-bs-toggle`, `data-bs-target`, `data-bs-html`.
- A BS4 backwards-compatibility layer (jQuery Bootstrap plugin support, BS4 SCSS helpers, silent data-attribute replacement) lasts until final deprecation in 6.0. `theme_boost/bs4-compat` (deprecated on arrival) rewrites old attributes: `import * as BS4compat from 'theme_boost/bs4-compat';` or `$PAGE->requires->js_call_amd('theme_boost/bs4-compat', 'init');`.
- **Since 5.3, AMD modules must not import `theme_boost/bootstrap/*`.** Use the public API instead:

```js
// Moodle 5.3 and later:
import {Tooltip} from 'bootstrap';
// Moodle 5.2 and earlier (still works until 7.0):
import {Tooltip} from 'theme_boost/index';
// util/dom helpers are not public API and are still imported directly (at your own risk):
import EventHandler from 'bootstrap/dom/event-handler';
```

(The bs5migration guide still shows `import Dropdown from 'theme_boost/bootstrap/dropdown'`; that is superseded by the 5.3 upgrade note.)

## jQuery policy

jQuery is strongly discouraged in new code and not generally accepted in core; use vanilla JS/ESM. When interfacing with legacy code, import it: `import jQuery from 'jquery';` (or AMD `define(['jquery'], ...)`). jQuery UI must not be used in core and is highly discouraged in plugins; if unavoidable: `$PAGE->requires->jquery_plugin('ui');` before any output. Since 5.0 `core/sortable_list` emits native events (jQuery listeners still work).

## Promises

- Use `then`/`catch` consistently; you must return at the end of a thennable.
- Never use `.done` or `.fail` — jQuery-only, absent from native Promises.
- A function whose main job is creating a Promise should return it:

```js
const getModal = questionBody => {
    return Modal.create({
        title: getString('my_title', 'mod_example'),
        body: Templates.render('mod_example/example_body', questionBody),
        removeOnClose: true,
    })
    .then(modal => {
        modal.show();
        return modal;
    });
};
```

## Deprecating JavaScript: core/deprecated (since 5.2, MDL-87867)

JS deprecation follows the PHP process: on `main` only, three stages (initial deprecation, final deprecation, removal), noted in an upgrade note, emitting debugging where possible. Use the default export of `core/deprecated` at file level (deprecates the whole file) or inside a method:

```js
import emitDeprecation from 'core/deprecated';

export const myFunction = (...args) => {
    emitDeprecation('mod_example/mymodule::myFunction', {
        replacement: 'myNewFunction',
        since: '5.3',
        mdl: 'MDL-12345',
        reason: 'To support some new technology',
        // emit: false,  // Silent: console only, no modal (non-final only).
        // final: true,  // Final deprecation: throws an Error instead of warning.
    });
    return myNewFunction(...args);
};
```

At least one of `reason`, `replacement` or `mdl` is required. A normal deprecation prints to the console and shows a modal (respecting `developerdebug`), and makes any Behat run that hits it fail. Suppress specific items with `$CFG->jsdeprecationignorelist = ['mod_example/mymodule::myFunction'];` (matches the first argument).

## Other core modules and changes

- `core/dropzone` (4.4+): accessible drop zones — `new Dropzone(container, 'image/*', files => {...})`, then `dropZone.setLabel('...')`, `dropZone.init()`.
- `core/page_title` (5.1): manipulate the current page title.
- `core/toast` (5.2): `visuallyHidden` config renders screen-reader-only toasts. `core/loadingicon` functions (5.2) accept an `options` parameter to display the icon as an `overlay`.
- `core/checkbox-toggleall` (5.1): `data-toggle=master`/`slave` → `toggler`/`target`; `updateSlavesFromMasterState()` → `updateTargetsFromTogglerState()` (old forms warn in the console until final deprecation).
- Removed in 5.2: `core/addblockmodal`; `request()` from `core/pending`; `M.util.set_user_preference()`, `M.util.init_toggle_class_on_click()`, `checkall()`, `checknone()`, `select_all_in()`, `confirm_if()`, `M.form.init_smartselect()` and other `javascript-static.js` globals; YUI `moodle-core-notification-confirm` (use `core/modal`).
- Reactive state (`core/reactive`, 4.0+) and combobox search: see `javascript-advanced.md`.
