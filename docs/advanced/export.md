# Export

> 4 Q&As · source: AE plugin dev community Discord

### What is the AEGP memory leak issue on Mac during export with Compute Cache?

When using AEGP_MemorySuite to create memHandles during ComputeCache threads, the memory may not be properly freed during export on Mac (works fine on Windows). The used memory grows past the RAM limit. Using new/delete instead of AEGP memHandles avoids the leak. The issue is that memHandles allocated in one thread and freed in another may not actually release memory during the render thread. The AEGP tools report the memory as freed, but virtual memory keeps growing.

*Tags: `aegp-memory-suite`, `compute-cache`, `export`, `mac`, `memory-leak`, `threading`*

---

### How can an AE plugin save a file (like CSV) when a button is clicked?

The AE SDK only deals with importing files into a project or rendering to custom file types. If you want to save arbitrary data to a file at the click of a button, just use standard C/C++ file I/O directly: fopen/fwrite/fclose. There's no need for any special SDK mechanism.

*Tags: `button-param`, `csv`, `export`, `file-io`, `fopen`*

---

### Is it possible to override or modify the After Effects Export to XFL command through the API?

No, the After Effects API does not offer means of overriding or modifying the export to XFL command. However, as an alternative, you could apply a script to the generated XML to correct it automatically after export, or run a script that exports data from AE directly in the format you need.

*Tags: `aegp`, `api`, `export`, `scripting`, `xfl`*

---

### What is the best approach to automate XFL transformation point corrections after exporting from After Effects?

Rather than trying to override the built-in XFL export, you can write a script that post-processes the exported XFL file's XML to correct placement of registration points, transformation points, and anchor/position data. This script can automatically transform the XML structure from the standard DOMFrame format to your desired output format, saving manual editing time.

*Tags: `automation`, `export`, `scripting`, `xfl`, `xml`*

---
