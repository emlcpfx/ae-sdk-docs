# Ae Effect Version

> 1 Q&A · source: AE plugin dev community Discord

### How does AE_Effect_Version work in the PiPL resource file?

AE_Effect_Version encodes version information using bit shifting. The formula is: (MAJOR_VERSION << 19) + (MINOR_VERSION << 15) + (BUG_VERSION << 11) + BUILD_VERSION. Note that MAJOR_VERSION may not be larger than 8 due to bit width constraints. The full PF_VERSION macro in AE_Effect.h provides more detail with additional fields for stage and build. If you get a PiPL version mismatch error, output the version you have in code for global setup and compare it to the one in the PiPL using a hex editor.

```cpp
#define PF_VERSION(vers, subvers, bugvers, stage, build)    \
    (PFVersionInfo)(    \
         ((((A_u_long)PF_Vers_VERS_HIGH(vers)) & PF_Vers_VERS_HIGH_BITS) << PF_Vers_VERS_HIGH_SHIFT) |   \
        ((((A_u_long)(vers)) & PF_Vers_VERS_BITS) << PF_Vers_VERS_SHIFT) |    \
        ((((A_u_long)(subvers)) & PF_Vers_SUBVERS_BITS)<<PF_Vers_SUBVERS_SHIFT) |\
        ((((A_u_long)(bugvers)) & PF_Vers_BUGFIX_BITS) << PF_Vers_BUGFIX_SHIFT) |\
        ((((A_u_long)(stage)) & PF_Vers_STAGE_BITS) << PF_Vers_STAGE_SHIFT) |    \
        ((((A_u_long)(build)) & PF_Vers_BUILD_BITS) << PF_Vers_BUILD_SHIFT)    \
    )
```

*Tags: `ae-effect-version`, `bit-shifting`, `pf-version`, `pipl`, `version`*

---
