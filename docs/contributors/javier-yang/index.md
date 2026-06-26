# Javier Yang

**1 contributions** to AE SDK community knowledge.

Top topics: `arb-param`, `extendscript`, `aegp-execute-script`, `string`, `interop`

---

## How can I get the string value of an effect's ARBITRARY parameter data through ExtendScript?

One approach: (1) Add a hidden supervised checkbox parameter. (2) From JavaScript, leave a message in global scope, then toggle the checkbox to trigger USER_CHANGED_PARAM. (3) From C side, use AEGP_ExecuteScript() to check for the JavaScript message, read the string from the arb, and pass it back via AEGP_ExecuteScript(). (4) The string value is then available in JavaScript global scope. An alternative trick is to use hidden UI parameters and encode the string into their controller names, then read the property names from ExtendScript.

*Source: adobe-forum-sdk · 2025-09-01 · Tags: `arb-param`, `extendscript`, `aegp-execute-script`, `string`, `interop` · [View in Q&A](../qa/arb-param/)*

---
