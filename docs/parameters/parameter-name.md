# Parameter Name

> 1 Q&A · source: AE plugin dev community Discord

### How to compile unicode characters (e.g., Chinese) in C++ plugin parameter names and category names for Mac AE plugins?

The name field is defined as A_char[32]. Convert your string to UTF-8 and feed the param macro the resulting string. Make sure the length of the resulting string is no more than 31 chars long (including multibyte characters), as the last char must be used for null termination.

*Tags: `localization`, `mac`, `parameter-name`, `unicode`, `utf8`, `xcode`*

---
