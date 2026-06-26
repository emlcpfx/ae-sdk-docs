# Import

> 2 Q&As · source: AE plugin dev community Discord

### Is it possible to import footage in the background using AEGP functions?

No, it is not possible to import footage in the background. AEGP functions must execute on the main thread only, so background import operations are not supported by the After Effects SDK.

*Tags: `aegp`, `import`, `main-thread`, `threading`*

---

### How do you open and read files in an AEIO plugin using the SDK API?

The After Effects SDK does not provide built-in file I/O functions. Instead, you use standard C library functions like fopen, fread, and fclose to open and read files. When you receive a file path as A_UTF16Char in functions like VerifyFileImportable(), you can use it directly with these standard functions.

*Tags: `aegp`, `aeio`, `import`, `sdk`*

---
