# Image

> 1 Q&A · source: AE plugin dev community Discord

### How can I validate image files before importing them into After Effects through a plugin?

There are two approaches: (1) Check the file using OS tools—both GDI+ (Windows) and Quartz (macOS) offer tools for reading image files in various formats to validate them before importing. (2) Use After Effects to check—use AEGP_StartQuietErrors to suppress user-visible errors, perform a test render to detect errors silently, then call AEGP_EndQuietErrors when done. Based on the result, you can decide whether to dispose of the image or notify the user.

*Tags: `aegp`, `debugging`, `image`, `macos`, `validation`, `windows`*

---
