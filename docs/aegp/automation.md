# Automation

> 1 Q&A · source: AE plugin dev community Discord

### What is the best approach to automate XFL transformation point corrections after exporting from After Effects?

Rather than trying to override the built-in XFL export, you can write a script that post-processes the exported XFL file's XML to correct placement of registration points, transformation points, and anchor/position data. This script can automatically transform the XML structure from the standard DOMFrame format to your desired output format, saving manual editing time.

*Tags: `automation`, `export`, `scripting`, `xfl`, `xml`*

---
