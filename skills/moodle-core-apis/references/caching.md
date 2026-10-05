# Cache API (MUC)

Compact reference for the Moodle 5.3 Universal Cache: defining caches in `db/caches.php`, getting a cache instance, get/set/delete, versioned get/set, invalidation events, and key rules. Distilled from the Moodle 5.3 developer docs and the 5.0–5.3 `UPGRADING.md` notes.

**What changed since 4.5:** the MUC page now recommends the versioned cache functions `set_versioned()` / `get_versioned()` for localized (multi-layer) caches and discourages the older "revision in the value" trick (see the last section). Since 5.2 (MDL-85336) Redis connection timeout settings for cache stores and sessions are split into a connection timeout and a read timeout, both accepting floats. The rest of the API is unchanged.

## Defining a cache (db/caches.php)

Cache definitions live in the component's `db/caches.php` (core: `lib/db/caches.php`; a module: `mod/myplugin/db/caches.php`; since 5.1 these sit under `public/` in a git checkout, e.g. `public/lib/db/caches.php`). Bump the plugin version number after adding/changing definitions so the upgrade processes the file.

Bare minimum — an area name (the array key) and a mode:

```php
// mod/myplugin/db/caches.php
$definitions = [
    'somedata' => [
        'mode' => cache_store::MODE_APPLICATION,
    ]
];
```

Modes:

- `MODE_APPLICATION` — shared cache, all users can access it.
- `MODE_SESSION` — scoped to a single user's session (but may not actually be stored in the session).
- `MODE_REQUEST` — like a static cache: only the owning request, alive until the end of the request.

Each definition requires a language string named `cachedef_<name>`:

```php
// mod/myplugin/lang/en/mod_myplugin.php
$string['cachedef_somedata'] = 'This is the description of the cache somedata';
```

## Getting a cache instance

```php
$cache = cache::make('mod_myplugin', 'somedata');
```

`cache::make()` is a factory; the returned object extends `cache` (implements `cache_loader`), normally one of `cache_application`, `cache_session`, or `cache_request` depending on the mode.

Ad-hoc cache (no definition; default settings only; only for insignificant caches used in a single place):

```php
$cache = cache::make_from_params(cache_store::MODE_APPLICATION, 'mod_myplugin', 'mycache');
```

Prefer a real definition unless you have a good reason not to.

## get / set / delete

Keys must be `int` or `string`. Values can be anything serializable.

```php
$result = $cache->set('key', 'value');   // true on success.
$data   = $cache->get('key');            // stored value, or false on miss.
$result = $cache->delete('key');         // true on success.
```

Multiple pairs in one transaction:

```php
$result = $cache->set_many([
    'key1' => 'data1',
    'key3' => 'data3',
]); // Number of pairs successfully set.

$result = $cache->get_many(['key1', 'key2', 'key3']);
// ['key1' => 'data1', 'key2' => false, 'key3' => 'data3']

$result = $cache->delete_many(['key1', 'key3']); // Number deleted.
```

## Full definition options (with defaults)

```php
$definitions = [
    // The area name is the key; component/plugin comes from the file location.
    'area' => [
        'mode' => cache_store::MODE_*,
        'simplekeys' => false,
        'simpledata' => false,
        'requireidentifiers' => ['ident1', 'ident2'],
        'requiredataguarantee' => false,
        'requiremultipleidentifiers' => false,
        'requirelockingread' => false,
        'requirelockingwrite' => false,
        'requirelockingbeforewrite' => false,
        'maxsize' => null,
        'overrideclass' => null,
        'overrideclassfile' => null,
        'datasource' => null,
        'datasourcefile' => null,
        'staticacceleration' => false,
        'staticaccelerationsize' => null,
        'ttl' => 0,
        'mappingsonly' => false,
        'invalidationevents' => ['event1', 'event2'],
        'canuselocalstore' => false,
        'sharingoptions' => cache_definition::SHARING_DEFAULT,
        'defaultsharing' => cache_definition::SHARING_DEFAULT,
    ],
];
```

### Requirements

- `simplekeys` (bool): true if keys are only `a-zA-Z0-9_`. Keys are then not hashed before hitting the store — better performance.
- `simpledata` (bool): true if data is scalar or array of scalars — values stored as-is instead of serialised.
- `requireidentifiers` (array): identifiers that must be provided when the cache is created.
- `requiredataguarantee` (bool): only use stores that guarantee data remains available once set.
- `requiremultipleidentifiers` (bool): only use stores supporting multiple identifiers.
- `requirelockingread` (bool): gain a lock before reading. Avoid unless absolutely required.
- `requirelockingbeforewrite` (bool): a lock must be gained and held during expensive computation before writing (prevents cache stampedes). After gaining the lock, code must check the cache wasn't already updated by another process. Used by course `modinfo` caches.

