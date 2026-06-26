# Cross Version

> 2 Q&As · source: AE plugin dev community Discord

### How can I make an After Effects plugin compiled with CS6 SDK work with CS5 and 5.5?

You need to do more than just swap headers—you must also update the API version in the .r (PiPL) file. The recommended approach is to copy your code to a working CS5 project and recompile with the CS5 SDK. While Adobe states that suites are never removed or altered, rare deprecations have occurred (e.g., in CS6). If you need to support multiple versions, use preprocessor definitions to conditionally select the correct suite version for each target SDK version.

```cpp
#define SOME_SUITE someSuiteVer6  // for CS4
#define SOME_SUITE someSuiteVer7  // for CS5

// In code:
suites.SOME_SUITE()->someFunction("yo yo!");
```

*Tags: `build`, `cross-version`, `pipl`, `sdk`, `suites`*

---

### Can I acquire multiple versions of the same After Effects suite in a single plugin?

Yes, you can acquire multiple versions of the same suite. Adobe's documentation states that new suite versions supersede old ones but never remove previously shipped suites. You can use AEFX_AcquireSuite() to explicitly request a specific suite version. However, be aware that rare exceptions exist where suites were deprecated between versions.

*Tags: `aegp`, `cross-version`, `suites`*

---
