# Advanced JavaScript in Moodle (5.3)

Covers the core reactive library (`core/reactive`), the combobox search component (`core/comboboxsearch`), and legacy YUI recognition/migration, per the Moodle 5.3 developer docs and 5.0–5.3 upgrade notes. For ESM modules, core/ajax, core/str, core/templates, core/modal, Bootstrap 5 and jQuery policy see `javascript.md`; for React/TypeScript components see `react.md`.

## Reactive library (`core/reactive`, since 4.0)

Moodle ships a framework-independent reactive library for small reactive apps on any page (the course editor and course index use it). Reference tutorial plugin: [moodle-mod_nosferatu](https://github.com/ferranrecio/moodle-mod_nosferatu/).

**Status in 5.3:** per the frontend development overview, with the adoption of React new reactive UI development should use React-based components (`react.md`). `core/reactive` remains supported for existing code (course editor, course index) but should not be used for new features.

Since 5.0 (MDL-83339, `core_courseformat`): passing a bare element ID to initialise course-format reactive components is deprecated in favour of a `querySelector` selector (`'#id'` instead of `'id'`).

Core pattern:

- UI is split into **components** (each extends `BaseComponent`, owns one DOM element and its children).
- UI always represents a data structure called the **state**; a state change is a **mutation**.
- User interaction never changes the UI directly — it dispatches mutations; only state changes (via **watchers**) update the DOM.
- Write web services are called inside mutations only; components may use read-only services (templates, strings, fragments).

Four pieces:

| Piece | Role |
|---|---|
| Component | Watches state, captures user events on its element, dispatches mutations |
| Reactive instance | Controller: registers components, triggers watchers, dispatches mutations, exposes read-only state |
| State manager | Stores/protects state, enables writes during mutations, emits state-change events |
| Mutations library | Object of methods that alter the state; the only place backend writes happen |

### Components

```js
import {BaseComponent} from 'core/reactive';
import {reactiveInstance} from 'YOUR_PLUGIN/reactive';

export default class extends BaseComponent {
    static init(target, selectors) {
        return new this({
            element: document.getElementById(target),
            reactive: reactiveInstance,
            selectors,
        });
    }
}
```

If no `reactive` is passed, the component inherits the instance from a parent DOM element (useful for subcomponents). Embed from mustache:

```handlebars
<div id="{{uniqid}}-your-plugin-{{id}}"> ... </div>
{{#js}}
require(['YOUR_PLUGIN/local/yourthing/childcomponent'], function(component) {
    component.init('{{uniqid}}-your-plugin-{{id}}');
});
{{/js}}
```

### Lifecycle hooks

- `create(descriptor)` — before registration. Define `this.name`, `this.selectors`, `this.classes`, read data from `this.element`. No `this.reactive` yet.
- `getWatchers()` — return `[{watch: 'event', handler: this._method}, ...]`. No manual `bind` needed; `this` is kept constant for watchers and listeners.
- `stateReady(state)` — initial state loaded (fires even if registered after load; the component "page ready"). Add event listeners, refresh from state, create subcomponents here.
- `destroy()` — before unregistering; clean up subcomponents (e.g. `this.dragdrop.unregister()`).

```js
create(descriptor) {
    this.name = 'courseindex_cm';
    this.selectors = {
        EXAMPLESELECTOR: `[data-for='example']`,
    };
    this.classes = {
        HIDDEN: 'dimmed',
    };
    this.id = this.element.dataset.id;
}

getWatchers() {
    const id = this.element.dataset.id;
    return [
        {watch: `cm[${id}]:created`, handler: this._createCm},
        {watch: `cm[${id}]:deleted`, handler: this._deleteCm},
    ];
}
```

### State events and watchers

Event name formats: `Element:Action` (`students:updated`), `Element.Attribute:Action` (`students.attempts:updated`), `SetName[ID]:Action` (`students[3]:created`), `SetName[ID].attribute:Action`. One change fires events from specific to generic; a mutation touching several values runs in one transaction ordered **creations, updates, deletes**, wrapped in `transaction:start` / `transaction:end`. Every change also fires `state:updated`.

Handlers receive `{action, state, element}`:

```js
getWatchers() {
    return [
        {watch: `students.attempts:updated`, handler: this._refreshStudent},
    ];
}

_refreshStudent({element}) {
    const domElement = this.getElement(this.selectors.ATTEMPTS, element.id);
    domElement.innerHTML = element.attempts;
}
```

Debugging: set Debug messages to DEVELOPER to get the reactive debug tools (state event log) in the page footer.

### Dispatching mutations from a component

Components cannot alter the state (exception if they try). They dispatch:

```js
stateReady() {
    this.addEventListener(
        this.getElement(this.selectors.SUBMITBTN),
        'click',
        this._submitAttempt
    );
}

_submitAttempt(event) {
    const userid = event.target.dataset.id;
    this.reactive.dispatch('submitAttempt', userid);
}
```

### BaseComponent helpers

- Listeners: `this.addEventListener(el, event, this.method)`, `removeEventListener`, `removeAllEventListeners()`, `this.dispatchEvent(name, detail)` (custom events; declare names in a static `getEvents()`, access via `this.events`).
- DOM (scoped to the component element): `this.element`, `this.getElement(QUERY[, DATAID])`, `this.getElements(QUERY[, DATAID])` — DATAID appends `[data-id='...']` to the query.
- Teardown: `this.unregister()` (keeps DOM), `this.remove()` (removes DOM); both run `destroy()`.
- Locking: `this.locked = true/false` on the whole component; `setElementLocked(el, bool)` / `getElementLocked(el)` per element. No styling is applied automatically.
- Subcomponents: `await this.renderComponent(target, 'your_plugin/your_template', data)` — renders a mustache template over `target` and returns a Promise of the new component. The template's component element must be at the root of the template.

### Setting up an application

1. `export const myReactiveInstance = new Reactive(description);` — description needs `eventName` and `eventDispatch` (mandatory, unique per app); optional `target` element, `mutations`, `state`.
2. Register mutations (`mutations` in the description, `setMutations()`, or `addMutations()`).
3. Set initial state: `state` in the description or `setInitialState(data)` (once only, e.g. after an ajax call). `stateReady` hooks then fire.

State root-level rules: only objects, or arrays of objects each having an `id` (arrays become maps indexed by id). Simple values, arrays of scalars, or arrays of id-less objects at root level raise exceptions. Nested attributes can be anything.

### Mutations

Every mutation method receives a `StateManager` as its first parameter. Manual state changes must toggle read-only mode; watchers are notified when read mode is restored:

```js
export default class {
    async submitAttempt(stateManager, userId) {
        const state = stateManager.state;
        stateManager.setReadOnly(false);
        const result = await ajax.call([{
            methodname: 'your_plugin_submit_attempt',
            args: {userid: userId},
        }])[0];
        if (result) {
            state.students.get(userId).attempts++;
        }
        stateManager.setReadOnly(true);
    }
}
```

Or let the backend drive: if the web service returns a state update message, just call `stateManager.processUpdates(updates)` (handles read-only mode itself). Update message format:

```js
[
    {"name": "students", "action": "put", "fields": {"id": 5, "name": "John Doe", "attempts": 4}},
    {"name": "students", "action": "delete", "fields": {"id": 3}},
]
```

Built-in actions: `put` (create-or-update, preserves extra fields), `override` (create-or-update, drops missing fields), `update` / `create` / `delete` (strict, exception if missing/existing), `remove` (delete, lenient). Add or override actions with `stateManager.addUpdateTypes({name: fn})` — each `fn(stateManager, name, fields)`. A reserved `prepareFields(stateManager, elementName, fields)` action normalizes incoming fields (frontend-only attributes, defaults) before storage.

Mutation names starting with `_` are private (cannot be dispatched). A mutation named `init(stateManager)` is reserved: auto-called when the mutations are registered (good place for `addUpdateTypes`).

### Drag & drop helper

`DragDrop` (exported from `core/reactive`) sets up HTML5 drag/drop; draggables only interact with drop-zones of the same reactive instance.

```js
import {BaseComponent, DragDrop} from 'core/reactive';

export default class extends BaseComponent {
    stateReady() {
        this.dragdrop = new DragDrop(this);
    }
    destroy() {
        if (this.dragdrop !== undefined) {
            this.dragdrop.unregister();
        }
    }
    // Draggable: mandatory. Return null/undefined to ignore the drag.
    getDraggableData() {
        return {id: 35, name: "Something"};
    }
    // Optional draggable hooks: dragStart(dragData, event), dragEnd(dragData, event).
}
```

Drop-zone methods on the parent component: `validateDropData(dropdata): boolean`, `drop(dropdata, event)` (only called after validation passes, so do not re-validate), optional `showDropZone(dragdata, event)` (valid data enters) / `hideDropZone(dragdata, event)` (valid data leaves). CSS classes toggled automatically: `draggable`, `dragging` (draggables); `dropready`, `dragover` (drop-zones).

## Combobox search (`core/comboboxsearch`, since 4.3)

Reusable searchable-dropdown component (used by the user, single view, and grader grade reports). Steps: add the dropdown HTML via the PHP output component, then init your JS module.

PHP side:

```php
use core\output\comboboxsearch;

$searchdropdown = new comboboxsearch(
    true,
    'Trigger button content',
    null,
    'parent-class',
    'trigger-button-class',
    'search-dropdown-class',
    null,
    false,
);
$data['templatevalue'] = $searchdropdown->export_for_template($OUTPUT);
$PAGE->requires->js_call_amd('yourplugin/yourjsfile', 'init');
```

JS side — extend the base class and implement the abstract parts:

```js
import search_combobox from 'core/comboboxsearch/search_combobox';

export default class extends search_combobox {
    // Required implementations:
    // fetchDataset()          - return the dataset (sync or Promise).
    // filterDataset(dataset)  - return results matching the search term.
    // filterMatchDataset()    - optionally mutate results (links, highlights).
    // renderDropdown()        - render the results however you like.
    // componentSelector()     - CSS selector for the component root.
    // dropdownSelector()      - CSS selector for the dropdown.
}

export const init = () => new YourComponent({});
```

`triggerSelector()` was removed from `core/comboboxsearch/search_combobox` in 5.2 (UPGRADING, MDL-87425) — do not implement or call it, even though the 5.3 comboboxsearch devdocs page still lists it.

Useful inherited helpers: `getDataset()` / `getDatasetSize()`, `getMatchedResults()` / `setMatchedResults()`, `getSearchTerm()` / `getPreppedSearchTerm()` (lowercased) / `setSearchTerms()`, `toggleDropdown()` / `closeSearch()` / `searchResultsVisible()`, `filterrenderpipe()` (filter + render in one call), `renderAndShow()`, keyboard navigation (`keyUpDown()`, `selectNode()`, `moveToFirstNode()` / `moveToLastNode()` / `moveToNode()`), and extensible `registerClickHandlers()` / `registerKeyHandlers()` / `registerInputHandlers()` (input is debounced).

## YUI (legacy — deprecated since 2.9)

Do NOT write new YUI code; core keeps converting YUI to native JS (e.g. 5.1 replaced `grade/report/singleview/js/singleview.js` with an AMD module). Moodle has been migrating from YUI to AMD/ESM modules since 2.9; new code should be vanilla ES2015+ modules (see `javascript.md`).

Recognizing YUI code:

- Wrappers: `YUI.add('moodle-block_fruit-fruitbowl', function(Y) {...}, '@VERSION@', {requires: ['panel']});` and `YUI().use('moodle-...', function(Y) {...});`
- Global `Y` (SimpleYUI) and `M.<frankenstyle>.<modname>` namespaces (JS objects like `M.block_fruit.fruitbowl`, with an `init` function); sandboxed modules under `Y.M.<component>.<modulename>` (e.g. `Y.M.mod_forum.inlinereply`).
- Module naming `moodle-FRANKENSTYLE-MODULENAME`; source lives in `plugin/yui/src/MODULENAME/js/*.js` with `build.json` and `meta/*.json`; built with `npx grunt yui`.
- PHP loading: `$PAGE->requires->yui_module('moodle-block_fruit-fruitbowl', 'M.block_fruit.fruitbowl.init', [$args]);`

Removed in 5.2: the `moodle-core-notification-confirm` YUI module (use AMD `core/modal`), the `keyDelegation` method and `lightbox` property of `moodle-core-notification` (no replacement), and `public/question/qengine.js`.

Migration direction: rewrite as an ESM module in `amd/src/`, replace `Y.delegate`/`Y.one` with native DOM APIs and event delegation, replace `yui_module()` with `$PAGE->requires->js_call_amd()`. Debug settings for legacy JS work: `$CFG->cachejs = false;`, `$CFG->yuicomboloading = false;`.
