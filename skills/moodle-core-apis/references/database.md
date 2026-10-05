# Moodle 5.3 Database APIs (DML, DDL, XMLDB)

Compact reference for reading/writing data via `$DB` (DML), altering schema in upgrades via the database manager (DDL), and defining tables with install.xml / the XMLDB editor. Distilled from the official Moodle 5.3 developer docs and the 5.0–5.3 `UPGRADING.md` notes.

## What changed since 4.5 (database-relevant)

- **Oracle removed in 5.0** (MDL-83172): Oracle support was dropped from the LMS. Do not write Oracle-specific workarounds; the 5.3 DML pages still list Oracle among supported servers/drivers, but that text is stale.
- **Supported databases for 5.3**: PostgreSQL 17, MySQL 8.4, MariaDB 11.4, SQL Server 2019 (minimum versions). PHP 8.3 minimum.
- **Read replica terminology** (5.0, MDL-71257): `moodle_database::want_read_slave()` and `perf_get_reads_slave()` are deprecated in favour of renamed versions substituting `replica` for `slave`; the trait `moodle_read_slave_trait` is deprecated in favour of the functionally identical `moodle_read_replica_trait` (`master` becomes `primary`).
- **`{user_preferences}.value` is TEXT** (5.0, MDL-46739): it used to be CHAR. Any WHERE/JOIN condition on it must use `$DB->sql_compare_text()` (see `sql_compare_text` below).
- **XMLDB editor `rename_field` code** (5.2, MDL-87158): the generated `rename_field(...)` upgrade step now checks for field existence first, so the step can run multiple times.
- **Code location** (5.1, MDL-83424): core and plugins moved under `public/` in a git checkout (e.g. `public/lib/dml/moodle_database.php`, `public/mod/forum/db/install.xml`). `$CFG->dirroot` still points at the code directory, so `"$CFG->dirroot/mod/forum/db/..."` paths keep working.

## General concepts

- Access data **exclusively** via the `$DB` global, an instance of `moodle_database`. It is set up automatically when config.php is included. Inside functions, use `global $DB;`.
- **Table prefix**: never hardcode `mdl_`. `$table` parameters take the unprefixed name (`'user'`). In custom SQL, wrap table names in curly braces — they are converted to the real prefixed name automatically:

```php
$user = $DB->get_record_sql('SELECT COUNT(*) FROM {user} WHERE deleted = 1 OR suspended = 1;');
```

- **Conditions**: `$conditions` parameters are arrays of `fieldname => fieldvalue`, combined with logical `AND`:

```php
$user = $DB->get_record('user', ['firstname' => 'Martin', 'lastname' => 'Dougiamas']);
```

### Placeholders

`$params` arrays fill placeholders in SQL. Placeholders prevent SQL injection and quoting problems and are required for cross-db code. Two styles:

- Question marks (`SQL_PARAMS_QM`):

```php
$DB->get_record_sql(
    'SELECT * FROM {user} WHERE firstname = ? AND lastname = ?',
    ['Martin', 'Dougiamas']
);
```

- Named (`SQL_PARAMS_NAMED`) — names **must be unique** even if the value is the same; repeat the value under distinct names if needed:

```php
$DB->get_record_sql(
    'SELECT * FROM {user} WHERE firstname = :firstname AND lastname = :lastname',
    ['firstname' => 'Martin', 'lastname' => 'Dougiamas']
);
```

### Strictness

Some methods accept a `$strictness` parameter:

- `MUST_EXIST` — record must exist and be unique; throws `dml_missing_record_exception` if not found, `dml_multiple_records_exception` if multiple match.
- `IGNORE_MISSING` (default) — returns `false` if not found; debugging message if multiple found.
- `IGNORE_MULTIPLE` — not recommended; silently returns the first of multiple matches.

## Getting a single record

```php
public function get_record($table, array $conditions, $fields = '*', $strictness = IGNORE_MISSING);
public function get_record_select($table, $select, array $params = null, $fields = '*', $strictness = IGNORE_MISSING);
public function get_record_sql($sql, array $params = null, $strictness = IGNORE_MISSING);
```

## Getting multiple records (array of objects)

The returned array is indexed by the **first column** of the query — always select `id` first (make `id` the first column/primary key when designing tables).

```php
public function get_records($table, array $conditions = null, $sort = '', $fields = '*', $limitfrom = 0, $limitnum = 0);
public function get_records_select($table, $select, array $params = null, $sort = '', $fields = '*', $limitfrom = 0, $limitnum = 0);
public function get_records_sql($sql, array $params = null, $limitfrom = 0, $limitnum = 0);
// Records where $field matches one of $values:
public function get_records_list($table, $field, array $values, $sort, $fields = '*', $limitfrom, $limitnum);
```

