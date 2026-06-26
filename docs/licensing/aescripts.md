# Aescripts

> 2 Q&As · source: AE plugin dev community Discord

### How should you handle shared C++ plugins that are used by multiple CEP extensions to avoid uninstall conflicts?

Several approaches: (1) Use the manifest system to flag dependencies for each product, along with a plugin version, and include the version in the plugin name to allow side-by-side installation. (2) Compile the same plugin source multiple times with different build flags that make each compiled plugin used only by one panel/product. (3) Bundle all products together with one merged versioning so users install/uninstall all at once. The core problem is that uninstalling any individual extension currently also uninstalls the shared plugin, breaking other extensions.

*Tags: `aescripts`, `cep-extensions`, `dependency-management`, `installation`, `shared-plugin`*

---

### Do Mac plugins need to be notarized in addition to codesigning for distribution via the aescripts app?

Yes, codesigning alone is not enough. The plugins also need to be notarized by Apple for distribution.

*Tags: `aescripts`, `code-signing`, `distribution`, `macos`, `notarization`*

---
