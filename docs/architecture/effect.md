# Effect

> 1 Q&A · source: AE plugin dev community Discord

### What should be passed for the AEGP_PluginID argument when calling AEGP_New from within an effect plugin?

Passing NULL for the AEGP_PluginID argument works in effect plugins, though this may not be the recommended approach. The AEGP_PluginID is normally required by AEGP functions, but NULL can be used as a workaround when calling AEGP_New from within an effect context.

*Tags: `aegp`, `effect`, `pluginid`, `sdk`*

---
