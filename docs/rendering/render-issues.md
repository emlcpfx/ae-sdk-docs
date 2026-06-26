# Render Issues

> 1 Q&A · source: AE plugin dev community Discord

### Why does Media Encoder not render plugins correctly or fail to load AEGP plugins?

Media Encoder does not load AEGP plugins. If your effect plugin relies on a companion AEGP plugin for rendering, it will fail in Media Encoder. Issues may also be related to static global variables or elements defined in GlobalData/GlobalSetup. The AEGP functionality would need to be incorporated directly into the effect plugin, or you'd need a standalone/command-line tool alternative.

*Tags: `aegp`, `global-data`, `media-encoder`, `render-issues`*

---
