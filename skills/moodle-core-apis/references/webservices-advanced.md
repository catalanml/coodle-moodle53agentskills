# Web Services: Advanced (Routing, Scopes, REST Style, Testing, Files) — Moodle 5.3

Covers the Routing subsystem (routes, mandatory scopes, parameters, responses, route testing, shortlinks), the REST API style rules for writing routes, unit testing external functions, and file upload/download in web services. Per Moodle 5.3 developer docs plus the 5.0–5.3 UPGRADING notes.

## Routing subsystem (new in 4.5, MDL-81301)

Built on Slim Framework + FastRoute. Routes are classes in a component's `route` L2 namespace, with methods tagged by the `\core\router\route` attribute. The L3 namespace is the route *group*; unknown groups are ignored. Supported group for plugin routes:

| Route Group | Namespace | URI Prefix | Purpose |
| --- | --- | --- | --- |
| API | `api` | `/api/rest/v2` | REST Web Services |

Full path = group prefix + component + attribute path. Route groups apply middleware (CORS, input/output sanitisation, path normalisation). Method parameters are resolved via Dependency Injection plus path/query/header parameter resolution. Class-level `route` attributes may define a prefix and defaults but cannot handle a route themselves. Routes may also describe HTTP methods (`method: 'GET'`), header parameters, responses, and examples in the attribute. The site must have the router configured (admin docs: "Configuring the Router").

**Since 5.3 (MDL-89710): every route in the `route\api` namespace MUST carry either one or more `#[scopeset(...)]` attributes or `#[unscoped_resource]`.** A core PHPUnit test scans every route in `route\api`; a route without one of these fails the core test suite. Always include it — including when copying the example below.

```php
// mod/example/classes/route/api/example.php  (public/mod/example/... in a git checkout)
namespace mod_example\route\api;

use core\router\route;
use core\router\schema\response\payload_response;
use core\router\scope\scopeset;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class example {
    #[route(
        // Resolves to https://example.com/moodle/api/rest/v2/mod_example/examples
        path: '/examples',
        method: 'GET',
    )]
    #[scopeset(
        new \mod_example\route\scope\example\read(),
    )]
    public function get_examples(
        ServerRequestInterface $request,
        ResponseInterface $response,
    ): ResponseInterface {
        // Scopes never replace capability checks: check the relevant context
        // with require_capability() (see access-roles.md) before returning data.
        return new payload_response(request: $request, response: $response, payload: []);
    }
}
```

Since 5.3 (MDL-87174) a route attribute can set `cookies: false` (or `true`) to control whether cookies/session are supported for that route (replaces checking `NO_MOODLE_COOKIES`; see `\core\session\manager::supports_cookies()`). The attribute class is `\core\router\route` (verified in MOODLE_503_STABLE source); the UPGRADING example's `\core\router\attributes\route` does not exist.

## Scopes (new in 5.3, MDL-89710)

API routes are protected by normal login and capability checks; when a request is authenticated with an **OAuth2 access token** or a **Personal Access Token**, the router *also* enforces scopes — named permissions such as `core_user:user:read` that limit what a client/integration can do. Scope checks are in addition to capability checks and have **no effect on cookie/session-authenticated requests**.

Personal access tokens (5.3, MDL-87706): minted by `\core\api\token_manager`; users manage their own at `/user/personalaccesstokens.php`, gated by the `moodle/api:createtoken` capability. OAuth2 server clients are managed by `\core\oauth2\server\client_manager` (MDL-89181).

### Declaring required scopes

`\core\router\scope\scopeset` goes on the route method next to `#[route]`:

```php
use core\router\scope\scopeset;
use core\router\scope\unscoped_resource;

// AND: every scope in one #[scopeset] is required together.
#[scopeset(
    new \core_user\route\scope\user\read(),
    new \core_user\route\scope\user\write(),
)]
public function set_preferences(/* ... */): ResponseInterface {}

// OR: the attribute is repeatable; satisfying any one set grants access.
#[scopeset(new \core_course\route\scope\course\content\read())]
#[scopeset(new \core_course\route\scope\course\content\write())]
public function categories(/* ... */): ResponseInterface {}

// Not resource-specific (e.g. a discovery/status endpoint): opt out explicitly.
#[route(path: '/status')]
#[unscoped_resource]
public function status(/* ... */): ResponseInterface {}
```

A route with no `#[scopeset]` currently behaves at runtime like `#[unscoped_resource]`, but that is incidental — the opt-out must be explicit, and the core test fails without it.

