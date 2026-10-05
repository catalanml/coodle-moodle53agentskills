# Moodle Behat Reference (Moodle 5.3)

Setup, running, and writing Behat acceptance tests for Moodle 5.3 plugins and
core, per the Moodle 5.3 developer docs and UPGRADING notes.

**Paths:** since 5.1 (MDL-83424) the Behat CLI is
`public/admin/tool/behat/cli/` (verified in MOODLE_503_STABLE source), so
commands below are run from the install root as `php public/admin/tool/behat/cli/...`,
matching the PHPUnit commands. The Behat docs and UPGRADING notes still write
`php admin/tool/behat/cli/init.php` and `lib/tests/fixtures/...` (those paths
are relative to `public/`).

## Setup

Behat runs against a parallel test site (own wwwroot/dataroot/prefix), never
production data. Needs a recent browser (Firefox/Chrome as standard) plus its
WebDriver (chromedriver/geckodriver) in `$PATH`; Selenium is optional but
recommended (needs a Java runtime). Safari/Safaridriver is not supported.

In `config.php`:

```php
$CFG->behat_dataroot = '/path/to/the/dataroot/you/created';  // new, dedicated dir
$CFG->behat_wwwroot = 'http://127.0.0.1/path/to/your/site';  // MUST differ from $CFG->wwwroot
$CFG->behat_prefix = 'beh_';
require_once('/path/to/moodle-browser-config/init.php');     // recommended profiles
```

`localhost` vs `127.0.0.1`, an extra `/etc/hosts` name, or
`host.docker.internal` are common ways to get a distinct `behat_wwwroot`.

Initialise (installs Composer deps, installs the test site, writes
`behat.yml`, prints the run command):

```bash
php public/admin/tool/behat/cli/init.php
```

