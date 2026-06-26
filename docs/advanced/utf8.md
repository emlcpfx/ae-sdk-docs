# Utf8

> 2 Q&As · source: AE plugin dev community Discord

### How to compile unicode characters (e.g., Chinese) in C++ plugin parameter names and category names for Mac AE plugins?

The name field is defined as A_char[32]. Convert your string to UTF-8 and feed the param macro the resulting string. Make sure the length of the resulting string is no more than 31 chars long (including multibyte characters), as the last char must be used for null termination.

*Tags: `localization`, `mac`, `parameter-name`, `unicode`, `utf8`, `xcode`*

---

### What are best practices for AE SDK string conversions between A_char, A_UTF16Char, and std::string?

Use MultiByteToWideChar with NULL as the last parameter to predict the required buffer size. For error handling, only use PF_Err_OUT_OF_MEMORY as the error return, because PF_Err_INTERNAL_STRUCT_DAMAGED tells AE the effect instance is bad and AE won't talk to it again. PF_Err_OUT_OF_MEMORY is the polite way of telling AE a certain operation has failed and its results should be ignored.

*Tags: `buffer-size`, `error-handling`, `string-conversion`, `utf16`, `utf8`*

---
