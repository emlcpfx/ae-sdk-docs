# Identification

> 1 Q&A · source: AE plugin dev community Discord

### How can I identify a specific effect instance using AEGP_EffectCallGeneric and PF_Cmd_COMPLETELY_GENERAL?

AEGP_EffectCallGeneric has a limitation: you cannot call it from an effect to another instance of the same effect type. Even bouncing through a separate AEGP via a custom suite doesn't work if the call chain originates from the same effect type. For instance identification, alternatives include: (1) Put an ID in sequence data and access via AEGP_EffectCallGeneric from a different effect type. (2) Change a hidden param's name to serve as identifier using AEGP_SetStreamName() (doesn't survive save/load). (3) Use a separate AEGP to handle identification during idle processing.

*Tags: `aegp`, `completely-general`, `effect-instance`, `identification`, `sequence-data`*

---
