# Project Structure

> 1 Q&A · source: AE plugin dev community Discord

### How can I store custom AEGP plugin data in a comp or project?

There is no official facility for storing AEGP data in a comp or project, but there are several workarounds: (1) Use layer and comp comments fields, which are rarely used by other plugins and not prominently displayed in the UI; (2) Add a null layer to the comp with a clear name like "my stuff, do not delete", lock it, and store data in its comments field while documenting its purpose to users; (3) Create a folder in the project with a clear name and store subfolders named with comp IDs containing the data, minimizing visible clutter.

*Tags: `aegp`, `arb-data`, `project-structure`*

---