## Key/value pairs (first two columns as associative array)

```php
public function get_records_menu($table, array $conditions = null, $sort = '', $fields = '*', $limitfrom = 0, $limitnum = 0);
public function get_records_select_menu($table, $select, array $params = null, $sort = '', $fields = '*', $limitfrom = 0, $limitnum = 0);
public function get_records_sql_menu($sql, array $params = null, $limitfrom = 0, $limitnum = 0);
```

## Counting and existence

```php
public function count_records($table, array $conditions = null);
public function count_records_select($table, $select, array $params = null, $countitem = "COUNT('x')");
public function count_records_sql($sql, array $params = null);

public function record_exists($table, array $conditions = null);
public function record_exists_select($table, $select, array $params = null);
public function record_exists_sql($sql, array $params = null);
```

## Single field values

```php
public function get_field($table, $return, array $conditions, $strictness = IGNORE_MISSING);
public function get_field_select($table, $return, $select, array $params = null, $strictness = IGNORE_MISSING);
public function get_field_sql($sql, array $params = null, $strictness = IGNORE_MISSING);

// Field values from multiple records, as an array:
public function get_fieldset(string $table, string $return, ?array $conditions = null);
public function get_fieldset_select($table, $return, $select, array $params = null);
public function get_fieldset_sql($sql, array $params = null);
```

## Writing data

```php
// Set one field on every matching record.
public function set_field($table, $newfield, $newvalue, array $conditions = null);
public function set_field_select($table, $newfield, $newvalue, $select, array $params = null);

// Insert; returns the new record's id.
public function insert_record($table, $dataobject, $returnid = true, $bulk = false);
// Fast bulk insert; inserted in order but NOT atomic — use a transaction if needed.
public function insert_records($table, $dataobjects);
// insert_record_raw(): rare cases where you must specify the id yourself.

// Update; $dataobject must have property "id" set.
public function update_record($table, $dataobject, $bulk = false);

// Delete.
public function delete_records($table, array $conditions = null);
public function delete_records_select($table, $select, array $params = null);
```

## execute — arbitrary SQL

Low-level; use only when no specialised method exists:

```php
public function execute($sql, array $params = null);
```

Danger: do NOT use `execute()` to change database structure — use the `database_manager` (DDL) methods instead.

## Recordsets

For large result sets, `get_records_*` loads everything into memory. Use `get_recordset_*` instead — they return an iterator. It is **absolutely important** to close the recordset after use:

```php
$rs = $DB->get_recordset(/* ... */);
foreach ($rs as $record) {
    // Do whatever you want with this record.
}
$rs->close();
```

You cannot test `$rs == true` / `!empty($rs)` to see whether records were found; use:

```php
if ($rs->valid()) {
    // The recordset contains some records.
}
```

```php
public function get_recordset($table, array $conditions = null, $sort = '', $fields = '*', $limitfrom = 0, $limitnum = 0);
public function get_recordset_select($table, $select, array $params = null, $sort = '', $fields = '*', $limitfrom = 0, $limitnum = 0);
public function get_recordset_sql($sql, array $params = null, $limitfrom = 0, $limitnum = 0);
public function get_recordset_list($table, $field, array $values, $sort, $fields = '*', $limitfrom, $limitnum);
```

## Delegated transactions

Nesting is emulated since 2.0; outer levels control inner ones. Not all engines support transactions (e.g. MyISAM), and some subsystems (e.g. messaging) cannot roll back — transactions are an emergency protection of data consistency, not expected code flow. Code must not rely on rollback happening.

```php
global $DB;
try {
    $transaction = $DB->start_delegated_transaction();
    $DB->insert_record('foo', $object);
    $DB->insert_record('bar', $otherobject);
    // Assuming both inserts work, we get to the following line.
    $transaction->allow_commit();
} catch (Exception $e) {
    $transaction->rollback($e);
}
```

Rules:

- Catching exceptions is optional, but if you catch, you **must** call `$transaction->rollback($e)` — it re-throws the exception.
- Once any level is marked for rollback, nothing can un-mark it; a level marked `allow_commit()` can still be rolled back by an outer level.
- Unfinished/unbalanced/double-finished transactions throw `transaction_exception` and the DB rolls back.
- To forbid transactions around sensitive code: `$DB->transactions_forbidden();` (throws if a transaction is active).

## Cross-DB SQL helpers

