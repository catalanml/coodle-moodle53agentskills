# React Components (Moodle 5.3)

Scope: writing React/TypeScript components in core or a plugin — file layout, build tooling (Grunt + esbuild), the browser import map, mounting from Mustache/PHP/JS, theme overrides and the swizzle CLI, the dev-mode profiler, and Jest unit tests. Per Moodle 5.3 developer docs (`guides/javascript/react/*`) plus the 5.3 UPGRADING notes.

Version history: React build tooling, import map, `{{#react}}` helper / `react_autoinit` (MDL-87765) and theme overrides (MDL-88507) arrived in **5.2**. **5.3** added the swizzle CLI (MDL-88509), Jest unit testing (MDL-88812 / MDL-87781), `html_writer::react_component()` and `react_component_renderable` (MDL-89296), and the `core/import` / `core/component` AMD bridges (MDL-88505). React is separate from the AMD pipeline (`amd/src` → `amd/build`, see `javascript.md`); both coexist.

## Where files live

Since 5.1 code lives under `public/` in a git checkout; paths below are shown as in the docs.

```text
<component>/
└── js/
    └── esm/
        ├── src/            # TypeScript/TSX source (.ts / .tsx)
        │   └── swizzle.json  # optional: override safety levels per component (5.3)
        ├── build/          # generated ES module output - do not edit
        └── tests/          # Jest tests: **/esm/tests/**/*.test.{ts,tsx} (5.3)
```

- Core: `public/lib/js/esm/src/…`; plugin: e.g. `public/mod/book/js/esm/src/viewer.tsx`.
- A module is addressed as `@moodle/lms/<component>/<module>` and resolves to `<componentdir>/js/esm/build/<module>.js`. Example: `@moodle/lms/mod_book/viewer` → `mod/book/js/esm/build/viewer.js`.
- Theme overrides: `public/theme/<themename>/js/esm/src/overrides/<component>/<path>` (see Theming).
- The build discovers every `js/esm/src/**/*.{ts,tsx}` across core and plugins automatically — no registration.

A component mounted automatically (template/PHP) must be a **default-exported React function component**:

```tsx
// mod/book/js/esm/src/viewer.tsx
type Props = {
    title?: string;
    chapter?: string;
};

export default function Viewer({title = 'Book', chapter = 'Chapter 1'}: Props) {
    return (
        <div>
            <h1>{title}</h1>
            <p>{chapter}</p>
        </div>
    );
}
```

## Build tooling

React is part of the Grunt JS pipeline alongside AMD and YUI (orchestration in `.grunt/tasks/javascript.js`; the esbuild driver is `.esbuild/build.mjs`, which generates TypeScript path aliases and then builds all core and plugin bundles).

| Command | Purpose |
| --- | --- |
| `grunt react` | Production build (minified, no sourcemaps) |
| `grunt react:dev` | Development build (inline sourcemaps, no minification) |
| `grunt react:watch` | esbuild native watch (`context.watch()`), incremental rebuilds; runs ESLint check-only on changed files after each rebuild |
| `grunt eslint:react` | ESLint on React sources with auto-fix |
| `grunt jsconfig` | Regenerate `tsconfig.aliases.json` (used by Jest and editors) |

- Running plain `grunt` inside a `js/esm/src` directory runs `grunt react` (e.g. `cd public/mod/book/js/esm/src && grunt`).
- **`grunt watch` does NOT monitor React files** — use `grunt react:watch`.
- `tsconfig.aliases.json` maps `@moodle/lms/<component>/*` aliases for TypeScript; it is generated (by `.esbuild/generate-aliases.mjs` from `.grunt/components.js`, only for components with real TS files under `js/esm/src`), gitignored, and must never be edited by hand. `tsconfig.json` reads it.
- These imports are marked external and never bundled into your output (they are shared at runtime via the import map): `react`, `react/*`, `react-dom`, `react-dom/*`, `@moodle/lms`, `@moodle/lms/*`, `@moodlehq/design-system`, `@moodlehq/design-system/*`.

## Import map

Bare specifiers are resolved by a native browser import map (`<script type="importmap">`) built by `page_requirements_manager::get_import_map()` during `<head>` rendering, so source can use standard imports:

