# Moodle 5.3 Forms API (formslib)

Compact reference for defining and processing forms with `moodleform` / `MoodleQuickForm` in Moodle 5.3, distilled from the Moodle 5.3 developer docs and the 5.0–5.3 upgrade notes. The formslib devdocs pages are unchanged since 4.5; 5.x changes come from the upgrade notes (see "Changes since 4.5" at the end).

## Form definition: moodleform subclass

Forms are split into (1) a form definition class extending `moodleform`, and (2) uses of that form. Place the class in an auto-loadable location: `[path/to/plugin]/classes/form/myform.php` (in a 5.1+ git checkout the plugin itself lives under `public/`, e.g. `public/local/myplugin/classes/form/`). The class name becomes the form's HTML `id` (trailing `_form` removed), so keep it unique.

```php
<?php
namespace [plugintype]_[pluginname]\form;

// moodleform is defined in formslib.php.
require_once("$CFG->libdir/formslib.php");

class simplehtml_form extends \moodleform {
    // Add elements to form.
    public function definition() {
        // A reference to the form is stored in $this->_form.
        $mform = $this->_form; // Don't forget the underscore!

        // Add elements to your form.
        $mform->addElement('text', 'email', get_string('email'));

        // Set type of element (how submitted values are cleaned).
        $mform->setType('email', PARAM_NOTAGS);

        // Default value.
        $mform->setDefault('email', 'Please enter email');
    }

    // Custom (server-side) validation goes here.
    public function validation($data, $files) {
        return []; // Return an empty array when there are no errors.
    }
}
```

## Using the form in a page

Default form `action` is `strip_querystring(qualified_me())`. `is_cancelled()` must be called before `get_data()`.

```php
// Process page parameters, set up $PAGE, etc.

$mform = new \plugintype_pluginname\form\myform();

// Initial values: array (or object) of name/value pairs matching element names.
$mform->set_data($toform);

if ($mform->is_cancelled()) {
    // Cancel button pressed: usually redirect back.
    redirect($returnurl);
} else if ($fromform = $mform->get_data()) {
    // Submitted AND validated: process data, then redirect.
    redirect($nexturl);
}

// First display, or resubmission after failed validation.
echo $OUTPUT->header();
$mform->display();
echo $OUTPUT->footer();
```

Inside a block (or anywhere you need HTML instead of direct output) use `$mform->render()` instead of `display()`.

## Elements

Add elements with `$mform->addElement($type, $name, $label, ...)`.

Basic types: `button`, `checkbox`, `radio`, `select`, `multi-select`, `password`, `hidden`, `html`, `static`, `text`, `textarea`, `header`.

Advanced types: `autocomplete`, `advcheckbox`, `choicedropdown`, `float`, `passwordunmask`, `recaptcha`, `selectyesno`, `selectwithlink`, `date_selector`, `date_time_selector`, `duration`, `editor`, `filepicker`, `filemanager`, `tags`, `addGroup`, `modgrade`, `modvisible`, `grading`, `questioncategory`.

```php
// Text field.
$mform->addElement('text', 'email', get_string('email'));
$mform->setType('email', PARAM_NOTAGS);

// Select.
$mform->addElement('select', 'display', get_string('displaymode', 'choice'), $CHOICE_DISPLAY);

// Header (fieldset). ALWAYS give headers a unique name — an empty or
// duplicate name causes flags like setAdvanced to leak into other sections.
$mform->addElement('header', 'miscellaneoussettingshdr', get_string('miscellaneoussettings', 'form'));

// Hidden.
$mform->addElement('hidden', 'optionid', 0);
$mform->setType('optionid', PARAM_INT);

// Grouped elements.
$tags = [
    $mform->createElement('text', 'tagsadd', get_string('addtags', 'blog')),
    $mform->createElement('submit', 'addtags', get_string('add')),
];
$mform->addGroup($tags, 'tagsgroup', get_string('addtags', 'blog'), [' '], false);
```