Moodle 5.3 supports MySQL, MariaDB, PostgreSQL and MS SQL Server (Oracle was removed in 5.0; the upstream DML page still lists it). Never write DB-specific syntax; build fragments with these `$DB` methods:

```php
sql_bitand($int1, $int2); sql_bitor($int1, $int2); sql_bitxor($int1, $int2); sql_bitnot($int1);
sql_null_from_clause();          // empty FROM clause some DBs require
sql_ceil($fieldname);
sql_modulo($int1, $int2);
sql_length($fieldname);          // length in characters
sql_position($needle, $haystack);// named placeholders required if both use placeholders
sql_substr($expr, $start, $length = false);
sql_cast_char2int($fieldname, $text = false);
sql_cast_char2real($fieldname, $text = false);
sql_cast_to_char(string $field);
sql_order_by_text($fieldname, $numchars = 32);  // bad for performance, avoid
sql_order_by_null(string $fieldname, int $sort = SORT_ASC);
sql_fullname($first = 'firstname', $last = 'lastname');
sql_isempty($tablename, $fieldname, $nullablefield, $textfield);
sql_isnotempty($tablename, $fieldname, $nullablefield, $textfield);
sql_group_concat(string $field, string $separator = ', ', string $sort = '');
sql_concat_join($separator = "' '", $elements = []);
sql_intersect($selects, $fields);
sql_equal($fieldname, $param, $casesensitive = true, $accentsensitive = true, $notequal = false);
```

### sql_like / sql_like_escape

```php
$DB->sql_like($fieldname, $param, $casesensitive = true, $accentsensitive = true, $notlike = false, $escapechar = '\\');
```

```php
// Hard-coded literal pattern:
$likeidnumber = $DB->sql_like('idnumber', ':idnum');
$DB->get_records_sql(
    "SELECT id, fullname FROM {course} WHERE {$likeidnumber}",
    ['idnum' => 'DEMO-%']
);

// User-submitted value: escape it so % and _ are literals.
$search = required_param('search', PARAM_RAW);
$likefullname = $DB->sql_like('fullname', ':fullname');
$DB->get_records_sql(
    "SELECT id, fullname FROM {course} WHERE {$likefullname}",
    ['fullname' => '%' . $DB->sql_like_escape($search) . '%']
);
```

### sql_compare_text

Comparing a TEXT (clob) column with a string/VARCHAR needs this on some RDBMSes. Since 5.0 this includes `{user_preferences}.value` (core uses it in `\core\task\send_new_users_password_task::execute`):

```php
$comparedescription = $DB->sql_compare_text('description');
$comparedescriptionplaceholder = $DB->sql_compare_text(':description');
$todogroups = $DB->get_records_sql(
    "SELECT id FROM {group} WHERE {$comparedescription} = {$comparedescriptionplaceholder}",
    ['description' => 'TODO']
);
```

### sql_concat

`sql_concat(...)` — gotcha: any NULL argument makes the whole result NULL. Coalesce nullable arguments:

```php
// Wrong:  sql_concat('requiredfield', 'optionalfield');
// Right:  sql_concat('requiredfield', "COALESCE(optionalfield, '')");
```

### get_in_or_equal

Builds an `IN (...)` fragment (falls back to `=` for a single item):

```php
public function get_in_or_equal($items, $type = SQL_PARAMS_QM, $prefix = 'param', $equal = true, $onemptyitems = false);
```

```php
$statuses = ['todo', 'open', 'inprogress', 'intesting'];
[$insql, $inparams] = $DB->get_in_or_equal($statuses);
$bugs = $DB->get_records_sql("SELECT * FROM {bugtracker_issues} WHERE status $insql", $inparams);

// Named params:
[$insql, $params] = $DB->get_in_or_equal($contexts, SQL_PARAMS_NAMED, 'ctx');
$contextsql = "AND rc.contextid {$insql}";
```

### Regex

```php
if ($DB->sql_regex_supported()) {
    $select = 'content ' . $DB->sql_regex() . ' :pattern';   // sql_regex($positivematch = true, $casesensitive = false)
} else {
    $select = $DB->sql_like('content', ':pattern', false);
}
```

Word boundary markers: `sql_regex_get_word_beginning_boundary_marker()` / `sql_regex_get_word_end_boundary_marker()` (default `[[:<:]]` / `[[:>:]]`, `\\b` on MySQL 8.0.4+).

## Exceptions and debugging

DML code throws exceptions rather than returning false on error: `dml_connection_exception` (cannot connect), `dml_read_exception` (error reading), `dml_write_exception` (error writing). See also strictness exceptions above and `transaction_exception` below.