```ts
import React from 'react';
import {jsx} from 'react/jsx-runtime';
import {BookViewer} from '@moodle/lms/mod_book/viewer';
```

`core\output\requirements\import_map` (a DI singleton, `JsonSerializable`) is the single source of truth. Built-in specifiers registered by `add_standard_imports()`:

| Specifier | Served from |
| --- | --- |
| `@moodle/lms/` | component's `js/esm/build/` (package scope, trailing slash) |
| `@moodlehq/design-system` | `lib/js/bundles/design-system.js` |
| `react`, `react/` | `lib/js/bundles/react/react.js`, `lib/js/bundles/react/` (covers `react/jsx-runtime`) |
| `react-dom`, `react-dom/` | `lib/js/bundles/react-dom/react-dom.js`, `lib/js/bundles/react-dom/` |

URLs are `{loaderBase}{specifier}`, where the loader is the ESM endpoint `/esm/{revision}/{scriptpath}` served by `core\route\controller\esm_controller::serve`. The revision is `page_requirements_manager::get_jsrev()`: a valid revision gets long-lived immutable cache headers + ETag (304s); `-1` (JS caching disabled / dev) gets short-lived headers so files are re-fetched each load. The controller calls `import_map::get_path_for_script()` (longest specifier prefix wins) and 404s if the file is missing.

Add or override specifiers from a `pre_render` hook (calling `add_import()` twice for the same specifier overwrites; hook order matters):

```php
use core\output\requirements\import_map;

$importmap = \core\di::get(import_map::class);

// Absolute URL (e.g. a CDN): written verbatim into the map; $path ignored.
$importmap->add_import('my-lib', loader: new \core\url('https://cdn.example.com/my-lib.js'));

// Filesystem path relative to $CFG->root; the map URL is '{loaderBase}@myplugin/'.
$importmap->add_import('@myplugin/', path: 'local_myplugin/js/esm/build');
```