How it is enforced: `\core\router\middleware\moodle_scope_attribute_middleware` attaches the resolved `scopeset[]` to the request; `\core\router\middleware\moodle_api_authentication_middleware` enforces them for API-key `Bearer` tokens and OAuth2 access tokens. If no scopeset is satisfied the request is rejected with an OAuth2 `access_denied` error and a hint naming the missing scope(s).

Unknown scopes fail safe: a `#[scopeset]` referencing a scope class that does not exist in the running Moodle version gets a `\core\router\scope\unknown_scope` placeholder which can never be satisfied; other scopesets on the same route still work.

Vary behaviour by granted scope inside a route (does not grant/deny access itself):

```php
use core\router\util;
use core_user\route\scope\user\read;
use core_user\route\scope\user\read_any;

#[scopeset(new read())]
#[scopeset(new read_any())]
public function serve(ServerRequestInterface $request, ResponseInterface $response, \stdClass $user): ResponseInterface {
    if (util::has_scope_granted($request, new read_any())) {
        // Check the capability to access this user.
    } else {
        // Token only has user:read - restrict to the caller's own record.
        $this->check_user($user);
    }
    // ...
}
```

### Defining scope classes

Scopes live in the component's `route\scope` L2 namespace (e.g. `mod/example/classes/route/scope/example/`). No registration: every component's `route\scope` namespace is scanned for concrete subclasses of `\core\router\scope\abstract_scope` and cached — **purge caches** after adding/renaming/removing one. Renaming or removing a scope breaks clients granted it; prefer adding new scopes and deprecate removals.

Hierarchy: abstract "grouping" classes each contribute one identifier segment; concrete leaf classes (`read`, `write`, `delete`, `backup`, …) are what `#[scopeset]` references. By convention each grouping class is called `abstract_scope` in its own namespace.

```php
// mod/example/classes/route/scope/example/abstract_scope.php
namespace mod_example\route\scope\example;

#[\core\router\scope\identifier_attribute('example')]
abstract class abstract_scope extends \core\router\scope\abstract_scope {
}

// mod/example/classes/route/scope/example/read.php  -> identifier mod_example:example:read
namespace mod_example\route\scope\example;

#[\core\router\scope\identifier_attribute('read')]
#[\core\router\scope\summary_attribute('example_read_scope_summary', 'mod_example')]
#[\core\router\scope\description_attribute('example_read_scope_desc', 'mod_example')]
class read extends abstract_scope {
}
```

Rules:

- Every class in the chain (abstract and leaf) needs `#[identifier_attribute]`; a missing one throws `coding_exception` when the identifier is resolved. Identifier: starts with a lowercase letter, then lowercase letters/digits/underscores only; no `:` and no component name (both added automatically). `abstract_scope::get_identifier()` walks up the hierarchy and prepends the frankenstyle component of the leaf's namespace.
- Concrete classes **must** have `#[summary_attribute]` (short label) and should have `#[description_attribute]` (consent screen / OpenAPI text). Both extend `\core\lang_string` (args as `get_string()`), resolved lazily by `get_summary()` / `get_description()`. A concrete class without a summary throws `coding_exception`.
- Lang string convention: key after the identifier without component, e.g. `$string['example_read_scope_summary'] = 'View examples';`.

### Testing scope declarations

The route testcase provides:

- `assert_route_is_scoped($callable)` — at least one `#[scopeset]`.
- `assert_route_is_unscoped($callable)` — explicitly `#[unscoped_resource]`.
- `assert_route_required_scopes($expectedscopesets, $callable)` — exact sets as array of arrays of identifier strings (outer = OR, inner = AND); unknown scopes are ignored.

```php
$this->assert_route_required_scopes(
    [['mod_example:example:read']],
    [\mod_example\route\api\example::class, 'get_examples'],
);
```

## Parameters

**Path parameters:** placeholders `{name}` must be described in `pathtypes` with `\core\router\schema\parameters\path_parameter`. Types are `\core\param` enum cases (`param::ALPHANUM`, `param::INT`, `param::TEXT`, …) and are validated before your method runs. Read them by named method argument or via `array $args`.

```php
#[route(
    path: '/users/{username}',
    pathtypes: [
        new path_parameter(name: 'username', type: param::ALPHANUM),
    ],
)]
public function get_user_information(string $username, array $args): ResponseInterface {}
```

