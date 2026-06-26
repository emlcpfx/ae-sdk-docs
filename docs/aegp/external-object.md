# External Object

> 1 Q&A · source: AE plugin dev community Discord

### How can I invoke AEGP functionality from ExtendScript (JSX) without creating a menu entry?

There is no direct way to invoke an AEGP from JavaScript without aid from a C external object. The simplest approach is to leave a flag in the JavaScript global scope and have the AEGP check for that flag on idle_hook calls (which happen 20-50 times per second). It's not immediate and synchronous, but it works easily. An alternative is using a C external object (ExternalObject in ExtendScript), though it's more complex to set up.

*Tags: `aegp`, `extendscript`, `external-object`, `idle-hook`, `interop`*

---
