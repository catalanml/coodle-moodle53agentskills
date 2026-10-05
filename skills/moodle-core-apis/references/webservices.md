# External API / Web Services (Moodle 5.3)

Scope: writing external functions (`\core_external\external_api`), declaring them in `db/services.php`, security (validate_parameters/validate_context/capabilities), calling them from JS via `core/ajax`, and service/protocol overview. Per Moodle 5.3 developer docs plus the 5.0–5.3 UPGRADING notes. For REST routes (`/api/rest/v2`), scopes, route testing and WS file transfer see `webservices-advanced.md`.

## Overview

Moodle's Web Service framework plus the External API provide self-describing endpoints. Moodle uses them internally for AJAX interactions in the web UI and for integrations. Flow: client authenticates (username/password against a login endpoint, receiving a token), then calls functions through a protocol server which checks access, invokes the external function, and validates the return values against the declared structure before responding. Since 4.5 there is also a routing subsystem exposing REST endpoints under `/api/rest/v2` (see Routing below and `webservices-advanced.md`).

Naming convention for function names: `[frankenstyle_component]_[verb]_[noun]`, verb usually one of get/create/delete/update, noun usually plural — e.g. `local_groupmanager_create_groups`.

**Paths:** since 5.1 (MDL-83424) code lives under `public/` in a git checkout, so `local/groupmanager/db/services.php` below is `public/local/groupmanager/db/services.php` at repository level. Paths relative to `$CFG->dirroot` are unchanged.

## Declaring functions: db/services.php

Functions must be declared in the plugin's `db/services.php`; a service discovery process stores them in the DB during install/upgrade (bump the plugin version after any change).

```php title="local/groupmanager/db/services.php"
$functions = [
    'local_groupmanager_create_groups' => [
        // The name of the namespaced class that the function is located in.
        'classname'   => 'local_groupmanager\external\create_groups',

        // A brief, human-readable, description of the web service function.
        'description' => 'Creates new groups.',

        // Options include read, and write.
        'type'        => 'write',

        // Whether the service is available for use in AJAX calls from the web.
        'ajax'        => true,

        // An optional list of predeclared services where the function will be included.
        'services'    => [
            'myintegration',
        ],
    ],
];
```

Advanced (optional / legacy) keys:

```php
'capabilities' => 'moodle/course:creategroups,moodle/course:managegroups', // advisory list for admins
// No longer recommended:
'methodname'  => 'execute',                          // defaults to 'execute'
'classpath'   => 'local/groupmanager/externallib.php', // do not use with autoloaded classes
'loginrequired' => false, // rare: callable without a session (100% public data only, e.g. core_get_string)
```

Since 5.3 (MDL-87150): a new `allowcorsrequests` property in the function definition allows specific AJAX web services to support CORS. The devdocs pages do not yet document its value format; check core `db/services.php` usages before relying on it.

### Custom service declarations (rare)

Moodle ships two built-in services your functions can be attached to. A `$services` array in the same file can pre-build a custom service; if omitted, an administrator must create one manually in the web UI. Admins cannot add/remove functions from a pre-built service:

```php
$services = [
    // The name of the service; does not need to include the component name.
    'myintegration' => [
        'functions' => ['local_groupmanager_create_groups'],
        'requiredcapability' => 'local_groupmanager/integration:access', // needed to call any function
        'restrictedusers' => 0, // 1 = admin must link users to the service in the web UI
        'enabled' => 1,
        'shortname' => '',      // optional, but required if restrictedusers is set
        'downloadfiles' => 0,   // allow file downloads
        'uploadfiles'  => 0,    // allow file uploads
    ],
];
```

## The external function class

Located in the component's `external` namespace: for `local_groupmanager_create_groups`, class `local_groupmanager\external\create_groups` in `local/groupmanager/classes/external/create_groups.php`. It:

- _must_ extend `\core_external\external_api`
- _must_ declare `execute_parameters()`, `execute()`, and `execute_returns()`
- _may_ declare `execute_is_deprecated()` (return true to mark deprecated)

