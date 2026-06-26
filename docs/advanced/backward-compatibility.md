# Backward Compatibility

> 3 Q&As · source: AE plugin dev community Discord

### How do you handle backward compatibility when adding new parameters to existing plugins?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag on new parameters. This flag tells AE to use the parameter's 'value' field (not 'dephault') when loading projects saved before this parameter existed. For more complex migration scenarios, add a hidden parameter storing a version number, and in sequence data setup, parse that version to reset values accordingly.

*Tags: `backward-compatibility`, `migration`, `params`, `pf-param-flag`, `versioning`*

---

### How can a plugin with a new mode parameter maintain backward compatibility with older projects that lack this parameter?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag on the parameter. This flag ensures that when opening projects created with older versions of the plugin that don't have the new parameter, After Effects will use the default value specified for that parameter rather than causing visual inconsistencies.

*Tags: `aegp`, `backward-compatibility`, `params`*

---

### How do you handle backward compatibility when adding new parameters with default values to an After Effects plugin?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag. This mechanism ensures that when opening older projects that don't have a new parameter, the plugin will use the specified default value without breaking the appearance of those projects. This is the standard way to handle parameter additions across plugin versions.

*Tags: `aegp`, `backward-compatibility`, `params`*

---