Signature: `add_import(string $specifier, ?\core\url $loader = null, ?string $path = null, bool $loadfromcomponent = false): void` (`$loadfromcomponent` resolves `<component>/<module>` to the component's `js/esm/build/`; used internally for `@moodle/lms/`).

## Mounting components

### From Mustache: `{{#react}}` (5.2, MDL-87765)

`mustache_react_helper` (`public/lib/classes/output/mustache_react_helper.php`) turns a JSON block plus optional fallback HTML into a mount point; `react_autoinit` (`public/lib/js/esm/src/react_autoinit.ts`) mounts it.

```mustache
{{#react}}
{
    "component": "@moodle/lms/mod_book/viewer",
    "props": {
        "title": "{{title}}",
        "confirmLabel": "{{#str}}confirm, core{{/str}}"
    },
    "id": "book-viewer",
    "class": "book-viewer-wrapper"
}
<p>Loading…</p>
{{/react}}
```

- `component` (required) → `data-react-component`; `props` → `data-react-props` (JSON-encoded); any other key → a plain HTML attribute (`id`, `class`, `aria-*`; `true` emits a bare attribute, `false` omits it).
- The block is rendered by Mustache first, so `{{var}}`, `{{#str}}`, `{{#quote}}` work inside values. Trailing commas are stripped.
- Invalid JSON: with fallback content a plain `<div>` with the fallback is rendered, otherwise an empty string; reported via `debugging()` at `DEBUG_DEVELOPER`.

DOM contract (you may also emit this markup yourself):

```html
<div data-react-component="@moodle/lms/mod_book/viewer"
     data-react-props='{"title":"My Book","chapter":"Chapter 1"}'></div>
```

The specifier must be the full `@moodle/lms/<component>/<path>`; invalid `data-react-props` JSON logs an error and falls back to `{}`.

`react_autoinit` lifecycle: on load waits for `DOMContentLoaded`, scans `[data-react-component]`, dynamically `import()`s each specifier through the import map, mounts `module.default` with `react-dom/client` `createRoot` (warns and skips if there is no default export), marks the element `data-react-mounted="1"` (mount guard), and installs one global `MutationObserver` on `document.documentElement` (`childList`, `subtree`). Added nodes (and descendants) are mounted, removed ones unmounted (cleanup stored in a `WeakMap`), so AJAX/fragment content needs no extra init call. Never call `createRoot` yourself on an element managed by `react_autoinit`.

### From PHP (5.3, MDL-89296)

- `\core\output\html_writer::react_component()` renders the mount point — use it instead of hand-writing the `div`. Pass props as PHP data; do **not** JSON-serialise them first. Signature: `public static function react_component(string $modulename, array|string|\stdClass|\JsonSerializable $props): string` (verified in MOODLE_503_STABLE source); it outputs a `div` with `data-react-component` and JSON-encoded `data-react-props`.
- Implement `\core\output\react_component_renderable` to render a renderable with React:

```php
final class my_renderable implements \core\output\react_component_renderable {
    #[\Override]
    public function get_react_component_name(): string {
        return 'block_timeline/Timeline';
    }

    #[\Override]
    public function get_react_component_props(
        \core\output\renderer_base $renderer,
    ): \stdClass {
        return (object) [
            // ...
        ];
    }
}

echo $OUTPUT->render(new my_renderable(...));
```

### From JavaScript

- ESM/TS: `@moodle/lms/core/mount` — use when the mount point is created dynamically or you need the unmount handle:

```ts
import {mountReactApp, unmountReactApp} from '@moodle/lms/core/mount';

const unmount = mountReactApp(container, MyComponent, props, {id: 'my-app'});
unmount();                    // or:
unmountReactApp(container);
```

- AMD (5.3, MDL-88505): `core/component` provides `appendToDom` and `prependToDom` to insert React components, which `react_autoinit` then mounts; `core/import` lets AMD code do a native ESM dynamic import without Babel rewriting it. (Signatures not documented in devdocs.)
- ESM code that needs an AMD module uses `requireAsync()` / `requireManyAsync()` from `@moodle/lms/core/amd` (e.g. `requireAsync('core/ajax')`). Redirects must go through `redirect()` from `@moodle/lms/core/location` (mockable in tests).

Debugging checklist: specifier in `@moodle/lms/<component>/<path>` form; built file exists in `js/esm/build/`; module has a default-exported function component; check console messages prefixed `[react_autoinit]`. Multiple mounts usually mean the surrounding template recreates the container.

## Theming and overrides (5.2, MDL-88507)

A request for `@moodle/lms/<component>/<path>` first checks the **current** theme, then the original:

1. `public/theme/<theme>/js/esm/src/overrides/<component>/<path>.tsx`
2. the component itself, e.g. `public/lib/js/esm/src/ExampleComponent.tsx` (core) or `public/mod/assign/js/esm/src/local/grader/GradingPanel.tsx` (plugin).

**There is no parent-theme fallback** — a child theme does not automatically pick up its parent's overrides.

Scope modifiers to reach a specific version (needed to wrap without infinite resolution loops):

- `@moodle/lms/theme-original/<component>/<path>` — the unthemed original.
- `@moodle/lms/theme-<name>/<component>/<path>` — that theme's override (e.g. `theme-classic`, `theme-boost`).

**Inside an override, never import the component you are overriding via bare `@moodle/lms/…`** (circular); other components may use the normal specifier.

```tsx
// public/theme/retro/js/esm/src/overrides/mod_myfirstplugin/ExampleComponent.tsx
import {
    ExampleComponent as OriginalComponent,
} from '@moodle/lms/theme-original/mod_myfirstplugin/ExampleComponent';
import {UnrelatedComponent} from '@moodle/lms/mod_myfirstplugin/UnrelatedComponent';

export const ExampleComponent = (props) => (
    <UnrelatedComponent>
        <OriginalComponent boostEnhanced={true} {...props}/>
    </UnrelatedComponent>
);
```

**Note:** the theming examples use named exports; an override of a component mounted by `react_autoinit` must still default-export, since autoinit mounts `module.default`.

### Swizzle CLI (5.3, MDL-88509)

`npm run swizzle` (or `./scripts/swizzle.mjs`) from the project root — or inside `public/theme/<themename>` to infer the theme — starts a wizard that scaffolds a **wrap** (decorates the original) or **eject** (full copy) override into a theme. Then run `grunt react` / `grunt react:watch`.

Plugin authors declare per-component safety in `js/esm/src/swizzle.json`:

```json
{
    "ExampleComponent": {
        "eject": "risky",
        "wrap": "safe"
    }
}
```

Levels: `safe` (minimal upgrade risk), `risky` (internals may change; CLI asks for confirmation), `prohibited` (CLI refuses). Every `grunt react` / `grunt esm` build adds undeclared components as `risky`/`risky`. Set levels with the CLI, not by hand (a typo like `"riskyy"` silently means unrestricted):

```bash
./scripts/swizzle.mjs manifest generate                    # initial manifest for a plugin
./scripts/swizzle.mjs manifest set                         # interactive
./scripts/swizzle.mjs manifest set @moodle/lms/mod_assign/local/grader/GradingPanel safe        # both
./scripts/swizzle.mjs manifest set @moodle/lms/mod_assign/local/grader/GradingPanel risky safe  # eject, wrap
```

## Profiler / developer mode

Dev mode = `jsrev === -1`, i.e. **Site administration > Development > Debugging > Cache JavaScript** disabled. Then:

- PHP: `import_map::resolve_react_dev_path()` serves `client.development.js` (unminified React DOM profiling build) instead of `client.js` — full warnings, clean stack traces.
- Browser: `isProfilerEnabled()` is true and `mountReactApp()` wraps every tree in React `<Profiler>`, logging `[mount] MyComponent - 2.34ms`; `console.warn` for renders over 16 ms, `console.error` over 50 ms; plus `[react_autoinit] Initializing...`, `Found N component(s) to mount`, `Mounted via default: <specifier>`, `Unmounted: <specifier>`.

## Unit testing with Jest (5.3)

Jest runs against ESM TypeScript source only (`ts-jest`, `jsdom` environment); AMD modules cannot run in Jest. A `Jest` job runs in core GitHub Actions (`.github/workflows/push.yml`) alongside Grunt and PHPUnit.

```bash
npm test                                             # pretest runs `grunt jsconfig`
npm test -- --testPathPatterns=public/lib/js/esm/tests/String.test.ts
npm test -- --coverage
```

Tests match `**/esm/tests/**/*.test.{ts,tsx}`, mirroring `src/` (e.g. `public/mod/forum/js/esm/src/output/ExampleComponent.tsx` → `public/mod/forum/js/esm/tests/output/ExampleComponent.test.ts`). Theme override tests import via `@moodle/lms/theme-<name>/…` (e.g. `public/theme/boost/js/esm/tests/overrides/core/ExampleComponent.test.ts`). If imports fail to resolve, run `grunt jsconfig`.

Global helpers (reset between tests automatically; use `beforeEach`/`afterEach` for your own state):

- Strings: all strings resolve to `[identifier, component]` unless mocked with `mockString(identifier, component, value)`.
- AMD: `mockAmdModule(name, mockObject)` — the `core/amd` mock returns it from `requireAsync` / `requireManyAsync`. An unmocked module throws `Unexpected call to requireAsync with module name: …` (intentional hard failure).
- Redirects: `expectRedirect({url: …})` or `expectRedirect({urlContains: …})` before triggering `redirect()` from `@moodle/lms/core/location`; an unexpected redirect throws.

```ts
import {getString} from '@moodle/lms/core/String';
import {requireAsync} from '@moodle/lms/core/amd';

describe('getString', () => {
    it('returns the resolved string', async () => {
        mockString('pluginname', 'mod_forum', 'Forum');
        await expect(getString('pluginname', 'mod_forum')).resolves.toBe('Forum');
    });
});

describe('my component', () => {
    it('fetches data via core/ajax', async () => {
        const mockAjax = {call: jest.fn().mockResolvedValue([{data: 'ok'}])};
        mockAmdModule('core/ajax', mockAjax);

        // ...code under test that calls requireAsync('core/ajax')...

        expect(mockAjax.call).toHaveBeenCalledWith(
            expect.arrayContaining([expect.objectContaining({methodname: 'my_ws_method'})]),
        );
    });
});
```

Component tests in the docs use `render`, `act` and `screen` (with `toBeInTheDocument()`); the docs do not show where these are imported from.
