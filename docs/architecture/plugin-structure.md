# Plugin Structure

> 1 Q&A · source: AE plugin dev community Discord

### Why does adding another PiPL resource with a different name and matchname only show one plugin in After Effects?

When multiple PiPL resources are added to the same Xcode project, After Effects may only recognize one of them. This is typically due to incorrect PiPL configuration, resource ID conflicts, or the build process not properly including all PiPL resources in the final plugin binary. Ensure each PiPL has a unique resource ID, verify the matchname and name fields are correctly differentiated, and check that all PiPL resources are included in the target's Copy Bundle Resources build phase.

*Tags: `build`, `pipl`, `plugin-structure`, `xcode`*

---