### choicedropdown

A dropdown where each option can carry an icon and description, driven by a `core\output\choicelist` (the same data definition used by `core\output\local\dropdown\status` and `core\output\local\action_menu\subpanel`):

```php
$options = new core\output\choicelist();
$options->add_option('option1', "Text option 1", [
    'description' => 'Option 1 description',
    'icon' => new pix_icon('t/hide', 'Eye icon 1'),
]);
$mform->addElement('choicedropdown', 'FIELDNAME', get_string('FIELDNAME', 'PLUGINNAME'), $options);
$mform->setDefault('FIELDNAME', $defaultvalue);
```

### Custom elements

```php
MoodleQuickForm::registerElementType(
    'course_competency_rule',                          // Name used in addElement().
    "$CFG->dirroot/$CFG->admin/tool/lp/classes/course_competency_rule_form_element.php",
    'tool_lp_course_competency_rule_form_element'      // Class name (no autoloading).
);
$mform->addElement('course_competency_rule', 'competency_rule', get_string('uponcoursemodulecompletion', 'tool_lp'), $options);
```

## Common per-element functions

- `setType($name, PARAM_*)` — how submitted values are cleaned (e.g. `PARAM_INT`, `PARAM_NOTAGS`, `PARAM_CLEANHTML`). Required for `text`/`hidden` elements.
- `setDefault($name, $value)` — default value.
- `addRule(...)` — client- and server-side validation rules (e.g. required, email, numeric).
- `addHelpButton(...)` — pop-up help button next to an element.
- `disabledIf(...)` — conditionally disable an element or group based on other elements' state.
- `hideIf(...)` — conditionally hide an element or group based on other elements' state.
- `setAdvanced($name, $bool = true)` — hide behind "Show more…". Passing a header name marks the whole section advanced. Use sparingly (see best practices below).
- `setExpanded($headername)` — expand a section by default.
- `$this->_form->disable_form_change_checker()` — suppress the "unsaved changes" leave-page warning.

## Action buttons

`add_action_buttons()` is on `moodleform` itself, not on `$this->_form`:

```php
public function definition() {
    $this->_form->addElement(/* ... */);
    // Standard Submit + Cancel buttons. $cancel controls the cancel button;
    // $submitlabel overrides the default 'savechanges' label.
    $this->add_action_buttons();               // add_action_buttons(bool $cancel = true, ?string $submitlabel = null)
    // Or sticky variant (4.3+): add_sticky_action_buttons($cancel, $submitlabel).
}
```

## Validation

Override `validation($data, $files)` in your form class; return an array of `elementname => errorstring` (an empty array means valid). `get_data()` only returns data once validation passes; otherwise the form is redisplayed with errors.

## Files in forms

Three file-related elements: `filepicker` (one throwaway file), `filemanager` (a permanent collection of files), `editor` (HTML textarea with embedded files).

### filepicker — single, transient file

Use for process-and-discard uploads (CSV import, backup restore). It does not store into the File API — use a `filemanager` restricted to one file if the file must persist and reappear on edit.

```php
$mform->addElement('filepicker', 'userfile', get_string('file'), null,
    ['maxbytes' => $maxbytes, 'accepted_types' => '*']);

// After submission:
$content = $mform->get_file_content('userfile');
$name = $mform->get_new_filename('userfile');
$success = $mform->save_file('userfile', $fullpath, $override);   // Beware on clustered servers.
$storedfile = $mform->save_stored_file('userfile', /* ... */);    // Store into the File API.
```

### filemanager — multiple, persistent files

```php
$mform->addElement('filemanager', 'attachments', get_string('attachment', 'moodle'), null, [
    'subdirs' => 0,
    'maxbytes' => $maxbytes,
    'areamaxbytes' => 10485760,
    'maxfiles' => 50,
    'accepted_types' => ['document'],
    'return_types' => FILE_INTERNAL | FILE_EXTERNAL,
]);
```

