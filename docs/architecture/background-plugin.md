# Background Plugin

> 1 Q&A · source: AE plugin dev community Discord

### How can I make an AEGP plugin invisible (not appear in the Window menu)?

Simply don't use AEGP_InsertMenuCommand or AEGP_RegisterCommandHook. These are only needed to create a menu entry. Without them, the AEGP will load and run in the background without any UI presence.

*Tags: `aegp`, `background-plugin`, `invisible-plugin`, `menu`*

---
