# Json

> 2 Q&As · source: AE plugin dev community Discord

### Why does JSON sometimes work and sometimes fail when called from a plugin via AEGP_ExecuteScript?

JSON is not natively defined in ExtendScript, though it may be available in CEP. The intermittent availability suggests that another script on the system may define the JSON standard globally, making it available to all subsequent scripts. Once defined, it persists until After Effects is restarted. To ensure consistent behavior, use the Douglas Crockford JSON-js library (json2.js) explicitly in your script.

*Tags: `aegp`, `extendscript`, `json`, `scripting`*

---

### What is a reliable library for JSON handling in ExtendScript?

The JSON-js library by Douglas Crockford (https://github.com/douglascrockford/JSON-js) provides a free, standard JSON implementation for ExtendScript. It allows you to use JSON.stringify() and other JSON methods consistently across different systems (macOS and Windows).

*Tags: `extendscript`, `json`, `library`, `open-source`*

---