### Modifiers

- `staticacceleration` (bool): instantiate the cache structure once per request and keep an in-memory array of items set/retrieved. Use when the same items are fetched many times per request. Costs memory.
- `staticaccelerationsize` (int): limits the number of items in that static array.
- `ttl` (int, seconds): discouraged — prefer event-driven invalidation; not all stores support it natively.
- `maxsize` (int): advisory maximum number of entries; stores may not enforce it.
- `canuselocalstore` (bool): cache can safely be local to each frontend in a cluster (avoids latency to a shared cache server). Code must be written carefully for this (see key strategies below). Example: `purify_html()` in `lib/weblib.php`.

### Data source

- `datasource` (string) / `datasourcefile` (string): a class implementing `cache_data_source`. On a miss the data source loads the value, the cache stores and returns it — get requests essentially never fail.
- Real example (5.2, MDL-86493): the `mod_quiz:quiz_overrides` cache is keyed by `quizid_userid` with datasource `\mod_quiz\cache\quiz_overrides_cache`; callers go through `\mod_quiz\local\quiz_overrides_cache_manager` (`get_overrides()`, `purge_for_user()`, `purge_for_group()`, ...), and hook callbacks on `\core_group\hook\after_group_membership_added` / `_removed` purge it when group membership changes.

### Loader override (avoid)

- `overrideclass` / `overrideclassfile`: custom cache loader class; must implement `cache_loader` and extend the default loader for its mode. "Should not be done. Ever," unless you have a very good reason.

### Misc

- `invalidationevents` (array): arbitrary strings (NOT Events API events). Trigger with:

  ```php
  cache_helper::purge_by_event('changesincoursecat');
  ```

  This marks all caches listing that event as invalid without callers knowing which caches are affected.
- `mappingsonly` (bool): only mapped store(s) used, never the default mode store. Super advanced.
- `sharingoptions` / `defaultsharing` (int): sharing options; leave at system default unless there's a specific reason.

## Cache key strategies for localized (per-frontend) caches

With `canuselocalstore` (or any multi-layer setup) each frontend may have its own store, so you cannot purge remotely. Instead make changed content resolve to something the local store does not have. Not every cache is a good candidate to localize; doing it for the wrong reasons can hurt.

### Versioned caches (recommended)

When the data has a "version" — a numeric time value or an incremented version number stored in the database — use the cache's `set_versioned()` and `get_versioned()` functions:

- They **must be used as a pair**: data stored with `set_versioned()` cannot be read with `get()`, and data stored with `set()` cannot be read with `get_versioned()`.
- `get_versioned()` takes a required version integer and only returns cached data whose version is equal to or newer than it. Optionally it can also report the actual version returned, if newer than requested.
- Multi-layer behaviour: if the right version is in the local cache it is returned without touching the shared cache; otherwise it is fetched from the shared cache and copied locally. `set_versioned()` writes to both local and shared caches. A new version is therefore usually computed on a single server (use locking if you must rule out races).
- Only one version is stored per key at any time, which matters for large items such as course modinfo.

The 5.3 docs describe the behaviour but give no full method signatures; check the methods on the cache loader class in core before relying on parameter order.

### Older approaches

- **Version number as key suffix**: works, but stores every version, using more cache space when items are large or change often. Works best with a store that has LRU garbage collection.
- **Version number inside the value**: avoids duplicates but is **not recommended** — it is hard to guarantee correct data on multi-level caches. Use the versioned functions instead.
- **Incrementing version numbers**: use a DB transaction or a lock so the same number is never issued twice.
- **Timestamp as version number**: no locking needed but two processes may clash in the same second; see `increment_revision_number()`. Theme revisions look like this but `theme_get_next_revision()` actually guarantees a unique integer close to the current time.
- **Content hash as key** (`sha1`/`sha256` of the content or of its dependencies): guaranteed unique and fully distributed. Good for nested fragments: hash the child hashes to key the combined item ("Russian Doll" caching), so changing one leaf only regenerates its branch.

MUC handles primary (local) + final (shared) layered caches automatically: a local miss falls back to the shared store and copies the value back; on generation the value is written to both. Stale key-suffixed items are never purged automatically — rely on store TTL/LRU garbage collection or the versioned functions.
