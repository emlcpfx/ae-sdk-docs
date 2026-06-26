# Pf Param Flag

> 1 Q&A · source: AE plugin dev community Discord

### How do you handle backward compatibility when adding new parameters to existing plugins?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag on new parameters. This flag tells AE to use the parameter's 'value' field (not 'dephault') when loading projects saved before this parameter existed. For more complex migration scenarios, add a hidden parameter storing a version number, and in sequence data setup, parse that version to reset values accordingly.

*Tags: `backward-compatibility`, `migration`, `params`, `pf-param-flag`, `versioning`*

---
