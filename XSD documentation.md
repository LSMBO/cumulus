# Schema documentation for the Cumulus tool descriptor XSD

## Purpose

This schema defines a compact XML format to describe GUI-configurable tools and their parameters.
A "tool" is composed of one or more "section" elements; each section contains parameters or conditional blocks
that drive display and behavior in the GUI and how to construct a command line or configuration file.


## Root element

### \<tool> element

* The document root that describes a single tool.
* Contains one or more section elements.

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| id | string | ✅ | Unique identifier for the tool |
| name | string | ✅ | Display name |
| version | string | ✅ | Version string |
| end_tag | string | ✅ | Output token used to detect process completion |
| end_tag_location | string | ❌ | File (or stream) where end_tag appears (%stderr%, etc.) |
| command | string | ✅ | Base command-line template |
| convert_config_to | string | ❌ | Output config file format (yaml/json), used with %config-file% in command |
| url | string | ❌ | Optional documentation link |
| description | string | ❌ | Short description |
| category | string | ❌ | For the moment only used to distinguish the apps in development (\_\_work_in_progress__) from the others |
| hidden | boolean | ❌ | Whether the tool is hidden from the UI |

## Layout elements

### \<section> elements

A section contains a list of parameters or conditionals, the title will be displayed on top.

In the GUI, a section can be expanded or collapsed, in order to reduce the quantity of visible information.

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| name | string | ✅ | Will be used to create the HTML identifier |
| title | string | ✅ | Section name displayed to the user |
| expanded | boolean | ❌ | If false, the section will collapsed by default |
| visibility | string | ❌ | If "hidden", the section will not be displayed. If "advanced", the section will only be displayed when the user activates the **advanced mode** |
| help | string | ❌ | The content of the tooltip text |

### \<conditional> elements

A conditional contains one primary parameter (only one of \<select>, \<filelist>, \<checkbox>, \<number> and \<string> for the moment), and at least one **\<when>** element.

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| name | string | ✅ | Will be used to create the HTML identifier |

### \<when> elements

The \<when> elementscan only appear within conditionals.

Each \<when> element contains a list of parameters. By default, the content of these elements is not visible, except if the value of the primary parameter of the conditional matches the value of the element.

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| value | string | ✅ | The parameter's value to activate this element, use 'true' of 'false' for checkboxes |
| allow_regex | boolean | ❌ | If true, treats when.value as a regular expression (useful when the parameter is a textbox). Default is false |

## Parameters

Different types of parameters are available, based on the most common HTML input tags. They have been added as they were needed in the apps we were interested in, in order to make sure that everything is tested thoroughly.

The parameters all share a list of generic attributes:

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| name | string | ✅ | Unique name (can be hierarchical via dot notation) |
| label | string | ✅ | Label displayed to the user |
| exclude_from_config | boolean | ❌ | If true, won't be included in config file |
| help | string | ❌ | The content of the tooltip text |
| visibility | string | ❌ | If "hidden", the parameter will not be displayed. If "advanced", it will only be displayed when the user activates the **advanced mode** |
| command | string | ❌ | CLI argument to inject this value |
| config_path | string | ❌ | Deprecated, to define the location of the parameter in the config file, use the dot notation in the name attribute |
| url | string | ❌ | External resource/documentation link |
| url_label | string | ❌ | Text to display before URL |


### \<select>

This parameter creates a dropdown list. The content of the list is defined by the \<option> element.

#### \<option> attributes

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| value | string | ✅ | The value displayed to the user |
| selected | boolean | ❌ | If true, the option will be selected (only one option can be selected) |
| command | string | ❌ | CLI argument to inject this value (may contain the displayed value or not) |


### \<checklist>

This parameter creates a dropdown list with checkboxes. The content of the list is defined by the \<option> element.

#### \<option> attributes

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| value | string | ✅ | The value displayed to the user |
| selected | boolean | ❌ | If true, the option will be selected |
| command | string | ❌ | CLI argument to inject this value (may contain the displayed value or not) |


### \<keyvalues>

This parameter creates a table with one row per key/value tuple. The content of the list is defined by the \<option> element.

#### \<keyvalues> attributes

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| label_key | string | ✅ | The label of the Key column displayed to the user |
| label_value | string | ✅ | The label of the Value column displayed to the user |
| placeholder_key | string | ❌ | The text used as a placeholder in the textboxes of the Key column |
| placeholder_value | string | ❌ | The text used as a placeholder in the textboxes of the Value column |
| type_of | boolean | ❌ | Defines the content allowed for the Value. Default is string, can be 'integer', 'float','string' |
| is_list | boolean | ❌ | If true, will gather the values with the same key into an array |
| repeated_command | string | ❌ | CLI argument used repeatedly for each option, variables allowed here are %key% and %value% |

#### \<option> attributes

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| key | string | ✅ | The name of the key |
| value | string | ✅ | The value of the key |


### \<checkbox>

A simple checkbox.

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| value | boolean | ✅ | Whether the box is checked or not |
| command_if_unchecked | string | ❌ | CLI argument to inject but only if the box is unchecked |

### \<string>

A simple textbox

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| value | string | ❌ | The content of the textbox |
| placeholder | string | ❌ | The text used as a placeholder in the textbox |
| allow_empty | boolean | ❌ | If true, the command line or config entry will be kept even if the textbox is empty ("" value will be used), if not it will be skipped. Default is false. |

### \<number>

A simple textbox for numbers

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| value | decimal | ❌ | The content of the textbox |
| min | decimal | ❌ | The minimum value allowed |
| max | decimal | ❌ | The maximum value allowed |
| step | decimal | ❌ | Increment or decrement the value with this number when clicking on the *up* and *down* buttons |
| placeholder | string | ❌ | The text used as a placeholder in the textbox |

### \<range>

Two textboxes for numbers, usually to determine a parameter between a minimum and a maximum

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| value | decimal | ❌ | The content of the first textbox |
| value | decimal | ❌ | The content of the second textbox |
| min | decimal | ❌ | The minimum value allowed |
| max | decimal | ❌ | The maximum value allowed |
| step | decimal | ❌ | Increment or decrement the value with this number when clicking on the *up* and *down* buttons |
| placeholder | string | ❌ | The text used as a placeholder in the first textbox |
| placeholder2 | string | ❌ | The text used as a placeholder in the second textbox |

### \<filelist>

A file browser, used to select the input files of the jobs.

If *multiple* is true, the client will display a table with one selected file per row, and a button to remove the file from the list. Another button will allow the user to clear the list entirely.

If *multiple* is false, the client will display a simple file browser with a textbox that will contain the path of the selected file.

| Attribute | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| multiple | boolean | ✅ | Whether the file browser allows one or multiple selections |
| is_folder | boolean | ✅ | Whether the file browser should target files or folders |
| is_raw_input | boolean | ✅ | Raw files will be sent to the shared data folder, any other file will be sent to the job folder |
| convert_to_mzml | boolean | ❌ | If true, the file will be converted to mzML format before starting the job. Only Thermo and Bruker formats are supported for the moment |
| value | string | ❌ | If multiple is false, allows a default file input to be used |
| format | string | ❌ | The formats to browse; multiple formats can be given, separated by ";". Only put the extensions, for instance 'fasta;txt' |
| repeated_command | string | ❌ | CLI argument used repeatedly for each file |
