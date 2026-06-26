# Single Instance

> 1 Q&A · source: AE plugin dev community Discord

### How can I prevent my effect from being added more than once to the same layer?

There's no outflag for this. The approach: (1) During global setup, find your effect's AEGP_InstalledEffectKey by iterating installed effects with AEGP_GetNumInstalledEffects, AEGP_GetNextInstalledEffect, and AEGP_GetEffectMatchName. (2) During UPDATE_PARAMS_UI (guaranteed to be called after applying), scan the effect's layer and check each effect's install key using AEGP_GetInstalledKeyFromLayerEffect. (3) If duplicates exist, alert the user, delete/disable the redundant, or skip rendering by copying input to output. You can also try AEGP_DisableCommand, but users can still duplicate or copy/paste the effect.

*Tags: `effect-detection`, `install-key`, `matchname`, `single-instance`, `update-params-ui`*

---
