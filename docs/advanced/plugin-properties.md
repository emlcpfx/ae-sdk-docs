# Plugin Properties

> 1 Q&A · source: AE plugin dev community Discord

### What is the function of the PiPL.r file in After Effects plugin development?

The PiPL.r (Plug In Property List) file was originally created to allow After Effects to get information about plug-ins without loading them, which was useful when computers had limited RAM (128MB). Nowadays it's mostly a legacy requirement. The values in the PiPL must correlate to the values set in the plugin's global setup call, or an error message will be sent during the plugin's launch indicating a mismatch. To set the correct values: put a breakpoint in the global setup function to see the numerical values applied to outflags and outflags2 variables, then copy these values to the PiPL file. Note that a clean rebuild is required for changes to take effect, as the pipl_tool only generates a new .rc file when one doesn't already exist.

*Tags: `build`, `deployment`, `pipl`, `plugin-properties`*

---
