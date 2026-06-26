# Plugin Interaction

> 1 Q&A · source: AE plugin dev community Discord

### How can you debug a plugin crash that occurs during project loading in After Effects on Windows but not on Mac, seemingly triggered by an external library like Cineware?

Use breakpoints in EffectMain to trace the plugin's initialization sequence across GlobalSetup and ParamsSetup. If the crash occurs well after the plugin's code executes (with no breakpoints hit for several seconds), it suggests the plugin may be inadvertently affecting how other libraries like Cineware load, possibly through symbol name clashing. A workaround is to pre-load the problematic library (e.g., Cineware) onto a clip before opening the project. Additionally, try importing the project into a new empty project rather than directly opening it, as project corruption or platform-specific state issues may be the root cause. Cross-platform differences (Mac vs Windows) suggest OS-specific library loading or initialization issues.

*Tags: `cross-platform`, `debugging`, `macos`, `plugin-interaction`, `windows`*

---
