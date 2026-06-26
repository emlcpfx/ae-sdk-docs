# Extendscript

> 10 Q&As · source: AE plugin dev community Discord

### How do you get the AE/Premiere version number programmatically?

in_data->version major/minor are not reliably updated between AE versions. Options: (1) Use ExtendScript via AEGP_ExecuteScript with 'app.version' - but may fail with 'cannot run script while modal dialog waiting'. (2) Use AEGP_GetPluginPaths with AEGP_GetPathTypes_APP to get the AE install folder path, then parse the version from the folder name or binary metadata. (3) For plugins with Premiere GPU entry points, PPixSuite's appinfo provides version data (format like '24.3', add 2000 for year).

*Tags: `aegp`, `compatibility`, `extendscript`, `premiere`, `version-detection`*

---

### What is the XMP Packet and can it be used to store plugin data in AE projects?

XMP Packets can be read/written via ExtendScript to store custom metadata in AE projects. However, limitations include: the data is not part of AE's undo stack, practical size limit around 500KB (though AE itself can bloat XMP to 50MB+), and it's not undoable. AE Viewer tools exist that can strip XMP to reduce file sizes dramatically (e.g., 50MB to 50KB).

*Tags: `extendscript`, `metadata`, `project-data`, `storage`, `xmp`*

---

### How can I get the string value of an effect's ARBITRARY parameter data through ExtendScript?

One approach: (1) Add a hidden supervised checkbox parameter. (2) From JavaScript, leave a message in global scope, then toggle the checkbox to trigger USER_CHANGED_PARAM. (3) From C side, use AEGP_ExecuteScript() to check for the JavaScript message, read the string from the arb, and pass it back via AEGP_ExecuteScript(). (4) The string value is then available in JavaScript global scope. An alternative trick is to use hidden UI parameters and encode the string into their controller names, then read the property names from ExtendScript.

*Tags: `aegp-execute-script`, `arb-param`, `extendscript`, `interop`, `string`*

---

### How can I invoke AEGP functionality from ExtendScript (JSX) without creating a menu entry?

There is no direct way to invoke an AEGP from JavaScript without aid from a C external object. The simplest approach is to leave a flag in the JavaScript global scope and have the AEGP check for that flag on idle_hook calls (which happen 20-50 times per second). It's not immediate and synchronous, but it works easily. An alternative is using a C external object (ExternalObject in ExtendScript), though it's more complex to set up.

*Tags: `aegp`, `extendscript`, `external-object`, `idle-hook`, `interop`*

---

### Is there an API to get the layer comment field from C++ code?

There is no C API for getting layer comments. Use AEGP_ExecuteScript() to run ExtendScript that reads the layer comment and retrieve the result back to the C side.

*Tags: `aegp`, `execute-script`, `extendscript`, `layer-comment`*

---

### Why does JSON sometimes work and sometimes fail when called from a plugin via AEGP_ExecuteScript?

JSON is not natively defined in ExtendScript, though it may be available in CEP. The intermittent availability suggests that another script on the system may define the JSON standard globally, making it available to all subsequent scripts. Once defined, it persists until After Effects is restarted. To ensure consistent behavior, use the Douglas Crockford JSON-js library (json2.js) explicitly in your script.

*Tags: `aegp`, `extendscript`, `json`, `scripting`*

---

### What is a reliable library for JSON handling in ExtendScript?

The JSON-js library by Douglas Crockford (https://github.com/douglascrockford/JSON-js) provides a free, standard JSON implementation for ExtendScript. It allows you to use JSON.stringify() and other JSON methods consistently across different systems (macOS and Windows).

*Tags: `extendscript`, `json`, `library`, `open-source`*

---

### How can you dynamically find After Effects menu command IDs instead of hardcoding them?

Use app.findMenuCommandId() to dynamically look up command IDs by name. This is the recommended approach because command numbers can change between After Effects versions. For example: id = app.findMenuCommandId("copy"); or app.executeCommand(app.findMenuCommandId("Close Project"));

```cpp
id = app.findMenuCommandId("copy");
app.executeCommand(app.findMenuCommandId("Close Project"));
```

*Tags: `cross-platform`, `extendscript`, `scripting`, `ui`*

---

### Is there an open-source searchable database of After Effects command IDs?

Justin Taylor at Hyper Brew maintains an updated searchable Command ID list for After Effects 2022 at https://hyperbrew.co/blog/after-effects-command-ids/, which is useful for scripting and tool development. There was also a Bitbucket snippet repository at https://bitbucket.org/justin2taylor/workspace/snippets/aLjjBE with command ID resources.

*Tags: `extendscript`, `open-source`, `reference`, `scripting`, `tool`*

---

### How can you add one composition as a layer into another composition using ExtendScript?

You can add a composition item as a layer to another composition using the `comp.layers.add()` method in ExtendScript. The syntax is `comp.layers.add(otherCompItem);` where `otherCompItem` is the composition you want to add. More details can be found in the official After Effects scripting guide.

```cpp
comp.layers.add(otherCompItem);
```

*Tags: `extendscript`, `layer-checkout`, `scripting`*

---
