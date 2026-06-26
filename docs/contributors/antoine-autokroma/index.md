# Antoine_Autokroma

**3 contributions** to AE SDK community knowledge.

Top topics: `premiere-pro`, `bit-depth`, `dogears`, `debugging`, `cep-extensions`, `shared-plugin`, `installation`, `dependency-management`, `aescripts`, `wrapper`

---

## How should you handle shared C++ plugins that are used by multiple CEP extensions to avoid uninstall conflicts?

Several approaches: (1) Use the manifest system to flag dependencies for each product, along with a plugin version, and include the version in the plugin name to allow side-by-side installation. (2) Compile the same plugin source multiple times with different build flags that make each compiled plugin used only by one panel/product. (3) Bundle all products together with one merged versioning so users install/uninstall all at once. The core problem is that uninstalling any individual extension currently also uninstalls the shared plugin, breaking other extensions.

*Source: aescripts discord · 2025-10-13 · Tags: `cep-extensions`, `shared-plugin`, `installation`, `dependency-management`, `aescripts` · [View in Q&A](../qa/cep-extensions/)*

---

## How do you view the current pixel bit depth in Premiere Pro?

Use the DogEars feature. You can look up the keyboard shortcut online, or edit the debug database to enable it. This feature is only available in Premiere Pro, not in After Effects.

*Source: aescripts discord · 2024-10-02 · Tags: `premiere-pro`, `bit-depth`, `dogears`, `debugging` · [View in Q&A](../qa/premiere-pro/)*

---

## Are there any good C++ wrapper libraries for the After Effects Filter GUI API?

No well-known general-purpose wrapper exists. Most wrappers that have been built end up being very complex themselves if you try to handle all use cases -- you end up with a non-standard system that is almost as complex as the original, just with more C++ features. Copy-pasting C chunks from the SDK examples works well in practice. That said, even a C wrapper with automatic memory management and state stored in structs would be an improvement over the current state machine. There is also a Rust binding project (https://github.com/virtualritz/after-effects/) that significantly reduces boilerplate compared to C/C++.

*Source: aescripts discord · 2024-09-27 · Tags: `wrapper`, `gui-api`, `cpp`, `rust`, `boilerplate`, `sdk` · [View in Q&A](../qa/wrapper/)*

---