Optional segments use `[]`: `'/users[/{username}]'` creates `/users` and `/users/{username}`. Unspecified optionals default to `null` (method arg must be nullable) or the `path_parameter`'s `default`. Nest for several: `'/users[/{name}/[{pet}]]'`. Everything after the first optional must be optional (`'/users[/{name}]/example'` is invalid); required parameters cannot have defaults.

**Query parameters:** `queryparams` with `query_parameter` instances; optional unless `required: true`. Read via `$request->getQueryParams()`.

**Header parameters:** `headerparams` with `\core\router\schema\parameters\header_object` (`required`, `default`, `multiple: true`). Read with `$request->getHeader('X-Name')` (array) or `$request->getHeaderLine('X-Name')` (comma-joined).

**Reusable parameters:** extend a parameter class and implement the marker interface `\core\router\schema\referenced_object` so the OpenAPI output uses a shared component reference (core example: `core\router\parameters\path_themename`, preset `name`, `type: param::ALPHANUMEXT`, `description`).

**Mapped parameters:** implement `\core\router\schema\parameters\mapped_property_parameter::add_attributes_for_parameter_value(ServerRequestInterface $request, string $value): ServerRequestInterface` to turn a raw value into an object attached as a request attribute of the same name (core's `path_course` also adds `{name}_context`, accepts `idnumber:`/`name:` prefixes, and throws `not_found_exception`). Override `get_schema_from_type(param $type)` to add an OpenAPI `pattern`. Since 5.2 (MDL-87264) core adds `core\router\parameter\query_course` and `core\router\parameter\query_coursemodule` (course / course-module IDs as query parameters, converted to objects and contexts) and a `path_module` type. **Note:** UPGRADING writes the namespace `core\router\parameter\` while the devdocs example uses `core\router\parameters\`.

**Examples:** any parameter accepts `examples: [new \core\router\schema\example(name: 'The Boost theme', value: 'boost')]`.

## Responses

Return a PSR-7 `ResponseInterface` or, for web service routes almost always, a `\core\router\schema\response\payload_response` — serialised to JSON and validated against the route's response schema. Do not return raw JSON.

```php
$response = $response->withHeader('X-Example', 'Added header'); // Body is always replaced by the payload.
return new payload_response(
    payload: ['users' => ['charlie', 'john', 'david']], // Any JSON-encodable array.
    request: $request,
    response: $response,
);
```

## REST API style rules (guides/rest/style.md, new in 5.3)

Normative rules for designing routes (MUST/SHOULD per RFC 2119):

- **Resources, not actions.** URIs are plural nouns; HTTP method carries the verb (Richardson level 2). Forbidden: RPC-style `/core_course_get_courses`, `/createCourse`, `/course/create`, action sub-paths like `/duplicate`, `/check`, `/calculate`, or `"action": "hide"` in bodies — set state declaratively (`PATCH … {"visible": false}`).
- **Stateless:** no dependence on server-side session state; prefer OAuth2 / Personal Access Tokens over session cookies.
- **Methods:** GET retrieve (never modifies state), POST create (never retrieval), PUT replace, PATCH partial update, DELETE remove. GET/PUT/DELETE idempotent; retryable POST SHOULD support `Idempotency-Key`. Cloning/imports = `POST` to the destination collection with the source as a query param (`?clonefrom=ID`). Delta checks = `GET …/updates?since=…`.
- **URI shape:** `/api/rest/v2/<component>/<resource>[/{identifier}]`. Core components drop `core_` (`/course/courses`, `/user/users`); plugins use the full frankenstyle name (`/mod_forum/discussions`, `/local_myplugin/settings`, subplugins `/assignsubmission_comments/comments/455`). Component prefixes are lowercase, keep underscores.
- **Resource names:** plural, lowercase, kebab-case (`/course-categories`, not `/courseCategories` or `/course_categories`). Nesting only for direct ownership, max depth ~2 (`/courses/{courseId}/sections`).
- **Path parameters:** camelCase ending in `Id` for identifiers (`{courseId}`, `{userId}`), never `{course_id}`, `{userid}`, `{user}`.
- **JSON fields:** camelCase (`courseId`, `shortName`, `createdAt`). Nullable fields explicit; no nulls in arrays; no generic `"data"` wrappers; no undocumented enum values.
- **Dates:** ISO-8601 with timezone (`2026-05-19T12:30:00+01:00`), SHOULD be in the current user's timezone; never Unix timestamps.
- **Collections:** respond `{"items": [...], "pagination": {"page": 1, "pageSize": 25, "totalItems": 100}}`; accept `?page=1&pagesize=25` (defaults 1 / 25, max pagesize 100, endpoint MAY override); sort with `?sort=createdat,-fullname` (`-` = descending; unsorted order not guaranteed); filter with query params (`?categoryid=3&visible=true`). Collections MUST apply authorization filtering.
- **Single resource:** authorization + capability checks, 404 handling. **Create:** `201 Created` with `Location` header and the created representation; validate input, handle duplicates, transactional.
- **Errors:** RFC 9457 problem details (`type`, `title`, `detail`, `instance`); validation errors include field-level details; error codes are stable lowercase concatenated words (`coursenotfound`, not `course_not_found` / `courseNotFound`).
- **Status codes:** 200, 201, 202, 204, 400 invalid, 401 unauthenticated, 403 unauthorized, 404, 409 conflict, 412 precondition failed (`ETag`/`If-Match` optimistic concurrency), 422 validation, 429 rate limited, 500.
- **Authorization:** scope-based; every endpoint defines its required scope sets (`core_course:course:content:read`).
- **Migrating an RPC function to REST:** the REST payload must be a superset of the legacy return structure; old fields removed only in a major version.

**Note:** the style guide is itself new and partly inconsistent — §12.1 (RFC 9457 shape) and §12.2 (`{"error": {"code", "message", "details"}}` shape) differ, and §16 is marked TO DO. Core routes are still migrating (5.2 moved question counts to `/api/rest/v2/question/counts`).

## Testing routes

Extend the route testcase. The autoloadable class is `\core\tests\router\route_testcase` (`public/lib/tests/classes/router/route_testcase.php`, verified in MOODLE_503_STABLE source), with `\core\tests\router\mocking_route_loader` (UPGRADING 5.0, MDL-83968). The 5.3 devdocs still write `\route_testcase`; use the namespaced name. `get_router()` returns a fully-configured router containing *all* routes; prefer the mocking route loader and add only the routes under test **before** calling `get_router()` (routes are injected via DI at router initialisation).

```php
final class example_test extends \core\tests\router\route_testcase {
    public function test_get_examples(): void {
        $this->resetAfterTest();

        // Either add one route...
        $this->add_route_to_route_loader(\mod_example\route\api\example::class, 'get_examples');
        // ...or all routes in a class. The route group argument is implied for classes
        // in an `api` namespace but must be set for other route groups.
        // $this->add_class_routes_to_route_loader(
        //     \mod_example\route\api\example::class,
        //     \core\router\route_loader_interface::ROUTE_GROUP_API,
        // );

        // Manual: create the request (default group is the REST API) and run it
        // through all middleware.
        $request = $this->create_request(
            'GET',
            '/path/to/route',
            \core\router\route_loader_interface::ROUTE_GROUP_API,
        );
        $response = $this->get_router()->get_app()->handle($request);

        // Shortcut (create request + router + handle). Also accepts headers and query params.
        $response = $this->process_api_request('GET', '/path/to/route');
    }
}
```

The request path is relative to the `/api/rest/v2` prefix; the only concrete devdocs example (`process_api_request('GET', "/bank/{$course->id}/question_counts")` for the core_question route `/api/rest/v2/question/bank/{course}/question_counts`) omits the component segment, so check against an existing core test. `process_request()` is the generic-group equivalent. Parameter classes (especially mapped parameters) can be unit tested for validation too. Add `assert_route_is_scoped()` / `assert_route_required_scopes()` tests for every route.

## Shortlinks (since 5.1)

Concise URLs (SMS etc.) routed through the routing system (which must be enabled): `https://site/s/X8fG56aa` (private, tied to user(s)) or `/p/…` (public). Short codes use ALPHANUMEXT minus ambiguous characters (`I`, `l`).

```php
$shortlink = \core\di::get(\core\shortlink::class);
$public = $shortlink->create_shortlink(
    component: 'mod_assign', linktype: 'view', identifier: $cm->id, userid: 0, minlength: 4, maxlength: 4,
);
$private = $shortlink->create_shortlink_for_users(
    component: 'mod_assign', linktype: 'view', identifier: $cm->id, userids: $userids, minlength: 4, maxlength: 4,
);
```

Each component using shortlinks needs a `shortlink_handler` class implementing `shortlink_handler_interface`:

```php
class shortlink_handler implements shortlink_handler_interface {
    #[\Override]
    public function get_valid_linktypes(): array {
        return ['view'];
    }

    #[\Override]
    public function process_shortlink(string $type, string $identifier): ?\core\url {
        return match ($type) {
            'view' => new \core\url('/mod/assign/view.php', ['id' => $identifier]),
            default => null,
        };
    }
}
```

The devdocs do not give the namespace of `shortlink_handler_interface` or where the handler class must live; check core implementations.

## Unit testing external functions

One testcase per external function file: `classes/external/get_fruit.php` → `tests/external/get_fruit_test.php`. Since 5.1 (MDL-86283) extend the autoloadable `\core_external\tests\externallib_testcase`, which replaces `\externallib_advanced_testcase` (the devdocs page still shows the old class plus `require_once($CFG->dirroot . '/webservice/tests/helpers.php')`, no longer needed).

```php
namespace mod_kitchen\external;

use core_external\external_api;

/**
 * Tests for get_fruit.
 *
 * @package    mod_kitchen
 * @covers     \mod_kitchen\external\get_fruit
 */
final class get_fruit_test extends \core_external\tests\externallib_testcase {
    public function test_capabilities(): void {
        $this->resetAfterTest(true);

        $course = $this->getDataGenerator()->create_course();
        $cm = $this->getDataGenerator()->create_module('mod_kitchen', ['course' => $course->id]);

        // Assign the capability the function requires.
        $contextid = \core\context\module::instance($cm->cmid)->id;
        $roleid = $this->assignUserCapability('moodle/CAPABILITYNAME', $contextid);

        $returnvalue = get_fruit::execute(['course' => $course->id, 'cmid' => $cm->id]);

        // Clean return values to simulate the web service server.
        $returnvalue = external_api::clean_returnvalue(get_fruit::execute_returns(), $returnvalue);
        $this->assertNotNull($returnvalue);
    }

    public function test_capabilities_missing(): void {
        $this->resetAfterTest(true);
        // ...create course/module and assign as above, then:
        $this->unassignUserCapability('moodle/CAPABILITYNAME', $contextid, $roleid);

        $this->expectException(\required_capability_exception::class);
        get_fruit::execute(['course' => $course->id, 'cmid' => $cm->id]);
    }
}
```

Key patterns: `resetAfterTest(true)`, data generators, `assignUserCapability()` / `unassignUserCapability()`, calling `execute()` directly, and always running `external_api::clean_returnvalue()` against `execute_returns()`.

## File handling in web services

Two mechanisms:

1. **Web service functions** — `core_files_get_files()` and `core_files_upload()`; file content is base64-encoded. Not recommended for large files (memory limits on many devices).
2. **Dedicated endpoints** (recommended) — `/webservice/upload.php` (upload) and `/webservice/pluginfile.php` (download), both authenticated by web service token.

### Upload: `/webservice/upload.php`

`POST` with a token; files land in the user's **draft file area**. Optional arguments: `itemid` (default `0` — a new draft item id is generated) and `filepath` (default `/`). To put several files in one draft area, take the `itemid` from the first upload's response and pass it to later uploads. Returns JSON (file info on success, error on failure):

```bash
curl -X POST -F "file_1=@users.csv" https://SITENAME/webservice/upload.php?token=TOKEN
# → [{"component":"user","contextid":567,"filearea":"draft","filename":"users.csv",
#     "filepath":"/","itemid":880413555,"license":"allrightsreserved",...}]
```

Then call a web service function (e.g. `mod_assign_save_submission`) with the draft `itemid` so it can process the files. The service must allow "files download" (Administration > Plugins > Web services > Manage services > Edit service > Advanced); in a `db/services.php` custom service this maps to `downloadfiles` / `uploadfiles`.

### Download: `/webservice/pluginfile.php`

Same URL structure as `/pluginfile.php` and `/tokenpluginfile.php`, but token-authenticated (plain `/pluginfile.php` needs a login session; `/tokenpluginfile.php` also works with an appropriate token). The service must allow "files download".

### Returning file areas from external functions

Use `\core_external\util::get_area_files()` (formerly `external_util`, deprecated in 5.3 by MDL-81225 — the devdocs still show the old name) plus the `external_files` structure:

```php
use core_external\external_files;
use core_external\util;

$forum->introfiles = util::get_area_files($context->id, 'mod_forum', 'intro', false, false);

public static function execute_returns(): external_multiple_structure {
    return new external_multiple_structure(
        new external_single_structure([
            'id' => new external_value(PARAM_INT, 'Forum id'),
            'introfiles' => new external_files('Files in the introduction text', VALUE_OPTIONAL),
        ])
    );
}
```
