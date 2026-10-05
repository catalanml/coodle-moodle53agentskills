# Upgrading a Plugin from Moodle 4.5 to 5.3

Migration guide for third-party plugin code written for Moodle 4.5 (LTS) that must run on Moodle 5.3 (LTS), crossing 5.0, 5.1, 5.2 and 5.3. Distilled from the four developer-update pages and the core `UPGRADING.md` notes for 5.0–5.3. Only APIs a plugin is likely to call are listed; for the full list read `UPGRADING.md` in each component directory of a 5.3 checkout (e.g. `public/course/format/UPGRADING.md`).

Terminology in the tables: **dep.** = deprecated (still works, emits debugging; normally removed in 6.0 or later), **removed** = gone (fatal/exception), **renamed** = new autoloaded name, old name still works for now.

## Part A — Must-do checklist

Work through these first; most plugins hit several.

1. **Platform.** PHP minimum 8.2 since 5.0 and **8.3 since 5.2** (8.4 supported); 64-bit only; `sodium` required; upgrades must start from **4.4 or later**. Oracle support removed in 5.0 (MDL-83172) — delete Oracle-specific SQL branches. MySQL 8.4, MariaDB 11.4, PostgreSQL 17, SQL Server 2019 minimums in 5.3.
2. **`public/` restructure (5.1, MDL-83424).** In a git checkout plugins live at `public/<type>/<name>` (e.g. `public/mod/forum`). `$CFG->dirroot` still points at the code directory, so `require_once($CFG->dirroot . '/mod/forum/lib.php')` is unchanged; new read-only `$CFG->root` is the install root. Web server DocumentRoot must point at `public/` (`$CFG->wwwrootendsinpublic` removed in 5.2, MDL-87072). Fix CI scripts, symlinks and docs that assume plugin paths at repository root.
3. **Bootstrap 5 (5.0, MDL-75669).** Theme_boost ships Bootstrap 5.3. A BS4 compatibility layer (jQuery plugins, BS4 SCSS helpers, `theme_boost/bs4-compat` data-attribute rewriter) works until **6.0** — migrate markup now:

   | BS4 | BS5 |
   | --- | --- |
   | `data-toggle`, `data-target`, `data-html` | `data-bs-toggle`, `data-bs-target`, `data-bs-html` |
   | `.ml-*`/`.mr-*`/`.pl-*`/`.pr-*`, `.float-left/right`, `.text-left/right`, `.border-left/right` | `.ms-*`/`.me-*`/`.ps-*`/`.pe-*`, `-start`/`-end` variants |
   | `.sr-only`, `.sr-only-focusable` | `.visually-hidden`, `.visually-hidden-focusable` |
   | `.badge-*`, `.badge-pill` | `.text-bg-*`, `.rounded-pill` |
   | `.font-weight-*`, `.font-italic` | `.fw-*`, `.fst-italic` |
   | `.no-gutters`, `.close`, `.rounded-sm/lg` | `.g-0`, `.btn-close`, `.rounded-1/3` |
   | `.dropdown-menu-left/right` | `.dropdown-menu-start/end` |
   | `.custom-select`, `.custom-check`, `.custom-switch`, `.custom-range` | `.form-select`, `.form-check`, `.form-check.form-switch`, `.form-range` |
   | `.form-group`, `.form-inline`, `.media`, `.card-deck`, `.input-group-append/prepend` | `.mb-3`, `.d-flex .flex-wrap .align-items-center`, flex utilities, `.row .row-cols-*`, direct children of `.input-group` |
   | `media-breakpoint-down(sm)` (= below md) | `media-breakpoint-down(md)` — argument is now the breakpoint itself |
   | `theme-color-level('primary', 1)` | `shift-color($primary, 10%)` |
   | jQuery `$('#x').dropdown('toggle')` | `import {Dropdown} from 'bootstrap'; new Dropdown('#x').toggle();` |

   Since 5.3 (MDL-88766) AMD code must not import `theme_boost/bootstrap/*`; use `import {Tooltip} from 'bootstrap'` (5.3+) or `from 'theme_boost/index'` (5.2 and earlier, supported until 7.0).
4. **PHPUnit 11.4 (5.0, MDL-83468).** Data providers must be `static` (no DB calls or object side effects inside); keyed provider arrays must match test parameter names; `MockBuilder::setMethods()` removed (use `onlyMethods()`/`addMethods()`); `assertTag`, `getName`, `isInIsolation` removed; `withConsecutive`, `willReturnConsecutive`, `onConsecutive` removed with no direct replacement (MDL-81308); `assertEqualsCanonicalizing` needs consistently sorted data. Since 5.1 `failOnDeprecation` and `failOnWarning` are on (MDL-86311). See `general/development/tools/phpunit/upgrading-11`.
5. **Test base-class moves.** `\externallib_advanced_testcase` → `\core_external\tests\externallib_testcase` (5.1, MDL-86283); `core_reportbuilder_testcase` → `\core_reportbuilder\tests\core_reportbuilder_testcase` (5.0); `\mod_assign\privacy\provider_test` is final → extend `\mod_assign\tests\provider_testcase` (5.0); quiz walkthrough → `\mod_quiz\tests\attempt_walkthrough_testcase` (5.0); `\phpunit_util`, `\phpunit_event_sink`, `\phpunit_message_sink`, `\testing_util` → `\core\test\phpunit\…` / `\core\test\testing_util` (5.2, old names kept until 6.0). Behat/Gherkin 4.11: backslashes in `.feature` files must be escaped (5.0, MDL-83848).
6. **Subplugins (5.0, MDL-83705/MDL-83703).** `db/subplugins.php` support removed. `db/subplugins.json` needs a `subplugintypes` object with paths relative to the plugin root; keep `plugintypes` (paths from the Moodle root) too if you still support 4.5, and keep both in sync:

   ```json
   {
       "plugintypes": {"quizaccess": "mod/quiz/accessrule"},
       "subplugintypes": {"quizaccess": "accessrule"}
   }
   ```
