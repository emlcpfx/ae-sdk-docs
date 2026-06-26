# Effectmain

> 1 Q&A · source: AE plugin dev community Discord

### What are the new entry points for After Effects plugins and how do they relate to PIPL?

After Effects plugins now have two entry points: EffectMainExtra and PluginDataEntryFunction, which are defined via PF_REGISTER_EFFECT in the samples. These new entry points may eventually replace PIPL, though there is no official announcement. PIPL is no longer used by Premiere Pro, so developers only need to write PIPL for After Effects. Some investigation suggests it may be possible to remove PIPL entirely by using these new entry points, though the exact mechanism is not yet fully documented.

*Tags: `EffectMain`, `aegp`, `entry-point`, `pipl`, `reference`*

---
