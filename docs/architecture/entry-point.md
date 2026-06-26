# Entry Point

> 2 Q&As · source: AE plugin dev community Discord

### Why might a plugin fail to load on some Macs with 'couldn't find main entry point' error, even without missing dependencies?

While 'couldn't find main entry point' usually means missing dependencies, it can also be caused by library collisions. For example, if your plugin links against curl, check whether you're providing your own dylib or linking against Apple's system version. On macOS, curl is normally relatively unproblematic -- you can usually safely use the lib provided by Apple. Also check whether the issue appears only on Silicon or Intel Macs.

*Tags: `curl`, `dylib`, `entry-point`, `library-collision`, `loading-error`, `macos`*

---

### What are the new entry points for After Effects plugins and how do they relate to PIPL?

After Effects plugins now have two entry points: EffectMainExtra and PluginDataEntryFunction, which are defined via PF_REGISTER_EFFECT in the samples. These new entry points may eventually replace PIPL, though there is no official announcement. PIPL is no longer used by Premiere Pro, so developers only need to write PIPL for After Effects. Some investigation suggests it may be possible to remove PIPL entirely by using these new entry points, though the exact mechanism is not yet fully documented.

*Tags: `EffectMain`, `aegp`, `entry-point`, `pipl`, `reference`*

---