- `$DB->set_debug(true)` outputs the SQL and timing of every query (remove before submitting code).
- Query logging via `$CFG->dboptions` in config.php (native drivers log to a DB table):

```php
$CFG->dboptions = [
    'dbpersist' => 0,
    // 'logall' => true,   // Log all queries - developers only, high load.
    'logslow' => 5,        // Log queries slower than N seconds (float ok).
    'logerrors' => true,   // Log all error queries.
];
```

- Native drivers in 5.3: MySQLi, MariaDB, PostgreSQL, Microsoft SQL Server (the drivers page still lists Oracle; removed in 5.0).
- Prefer `get_course($id)` over `$DB->get_record('course', ...)` — the current/site course is probably already loaded.

## DDL: database manager (upgrade-only)

DDL functions are **restricted to install/upgrade processes**. All `$table`, `$field`, `$index` parameters are XMLDB objects; `$newname` parameters are plain strings. `*_exists()` return bool; any failure throws an exception and stops the upgrade. Always use the XMLDB editor to generate this PHP code.

```php
function xmldb_xxxx_upgrade($oldversion) {
    global $DB;
    // Load the DDL manager and xmldb API.
    $dbman = $DB->get_manager();
    // Your upgrade code goes here.
}
```

```php
// Tables.
$dbman->table_exists($table);
$dbman->create_table($table, $continue = true, $feedback = true);
$dbman->drop_table($table, $continue = true, $feedback = true);
$dbman->rename_table($table, $newname, $continue = true, $feedback = true);

// Fields.
$dbman->field_exists($table, $field);
$dbman->add_field($table, $field, $continue = true, $feedback = true);
$dbman->drop_field($table, $field, $continue = true, $feedback = true);
$dbman->change_field_type($table, $field, $continue = true, $feedback = true);
$dbman->change_field_precision($table, $field, $continue = true, $feedback = true);
$dbman->change_field_notnull($table, $field, $continue = true, $feedback = true);
$dbman->change_field_default($table, $field, $continue = true, $feedback = true);
$dbman->rename_field($table, $field, $newname, $continue = true, $feedback = true);

// Indexes.
$dbman->index_exists($table, $index);
$dbman->find_index_name($table, $index);
$dbman->add_index($table, $index, $continue = true, $feedback = true);
$dbman->drop_index($table, $index, $continue = true, $feedback = true);
```

## XMLDB editor and install.xml

Location: *Site administration > Development > XMLDB editor*. It edits the database-neutral `db/install.xml` file of each plugin. Never edit install.xml by hand.

Workflow for a new plugin:

1. Create a `db` directory under the plugin (e.g. `mod/mymodule/db`); the web server needs **write access** to it (otherwise the Create/Load links are not clickable).
2. In the XMLDB editor, find your `db` location and click **Create** (blank install.xml), then **Load**, then **Edit**.
3. Create/edit tables, fields, keys and indexes (add/edit/delete/move). Fields let you set name, type, length, decimals, null-ability, defaults; same for keys and indexes. Fields used by a key/index are non-editable until freed.
4. Click **Back** / **Back to Main**, then **Save**.

install.xml creates the tables at plugin installation (with version.php). In a 5.1+ git checkout the file sits under `public/` (e.g. `public/mod/mymodule/db/install.xml`); inside Moodle it is still `$CFG->dirroot . '/mod/mymodule/db/install.xml'`. For **subsequent structure changes**, write `db/upgrade.php` with an `xmldb_<component>_upgrade($oldversion)` function; get the change code by loading install.xml in the XMLDB editor and using the **View PHP Code** option, then copy/paste the generated block into the upgrade function. Since 5.2 the generated `rename_field()` block is wrapped in a field-existence check so it is safe to re-run.

### XMLDB conventions

- All names lowercase; tables/fields use only `a-z`, `0-9`, `_`. Max 53 chars for table names, 63 for fields (28/30 before Moodle 4.3 — keep the lower limits for plugins targeting older versions). Avoid SQL reserved words.
- Key/index names are the field names joined with `-`; exception: the primary key is always named `primary`.
- NULLs: don't make everything NOT NULL with a default empty string — the XMLDB structure must reflect reality.
- Foreign keys: define existing FKs properly; a field combination defined as FK gets its index created automatically — do not add one yourself.
- Unique keys: declare a UNIQUE KEY only if it is the target of a FK; otherwise use a unique index.
- Enum fields are deprecated since Moodle 2.0 — avoid them.
- Every XMLDB element (table, field, key, index, statement) accepts a comment — use it.
