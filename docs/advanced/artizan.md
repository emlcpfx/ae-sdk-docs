# Artizan

> 1 Q&A · source: AE plugin dev community Discord

### Can I write a 3D importer plugin for After Effects that behaves like the .obj importer with 3D transforms and depth buffer?

No, this is not possible. Importer AEGPs (known as IO and FBIO) only parse a requested frame and pass it to AE as an image. 3D behavior is handled by the Artizan plugin which does full comp rendering. You could try writing an Artizan plugin, but you'd have to handle all other rendering as well, making it impractical.

*Tags: `3d-import`, `artizan`, `fbio`, `importer`, `io-plugin`*

---
