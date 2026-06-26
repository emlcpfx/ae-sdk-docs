# Fopen

> 1 Q&A · source: AE plugin dev community Discord

### How can an AE plugin save a file (like CSV) when a button is clicked?

The AE SDK only deals with importing files into a project or rendering to custom file types. If you want to save arbitrary data to a file at the click of a button, just use standard C/C++ file I/O directly: fopen/fwrite/fclose. There's no need for any special SDK mechanism.

*Tags: `button-param`, `csv`, `export`, `file-io`, `fopen`*

---
