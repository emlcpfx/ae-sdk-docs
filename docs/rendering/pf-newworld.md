# Pf_Newworld

> 1 Q&A · source: AE plugin dev community Discord

### Why does my After Effects plugin cause memory to balloon to 4-5 GB when changing parameters?

This can happen for two main reasons: (1) After Effects caching images as a user preference, which generally shouldn't be a concern, or (2) your plugin uses PF_NewWorld, which in some AE versions doesn't free memory even when PF_DisposeWorld is called. The memory is only freed when the user manually purges all caches through the menu. To fix this, consider allocating memory through the memory suite instead of PF_NewWorld, and release it after processing. Alternatively, you can point a PF_EffectWorld's 'data' parameter to arbitrarily allocated memory by filling in the other relevant data fields.

*Tags: `aegp`, `caching`, `debugging`, `memory`, `pf_newworld`*

---