7. **Activity modules.**
   - 5.0 (MDL-83872): a course "Activities" overview page replaces per-module `index.php`; add `mod_PLUGINNAME\courseformat\overview` extending `core_courseformat\activityoverviewbase` (file `classes/courseformat/overview.php`).
   - 5.0: `\core\event\course_module_instances_list_viewed` removed (MDL-84593); 5.1 deprecates per-module `course_module_instance_list_viewed` events in resource modules → `\core\event\course_resources_list_viewed` (MDL-84632).
   - 5.1 (MDL-85598): optional `FEATURE_MOD_OTHERPURPOSE` (secondary activity-chooser category) alongside mandatory `FEATURE_MOD_PURPOSE`.
   - 5.2 (MDL-87117): optional lang strings `modulename_summary` and `modulename_tip` beside `modulename_help` (structure the help as "Key features" / "Ways to use it").
   - 5.3 (MDL-83231): remove `FEATURE_GROUPMEMBERSONLY`.
   - 5.0 (MDL-83892): `cm_info::get_instance_record()` returns the cached activity record — avoid an extra `$DB` call.
8. **Course formats.**
   - Formats that do not override `supports_components()` (or return `false`) still use the 3.11 YUI editor: deprecated in 5.0, **will not work in 6.0** (MDL-83527, MDL-82341). Follow `apis/plugintypes/format/migration`.
   - Activity/section menus: extend `core_courseformat\output\local\content\cm\controlmenu` / `…\section\controlmenu`; return `action_menu_link` objects, not arrays (5.0).
   - Webservice `core_courseformat_create_module` → `core_courseformat_new_module` (section id, not number); `data-action="addModule"` → `data-action="newModule"` plus `data-sectionid` (5.0, MDL-83469).
   - Activity chooser moved from core_course to core_courseformat (5.1, MDL-85597/MDL-86337); `maxsections` and `get_max_sections()` deprecated, removal in 6.0 (5.1, MDL-84291).
   - Course/module manipulation functions → `core_courseformat\local\cmactions` / `sectionactions` (5.2).
   - Linear navigation is opt-in via `uses_linear_navigation()` (5.3).
9. **Routes in `route\api` (5.3, MDL-89710).** Each must declare `#[\core\router\scope\scopeset(...)]` or `#[\core\router\scope\unscoped_resource]`; a core PHPUnit test fails otherwise.
10. **Removed from core — plugins that depended on them break:** Atto editor and subplugins, mod_chat, mod_survey, auth_cas, auth_mnet, enrol_mnet, block_mnet_hosts, portfolio_mahara, mlbackend_php (5.0); block_section_links (5.1); block_activity_modules, `qtype_random`, tool_moodlenet and MoodleNet outbound sharing (5.2); theme_classic (5.3). The `mnetservice` plugin type is deprecated (5.0).
11. **Database quirks.** `{user_preferences}.value` is now TEXT (5.0, MDL-46739): conditions on it need `$DB->sql_compare_text()`. `arg_separator.output` now defaults to `&` (5.2, MDL-71368): pass `'&amp;'` explicitly to `http_build_query()` if you relied on the old default. `\core\persistent::get_records()` results are keyed by record id (5.2, MDL-79574).
12. **Dark colour mode (5.3, MDL-68037).** Plugin CSS must not assume a light page; use `--bs-body-bg`/`--bs-body-color`/… custom properties or `bg-body`/`text-body`/`text-bg-*` utilities, scope exceptions with `[data-bs-theme="dark"]`.

## Part B — Changes by version

### Moodle 5.0

**core / output**

