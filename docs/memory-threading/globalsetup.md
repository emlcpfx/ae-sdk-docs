# Globalsetup

> 2 Q&As · source: AE plugin dev community Discord

### Can compute cache data be accessed in globalsetup or paramssetup?

Compute cache must be defined during global setup, and the first access possible is during param setup. However, compute cache cannot be flattened. For persisting data in project files, sequence data or arb data should be used instead.

*Tags: `compute-cache`, `globalsetup`, `params`*

---

### How can instance-specific data be stored in a project file and accessed in global or paramssetup?

Arb data can be used for this purpose. Arb data can be accessed using special group functions for access and save operations, allowing you to store and retrieve instance-specific data that persists in the project file.

*Tags: `arb-data`, `globalsetup`, `params`, `sequence-data`*

---