Re-run init (or `php public/admin/tool/behat/cli/util.php --enable`) after adding
step definitions or features. Use the
[chromedriver-wrapper](https://github.com/andrewnicols/chromedriver-wrapper)
to keep chromedriver matched to Chrome.

## Running tests

Start Selenium (`java -jar selenium-server-standalone-3.141.59.jar`, or
`java -jar selenium-server-<4.x>.jar standalone`) or a driver directly, then:

```bash
vendor/bin/behat --config /path/to/behat_dataroot/behatrun/behat/behat.yml

vendor/bin/behat --config .../behat.yml --tags=@mod_forum
vendor/bin/behat --config .../behat.yml `pwd`/mod/forum/tests/behat/private_replies.feature
vendor/bin/behat --config .../behat.yml `pwd`/mod/forum/tests/behat/private_replies.feature:38
vendor/bin/behat --config .../behat.yml --name="As a teacher I can see my own response"
vendor/bin/behat --config .../behat.yml --profile=headlesschrome   # default: firefox
```

moodle-browser-config profiles: `firefox`, `headlessfirefox`, `chrome`,
`headlesschrome`, `edge`, `headlessedge` (via Selenium); `geckodriver`,
`headlessgeckodriver`, `chromedriver`, `headlesschromedriver`, `edgedriver`,
`headlessedgedriver` (no Selenium; geckodriver uses port 4444, so not
alongside Selenium).

Other options:

- `--rerun` reruns previously failed scenarios; the Moodle runner also has
  `--auto-rerun` (reruns failures exactly once).
- `--format=pretty --out=/tmp/pretty.txt --format=moodle_progress --out=std`;
  formats include `progress`, `pretty`, `junit`, `moodle_progress`,
  `moodle_list`, `moodle_stepcount`, `moodle_screenshot`.
- `-vv` for verbose errors.
- Themes: `init.php --add-core-features-to-theme`, then `--suite=classic`
  (Boost is suite `default`).
- Parallel: `php public/admin/tool/behat/cli/init.php --parallel=3` (`-j`;
  also `--maxruns`, `--fromrun`, `--torun`, `-o/--optimize-runs`), then
  `php public/admin/tool/behat/cli/run.php --tags="@tool_myplugin" --suite="classic"`
  (runner accepts `--tags`, `--name`, `--feature`, `--suite`, `--replace`,
  `--fromrun`, `--torun`). Per-run DB/wwwroot/Selenium overrides via
  `$CFG->behat_parallel_run`.
- Slow machines: `$CFG->behat_increasetimeout = 2;`
- **Colour modes (since 5.3, MDL-68037):** `php public/admin/tool/behat/cli/init.php
  --colourmode=dark` runs the whole suite in that mode (reported in the run
  header; themes read it via `behat_get_colour_mode()`). The option also turns
  colour modes on. Precedence: user preference > run option > site settings,
  so a scenario asserting a mode must pin it with a user preference; a
  scenario about modes being off must skip itself with
  `Given the run is not using a colour mode`. Example:
  `theme/boost/tests/behat/colour_mode_accessibility.feature`.

## Feature files

Features live in the plugin's `tests/behat/*.feature`, in Gherkin:

```gherkin
@mod @mod_yourplugin @javascript
Feature: Here comes a description of your user story.

  Background:
    Given the following "courses" exist:
      | fullname | shortname | category | groupmode |
      | Course 1 | C1        | 0        | 1         |
    And the following "users" exist:
      | username | firstname | lastname | email                |
      | teacher1 | Theo      | Teacher  | teacher1@example.com |

  Scenario: Description of your scenario, which you want to test.
    When I log in as "teacher1"
    And I am on "Course 1" course homepage
    Then I should see "Course 1"
```

Rules and conventions:

- **Given** = set-up, **When** = actions, **Then** = assertions. Use each
  keyword only ONCE per scenario and chain with **And** / **But**
  (MDLSITE-3778). Tear-down is automatic.
- Test one thing per scenario — the failing scenario name should name the bug.
- Set-up must not use the UI: use generator steps
  (`the following "courses" exist:`).
- Multiple scenarios per file only for the same feature; shared Given steps go
  in `Background:`. Feature-level tags apply to all scenarios.
- Avoid XPath/CSS selectors (`xpath_element`, `css_element`) — needing them
  usually signals an accessibility bug. Prefer `I set the field "Answer" to
  "frog"` and `I click on "True" "radio" in the "First question" "question"`.
- Since 5.0 (MDL-83848, behat/gherkin 4.11.0) **backslashes in feature files
  must be escaped**.

### Tags

- `@javascript` — needs a real browser; much slower, use only when needed.
- `@_file_upload` — file uploads; run only with Selenium on the same machine.
- `@_alert`, `@_switch_window`, `@_switch_iframe`, `@_cross_browser` — mark
  OS/browser-sensitive steps.
- `@componentname` — the Frankenstyle component (e.g. `@mod_forum`).
- TinyMCE tests **must** carry `@editor_tiny` and/or `@tiny_[pluginname]`
  (forces TinyMCE as default editor for the test). Tests that merely *use* an
  editor should use generic steps, e.g.
  `I set the field "Description" to "<p>Some value</p>"` or
  `I set the field "Description" to multiline:` with a PyString.

### Providing values to steps

- Strings in double quotes (`I press "Save and return to course"`); escape
  quotes as `\"`. `_NUMBER` placeholders take digits.
- Tables for forms/data (steps ending in `:`); PyStrings for multiline values.
- Field values: text for text fields; `1`/`""` for checkboxes; option text or
  value for selects (comma-separated for multi-select); option text for radios.
- Selectors pair a locator with a type: `I click on "User Name" "link"`.
  Types: field, link, button, link_or_button, select, checkbox, radio, file,
  optgroup, option, dialogue, filemanager, block, section, activity, region,
  table_row, table, icon, fieldset, css_element, xpath_element. Container
  types for `... in the "Community finder" "block"`: dialogue, block, section,
  activity, region, table_row, table, fieldset, list_item, css_element,
  xpath_element. Since 5.2 there is a `toast_message` named selector for
  Toast messages (MDL-87443).
- Table checks:

  ```gherkin
  Then "STRING_IN_ROW" row "COLUMN_HEADER" column of "TABLE_ID" table should contain "VALUE_TO_CHECK"
  Then the following should exist in the "TABLE_ID" table:
    | COLUMN_HEADER1 | COLUMN_HEADER2 |
    | VALUE_IN_ROW_1 | VALUE_IN_ROW_1 |
  ```

- Field groups: `myDate[day]`, `myDuration[unit]` in
  `I set the following fields to these values:` tables.
- Relative dates: `##yesterday##`, `##2 days ago##`, with a format:
  `##yesterday##%d##`.
- File uploads: `I upload "lib/tests/fixtures/users.csv" file to "Files"
  filemanager`; fixtures in `COMPONENT/tests/fixtures/`, path relative to the
  codebase root with `/` separators; tag `@_file_upload`.
- Navigation (faster, immune to navigation changes):

  ```gherkin
  Given I am on the "Quiz 1" "mod_quiz > View" page logged in as "manager"
  Given I am on the "C1" "Course" page
  ```

Browse all steps at Site administration > Development > Acceptance testing
(behat site must be installed); PhpStorm's Behat extension autocompletes them.

### Steps added in 5.0–5.3

| Since | Step / feature |
|---|---|
| 5.0 (MDL-75971) | Mailpit email checks: `Given an email catcher server is configured`, then `Then the email to "user@example.com" with subject containing "subject" should contain "content"`. Mailpit must run; define `TEST_EMAILCATCHER_MAIL_SERVER` (e.g. `0.0.0.0:1025`) and `TEST_EMAILCATCHER_API_SERVER` (e.g. `http://localhost:8025`) in `config.php`. |
| 5.0 | `behat_general::the_url_should_match()` — assert the current URL (e.g. after a redirect). |
| 5.0 | `the following element should download a file that:` replaces `behat_admin_presets::following_in_the_should_download_between_and_bytes`. |
| 5.0 | `the following "core_competency > [competency\|framework\|plan...]" exist:` replaces `the_following_lp_exist`. |
| 5.0 | Named selectors `qbank_comment > Comment count link` / `... text` replace the qbank_comment custom steps. |
| 5.3 (MDL-84065) | `I set the focus on the "<element>" "<selector>"` — focus without activating. |
| 5.3 (MDL-87575) | `the course linear navigation should be visible` / `should not be visible`. |

Removed steps: `behat_gradereport_grader::get_grade_item_id` (use the
`behat_grades` equivalent); `behat_grade::select_in_gradebook_tabs()` and
`select_in_gradebook_navigation_selector()` (5.2); qbank_columnsortorder
`column move handle` selector (5.2). Deprecated in 5.3:
`behat_mod_workshop::i_set_portfolio_instance_to` -> use
`behat_portfolio::i_set_the_portfolio_instance_to`.

## Custom step definitions

Steps are PHP methods in class `behat_plugintype_pluginname` in
`tests/behat/behat_plugintype_pluginname.php`, with a `@Given`/`@When`/`@Then`
regex in the PHPDoc. Choose the keyword by meaning (matching ignores it), and
name steps so they are clearly plugin-specific ("... in the CodeRunner question
type", not "I disable UI plugins").

```php
/**
 * Toggles the specified admin switch.
 *
 * @When /^I toggle the "(?P<element_string>(?:[^"]|\\")*)" admin switch "(?P<state_string>on|off)"$/
 * @param string $element Element we look for
 * @param string $state The state of the switch
 */
public function i_toggle_admin_switch($element, $state) {
    if (!$this->running_javascript()) {
        throw new \Behat\Mink\Exception\DriverException('Switches are only available with JavaScript enabled');
    }

    $node = $this->get_selected_node('checkbox', $element);
    $this->ensure_node_is_visible($node);

    $field = $node->getAttribute('id');
    if ($state == "on") {
        $this->execute('behat_forms::i_set_the_field_to', [$field, 1]);
    } else if ($state == "off") {
        $this->execute([\behat_forms::class, 'i_set_the_field_to'], [$field, 0]); // Array callable, MDL-86231.
    } else {
        throw new \Behat\Mink\Exception\ExpectationException('Invalid state for switch: ' . $state, $this->getSession());
    }
}
```

- To click, press, fill or reuse steps, call `\behat_session_trait::execute()`
  — never `->click()` on a `NodeElement` (bypasses error detection and JS
  waits). It takes the string `'class::method'` or, since MDL-86231 (5.1;
  backported to 4.1.21, 4.4.11, 4.5.7, 5.0.3), an array callable
  `[\behat_forms::class, 'i_set_the_field_to']`. Closures are not supported.
- Since 5.2 (MDL-75067) the `behat_session_trait::ensure_element_[does_not_]exists(...)`
  methods accept an optional `$container` parameter (parent node to search in).
- Custom selectors (`core_message > Message`): implement
  `behat_component_named_selector`-style methods (PHPDoc in
  `lib/behat/behat_base.php`).
- Custom navigation targets (`I am on the "..." "..." page`): implement
  `resolve_page_url` and `resolve_page_instance_url`.

### Deprecating a step

Create `tests/behat/behat_<component>_deprecated.php` with a class extending
`behat_deprecated_base`; the method calls `$this->deprecated_message()` with
the replacement, then keeps working:

```php
<?php
// In question/bank/comment/tests/behat/ (GPL boilerplate omitted).
// Require the base class explicitly, as the docs example does.
require_once(__DIR__ . '/../../../../../lib/behat/behat_deprecated_base.php');

class behat_qbank_comment_deprecated extends behat_deprecated_base {
    /**
     * Looks for the appropriate hyperlink comment count in the column.
     *
     * @Then I should see :arg1 on the comments column
     * @param string $linkdata
     * @deprecated Since Moodle 5.0 MDL-79122 in favour of the "qbank_comment > Comment count link" named selector.
     * @todo Final removal in Moodle 6.0 MDL-82413.
     */
    public function i_should_see_on_the_column(string $linkdata): void {
        $this->deprecated_message("Use '\"{$linkdata}\" \"qbank_comment > Comment count link\" should exist'");
        $this->execute('behat_general::should_exist', [$linkdata, 'qbank_comment > Comment count link']);
    }
}
```

Deprecated steps fail tests unless `$CFG->behat_usedeprecated` is set.

### Theme overrides

Contexts in `/theme/{MYTHEME}/tests/behat/` named
`behat_theme_{MYTHEME}_<context>` (e.g.
`mod_forum/behat_theme_{MYTHEME}_behat_mod_forum.php`); selectors via
`behat_theme_{MYTHEME}_behat_selectors` extending `behat_selectors`; disable
contexts/features for the theme suite in
`/theme/{MYTHEME}/tests/behat/blacklist.json`
(`{"contexts": [...], "features": [...]}`).

## Generators in Given steps

Reuse the PHPUnit generator (`tests/generator/lib.php`) via a Behat generator
in `tests/generator/behat_<component>_generator.php` (full options: PHPDoc of
`behat_generator_base`; core example: `mod/quiz/tests/generator`):

```php
// tests/generator/lib.php
class local_myplugin_generator extends component_generator_base {
    public function create_thing($thing) {
        global $DB;
        $DB->insert_record('local_myplugin_things', $thing);
    }
}

// tests/generator/behat_local_myplugin_generator.php
class behat_local_myplugin_generator extends behat_generator_base {
    protected function get_creatable_entities(): array {
        return [
            'things' => [
                'datagenerator' => 'thing',   // -> create_thing()
                'required' => ['name'],
            ],
        ];
    }
}
```

```gherkin
Given the following "local_myplugin > things" exist:
  | name   |
  | thing1 |
  | thing2 |
```

## Generator tool (test data outside Behat runs)

With debugging at DEVELOPER, `admin/tool/generator` builds test data:

```bash
php admin/tool/generator/cli/maketestcourse.php --shortname=SIZE_S --size=S

# Run a Behat generator .feature (generator/set-config steps only, scenarios
# and outlines, no Background, no browser) against the current instance:
php admin/tool/generator/cli/runtestscenario.php --feature=/path/to/some/testing/scenario.feature
```

Web equivalents: Site administration > Development > Make test course /
Create testing scenarios. Example: `admin/tool/generator/tests/fixtures/testscenario/scenario.feature`.

## Troubleshooting

- New steps/features not picked up: re-run `init.php` (or `util.php --enable`).
- Useless error message: rerun with `-vv`.
- Timeouts on slow machines: `$CFG->behat_increasetimeout`.
- Time-dependent behaviour in `mod_assign` is mockable in Behat and PHPUnit
  since 5.1 (it now uses the core clock).
