# Interop

> 3 Q&As · source: AE plugin dev community Discord

### How can I get the string value of an effect's ARBITRARY parameter data through ExtendScript?

One approach: (1) Add a hidden supervised checkbox parameter. (2) From JavaScript, leave a message in global scope, then toggle the checkbox to trigger USER_CHANGED_PARAM. (3) From C side, use AEGP_ExecuteScript() to check for the JavaScript message, read the string from the arb, and pass it back via AEGP_ExecuteScript(). (4) The string value is then available in JavaScript global scope. An alternative trick is to use hidden UI parameters and encode the string into their controller names, then read the property names from ExtendScript.

*Tags: `aegp-execute-script`, `arb-param`, `extendscript`, `interop`, `string`*

---

### How can I invoke AEGP functionality from ExtendScript (JSX) without creating a menu entry?

There is no direct way to invoke an AEGP from JavaScript without aid from a C external object. The simplest approach is to leave a flag in the JavaScript global scope and have the AEGP check for that flag on idle_hook calls (which happen 20-50 times per second). It's not immediate and synchronous, but it works easily. An alternative is using a C external object (ExternalObject in ExtendScript), though it's more complex to set up.

*Tags: `aegp`, `extendscript`, `external-object`, `idle-hook`, `interop`*

---

### How can After Effects GPU features be utilized with Vulkan?

Vulkan has interoperability features that allow you to import textures from native After Effects GPU features (which may use CUDA, OpenCL, DirectX, or Metal) into Vulkan for processing, providing a way to leverage both native AE GPU capabilities and Vulkan's explicit control.

*Tags: `cuda`, `gpu`, `interop`, `metal`, `vulkan`*

---
