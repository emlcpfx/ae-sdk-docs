# Importer

> 3 Q&As · source: AE plugin dev community Discord

### Can I write a 3D importer plugin for After Effects that behaves like the .obj importer with 3D transforms and depth buffer?

No, this is not possible. Importer AEGPs (known as IO and FBIO) only parse a requested frame and pass it to AE as an image. 3D behavior is handled by the Artizan plugin which does full comp rendering. You could try writing an Artizan plugin, but you'd have to handle all other rendering as well, making it impractical.

*Tags: `3d-import`, `artizan`, `fbio`, `importer`, `io-plugin`*

---

### Can a custom 3D importer be written to behave like the .obj importer with 3D transforms and depth buffer support?

No, this is not possible with standard importer AEGPS (IO and FBIO plugins). Importers only parse a requested frame and pass it to After Effects as an image. True 3D behavior is handled by Artisan plugins, which perform full composition rendering. To achieve 3D import functionality, you would need to write an Artisan plugin, but this requires handling all rendering yourself, not just the 3D import.

*Tags: `3d`, `aegp`, `importer`, `reference`*

---

### How should an importer plugin interact with an effect plugin to share custom file data?

An importer plugin allows custom file types to be imported into After Effects and placed in compositions. There are two main scenarios for importer-to-effect interaction: (1) The importer parses the file on demand and populates an image buffer, which can then be read by the effect like any other layer source. After Effects can cache the result for faster access on subsequent frame renders. This approach expands usability for custom file types. (2) The effect uses the layer selector only to get the selected layer's ID, retrieves the source project item and file path from it, and then parses the file directly in the effect. This approach allows for data files without graphic interpretation and enables selective storage of data rather than loading all file data into RAM. Both scenarios are valid, and you can combine both approaches—parsing in the importer while also allowing the effect to fetch and parse the file path independently.

*Tags: `aegp`, `arb-data`, `caching`, `importer`, `layer-checkout`*

---