Uploads land in the user's draft area; you must move them. When editing existing content, first copy existing files into the draft area:

```php
$draftitemid = file_get_submitted_draft_itemid('attachments');
file_prepare_draft_area($draftitemid, $context->id, 'mod_glossary', 'attachment', $entry->id,
    ['subdirs' => 0, 'maxbytes' => $maxbytes, 'maxfiles' => 50]);
$entry->attachments = $draftitemid;
$mform->set_data($entry);
```

On submission, persist the draft files:

```php
if ($data = $mform->get_data()) {
    // ... store or update $entry, then:
    file_save_draft_area_files($data->attachments, $context->id, 'mod_glossary', 'attachment',
        $entry->id, ['subdirs' => 0, 'maxbytes' => $maxbytes, 'maxfiles' => 50]);
}
```

### editor — HTML content with embedded files

Simplest route: store the content in DB fields `textfield`, `textfieldformat` (and optionally `textfieldtrust`), then use the standard helpers:

```php
$textfieldoptions = [
    'trusttext' => true, 'subdirs' => true,
    'maxfiles' => $maxfiles, 'maxbytes' => $maxbytes, 'context' => $context,
];

$mform->addElement('editor', 'textfield_editor', get_string('fieldname', 'somemodule'), null, $textfieldoptions);

// Before display: copies existing files to draft area, rewrites @@PLUGINFILE@@ URLs.
$data = file_prepare_standard_editor($data, 'textfield', $textfieldoptions,
    $context, 'mod_somemodule', 'somearea', $data->id);   // $data->id = null if new.

// After submit: moves draft files to permanent storage, restores @@PLUGINFILE@@ placeholders.
$data = file_postupdate_standard_editor($data, 'textfield', $textfieldoptions,
    $context, 'mod_somemodule', 'somearea', $data->id);
```

Submitted editor values are arrays: `$fromform->entry['text']`, `$fromform->entry['format']`, `$fromform->entry['itemid']`. Before displaying stored content, rewrite placeholders back to URLs with `file_rewrite_pluginfile_urls($text, 'pluginfile.php', $context->id, $component, $filearea, $itemid)` and serve files via a `[componentname]_pluginfile()` function in the component's `lib.php`.

## Repeat elements

`repeat_elements()` repeats a group of elements (e.g. quiz answers), with an "Add more" button that reloads the page; submitted data arrives as zero-indexed arrays.

```php
$repeatarray = [
    $mform->createElement('text', 'option', get_string('optionno', 'choice')),
    $mform->createElement('text', 'limit', get_string('limitno', 'choice')),
    $mform->createElement('hidden', 'optionid', 0),
    $mform->createElement('submit', 'delete', get_string('deletestr', 'choice'), [], false),
];

$repeatno = $this->_instance
    ? $DB->count_records('choice_options', ['choiceid' => $this->_instance]) + 2
    : 5;

// Per-element options: 'default', 'type', 'helpbutton', 'disabledif', 'rule',
// 'expanded', 'advanced'. '{no}' in strings is replaced by the (1-indexed) repeat number.
$repeateloptions = [
    'limit' => [
        'default' => 0,
        'disabledif' => ['limitanswers', 'eq', 0],
        'rule' => 'numeric',
        'type' => PARAM_INT,
    ],
    'option' => ['helpbutton' => ['choiceoptions', 'choice']],
];

$mform->setType('option', PARAM_CLEANHTML);
$mform->setType('optionid', PARAM_INT);

$this->repeat_elements(
    $repeatarray,        // Elements to repeat.
    $repeatno,           // Initial number of repeats.
    $repeateloptions,    // Options per element.
    'option_repeats',    // Hidden element storing repeat count.
    'option_add_fields', // Name of the 'Add more' button.
    3,                   // Fields added per click.
    null,                // Add-button label ({no} = number added).
    true,                // Add button inside the section.
    'delete',            // No-submit delete button name per repeat.
);
```

