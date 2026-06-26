# Javascript

> 1 Q&A · source: AE plugin dev community Discord

### How can a C/C++ plugin propagate internal data to JavaScript in After Effects?

Use the AEGP_ExecuteScript() function from the Utility Suite to execute JavaScript code from the C/C++ plugin side. For simple cases, call suites.UtilitySuite5()->AEGP_ExecuteScript(NULL, "alert('hello world!')", FALSE, NULL, NULL); For more elaborate work where you need to retrieve return values from the script, refer to the detailed example in the Adobe forum thread at https://forums.adobe.com/message/3625857

```cpp
suites.UtilitySuite5()->AEGP_ExecuteScript(NULL, "alert('hello world!')", FALSE, NULL, NULL);
```

*Tags: `aegp`, `c-plugin`, `data-communication`, `javascript`, `scripting`*

---
