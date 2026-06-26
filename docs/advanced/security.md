# Security

> 1 Q&A · source: AE plugin dev community Discord

### Should .aex plugins be code signed on Windows?

.aex plugins are regular DLLs (just renamed), so they can be codesigned. Microsoft currently allows unsigned DLLs to be loaded from a signed process, but they have hinted this might change in the future (was supposed to happen with Win11). On macOS, a plugin is a bundle (renamed folder), and you don't need to sign the actual plugin binary inside it as long as the bundle itself is signed.

*Tags: `aex`, `code-signing`, `dll`, `macos`, `security`, `windows`*

---
