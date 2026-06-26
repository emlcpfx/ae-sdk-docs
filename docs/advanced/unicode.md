# Unicode

> 3 Q&As · source: AE plugin dev community Discord

### How to compile unicode characters (e.g., Chinese) in C++ plugin parameter names and category names for Mac AE plugins?

The name field is defined as A_char[32]. Convert your string to UTF-8 and feed the param macro the resulting string. Make sure the length of the resulting string is no more than 31 chars long (including multibyte characters), as the last char must be used for null termination.

*Tags: `localization`, `mac`, `parameter-name`, `unicode`, `utf8`, `xcode`*

---

### How do you handle Unicode characters in After Effects plugin parameter names and categories on macOS?

The name field is defined as 'A_char[32]', so you need to convert your string to UTF-8 and feed it to the param macro. Make sure the length of the resulting UTF-8 string is no more than 31 characters long (including multibyte characters), as the last character must be reserved for null termination. One successful approach mentioned was using ConvertUTF8ToGBK() in a UTF-8 encoded .cpp file when working with XCode.

*Tags: `build`, `macos`, `params`, `unicode`, `xcode`*

---

### How do I fix the 'could not convert Unicode characters' error when setting an output file path in After Effects plugins?

Use A_UTF16Char instead of A_char for the output path parameter. The correct approach is to cast a wide string literal to A_UTF16Char and pass it to AEGP_SetOutputFilePath. Example: const A_UTF16Char *outPath = reinterpret_cast<const A_UTF16Char *>(L"C:\\whee.mov"); ERR(suites.OutputModuleSuite4()->AEGP_SetOutputFilePath(0, 0, (A_char*)outPath));

```cpp
const A_UTF16Char *outPath = reinterpret_cast<const A_UTF16Char *>(L"C:\\whee.mov");
ERR(suites.OutputModuleSuite4()->AEGP_SetOutputFilePath(0, 0, (A_char*)outPath));
```

*Tags: `aegp`, `output-rect`, `scripting`, `unicode`*

---