**Deprecated in 5.3 (MDL-81225):** the legacy non-namespaced classes and functions from `lib/externallib.php` are deprecated. Always use the `\core_external\` names:

| Old (deprecated) | Use instead |
| --- | --- |
| `\external_api` | `\core_external\external_api` |
| `\external_value`, `\external_single_structure`, `\external_multiple_structure` | `\core_external\external_value`, `\core_external\external_single_structure`, `\core_external\external_multiple_structure` |
| `\external_function_parameters`, `\external_description`, `\external_format_value` | `\core_external\external_function_parameters`, `\core_external\external_description`, `\core_external\external_format_value` |
| `\external_files`, `\external_warnings`, `\external_settings` | `\core_external\external_files`, `\core_external\external_warnings`, `\core_external\external_settings` |
| `\external_util`, `\restricted_context_exception` | `\core_external\util`, `\core_external\restricted_context_exception` |
| `external_format_text()`, `external_format_string()` | `\core_external\util::format_text()`, `\core_external\util::format_string()` |
| `external_validate_format()`, `external_delete_descriptions()` | `\core_external\util::validate_format()`, `\core_external\util::delete_service_descriptions()` |
| `external_generate_token()`, `external_create_service_token()` | `\core_external\util::generate_token()` |
| `external_generate_token_for_current_user()`, `external_log_token_request()` | `\core_external\util::generate_token_for_current_user()`, `\core_external\util::log_token_request()` |

```php title="local/groupmanager/classes/external/create_groups.php"
<?php
// This file is part of Moodle - http://moodle.org/
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
// along with Moodle.  If not, see <http://www.gnu.org/licenses/>.

namespace local_groupmanager\external;

use core_external\external_api;
use core_external\external_function_parameters;
use core_external\external_multiple_structure;
use core_external\external_single_structure;
use core_external\external_value;

/**
 * External function to create groups.
 *
 * @package    local_groupmanager
 * @copyright  2026 Your Name
 * @license    http://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */
class create_groups extends external_api {
    /**
     * Describe the parameters.
     *
     * @return external_function_parameters
     */
    public static function execute_parameters(): external_function_parameters {
        return new external_function_parameters([
            'groups' => new external_multiple_structure(
                new external_single_structure([
                    'courseid' => new external_value(PARAM_INT, 'The course to create the group for'),
                    'idnumber' => new external_value(
                        PARAM_RAW,
                        'An arbitrary ID code number perhaps from the institution',
                        VALUE_DEFAULT,
                        null
                    ),
                    'name' => new external_value(PARAM_RAW, 'The name of the group'),
                    'description' => new external_value(PARAM_TEXT, 'A description', VALUE_OPTIONAL),
                ]),
                'A list of groups to create'
            ),
        ]);
    }

    /**
     * Create the groups.
     *
     * @param array $groups
     * @return array
     */
    public static function execute(array $groups): array {
        // Validate all of the parameters.
        [
            'groups' => $groups,
        ] = self::validate_parameters(self::execute_parameters(), [
            'groups' => $groups,
        ]);

        // Perform security checks for every course context touched.
        foreach ($groups as $group) {
            $coursecontext = \core\context\course::instance($group['courseid']);
            self::validate_context($coursecontext);
            require_capability('moodle/course:creategroups', $coursecontext);
        }

        // Create the group using existing Moodle APIs.
        $createdgroups = \local_groupmanager\util::create_groups($groups);

        // Return a value as described in the returns function.
        return [
            'groups' => $createdgroups,
        ];
    }

    /**
     * Describe the return structure.
     *
     * @return external_single_structure
     */
    public static function execute_returns(): external_single_structure {
        return new external_single_structure([
            'groups' => new external_multiple_structure(
                new external_single_structure([
                    'id' => new external_value(PARAM_INT, 'Id of the created group'),
                    'name' => new external_value(PARAM_RAW, 'The name of the group'),
                ])
            ),
        ]);
    }
}
```

**Note:** the upstream devdocs example passes a bare array to `external_multiple_structure` in `execute_returns()` and uses an undefined `$courseid`; the version above is corrected.

### Describing parameters and returns

- list => `external_multiple_structure`
- object => `external_single_structure`
- primitive => `external_value(PARAM_*, 'description')` — PARAM types are defined in `lib/moodlelib.php` (PARAM_INT, PARAM_TEXT, PARAM_RAW, PARAM_BOOL, ...)
- `execute_parameters()` returns an `external_function_parameters` (empty array if the function takes none); `execute_returns()` describes the return the same way.

Requiredness: third arg of `external_value` is `VALUE_REQUIRED` (default), `VALUE_OPTIONAL`, or `VALUE_DEFAULT` (fourth arg is the default):

```php
'yearofstudy' => new external_value(PARAM_INT, 'year of study', VALUE_DEFAULT, 1979),
```

`VALUE_OPTIONAL` cannot be used on top-level parameters (some protocols are strict about argument counts) — nest optional values inside an `external_single_structure`, or use `VALUE_DEFAULT` at top level.

Return-value validation by the server: undescribed values are skipped; missing `VALUE_REQUIRED` values or type mismatches cause a server error. Cast returned objects to arrays.

### Deprecating an external function

Follow the standard deprecation policy **and** add to the class:

```php
/**
 * Mark the function as deprecated.
 *
 * @return bool
 */
