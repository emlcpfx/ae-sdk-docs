# Multiple Plugins

> 1 Q&A · source: AE plugin dev community Discord

### Can you have multiple effect plugins in one .aex file?

The SDK documentation says it's possible using multiple PiPLs, with AEGPs placed first. However, it's not recommended because: (1) No other hosts including Premiere Pro support multiple PiPLs. (2) If you need to update one plugin, you'd have to ship a new build of all plugins. In practice, adding another PiPL in Xcode with a different name/matchname may only show one plugin. Adobe recommends one PiPL per code fragment.

*Tags: `aex`, `architecture`, `multiple-plugins`, `pipl`*

---
