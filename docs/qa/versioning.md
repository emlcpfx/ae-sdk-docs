# Q&A: versioning

**5 entries** tagged with `versioning`.

---

## How do you handle backward compatibility when adding new parameters to existing plugins?

Use the PF_ParamFlag_USE_VALUE_FOR_OLD_PROJECTS flag on new parameters. This flag tells AE to use the parameter's 'value' field (not 'dephault') when loading projects saved before this parameter existed. For more complex migration scenarios, add a hidden parameter storing a version number, and in sequence data setup, parse that version to reset values accordingly.

*Contributors: [**rowbyte**](../contributors/rowbyte/), [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/) · Source: adobe-plugin-devs · 2025-01-16 · Tags: `backward-compatibility`, `params`, `versioning`, `pf-param-flag`, `migration`*

---

## How can you handle migration of sequence data when the plugin format changes over versions?

Add an extra hidden parameter that stores the version number when sequence data is saved. When loading, parse this version information and reset values accordingly. This approach becomes especially useful when you have complex migration logic to handle multiple format versions.

*Tags: `sequence-data`, `params`, `arb-data`, `versioning`*

---

## How should parameter IDs be numbered to avoid issues when reordering parameters between plugin versions?

Parameter IDs in the non-_ID enum should start from 1, not 0. Value 0 is reserved for the default invisible layer parameter that serves as the effect's input. If you start numbering IDs from 0 (NULL), After Effects may assume you haven't bothered to ID params and will auto-assign IDs, which causes problems when reordering parameters in new versions. Always explicitly set parameter IDs starting from 1 to maintain backwards compatibility.

```cpp
enum
{
  MOVEMENT_DIRECTION_ID = 1,
  AMOUNT_SLIDER_ID = 2,
  NEW_FIELD_SLIDER_ID = 3
}
```

*Tags: `params`, `sdk`, `versioning`, `backwards-compatibility`*

---

## What is the recommended approach for changing parameter order in newer versions of an After Effects plugin while preserving saved project data?

Adobe's AE Plugin SDK Guide provides detailed documentation on changing parameter orders: https://ae-plugin-sdk-guide.readthedocs.io/effect-details/changing-parameter-orders.html. The guide explains how to properly handle parameter ID assignment and reordering to maintain backwards compatibility with projects saved using older plugin versions.

*Tags: `params`, `versioning`, `reference`, `sdk`*

---

## How can you track which version of a plugin was used to save an After Effects project?

There is no dedicated API in the After Effects SDK to retrieve the plugin version that saved a project. The recommended approach is to store the version number in the sequence data of your effect. This allows you to detect version changes when reopening projects and handle updates accordingly.

*Tags: `sequence-data`, `params`, `aegp`, `versioning`*

---