public static function execute_is_deprecated(): bool {
    return true;
}
```

## Security inside execute()

1. **Always** validate parameters first — returns validated/cleaned values, throws on missing/malformed input:

```php
$params = self::validate_parameters(self::execute_parameters(), ['groups' => $groups]);
```

Parameters of `execute()` must be in the same order as declared in `execute_parameters()`.

2. **Always** call `self::validate_context($context)` with the _most specific_ context of the data before operating on it (activity context for activity data, course context for course data). For multiple contexts (e.g. a list of courses), validate each one right before producing data for it. It also sets up `$PAGE`/`$OUTPUT`, theme, language, and filters for rendering. Do NOT use `require_login()` or `$PAGE->set_context()` in an external function.

3. Check capabilities with the standard functions: `has_capability()`, `has_any_capability()`, `has_all_capability()`, `require_capability()`, `require_all_capabilities()`.

```php
$context = \core\context\course::instance($group->courseid);
self::validate_context($context);
require_capability('moodle/course:managegroups', $context);
```

Exceptions thrown in `execute()` (e.g. `invalid_parameter_exception`) are handled automatically by the web service servers; put details in `$debuginfo` — generic messages are shown when DEBUG < NORMAL. `$DB->start_delegated_transaction()` / `$transaction->allow_commit()` can wrap multi-record writes so exceptions roll everything back.

## Calling from JavaScript (core/ajax)

Requires `'ajax' => true` in the function's `db/services.php` entry. Convention: put all `core/ajax` usage in a `repository.js` module.

```js title="mod/assign/amd/src/repository.js"
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
const response = await submitGradingForm(assignmentId, userId, data);
```

`fetchMany([...])` accepts multiple `{methodname, args}` requests chained in a single HTTP request and returns an array of promises. Considerations:

- Text returned from a web service must be filtered via `\core_external\util::format_text()` / `\core_external\util::format_string()` (formerly `external_format_text()` / `external_format_string()`, deprecated in 5.3) with the correct (most specific) context.
- Notify filters after adding dynamic content: `M.core.event.FILTER_CONTENT_UPDATED`.
- After changing `db/services.php`, bump the version and run the upgrade or AJAX cannot find the function.
- Session-less calls (rare, public data only): `'loginrequired' => false` in the declaration and pass `false` as the 3rd argument to `call`.
- React/ESM code (5.2+) reaches `core/ajax` through `requireAsync()` from `@moodle/lms/core/amd`; see `react.md`.

## Unit testing (summary)

One testcase per external function: `classes/external/get_fruit.php` → `tests/external/get_fruit_test.php`. Since 5.1 (MDL-86283) extend the autoloadable `\core_external\tests\externallib_testcase` (replaces `\externallib_advanced_testcase`; no `require_once` of `webservice/tests/helpers.php` needed). Call `execute()` directly and always pass the result through `\core_external\external_api::clean_returnvalue(get_fruit::execute_returns(), $result)`. Full example in `webservices-advanced.md`.

## Protocols and endpoints overview

- Protocol servers validate declared parameters and return values and can build WSDL/other protocol documents from the descriptions.
- Live API documentation for a site: **Site administration > Server > Web services > API Documentation**. New services appear under *Administration > Plugins > Web Services > Manage services* after the version bump/upgrade.
- Dedicated endpoints exist for file upload and download (`/webservice/upload.php`, `/webservice/pluginfile.php`; custom services opt in via `downloadfiles`/`uploadfiles`) — see `webservices-advanced.md`.
- Since 5.3 (MDL-87010): token endpoint hardening (earlier service validation, GET parameter sanitisation); the `appsitecheck` parameter was removed from the token endpoint.
- **Routing (new in 4.5, MDL-81301)**: REST routes are defined as classes in a component's `route\api` namespace using the `#[\core\router\route(path: '/example')]` attribute on methods; the `api` route group maps to the `/api/rest/v2` URI prefix (full path `/api/rest/v2/{component}{path}`) and applies middleware for CORS, input/output sanitisation, etc. Method parameters are resolved via dependency injection plus path/query/header parameters; return a `core\router\schema\response\payload_response`. **Since 5.3 (MDL-89710) every `route\api` method must also carry `#[scopeset(...)]` or `#[unscoped_resource]`** — enforced by a core PHPUnit test. Details in `webservices-advanced.md`.
