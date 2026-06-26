# Loading Error

> 1 Q&A · source: AE plugin dev community Discord

### Why might a plugin fail to load on some Macs with 'couldn't find main entry point' error, even without missing dependencies?

While 'couldn't find main entry point' usually means missing dependencies, it can also be caused by library collisions. For example, if your plugin links against curl, check whether you're providing your own dylib or linking against Apple's system version. On macOS, curl is normally relatively unproblematic -- you can usually safely use the lib provided by Apple. Also check whether the issue appears only on Silicon or Intel Macs.

*Tags: `curl`, `dylib`, `entry-point`, `library-collision`, `loading-error`, `macos`*

---
