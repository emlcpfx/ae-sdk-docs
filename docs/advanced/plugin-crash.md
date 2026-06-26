# Plugin Crash

> 1 Q&A · source: AE plugin dev community Discord

### How can I move a layer from one composition to another or create a precomposition from a layer using AEGP calls?

You can trigger the precompose command just like any other After Effects command through AEGP. However, if you attempt to execute this during a PF_Cmd_USER_CHANGED_PARAM callback, it will crash because precomposing invalidates the effect from which you're currently operating. To avoid this crash, you need to call the precompose command from an AEGP idle hook instead, which executes the operation outside of the current effect's execution context.

*Tags: `aegp`, `callback`, `idle-hook`, `plugin-crash`, `precomp`*

---
