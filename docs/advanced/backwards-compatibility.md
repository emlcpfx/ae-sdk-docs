# Backwards Compatibility

> 2 Q&As · source: AE plugin dev community Discord

### Should obsolete CPU architecture fields have been removed from in_data when After Effects went 64-bit only?

Yes, the fields in in_data related to old CPU architecture detection (such as Gestalt values for 68x and PowerPC) should have been removed from the plugin API when Adobe transitioned to 64-bit only support around CS5, rather than keeping legacy fields that are no longer used or meaningful.

*Tags: `aegp`, `backwards-compatibility`, `deprecated`, `macos`, `windows`*

---

### How should parameter IDs be numbered to avoid issues when reordering parameters between plugin versions?

Parameter IDs in the non-_ID enum should start from 1, not 0. Value 0 is reserved for the default invisible layer parameter that serves as the effect's input. If you start numbering IDs from 0 (NULL), After Effects may assume you haven't bothered to ID params and will auto-assign IDs, which causes problems when reordering parameters in new versions. Always explicitly set parameter IDs starting from 1 to maintain backwards compatibility.

```cpp
enum
{
  MOVEMENT_DIRECTION_ID = 1,
  AMOUNT_SLIDER_ID = 2,
  NEW_FIELD_SLIDER_ID = 3
}
```

*Tags: `backwards-compatibility`, `params`, `sdk`, `versioning`*

---
