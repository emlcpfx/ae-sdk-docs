# Ae Transition Extensions

> 1 Q&A · source: AE plugin dev community Discord

### How can I set default values for PF_Param_POINT parameters based on sequence resolution in After Effects plugins?

Point parameters in After Effects use percentage-based coordinates for their defaults, so a default value of {50, 50} will always represent the center of the composition or sequence, regardless of resolution. This means you don't need to dynamically set defaults based on sequence information during PF_Cmd_USER_CHANGED_PARAM or other callbacks—the percentage-based system handles this automatically.

*Tags: `ae transition extensions`, `params`, `pf_paramdef`, `premiere`*

---
