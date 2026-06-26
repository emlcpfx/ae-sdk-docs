# Install Key

> 2 Q&As · source: AE plugin dev community Discord

### Why does my std::map of install keys and matchnames return wrong effect keys?

The map was declared as std::map<A_char, A_long> which stores only a single A_char character, not the full string. The map key type should be a string type (like std::string) or at least an array of A_char to store the full matchname. Using just A_char means all matchnames starting with the same character would overwrite each other.

*Tags: `aegp`, `install-key`, `matchname`, `std-map`, `string-handling`*

---

### How can I prevent my effect from being added more than once to the same layer?

There's no outflag for this. The approach: (1) During global setup, find your effect's AEGP_InstalledEffectKey by iterating installed effects with AEGP_GetNumInstalledEffects, AEGP_GetNextInstalledEffect, and AEGP_GetEffectMatchName. (2) During UPDATE_PARAMS_UI (guaranteed to be called after applying), scan the effect's layer and check each effect's install key using AEGP_GetInstalledKeyFromLayerEffect. (3) If duplicates exist, alert the user, delete/disable the redundant, or skip rendering by copying input to output. You can also try AEGP_DisableCommand, but users can still duplicate or copy/paste the effect.

*Tags: `effect-detection`, `install-key`, `matchname`, `single-instance`, `update-params-ui`*

---
