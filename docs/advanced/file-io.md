# File Io

> 2 Q&As · source: AE plugin dev community Discord

### How can an AE plugin save a file (like CSV) when a button is clicked?

The AE SDK only deals with importing files into a project or rendering to custom file types. If you want to save arbitrary data to a file at the click of a button, just use standard C/C++ file I/O directly: fopen/fwrite/fclose. There's no need for any special SDK mechanism.

*Tags: `button-param`, `csv`, `export`, `file-io`, `fopen`*

---

### How can you persist sequence data without consuming memory?

You can dump sequence data to a file (encoded or not) and read it back as needed. This approach allows you to save a sequence of data without loading it all into memory at once.

*Tags: `caching`, `file-io`, `memory`, `sequence-data`*

---