| Old | New / action | Since |
| --- | --- | --- |
| `core_renderer::sr_text()` | `core_renderer::visually_hidden_text()` | 5.0 dep. (MDL-81825) |
| `new single_button(..., true)` (boolean 4th "primary" arg) | 4th parameter is now a string, not a boolean | 5.0 removed (MDL-75875) |
| `action_menu::set_alignment()`, `set_constraint()`, `do_not_enhance()` | remove calls | 5.0 removed (MDL-83765) |
| `core_renderer::activity_information()`, `core_course\output\activity_information`, `core_course_renderer::render_activity_information()` | none listed | 5.0 removed (MDL-78926) |
| `task_base::is_blocking()`, `set_blocking()` | remove | 5.0 removed (MDL-81509) |
| `\core\encryption::METHOD_OPENSSL`, `OPENSSL_CIPHER`, `is_sodium_installed()` | sodium only | 5.0 removed (MDL-78869) |
| `moodle_database::want_read_slave()`, `perf_get_reads_slave()`, trait `moodle_read_slave_trait` | `*_replica` names, `moodle_read_replica_trait` | 5.0 dep. (MDL-71257) |
| `imagecopybicubic()` | `imagecopyresampled()` | 5.0 dep. (MDL-84449) |
| WS `core_output_load_fontawesome_icon_map` | `core_output_load_fontawesome_icon_system_map` | 5.0 removed (MDL-84036) |
| `course/view.php` externservercourse override | hook `\core_course\hook\before_course_viewed` | 5.0 dep. (MDL-83764) |

New: `\core\output\local\collapsable_section` (template `core/local/collapsable_section`), `core\output\local\properties\iconsize` enum, `\core\output\task_indicator`, `url::get_slashargument()`, `core_text::trim_ctrl_chars()`, hooks `after_role_switched` and `after_cm_completion_updated`.

**core_calendar** — human date renderers `humandate` / `humantimeperiod` (MDL-83873)

| Old | New | Since |
| --- | --- | --- |
| `calendar_day_representation()`, `calendar_time_representation()`, `calendar_format_event_time()` | `humandate` / `humantimeperiod` output classes | 5.0 dep. |
| `calendar_sub_month()` / `calendar_add_month()` | `\core_calendar\type_factory::get_calendar_instance()->get_prev_month()` / `get_next_month()` | 5.0 dep. |
| `calendar_get_courselink()`, `calendar_events_by_day()` | none | 5.0 dep. |
| `calendar_top_controls()`, `calendar_get_link_previous/next()`, `prepare_for_view()`, `calendar_add_event_metadata()`, `core_calendar_renderer::event()` | none | 5.0 removed |

**core_course / core_courseformat**

| Old | New | Since |
| --- | --- | --- |
| `course_get_cm_edit_actions()` | extend `core_courseformat\output\local\content\cm\controlmenu` | 5.0 dep. (MDL-83527) |
| Arrays in section/cm control menus | `action_menu_link` objects | 5.0 dep. |
| `format_topics` section controlmenu `get_highlight_control()` / `get_course_url()` | `get_section_highlight_item()` / `$this->format->get_update_url()` | 5.0 dep. |
| WS `core_course_edit_module`, `core_course_edit_section` | `core_courseformat_update_course` | 5.0 dep. (MDL-82342) |
| WS `core_course_get_module` | fragment API, component `core_courseformat`, fragment `cmitem` | 5.0 dep. |
| `course_format_ajax_support()` | `course_get_format($course)->supports_ajax()` | 5.0 dep. (MDL-82351) |
| `base::get_non_ajax_cm_action_url()`; GET actions in `course/mod.php`/`course/view.php` (indent, duplicate, hide, show, stealth, delete, groupmode, marker) | `get_update_url()`; `course/format/update.php` (same params as `core_courseformat_course_update`) | 5.0 dep. (MDL-82767) |
| state action `section_move`, `base::get_section_number()`/`set_section_number()` | `section_move_after`, `get_sectionnum()`/`set_sectionnum()` | 5.0 final dep. (MDL-80116) |
| `core_courseformat_create_module`, state mutation `addModule` | `core_courseformat_new_module`, `newModule` | 5.0 dep. (MDL-83469) |
| `section\availability::availability_info()` | `get_availability_data()` | 5.0 removed |
| Reactive component init by bare id `'id'` | selector `'#id'` | 5.0 dep. (MDL-83339) |
| Template ids `#collapssesection{{num}}`, `#coursecontentcollapse{{num}}` | `#collapsesectionid{{id}}`, `#coursecontentcollapseid{{id}}` | 5.0 changed |

