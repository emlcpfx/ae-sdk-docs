# Suite Manager

> 1 Q&A · source: AE plugin dev community Discord

### What causes AE error code 1397908844?

Error 1397908844 is a generic exception handler/catch-all in Adobe's suites manager, dating back to CS2/CS3. It commonly appears when double-freeing/releasing a suite pointer. It appears more often in newer AE versions due to MFR with global suite handles shared between threads. Known triggers include: problems with arb params, crashes after exporting RAM previews, and double-releasing suites. The underlying issue is usually in the plugin's suite management code.

*Tags: `debugging`, `double-free`, `error-code`, `mfr`, `suite-manager`*

---
