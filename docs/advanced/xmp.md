# Xmp

> 1 Q&A · source: AE plugin dev community Discord

### What is the XMP Packet and can it be used to store plugin data in AE projects?

XMP Packets can be read/written via ExtendScript to store custom metadata in AE projects. However, limitations include: the data is not part of AE's undo stack, practical size limit around 500KB (though AE itself can bloat XMP to 50MB+), and it's not undoable. AE Viewer tools exist that can strip XMP to reduce file sizes dramatically (e.g., 50MB to 50KB).

*Tags: `extendscript`, `metadata`, `project-data`, `storage`, `xmp`*

---