## No-submit buttons

Buttons that trigger a form "sub action" (e.g. add an option) without full submission. Register them in `definition()`:

```php
$mform->registerNoSubmitButton('addtags');
```

Then in the page, check `no_submit_button_pressed()` before checking submitted data:

```php
if ($mform->is_cancelled()) {
    // ...
} else if ($mform->no_submit_button_pressed()) {
    $data = $mform->get_submitted_data();  // Unvalidated submitted data.
} else if ($fromform = $mform->get_data()) {
    // Process validated data.
}
```

## Checkbox controller

Group `advcheckbox` elements with an integer `group` attribute and add a select-all/none control:

```php
$mform->addElement('advcheckbox', 'test1', 'Test 1', null, ['group' => 1]);
$mform->addElement('advcheckbox', 'test2', 'Test 2', null, ['group' => 1]);
$this->add_checkbox_controller(1);
// Optional custom label: $this->add_checkbox_controller(3, get_string('checkall', 'plugintype_pluginname'));
```

Signature: `add_checkbox_controller($groupid, ?string $text = null, $attributes = null, int $originalvalue = 0)` — `$originalvalue` sets the initial toggle direction.

## Activity module (mod_form) forms

- Class must be named `mod_[modname]_mod_form`, extend `moodleform_mod`, and live in `mod/[modname]/mod_form.php` or `mod/[modname]/classes/mod_form.php` (repository path `public/mod/[modname]/...` since 5.1).
- Call `$this->standard_coursemodule_elements()` before `$this->add_action_buttons()` to add the shared visibility/group-mode elements.
- Pre-process DB defaults in `defaults_preprocessing()`; post-processing happens in `[modname]_add_instance` / `[modname]_update_instance` in `lib.php`.

## Other useful methods

- `filter_shown_headers(array $headernames)` (4.3+) — show only the named headers, expanded; other headers hidden. Often driven by a `showonly` page param.
- `mock_submit($data)` — for unit tests: simulates a user submission; data passes through validation so `get_data()` returns parsed values plus defaults.

## Form usability best practices

- Every field needs a clear, concise label; never rely on Help popups to fix a poor label.
- Name sections so users can predict their content without expanding them; avoid "Miscellaneous", "Advanced settings", "Additional options", and words like "Options"/"Settings" in section names.
- Reuse standard section names: **General** (first, expanded), **Content**, **Appearance**, **Availability**.
- Aim for roughly 5-6 settings per section; split overloaded sections, merge tiny related ones.
- Use `setAdvanced` / "Show more" sparingly; a section must never contain only advanced settings.
- Forms with more than 2 sections collapse all but the first (plus any with required fields) — keep that default.
- Order matters: important sections and settings first; group related things; expanded sections at the top.

## Changes since 4.5 (from upgrade notes)

- **`duration` element (5.3):** units are now strictly enforced as defined by the caller; an exception is thrown when the `defaultunit` option is not one of the element's `units`. Check every `duration` element that sets both.
- **`cohort` element (5.0, MDL-83641):** new `includes` option, passed to the underlying external service to choose which cohorts are returned (`self`, `parents`, `all`).
- **Removed (5.2):** `\core_form\filetypes_util::is_whitelisted()` and `get_not_whitelisted()` (final deprecation); `M.form.init_smartselect()` from `javascript-static.js`.
- **AI provider forms (5.0):** AI provider plugin form classes should no longer call `$this->add_action_buttons();` (it is no longer required there).
- **Bootstrap 5 (5.0):** if you hand-write form markup in templates or custom elements, `.form-group` is replaced by margin utilities (e.g. `mb-3`) and `.form-inline` by flex utilities (`d-flex flex-wrap align-items-center`); use `visually-hidden` instead of `sr-only` for hidden labels. A BS4 compatibility layer remains until 6.0.
