# Ae

> 3 Q&As · source: AE plugin dev community Discord

### What are the new entry points in After Effects plugins and how do they relate to PIPL?

After Effects plugins now have two entry points: EffectMainExtra and PluginDataEntryFunction, which are defined via PF_REGISTER_EFFECT in the samples. These new entry points may eventually replace PIPL, though there is no official announcement. Currently, PIPL is still used by After Effects but not by Premiere Pro anymore, so you only need to write PIPL for AE. The new entry points potentially allow for removing PIPL/rsrc resources entirely, though the exact mechanism needs further investigation in the samples.

*Tags: `ae`, `aegp`, `pipl`, `plugin-architecture`*

---

### How did Adobe's variable font support in After Effects affect third-party VariFont plugins?

Tobias Fleischer (reduxFX) developed the VariFont plugin as the only way to use variable fonts in After Effects for an extended period. After years of waiting, Adobe finally implemented native variable font support in AE, making the third-party plugin less essential. Plugin developers anticipated this would eventually happen.

*Tags: `ae`, `deployment`, `reference`*

---

### When should I use PreRender and SmartRender in After Effects plugins, and what are their benefits?

PreRender (PF_Cmd_SMART_PRE_RENDER) and SmartRender (PF_Cmd_SMART_RENDER) are supported only in After Effects and are mandatory if you wish to support 32bpc (32-bit per channel) rendering. They also offer rendering pipeline improvements. However, if your plugin needs to work in Premiere Pro as well, you must support the old rendering pipeline with FrameSetup() and regular Render() calls instead.

*Tags: `ae`, `mfr`, `reference`, `render-loop`, `smart-render`*

---
