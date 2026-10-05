# Moodle CSS/SCSS Coding Style (Moodle 5.3)

Style rules for CSS in Moodle plugins and themes, per the Moodle CSS coding
style policy page, plus the colour-mode guidance added in Moodle 5.3.
Goals: simplicity, readability, tool friendliness.

**Note:** the upstream CSS policy page is old — it still describes LESS,
Bootstrap 2 variables, the Clean/More themes and IE hacks. Core themes today
compile SCSS (`grunt css` = `scss` + `rawcss` tasks) and Moodle 5.0 moved to
Bootstrap 5. The general rules below still apply; LESS-specific advice has been
carried over as SCSS principles only, and obsolete parts are dropped.

## Tooling

Conformance is checked with [stylelint](https://stylelint.io), run through
Grunt from the Moodle root (after `nvm install && nvm use && npm install`):

```bash
npx grunt stylelint     # lint CSS/SCSS
npx grunt css           # compile: alias for the "scss" and "rawcss" tasks
```

The plugin prechecks (Moodle Marketplace / Plugins directory) include a **css**
test against this coding style. Compiling a plugin's own SCSS needs a local
`package.json` + `Gruntfile.js` in the plugin root (see `nodejs.md` in the dev
docs; `grunt-contrib-sass` with `scss/styles.scss` -> `styles.css`).

## File naming

- Plugins: `styles.css` (recommended). `styles_<theme name>.css` exists but is
  not recommended (reserved for third-party plugins).
- Themes: names chosen by the designer, but lowercase with `-` between words,
  succinct and descriptive. The policy page says CSS goes in `style/`; SCSS
  sources conventionally live in `scss/` compiled to `style/*.css`.

## Blocks

- Each selector on its own line; after a comma in a selector list, break the
  line.
- Each property-value pair on its own line, indented **4 spaces**, ending with
  a semicolon.
- Closing brace at the same indentation as the opening selector.
- One blank line between blocks.

```css
@media only screen and (min-width: 768px) {
    .selector-one,
    .selector-two {
        color: #fff;
        background-color: #000;
    }
}
```

Invalid:

```css
.selector_one, .selector_two { color: #fff; background-color: #000; }
```

## Selectors

- Lowercase only; separate words with hyphens (preferred) or underscores.
- Simple English words; verbosity is encouraged when it makes names clearer.
- **Semantic names**: say what the thing *is*, not what it looks like
  (`.forum-post-header`, not `.blue-box`).
- Avoid IDs — harder to maintain and override.
- Do not over-qualify: no `div#selName`, `div.some-class`.

```css
/* Valid. */
.selector-name {
    color: #fff;
}

/* Invalid: ID, tag qualification, capitals. */
div#selName {
    color: #fff;
}

.Color-White {
    color: #fff;
}
```

### Namespace plugin selectors

All plugin `styles.css` files are concatenated, cached and served as one
resource on every page, so plugin selectors must be scoped to the plugin. The
`.path-*` classes are added to `<body>` by core renderers:

```css
/* Instead of .contentarea: */
.path-mod-mymodule .contentarea {
    margin-top: 0;
}
```

## Properties and values

- Colon followed by a single space; never minify source files.
- Lowercase, except font names and vendor-specific properties.
- Colour codes: lowercase preferred, shorthand where possible (`#fff`).
- If you use RGBA/HSLA, also provide a hex fallback.
- Use shorthand (except when overriding) for `background`, `border`, `font`,
  `list-style`, `margin`, `padding`.
- **Never use `!important`.** If nothing else works, the CSS you are
  overriding is wrong.
- Vendor-prefixed properties go directly before the standard property they
  refer to; use prefixes only when a supported browser lacks the unprefixed
  property.

## Units

- Allowed units: `px`, `rem`, `em`.
- No unit on zero values.

```css
/* Valid. */
.something {
    margin-top: 0;
    font-size: 1.25em;
}

/* Invalid. */
.something {
    margin-top: 0px;
    font-size: 1pt;
}
```

## Comments

Comments start with a capital letter and end with a period (as in the PHP
coding style). A block comment at the top of each file explains its purpose:

```css
/**
 * File base.css.
 * Contains base styles for theme basic.
 */
```

Block comments may also mark a section for one component or view:

```css
/**
 * SCORM Navigation Sidebar.
 */
```

Single-line comments explain one rule or a small group:

```css
/* Required because the table reset adds a border to all tables. */
```

## Progressive enhancement and browser hacks

- Always provide fallbacks (e.g. a `background-color` behind background images
  and gradients).
- The standard property comes after any vendor-prefixed variants.
- No browser-specific hacks. Moodle adds browser classes to `<body>` for
  browser-specific CSS when really needed.
- Do not add hacks for browsers Moodle core does not support.

## Plugins: keep it bare-bones

Plugins published to the Marketplace should define the strict minimum: no
text sizes, colours etc. that belong to the theme — reuse core/Bootstrap
classes instead, so themes can restyle the plugin. Plugins not intended for
publication may define their own styles. (The policy page still names
Bootstrap 2; Moodle 5.0+ ships Bootstrap 5 with a BC layer — use Bootstrap 5
classes.)

## Colour modes (since 5.3, MDL-68037)

Boost has a dark colour mode, enabled by the experimental
`theme_boost | enablecolourmodes` setting. A plugin's `styles.css` can no
longer assume the page is light.

- Take colours from the Bootstrap custom properties, which follow the mode:
  `--bs-body-bg`, `--bs-body-color`, `--bs-secondary-bg`, `--bs-tertiary-bg`,
  `--bs-border-color`, `--bs-emphasis-color`, `--bs-link-color`; or from the
  design-system tokens `--mds-bg-surface-*`, `--mds-text-*`, `--mds-border-*`.
- Prefer the matching utility classes (`bg-body`, `bg-body-secondary`,
  `bg-body-tertiary`, `text-body`, `text-body-secondary`, `text-bg-*`,
  `border`) over `bg-white` or `text-dark`.
- Where a colour genuinely must differ between modes, scope it with the
  standard Bootstrap attribute selector `[data-bs-theme="dark"]`.
- Watch half-pairs: a fixed light background behind text that follows the
  mode (or fixed dark text on a surface that follows the mode) loses contrast.
  If a colour carries meaning and must stay (e.g. a status tint), fix the
  other half of the pair too.
- An SVG shown through `<img>` paints with the fill baked into the file;
  render icons through the icon API so they inherit `currentColor`.

```css
.path-mod-mymodule .summary-card {
    background-color: var(--bs-tertiary-bg);
    color: var(--bs-body-color);
    border: 1px solid var(--bs-border-color);
}

[data-bs-theme="dark"] .path-mod-mymodule .summary-card .brand-mark {
    opacity: 0.8;
}
```

Behat runs can exercise a mode with `php public/admin/tool/behat/cli/init.php
--colourmode=dark` (see the moodle-testing skill).

## Right-to-left (RTL)

Since Moodle 3.2 styles are flipped automatically for RTL languages by
[RTLCSS-PHP](https://github.com/moodlehq/rtlcss-php). Special comments control
exceptions:

```css
.hello-world {
    margin-left: 10px;
    /*rtl:ignore*/
    padding-left: 10px;

    background: url('logo.png') 0 0 no-repeat;
    /*rtl:raw:
    background-image: url('logo-rtl.png');
    */

    color: blue;
}
```

becomes, in RTL:

```css
.hello-world {
    margin-right: 10px;
    padding-left: 10px;

    background: url('logo.png') 0 100% no-repeat;
    background-image: url('logo-rtl.png');

    color: blue;
}
```

Caveats:

- The special comments must **precede** a statement — they must not end a
  file or a rule block; comments not followed by a rule/declaration are
  ignored.
- In SCSS (or LESS) they often do not survive compilation as intended: attach
  RTL comments to plain CSS declarations — no `@extend`, nested rules, etc.

Do not use `.dir-rtl` any more. To force left-to-right text (URLs, file
paths, hex colours, English DB field names, emails, numbers, regexes, code or
config samples), use the `.text-ltr` class.

## SCSS (carried over from the LESS guidance)

SCSS follows all the CSS rules above, plus:

- Encapsulate (nest) selectors rather than repeating the same prefix:

  ```scss
  .path-mod-mymodule {
      .something {
          color: var(--bs-emphasis-color);

          a {
              color: var(--bs-link-color);
          }
      }

      .something-else a {
          color: var(--bs-body-color);
      }
  }
  ```

- Never hard-code colours, font sizes etc. in themes — use a variable (or, for
  plugin CSS, the CSS custom properties above).
- Declare new variables sparingly and with semantic names; default a new
  variable from an existing one; don't create two variables that will always
  share a value.
- Prefer mixins/placeholders to duplicating the same set of declarations.

The page's LESS variable naming (camelCase, following Bootstrap 2) is
obsolete; the docs give no SCSS-specific variable naming rule.
