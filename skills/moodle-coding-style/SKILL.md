---
name: moodle-coding-style
description: Use when writing or editing any Moodle 5.3 PHP, JavaScript, CSS/SCSS, or SQL code - enforces the official Moodle coding style covering naming, whitespace, PHPDoc, file boilerplate, CSS, and SQL conventions
---

# Moodle 5.3 Coding Style

Moodle style guidelines for PHP and SQL, per the Moodle coding style policy
pages used for Moodle 5.3. Unless otherwise specified here, Moodle defers to
**PSR-12**, then **PSR-1**. New code must conform; old non-conforming code is
upgraded progressively, not paranoidly.

Check with the `moodle`/`moodle-extra` phpcs rulesets (moodle-cs) and the
PHPdoc checker (`local_moodlecheck`). For CSS/SCSS see `references/css.md`.

## File boilerplate

Every PHP file starts with the GPL header, immediately after the open tag
(no blank line between `<?php` and the header):

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
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
// GNU General Public License for more details.
//
// You should have received a copy of the GNU General Public License
// along with Moodle. If not, see <https://www.gnu.org/licenses/>.

/**
 * This is a one-line short description of the file.
 *
 * Longer description of the file (optional, may span multiple lines).
 *
 * @package    mod_mymodule
 * @category   backup
 * @copyright  2008 Kim Bloggs
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */
```

- Always use "long" PHP tags (`<?php`). **Never** include the closing `?>` tag
  at the end of the file.
- Files accessed via browser start with `require(__DIR__ . '/../../config.php');`
  (always `require`, not `require_once`, for config.php).
- All other scripts — except imported 3rd-party libraries and files **without
  side-effects** (single class/interface/trait definitions) — must start with:

  ```php
  defined('MOODLE_INTERNAL') || die();
  ```

  "Side-effects" means any global-scope code other than: the `namespace`
  declaration, `use` statements, namespace constants, or declarations such as
  `strict_types`. A file containing only one class definition needs **no**
  `MOODLE_INTERNAL` check. Class files may only contain extra code
  (side-effects) to require artifacts not available via autoloading — then the
  `MOODLE_INTERNAL` check is required.
- Other includes use `__DIR__` or absolute paths (`$CFG->dirroot`,
  `$CFG->libdir`); never relative `../` paths. Library files use
  `require_once`. CLI scripts must not use relative config.php paths.
- Includes go at the top of files or inside the functions that need them —
  never mid-file in global scope.
- Since 5.1 (MDL-83424) code lives under `public/` in a git checkout, but
  `$CFG->dirroot` still points at it, so includes are unchanged; new
  read-only `$CFG->root` is the install root.

## File formatting

- Unix LF line endings only; no CR. No trailing whitespace on any line
  (including inside multiline strings such as SQL). File ends with a single LF,
  no extra blank lines.
- Indent with **4 spaces**. No tabs.
- Line length: aim for **132** characters; never more than **180**.
  Exception: `/lang` string files — each `$string['id'] = 'value';` value is a
  single quoted string of any length (no concatenation, heredoc, or nowdoc).
- Wrapped lines of control-statement conditions or function/class declarations
  get **4 additional** spaces (8 total) to distinguish them from the body:

  ```php
  if (($userenrol->timestart && $userenrol->timestart < $limit) ||
          (!$userenrol->timestart && $userenrol->timecreated < $limit)) {
      return false;
  }
  ```

  Prefer extracting complex conditions into well-named helper variables over
  long wrapped conditions.
- Assignment: exactly one space after `=`; one or more spaces before it is
  allowed (aligned assignments are fine — match surrounding code).

## Naming

| Thing | Convention | Example |
|---|---|---|
| Files | short, whole English words, lowercase only; `.php`, `.html`, `.js`, `.css`, `.xml` | `locallib.php` |
| Classes | lowercase words separated by underscores | `some_custom_class` |
| Functions/methods | lowercase words separated by underscores; legacy (non-class) functions prefixed with frankenstyle component | `report_participation_get_overviews()` |
| Variables | lowercase words run together, **no underscores**, no camelCase; plural for arrays of objects; positive names (`$allowfilelocking`, not `$preventfilelocking`) | `$errorstring`, `$assignments` |
| Constants | UPPER_CASE with underscores, frankenstyle prefix (activities: module name only) | `FORUM_MODE_FLATOLDEST`, `BLOCK_COURSE_OVERVIEW_SHOWCATEGORIES_NONE` |
| `true`/`false`/`null` | always lowercase | |

- Activity modules use only the module name as prefix (`forum_...`) for
  legacy reasons; all other plugins use the full frankenstyle prefix.
- Global (non-class) functions and `define()` constants are not recommended:
  prefer methods and class constants on an autoloaded class, which autoload
  without any manual `require`.
- Uppercase globals (`$CFG`, `$USER`, `$DB`, `$SESSION`, `$COURSE`, `$SITE`,
  `$PAGE`, `$PERF`, `$THEME`) are reserved for core — never create more.
- For plain data objects use `stdClass` (`new stdClass()` or
  `(object) [...]`); never the deprecated `object` class.
- Web service functions: `{fullcomponent}_{verb}_{noun}`, verb preferably one
  of `get`, `create`, `delete`, `update` — e.g. `core_cohort_create_cohorts`.

## Frankenstyle (component names)

`{plugintype}_{pluginname}`, all lowercase, e.g. `mod_quiz`,
`gradereport_xls`. Core subsystems are `core_xxxx` (e.g. `core_enrol`,
`core_group`); plain `core` for the few core files outside any subsystem.
Used in: function name prefixes, class namespaces, constants prefix
(`MOD_FORUM_XXXX`), DB table names (`local_coolreport_users`; activities drop
the `mod_` prefix: table `forum`), capabilities (`mod/quiz:viewattempt` — `/`
instead of `_`), lang file names (`block_participants.php`; activities use
just `quiz.php`), `@package` tags, JS module names, template names, web
service function names.

## Namespaces & autoloading

Formal namespaces are **required** for all new classes. Frankenstyle-prefixed
class names (`mod_forum_example`) are deprecated — write `mod_forum\example`.

- Namespaced classes live under the component's `classes/` directory
  (`mod/forum/classes/`, `lib/classes/` for core, `subsystemdir/classes/`),
  with filename/path following the automatic class loading rules.
- Level 1 of the namespace **must** be the full component name
  (`mod_forum`) or `core`. Level 2, when used, must be a core API short name
  or `local` (for the plugin's own organisation). Level 3+ is free.
- One namespace declaration per file, first non-comment line (a docblock may
  precede it), followed by one blank line, then `use` statements — one class
  per line, single blank line after the last one.
- Never: import whole namespaces (`use mod_porridge\local;`), alias without
  need (`use ... as ...`), use `__NAMESPACE__`, use the `namespace` keyword
  outside the declaration, use bracketed namespace blocks, or use a leading
  backslash in `namespace`/`use` statements.
- Global functions called from namespaced code take **no** leading backslash;
  classes outside the current scope are either imported with `use` or written
  fully qualified with a leading backslash:

  ```php
  namespace mod_breakfast\local;

  use moodle_url;

  echo get_string('goodmorning', 'mod_breakfast'); // No backslash on global functions.
  $url = new moodle_url(...);                      // Imported via use.
  $a = new \stdClass();                            // Leading slash when not imported.
  ```

- Namespaces only for classes in `classes/` (and `tests/`); use the maximum
  level of detail. `use stdClass;` and `\stdClass` are both acceptable.
- Unit tests (MDLSITE-4800): namespace strongly recommended and matching the
  code under test; level 1 **must** be the component; class named after the
  class under test, file suffixed `_test.php`; directory structure mirrors the
  namespace under `tests/` instead of `classes/`. No `\test` level 2 unless
  the covered code has one. Tests themselves are not autoloaded.

## Classes

- One class (or interface/trait) per file, in `classes/` for autoloading.
- Member variables at the top of the class, above methods. `var` is
  forbidden — always declare visibility (`private`/`protected`/`public`).
  Public properties are permitted but discouraged in favour of get/set methods.
- Always use `()` when instantiating, even with no arguments:
  `new some_custom_class();`.
- Opening brace on the same line as the class/function declaration; body
  indented 4 spaces.

## Functions and methods

- Methods always declare visibility. No space between function name and `(`.
- Type hints and return types are **required** in all possible locations for
  new code. No space around the nullable `?` or before the `:` return type:

  ```php
  function report_participation_get_overviews(string $action, ?int $userid): ?array {
  ```

- Parameters: lowercase words, sensible defaults where possible; prefer
  `null` over `false` as a "not provided" default (`bool` defaults of
  true/false are fine when logical). Nullable hint on optional params
  (`?string $default = null`) is highly recommended.
- Do not enclose the return value in parentheses. Prefer a single return type.
- Arguments separated by a single space after each comma:
  `three_arguments(1, 2, 3);`.
- Magic methods are heavily discouraged — justification required.
- Avoid generic `array $options = []` parameters: list every option as a typed
  parameter with a default, or use a dedicated options class.
- When overriding a parent/interface method with no change to its meaning,
  apply `#[\Override]` and omit the docblock entirely. `@inheritdoc` as a
  docblock replacement is explicitly forbidden.

## Strings

- **Single quotes** for literal strings and strings full of double quotes
  (HTML). **Double quotes** when interpolating variables, using many single
  quotes, and always for complex SQL.
- Interpolation: `"Hello $name"` or `"Hello {$name}"` — both fine.
- Concatenate with `.`; when wrapping, put the dot at the **end** of each
  line. Spaces around the dot are optional (be consistent).
- Language strings: sentence case only (capital at sentence start and proper
  names). Each string must stand alone — never design strings to be
  concatenated into sentences. No leading/trailing whitespace.

## Arrays

- **Short array syntax** `[]` for all new code (long `array()` only to match
  surrounding old code).
- Space after each comma: `$myarray = [1, 2, 3, 'Stuff', 'Here'];`
- Multi-line arrays: 4-space indent per line, balanced brackets/newlines, and
  a trailing comma on the last element for cleaner diffs:

  ```php
  $myarray = [
      'firstkey' => 'firstvalue',
      'secondkey' => 'secondvalue',
  ];
  ```

## Control structures

- Space before and after the condition parentheses; spaces around operators
  inside. Always use braces, even for one-line blocks; opening brace on the
  same line. Indent 4 spaces.
- Use `else if`, **never** `elseif`:

  ```php
  if ($x == $y) {
      $a = $b;
  } else if ($x == $z) {
      $a = $c;
  } else {
      $a = $d;
  }
  ```

- `switch`: cases indented 4, case bodies a further 4.
- Ternary operator only for short, one-sentence-understandable statements,
  with whitespace around `?` and `:`. Prefer `??` over `isset()` ternaries.

## Documentation (PHPDoc)

- Every class and every function/method needs a docblock with a description —
  even when the tags seem self-explanatory. Exception: pure overrides with
  `#[\Override]` (see above).
- The `@package`, `@copyright` and `@license` tags (plus optional
  `@category`) must always be present in the file — in the file docblock, or
  the class docblock for single-artifact files (where the file docblock is
  optional).

Types in `@param`/`@return`/`@var`: short lowercase primitives (`bool`,
`int`, not `boolean`/`integer`), `stdClass` not `object`, `array` not `Array`;
prefer `int[]`/`stdClass[]` over bare `array`; alternatives separated by `|`
(`int|false`); `static`/`self`/`$this`/`void` where applicable.

Key tags:

- `@package` — frankenstyle component (`mod_quiz`, `core_enrol`, or `core`);
  exactly one package per file. No standard for `@subpackage`.
- `@copyright 2008 Kim Bloggs` — year + creator; never change in existing files.
- `@license https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later` —
  exactly this format.
- `@param type $name Description.` — no hyphens after the variable name.
- `@return type Description.` — mandatory if the function returns something;
  omit if not. Description optional for simple functions.
- `@var type Description.` — for properties **and** class constants (never
  `@const`). Inline `@var` in method bodies only exceptionally for type hints.
- `@throws` — optional, to flag thrown exceptions.
- `@uses exit` — if the function may call die/exit.
- `@since Moodle 5.3` — Moodle version that added a core class/function/method.
- `@deprecated since Moodle 2.0 MDL-12345 - ...` with `@todo`/`@see` where
  possible.
- `@see some_other_function()` for related elements; `@link` for URLs; both
  usable inline as `{@see ...}` / `{@link ...}`.
- `@category` — only for public artifacts of a Core API.
- Test-only tags: `@Given`/`@When`/`@Then` (Behat steps); `@covers`,
  `@coversDefaultClass`, `@coversNothing`, `@uses`, `@dataProvider`,
  `@testWith`, `@depends`, `@group`, `@requires`,
  `@runTestsInSeparateProcesses`/`@runInSeparateProcess` (unit tests). Since
  5.0 (PHPUnit 11) coverage may use
  `#[\PHPUnit\Framework\Attributes\CoversClass(example::class)]` (the only
  attribute the docs show); data providers must be `public static`.
- Document each property explicitly (`/** @var int The number of white
  stripes */`); no DocBlock templates.

### Inline comments

`// ` style (2 slashes + space) only — never `/* */` or `#` for inline
comments. Start with a capital letter (or digit or `...`), end with `.`, `?`
or `!`. No MDL tracker references in ordinary comments; TODOs **must** include
one: `// TODO MDL-12345 This works but is a hack.`

## Exceptions

- Use exceptions to report errors, especially in library code; never abuse
  them for normal flow control. `print_error()` is deprecated — throw
  `moodle_exception` instead.
- Base class `moodle_exception`; subclass it where appropriate. Since 4.5
  (MDL-81903) custom exception classes may live in `classes/exception/`
  (namespace `<plugin>\exception\`).
- Notable types: `moodle_exception` (generic), `coding_exception` (developer
  mistake — make the message helpful), `dml_exception` (DB failure),
  `file_exception` (File API).

## Dangerous constructs — never use

`eval()` (lang packs excepted), `preg_replace()` with `/e`, backticks for
shell execution, `goto` (operator or labels), `unserialize()` on user data.

## SQL coding style

- All variable parameters via **placeholders**: `:named`, `?` or `$1` —
  prefer named when there is more than one parameter.
- Table names in `{tablename}` syntax (the prefix is added automatically).
- SQL keywords in **UPPER CASE**; queries and fragments enclosed in **double
  quotes** (single quotes are for SQL string literals); do not concatenate SQL
  from parts if avoidable.
- Complex queries on multiple lines, **right-aligned** on SELECT, FROM, JOIN,
  WHERE, GROUP BY, HAVING:

  ```php
  $sql = "SELECT e.*, ue.userid
            FROM {user_enrolments} ue
            JOIN {enrol} e ON (e.id = ue.enrolid AND e.enrol = 'self')
           WHERE :now - u.lastaccess > e.customint2";
  $records = $DB->get_records_sql($sql, ['now' => $now]);
  ```

- `JOIN`, not `INNER JOIN`; no right joins. `AS` **always** for column
  aliases, **never** for table aliases. `<>` for inequality, never `!=`.
- Subquery indentation: no strict rule — optimise readability.

## PHP versions (policy summary)

Policy (since 3.5, MDL-59159): an LTS always requires the previous LTS (or
later) to upgrade; a branch's maximum PHP is the highest reached during its
life; its minimum PHP is the **lower of** the oldest PHP version still
supported upstream for 12+ more months at release, or the previous LTS's
maximum PHP (so LTS-to-LTS upgrades never force a PHP upgrade).

| PHP | Usable from | Minimum for |
|---|---|---|
| 8.1 | 4.1.2, 4.2 | 4.4, 4.5 |
| 8.2 | 4.2.3, 4.3 | 5.0, 5.1 |
| 8.3 | 4.4 | **5.2, 5.3** |
| 8.4 | 5.0 (MDL-80117) | — |
| 8.5 | in development for 5.3+ (MDL-84684): incomplete, development only | — |

Moodle 5.3 runs on PHP 8.3 (minimum) and 8.4. Write code for the target
branch's minimum PHP (a plugin also supporting 4.5 must stay PHP 8.1-safe).
The policy pages say nothing else about allowed language features.

## Git commits

Format: `MDL-xxxx CODE AREA: short summary` (72-char soft limit), blank
second line, then an optional detailed explanation. Present a cleaned-up
history — no "fix review comments" commits, no arbitrary splitting of one
logical change.

## Coding-style-only fixes policy

Related style fixes (same lines/method) may ride along with a real issue.
Unrelated style fixes go in a separate commit, main branch only. Style-only
issues are accepted only in the first 2 months of each release cycle.
