# String Conversion

> 1 Q&A · source: AE plugin dev community Discord

### What are best practices for AE SDK string conversions between A_char, A_UTF16Char, and std::string?

Use MultiByteToWideChar with NULL as the last parameter to predict the required buffer size. For error handling, only use PF_Err_OUT_OF_MEMORY as the error return, because PF_Err_INTERNAL_STRUCT_DAMAGED tells AE the effect instance is bad and AE won't talk to it again. PF_Err_OUT_OF_MEMORY is the polite way of telling AE a certain operation has failed and its results should be ignored.

*Tags: `buffer-size`, `error-handling`, `string-conversion`, `utf16`, `utf8`*

---
