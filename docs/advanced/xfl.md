# Xfl

> 2 Q&As · source: AE plugin dev community Discord

### Is it possible to override or modify the After Effects Export to XFL command through the API?

No, the After Effects API does not offer means of overriding or modifying the export to XFL command. However, as an alternative, you could apply a script to the generated XML to correct it automatically after export, or run a script that exports data from AE directly in the format you need.

*Tags: `aegp`, `api`, `export`, `scripting`, `xfl`*

---

### What is the best approach to automate XFL transformation point corrections after exporting from After Effects?

Rather than trying to override the built-in XFL export, you can write a script that post-processes the exported XFL file's XML to correct placement of registration points, transformation points, and anchor/position data. This script can automatically transform the XML structure from the standard DOMFrame format to your desired output format, saving manual editing time.

*Tags: `automation`, `export`, `scripting`, `xfl`, `xml`*

---