New: `base::get_generic_section_name()` (use instead of a direct `get_string()` for the format's section name), `base::invalidate_all_session_caches()`, hook `after_course_content_updated`, `core_course\output\activity_icon`. Non-ajax activity deletion now always uses the recycle bin.

**core_question / mod_quiz** (question banks are now activity-module based; course and category contexts no longer hold question categories — MDL-71378)

| Old | New | Since |
| --- | --- | --- |
| `question_make_default_categories()` | `question_get_default_category()` (can create) | 5.0 dep. |
| `question_delete_course()`, `question_delete_course_category()`, `question_type::generate_test()` | none | 5.0 dep. |
| `mod_quiz\question\bank\qbank_helper::get_version_options()` | `core_question\local\bank\version_options::get_version_options()` | 5.0 dep. |
| `action_column_base`, `menu_action_column_base`, `menuable_action`, `view::print_choose_category_message()` and other 4.x bank view methods | none listed | 5.0 removed (MDL-78090) |
| `quiz_attempt::process_finish()` | `process_submit()` + `process_grade_submission()` (new states NOT_STARTED, SUBMITTED) | 5.0 dep. (MDL-68806) |
| WS `mod_quiz_get_user_attempts` | `mod_quiz_get_user_quiz_attempts` | 5.0 dep. |
| `quiz_delete_override()`, `quiz_delete_all_overrides()` | none listed | 5.0 removed |

Qtype backup/restore (MDL-83541): restore now hashes question data to detect duplicates. Override `restore_qtype_plugin::convert_backup_to_questiondata()` if your qtype adds backup data, `define_excluded_identity_hash_fields()` if it has its own tables with id fields, and `remove_excluded_question_data()` for data present on only one side.

**Other components**

| Old | New | Since |
| --- | --- | --- |
| `enrol_self_enrol_form`, `enrol_guest_enrol_form` | `enrol_self\form\enrol_form`, `enrol_guest\form\enrol_form` | 5.0 dep. |
| `enrol_plugin::update_communication()`; `enrol_self_plugin::get_welcome_email_contact()` | none; `enrol_plugin::get_welcome_message_contact()` | 5.0 removed |
| `enrol_page_hook()` raw HTML | renderable `\core_enrol\output\enrol_page`, forms in a modal | 5.0 recommended |
| `badges/lib/bakerlib.php` class | `core_badges\png_metadata_handler` | 5.0 moved |
| `print_graded_users_selector()` | none | 5.0 dep. |
| `grade_structure::get_element_type_string/get_element_header/get_element_icon/get_activity_link()`, `grade_helper::get_lang_string()` | none | 5.0 removed |
| `user_preference_allow_ajax_update`, `M.util.set_user_preference`, `lib/ajax/setuserpref.php` | `core_user/repository` JS module | 5.0 removed |
| `repository::get_file_size()` | none | 5.0 removed |
| report entity `get_default_table_aliases()` | `get_default_tables()` (abstract) | 5.0 removed |
| `assign_course_index_summary` | `mod_assign\course\overview` | 5.0 dep. |
| `theme_boost` SCSS deprecated in 4.4; `$activity-icon-*-filter` | removed; `$activity-icon-{administration,assessment,collaboration,communication,content,interactivecontent}-bg` | 5.0 |

New: `file_clear_draft_area()`; `grade_regrade_final_grades(..., async)`; hook `after_user_enrolled` has `roleid`; `assign_plugin::settings_validation()`; `\core_customfield\external\field_data_exporter`; `editor_tiny_get_configuration` WS (TinyMCE subplugins implement `plugin_with_configuration_for_external`); AI action settings forms no longer call `$this->add_action_buttons()`.

### Moodle 5.1

**core** — autoloaded renames (old names still work unless noted)

| Old | New | Since |
| --- | --- | --- |
| `\global_navigation`, `\settings_navigation`, `\navigation_node`, `\navbar`, `\navigation_node_collection`, `\navigation_cache` (+ `_for_ajax`, `navigation_json`, `breadcrumb_navigation_node`) | `\core\navigation\…` same short name | 5.1 renamed (MDL-82159) |
| `NAVIGATION_CACHE_NAME`, `NAVIGATION_SITE_ADMIN_CACHE_NAME` | `\core\navigation\navigation_node::CACHE_NAME`, `::SITE_ADMIN_CACHE_NAME` | 5.1 dep. |
| `\cm_info`, `\cached_cm_info`, `\section_info`, `\course_modinfo` | `\core_course\cm_info`, `\core_course\cached_cm_info`, `\core_course\section_info`, `\core_course\modinfo` | 5.1 renamed (MDL-86155) |
| `MAX_MODINFO_CACHE_SIZE`; `cm_info::$extra`, `::$score` | class constant; now emit debugging | 5.1 dep. |
| `ajax_capture_output()`, `ajax_check_captured_output()` | `\core\ajax::capture_output()`, `::check_captured_output()` (no need to include `lib/ajax/ajaxlib.php`) | 5.1 (MDL-86168) |
| `\core_xml_parser`, `\xml_format_exception`, `xmlize()` | `\core\xml_parser`, `\core\exception\xml_format_exception` | 5.1 (MDL-86256) |
| `\comment`, `\comment_exception`, `comment/locallib.php` `comment_manager`; `comment/lib.php` | `\core_comment\manager`, `\core_comment\comment_exception`; stop including `comment/lib.php` (removed 6.0) | 5.1 (MDL-86254) |
| `\course_request` | `\core_course\course_request` | 5.1 renamed |
| `file_encode_url()` | `\core\url` factory methods | 5.1 dep. (MDL-31071) |
| checkbox-toggleall templates `-master`, `-master-button`, `-slave`; `data-toggle=master/slave`; `updateSlavesFromMasterState()` | `-toggler`, `-toggler-button`, `-target`; `data-toggle=toggler/target`; `updateTargetsFromTogglerState()` | 5.1 dep. (MDL-79756) |
| `core\hook\manager::is_deprecated_plugin_callback()` | `get_hooks_deprecating_plugin_callback()` | 5.1 removed |
| `core_useragent::get_device_type_theme()`, `get_device_type_cfg_var_name()`, `get_device_type_list()` | none | 5.1 removed |
| `toggle.mustache` `dataattributes` | `extraattributes` | 5.1 dep. |
| Plugin type names via other strings | lang strings `type_<type>` and `type_<type>_plural` required | 5.1 changed (MDL-84948) |

New: `\core\deprecation::emit_deprecation([self::class, __FUNCTION__])` (with `#[\core\attribute\deprecated]`, whose `$replacement` may now be omitted); `core\output\externable` interface; `core/page_title` JS module.

**core_course / core_courseformat** (activity chooser now in core_courseformat — MDL-80295, MDL-86337)

| Old | New | Since |
| --- | --- | --- |
| `core_course_renderer::course_section_add_cm_control()` | `section_add_cm_controls()` | 5.1 dep. |
| `core_course_renderer::course_activitychooser()` | `section_renderer::add_cm_controls()` | 5.1 dep. |
| `core_course\output\activitychooserbutton` (int section number) | `core_courseformat\output\local\activitychooserbutton` (pass a `section_info`) | 5.1 moved/dep. |
| Templates `core_course/activitychooser`, `core_course/activitychooserbutton`, `core_course/local/activitychooser/*` | `core_courseformat/activitychooser`, `core_courseformat/local/content/activitychooserbutton`, `core_courseformat/local/activitychooser/*` | 5.1 moved |
| WS `core_course_get_course_content_items`; JS `getModulesData`/`activityModules` | `core_courseformat_get_section_content_items`; `getSectionModulesData`/`sectionActivityModules` | 5.1 dep. |
| `custom_chooser_footer` AMD calling `modal.setFooter(...)` | remove the call; footer managed by the chooser dialogue | 5.1 changed |
| `maxsections` setting, `base::get_max_sections()`, "numsections" increment/decrement | own format setting (see format_weeks) | 5.1 dep., removed 6.0 |
| `course/view.php?duplicatesection`, `course/changenumsections.php` | `course/format/update.php` (e.g. action `section_duplicate`) | 5.1 dep. |
| setting `activitychoosertabmode` | none | 5.1 removed |

New: `main_activity_interface` (single-activity formats), `overviewfactory::activity_has_overview_integration()`, `activityoverviewbase::needs_filtering_by_groups()` / `get_groups_for_filtering()` / `has_error()`, `overviewaction` and `overviewdialog` outputs, `modinfo::get_instance_of()` and `modinfo::sort_cm_array()` (replacing single-instance uses of `get_course_and_cm_from_instance()`).

**core_question / mod_quiz**

| Old | New | Since |
| --- | --- | --- |
| `mod/quiz/attemptlib.php`, `accessmanager.php`, `accessmanager_form.php`, `accessrule/accessrulebase.php`, `renderer.php`, `cronlib.php`, `override_form.php`, `report/attemptsreport*.php`, `report/default.php` | autoloaded classes (e.g. access rules extend `mod_quiz\local\access_rule_base`) — do not `require_once` these files | 5.1 removed (MDL-76612) |
| `quiz_has_question_use`, `quiz_update_sumgrades`, `quiz_set_grade`, `quiz_save_best_grade`, `quiz_calculate_best_grade`, `quiz_calculate_best_attempt`, `quiz_update_all_*` | none listed | 5.1 final dep., removed 5.2 |
| classes `mod_quiz_overdue_attempt_updater`, `moodle_quiz_exception` | none | 5.1 removed |
| `quiz_add_random_questions()`, `mod_quiz\form\add_random_form`, `slot_random::set_tags*()` | none | 5.1 final dep. |
| callbacks `quiz_structure_modified`, `quiz_attempt_deleted` | hooks `structure_modified`, `attempt_state_changed` | 5.1 removed (MDL-80327) |
| `random_question_loader::get_next_question_id()` and siblings; `question_finder::get_questions_from_categories_*_with_usage_counts()` | none | 5.1 final dep. |

`question_edit_contexts` now only checks the given context, not parents (MDL-85754). Activities using question *set* references (random questions) must call `backup_question_set_reference_trait::annotate_set_reference_bank_entries()` next to `add_question_set_references()` (MDL-41924).

**Other components**

| Old | New | Since |
| --- | --- | --- |
| `core_badges_observer` | `core_badges\event\observer` | 5.1 dep. (MDL-83904) |
| `grade_structure::get_*_icon()` (analysis, reset, edit, hiding, locking, calculation) | none | 5.1 removed |
| `\core_customfield\api::get_instance[s]_fields_data()` | new `component`, `area`, `itemid` params (shared categories) | 5.1 changed |
| `workshop::count_submissions()` / `count_assessments()` | `count_all_submissions()` / `count_all_assessments()` | 5.1 dep. |
| report builder `schedule::create_schedule()`, `get_report_empty_options()`, `send_schedule_message()` | schedule types in `<component>\reportbuilder\schedule\<type>` | 5.1 dep. |
| icons `core:e/text_highlight`, `core:e/text_highlight_picker` | none (removed 6.0) | 5.1 dep. |

New: `grade_item::is_gradable()`; `\core_user::get_dummy_fullname()`; themes inherit layouts from grandparents (MDL-79319); `.table-reboot` opt-out class.

### Moodle 5.2

**core**

| Old | New | Since |
| --- | --- | --- |
| AMD `core/modal_factory`, `core/modal_registry` | use `core/modal`; manually registering modal types is no longer possible or necessary (MDL-78324) | 5.2 removed (MDL-79182) |
| YUI `moodle-core-notification-confirm`; `keyDelegation`, `lightbox` | AMD `core/modal` | 5.2 removed |
| `upgrade_ensure_not_running()` | `\core\setup::ensure_upgrade_is_not_running()` / `warn_if_upgrade_is_running()` / `is_upgrade_running()` | 5.2 dep. |
| `\core_shutdown_manager` | `\core\shutdown_manager` | 5.2 renamed |
| `MOD_PURPOSE_INTERFACE` | none | 5.2 removed |
| `get_context_instance()`, `get_system_context()`, `print_arrow()`, `can_use_rotated_text()`, `cron_run*()`, `cron_setup_user()`, `print_course_request_buttons()`, `plagiarism_get_file_results()`, `plagiarism_update_status()` | none listed (long deprecated) | 5.2 removed |
| `core_renderer::htmllize_file_tree()`; `renderer_base::should_display_main_logo()`; `action_menu\link::$instance` | none | 5.2 removed |
| `\core\event\content_viewed`; `webservice_parameter_exception` | none | 5.2 removed |
| `javascript-static.js` helpers `checkall()`, `checknone()`, `confirm_if()`, `M.util.init_toggle_class_on_click()`, `M.form.init_smartselect()`, … | none listed | 5.2 removed |
| `\admin_setting_configselect::output_select_html()` | none | 5.2 removed |
| `get_moodlenet_info()` | none | 5.2 dep. |
| Hook manager caching via MUC | local cache | 5.2 changed |

New: template names ending in `!` ignore theme overrides (MDL-77894); `clean_string()` for Mustache; JS `core/deprecated` (`emitDeprecation('…', {mdl, since, replacement, reason, final})`); router parameter types `query_course`, `query_coursemodule`, `path_module`, `path_section`; `confirm_action` accepts `$title`, `$dialogtype`; checks may return several results via `get_results()`; `\core\attribute\description`.

**core_course / core_courseformat** (central action classes — MDL-86854…86862)

The classes are `core_courseformat\local\cmactions` and `core_courseformat\local\sectionactions` (verified in MOODLE_503_STABLE source); some upstream notes drop the `local\` segment.

| Old | New | Since |
| --- | --- | --- |
| `moveto_module()` | `core_courseformat\local\cmactions::move_before()` / `move_end_section()` | 5.2 dep. |
| `course_delete_module()`, `course_module_flag_for_async_deletion()` | `cmactions::delete()` / `delete_async()` | 5.2 dep. |
| `set_coursemodule_groupmode()` | `cmactions::set_groupmode()` | 5.2 dep. |
| `duplicate_module()` | `cmactions::duplicate()` | 5.2 dep. |
| `course_set_marker()` | `sectionactions::set_marker()` / `remove_all_markers()` | 5.2 dep. |
| `set_section_visible()` | `sectionactions::set_visibility()` | 5.2 dep. |
| `move_section_to()`, `reorder_sections()` | `sectionactions::move_after()` / `move_at()` | 5.2 dep. |
| `base::get_section_number()`, `stateactions::section_move()`, `section\availability::availability_info()`, `core_course_renderer::render_activity_information()` | see 5.0 rows | 5.2 removed |
| JS `replaceSectionActionItem()` (`course/amd/src/actions.js`) | none | 5.2 removed |

New: `cm_info::get_navigation_url()` / `set_navigation_url(?url)` / `reset_navigation_url()` (MDL-87984); routes `course/cms/<cmid>/next`; `$PAGE` `set_hide_settings()`; `course_section_view(..., restricted)`; `base::set_show_restrictions_expanded()`; `activityoverviewbase::$cm` public; `overviewtable::is_cm_displayable()` / `is_cm_available()`. Subsections always render inline; delegated sections lose descriptions (mod_subsection migration tasks). Theme layout option `activityinfoinheader` (default on) moves completion button and activity dates into the activity header (MDL-87662).

**Other components**

| Old | New | Since |
| --- | --- | --- |
| `\badge_award_selector_base`, `\badge_potential_users_selector`, `\badge_existing_users_selector`, `process_manual_award()`, `process_manual_revoke()` | `\core_badges\award_selector_base`, `potential_award_selector`, `existing_award_selector`, `\core_badges\award_manager::process_manual_award()` / `::process_manual_revoke()` | 5.2 new locations (MDL-83902) |
| `core_badges_assertion`, `badges_get_default_issuer()` | `\core_badges\achievement_credential`; none | 5.2 dep. (MDL-85621) |
| `core_badges_renderer::print_badge_table_actions/render_badge_collection/render_badge_management/render_badge_recipients()` | none | 5.2 removed |
| `get_next_version()` (questionlib) | `\core_question\versions::get_next_version()` (call in a transaction) | 5.2 dep. (MDL-86798) |
| `\core_question\output\switch_question_bank`; `mod_quiz_output_fragment_switch_question_bank()` | JS `core_question/bank_switcher`; route `/api/rest/v2/core/question/banks?courseid=X` | 5.2 dep. |
| `question_category_selector::question_count_sql` | `core_question\local\bank\question_counts\by_category_query` | 5.2 replaced |
| YUI `moodle-question-chooser`, template `core_question/qbank_chooser` | none | 5.2 removed |
| `\core_customfield\handler::create()` custom caching | base caching (but see 5.3 reversal) | 5.2 changed |
| JS `core_customfield/repository/toggle_shared` | `core_customfield/repository` | 5.2 dep. |
| `\core_customfield\api::is_shortname_unique()` | new | 5.2 added |
| `\core_form\filetypes_util::is_whitelisted()` / `get_not_whitelisted()` | none listed | 5.2 removed |
| `grade_structure::get_*`, `grade_report::get_lang_string()`, `grade_helper::get_lang_string()`, `core/grades/basewidget.js` | none | 5.2 removed |
| `enrol_manual` WS `unenrol_users()` `roleid` param | omit | 5.2 removed |
| `\profile_field_base::profile_field_base()` (PHP 4 constructor), `core_user_renderer::unified_filter()` | none (PHP 4-style constructors removed, MDL-82284) | 5.2 removed |
| user `moodlenetprofile` field | custom profile field | 5.2 dep. |
| `core_message\api::search_users()`, `get_contacts()`, `get_messages()`, `can_post_message()`, `is_user_blocked()`, … | none listed | 5.2 removed |
| report entity filter `user_filter_manager::reset_all()` | `reset()` | 5.2 dep. |
| forum report entity `get_context_joins()` | `get_course_modules_joins(...)` | 5.2 dep. |
| quiz overrides cache | `\mod_quiz\local\quiz_overrides_cache_manager` | 5.2 replaced |
| `theme_boost` templates `flat_navigation`, `nav-drawer` | none | 5.2 removed |
| icons `core:t/blocks_drawer`, `core:t/blocks_drawer_rtl`, `core:t/index_drawer` | none | 5.2 dep. |

New: `groups_print_activity_menu()` / `groups_get_activity_group()` gain `$participationonly`; `curl::removeopt()`; report builder base entity default `initialise()`; entities `core_course\reportbuilder\local\entities\course_module[_base]`; qbank conditions implement `restore_filtercondition()`; `restore_qtype_plugin::unset_excluded_fields()` returns the modified structure.

### Moodle 5.3

**core**

| Old | New | Since |
| --- | --- | --- |
| `user_create_user()`, `user_update_user()`, `user_delete_user()`, `user_get_users_by_id()`, `user_get_user_details()`, `user_can_view_profile()`, `user_get_default_fields()` (+ ~18 more in `user/lib.php`) | `\core\user::create_user()`, `::update_user()`, `::delete_user()`, `::get_users_by_id()`, `::get_user_details()`, `::can_view_profile()`, `::get_default_fields()` | 5.3 dep. (MDL-82650) |
| `user_update_device_public_key()` | `\core_user\devicekey::update_device_public_key()` | 5.3 dep. |
| `\admin_setting_*`, `\admin_settingpage`, `\admin_category`, `\admin_externalpage`, `\admin_root`, `\admin_setting` | `\core_admin\setting\setting\*`, `\core_admin\setting\settingpage\settingpage`, `\core_admin\setting\tree\category` / `externalpage` / `root`, `\core_admin\setting` | 5.3 renamed (MDL-81935) |
| `validate_internal_user_password()`, `hash_internal_user_password()`, `update_internal_user_password()`, `get_auth_plugin()`, `is_enabled_auth()`, `get_enabled_auth_plugins()` | `\core\authentication\password::validate()` / `::hash()` / `::update()`, `\core\authentication::get_plugin()` / `::is_enabled()` / `::get_enabled_plugins()` (wrappers kept) | 5.3 (MDL-88580) |
| `NO_MOODLE_COOKIES` checks | `\core\session\manager::supports_cookies()`, `set_cookies_supported()`; route option `cookies:` | 5.3 (MDL-87174) |
| `FEATURE_GROUPMEMBERSONLY` | remove | 5.3 dep. (MDL-83231) |
| `\core\task\manager::task_is_scheduled()` | `get_queued_adhoc_task_record()` | 5.3 dep. (MDL-86422) |
| `\core\hub\registration::get_dataroot_size()` | `get_filepool_usage()` | 5.3 dep. |
| template `core/external_content_banner` | none (removed 6.0) | 5.3 dep. |
| `core/modal` title `<h5>` | `<h2 class="modal-title fs-5">` — nest content headings from `<h3>` | 5.3 changed (MDL-75699) |
| Primary/secondary nav `.moremenu` / `a.nav-link.active` markup | React `core/nav/PrimaryNav` / `core/nav/Nav` (`a.mds-nav-pill`, `.mds-nav-pill--selected`) | 5.3 changed |

New: `moodle_page::set_supplementary_content()`, `set_show_navigation_footer()`, `set_has_sticky_footer()`; hook `before_email_to_user`; `moodle_exception` accepts `$previous`; `core\output\submenu` and `url` on `action_menu\subpanel`; `flexible_table::set_columnheadersattributes()`; adhoc `set_soft_retry_delay()`; DI via `#[\DI\Attribute\Inject]`; React `react_component_renderable` and `html_writer::react_component()`; `core/imagedetails/modal`; `\core_auth\validate_user`.

**core_external / core_webservice** (MDL-81225)

| Old | New | Since |
| --- | --- | --- |
| `\external_api`, `\external_value`, `\external_format_value`, `\external_single_structure`, `\external_multiple_structure`, `\external_function_parameters`, `\external_description`, `\external_files`, `\external_warnings`, `\external_settings`, `\restricted_context_exception` | `\core_external\` + same name (e.g. `\core_external\external_api`) | 5.3 dep. |
| `\external_util` | `\core_external\util` | 5.3 dep. |
| `external_format_string()`, `external_format_text()`, `external_validate_format()` | `\core_external\util::format_string()`, `::format_text()`, `::validate_format()` | 5.3 dep. |
| `external_generate_token()`, `external_create_service_token()`, `external_generate_token_for_current_user()`, `external_log_token_request()`, `external_delete_descriptions()` | `\core_external\util::generate_token()`, `::generate_token_for_current_user()`, `::log_token_request()`, `::delete_service_descriptions()` | 5.3 dep. |

New: `allowcorsrequests` in `db/services.php` definitions (MDL-87150). Route methods in `route\api` must carry `#[scopeset(...)]` or `#[unscoped_resource]` (MDL-89710).

**core_course / core_courseformat**

| Old | New | Since |
| --- | --- | --- |
| `base::get_return_section()` | `get_page_section()` | 5.3 dep. (MDL-86284) |
| `base::get_view_url()` option `sr` | `pagesectionid` | 5.3 dep. |
| Section collapse-all `collapsemenu` in `section` output / `core_courseformat/local/content/section/content` | toggle in `core_courseformat\output\local\content` and `core_courseformat/local/content` | 5.3 dep. (MDL-89496) |
| `#collapsesections` id | `[data-toggle="toggleall"]` selector | 5.3 changed |
| Course index subsection tree semantics on section template | `aria-owns`/`aria-labelledby`/`aria-expanded` on the delegating cm `li[role="treeitem"]` | 5.3 changed (MDL-88949) |

New: `base::uses_linear_navigation()` (opt-in); `course_navigation::get_all_section_cms()`, `get_adjacent_section()`, `is_first_navigable()`, `is_last_navigable()`, `get_section()`; `content_item` `disabled` / `disabledreason`; `inline_help` flag on format settings; Behat steps "the course linear navigation should/should not be visible".

**Other components**

| Old | New | Since |
| --- | --- | --- |
| `\core_customfield\handler::create()` cached (5.2) | new instance by default again; opt in with `protected const CACHE_HANDLER_INSTANCES = true;` | 5.3 changed (MDL-89238) |
| `\core_user\hook\extend_user_menu::add_navitem()` | `add_menu_item(\core_user\output\user_action_menu\base $item)` | 5.3 dep. (MDL-88938) |
| `duration` form element with `defaultunit` outside `units` | exception — keep `defaultunit` in `units` | 5.3 changed (MDL-89434) |
| `grade_item::update_deducted_mark()` | none (penalties applied in `penalty_manager`) | 5.3 dep. |
| templates `core_grades/general_action_bar`, `gradereport_grader/action_bar` | `core/navigation_action_bar`, `core/action_bar` | 5.3 changed |
| `bulk_action_base` without icon | override `get_action_icon(): string` (base returns `'i/empty'`) | 5.3 added (MDL-73051) |
| Quiz report `report_base::print_header_and_tabs()` printing the group selector | call `print_action_bar()`; implement `setup_report_data()` and `has_permission()` or pass `null` options | 5.3 changed (MDL-81096) |
| `override_manager::get_effective_open_close_times()` | `get_effective_times()` | 5.3 dep. |
| assign `delete_override`, `delete_all_overrides`, `move_group_override`, `reorder_group_overrides` | `override_manager::` same names | 5.3 dep. (MDL-86513) |
| `assign::get_allocated_markers()` / `update_allocated_markers()`; `ASSIGN_MULTIMARKING_MAX_MARKERS`; event `marker_updated` | `get_marker_allocations()` / `update_marker_allocations()`; `ASSIGN_MULTIMARKING_DEFAULT_MAX_MARKERS`; `marker_added` / `marker_removed` | 5.3 dep. |
| `assign::calculate_penalised_grade()` caller-side scaling | remove (method now scales) | 5.3 changed |
| `enrol_manual` instance `enrol_cohort()` | none | 5.3 dep. |
| `block_timeline\output\main`, `renderer`, external services | removed, no stub (React rewrite) | 5.3 removed |
| `import X from 'theme_boost/bootstrap/x'` | `import {X} from 'bootstrap'` | 5.3 dep. (MDL-88766) |
| `theme_boost/drawer` block `drawerheadercontent` | `drawercontrols` | 5.3 changed |
| Boost `core/loginform` template | moved to core | 5.3 changed |
| theme_classic | removed from core; install from GitHub before upgrade | 5.3 removed (MDL-88351) |

Other 5.3 behaviour: `badge_message_from_template()` accepts `$user`; `grade_outcome::add_outcome_to_module()` etc. for scale-less outcomes; Boost default font Noto Sans; dark colour mode (`theme_boost\colour_mode::render_menu()` for child themes with their own navbar).
