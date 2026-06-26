# Plugin Version Encoding: PF_VERSION Macro

## Overview

Every After Effects plugin reports a version number to the host. This version is packed into a single 32-bit unsigned integer (`A_u_long`) using the `PF_VERSION` macro defined in `AE_Effect.h`. The encoding uses bit fields to store five components: major version, minor version (subvers), bug fix version, development stage, and build number.

This version number is set during `PF_Cmd_GLOBAL_SETUP` via `out_data->my_version` and must match the version declared in the plugin's PiPL resource under the `AE_Effect_Version` property.

## The PF_VERSION Macro

```c
#define PF_VERSION(vers, subvers, bugvers, stage, build) \
    (PFVersionInfo)(                                      \
        ((((A_u_long)PF_Vers_VERS_HIGH(vers)) & PF_Vers_VERS_HIGH_BITS) << PF_Vers_VERS_HIGH_SHIFT) | \
        ((((A_u_long)(vers)) & PF_Vers_VERS_BITS) << PF_Vers_VERS_SHIFT) |                            \
        ((((A_u_long)(subvers)) & PF_Vers_SUBVERS_BITS) << PF_Vers_SUBVERS_SHIFT) |                    \
        ((((A_u_long)(bugvers)) & PF_Vers_BUGFIX_BITS) << PF_Vers_BUGFIX_SHIFT) |                     \
        ((((A_u_long)(stage)) & PF_Vers_STAGE_BITS) << PF_Vers_STAGE_SHIFT) |                         \
        ((((A_u_long)(build)) & PF_Vers_BUILD_BITS) << PF_Vers_BUILD_SHIFT)                            \
    )
```

### Parameters

| Parameter | Type | Range | Description |
|-----------|------|-------|-------------|
| `vers` | Major version | 0--127 | Primary version number |
| `subvers` | Minor version | 0--15 | Minor/sub version |
| `bugvers` | Bug fix version | 0--15 | Patch/bug fix level |
| `stage` | Development stage | 0--3 | One of the `PF_Stage_*` constants |
| `build` | Build number | 0--511 | Build/revision number |

### Return Type

The macro returns a `PFVersionInfo`, which is a typedef for `A_u_long` (32-bit unsigned integer).

## Development Stage Constants

```c
enum {
    PF_Stage_DEVELOP,   // 0
    PF_Stage_ALPHA,     // 1
    PF_Stage_BETA,      // 2
    PF_Stage_RELEASE    // 3
};
typedef A_long PF_Stage;
```

Use `PF_Stage_RELEASE` for shipping plugins. The other values are primarily for your own tracking -- After Effects does not change behavior based on the stage value, but it is encoded into the version integer and can be inspected by other tools.

## Bit-Packing Layout

The 32-bit version integer is divided into the following fields:

```
Bit 31  30  29  28  27  26  25  24  23  22  21  20  19  18  17  16  15  14  13  12  11  10   9   8   7   6   5   4   3   2   1   0
 |---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
 |unused | VERS_HIGH (4 bits) |  unused  | VERS_LOW (3) | SUBVERS (4 bits)  | BUGFIX (4 bits)   |STAGE| BUILD (9 bits)        |
```

### Field Definitions

| Field | Bits | Shift | Mask | Width | Max Value |
|-------|------|-------|------|-------|-----------|
| BUILD | 0--8 | 0 | `0x1FF` | 9 bits | 511 |
| STAGE | 9--10 | 9 | `0x3` | 2 bits | 3 |
| BUGFIX | 11--14 | 11 | `0xF` | 4 bits | 15 |
| SUBVERS | 15--18 | 15 | `0xF` | 4 bits | 15 |
| VERS (low) | 19--21 | 19 | `0x7` | 3 bits | 7 |
| (bits 22--25) | 22--25 | -- | -- | -- | (skipped/unused) |
| VERS_HIGH | 26--29 | 26 | `0xF` | 4 bits | 15 |
| (bits 30--31) | 30--31 | -- | -- | -- | (unused) |

### How the Major Version is Split

The major version field is the most unusual part of the encoding. It is split across two non-contiguous bit fields:

- **VERS (low 3 bits):** bits 19--21, holding `vers & 0x7`
- **VERS_HIGH (upper 4 bits):** bits 26--29, holding `vers >> 3`

The full major version is reconstructed as:

```
major = VERS_LOW + (VERS_HIGH << 3)
```

This allows major versions from 0 to 127 (7 bits total). The split exists for backward compatibility -- the original encoding only had 3 bits for the version (max 7). The `VERS_HIGH` bits were added later to extend the range without breaking existing projects that only read the low 3 bits.

The helper macro used during encoding:

```c
#define PF_Vers_VERS_LOW_SHIFT  3
#define PF_Vers_VERS_HIGH(vers) ((vers) >> PF_Vers_VERS_LOW_SHIFT)
```

## Extraction Macros

To decompose a packed version integer back into its components:

```c
#define PF_Version_VERS(vers)    \
    (((((A_u_long) vers) >> PF_Vers_VERS_SHIFT) & PF_Vers_VERS_BITS) + \
     (((vers >> PF_Vers_VERS_HIGH_SHIFT) & PF_Vers_VERS_HIGH_BITS) << PF_Vers_VERS_LOW_SHIFT))

#define PF_Version_SUBVERS(vers)  \
    ((((A_u_long) vers) >> PF_Vers_SUBVERS_SHIFT) & PF_Vers_SUBVERS_BITS)

#define PF_Version_BUGFIX(vers)   \
    ((((A_u_long) vers) >> PF_Vers_BUGFIX_SHIFT) & PF_Vers_BUGFIX_BITS)

#define PF_Version_STAGE(vers)    \
    ((((A_u_long) vers) >> PF_Vers_STAGE_SHIFT) & PF_Vers_STAGE_BITS)

#define PF_Version_BUILD(vers)    \
    ((((A_u_long) vers) >> PF_Vers_BUILD_SHIFT) & PF_Vers_BUILD_BITS)
```

| Macro | Returns | Description |
|-------|---------|-------------|
| `PF_Version_VERS(v)` | 0--127 | Major version (recombined from split fields) |
| `PF_Version_SUBVERS(v)` | 0--15 | Minor version |
| `PF_Version_BUGFIX(v)` | 0--15 | Bug fix version |
| `PF_Version_STAGE(v)` | 0--3 | Development stage |
| `PF_Version_BUILD(v)` | 0--511 | Build number |

## Relationship to PiPL AE_Effect_Version

The PiPL resource for an After Effects effect plugin must contain an `AE_Effect_Version` property whose value is the same packed 32-bit integer produced by `PF_VERSION`. This value is read by After Effects at plugin load time, before `PF_Cmd_GLOBAL_SETUP` is ever called.

**The PiPL version and the version set in `PF_Cmd_GLOBAL_SETUP` must match.** If they differ, After Effects may exhibit unpredictable behavior, including refusing to load the plugin or misidentifying it.

In a PiPL resource file (`.r` format):

```c
AE_Effect_Version {
    PF_VERSION(2, 5, 1, PF_Stage_RELEASE, 42)
}
```

In the Windows `.rc` resource equivalent, this appears as the raw 32-bit integer value.

## Code Examples

### Setting the Version in Global Setup

```c
static PF_Err
GlobalSetup(
    PF_InData   *in_data,
    PF_OutData  *out_data)
{
    out_data->my_version = PF_VERSION(
        3,                  // Major version 3
        1,                  // Minor version 1
        0,                  // Bug fix 0
        PF_Stage_RELEASE,   // Release build
        128                 // Build number 128
    );

    return PF_Err_NONE;
}
```

### Extracting Version Components

```c
static void
LogVersion(
    PF_InData   *in_data,
    PFVersionInfo packed_version)
{
    A_char buf[128];

    A_long major = PF_Version_VERS(packed_version);
    A_long minor = PF_Version_SUBVERS(packed_version);
    A_long bugfix = PF_Version_BUGFIX(packed_version);
    A_long stage = PF_Version_STAGE(packed_version);
    A_long build = PF_Version_BUILD(packed_version);

    const char *stage_str;
    switch (stage) {
        case PF_Stage_DEVELOP: stage_str = "dev";     break;
        case PF_Stage_ALPHA:   stage_str = "alpha";   break;
        case PF_Stage_BETA:    stage_str = "beta";    break;
        case PF_Stage_RELEASE: stage_str = "release"; break;
        default:               stage_str = "unknown"; break;
    }

    PF_SPRINTF(buf, "Version %d.%d.%d %s (build %d)",
               major, minor, bugfix, stage_str, build);
}
```

### Defining Version Constants for Consistency

A common pattern is to define your version components as constants and use them in both the PiPL and the code:

```c
// In a shared header, e.g., MyPlugin_Version.h
#define MYPLUGIN_MAJOR      2
#define MYPLUGIN_MINOR      5
#define MYPLUGIN_BUGFIX     1
#define MYPLUGIN_STAGE      PF_Stage_RELEASE
#define MYPLUGIN_BUILD      42

#define MYPLUGIN_VERSION    PF_VERSION(             \
    MYPLUGIN_MAJOR,                                 \
    MYPLUGIN_MINOR,                                 \
    MYPLUGIN_BUGFIX,                                \
    MYPLUGIN_STAGE,                                 \
    MYPLUGIN_BUILD)
```

Then in your PiPL resource:

```c
#include "MyPlugin_Version.h"

AE_Effect_Version { MYPLUGIN_VERSION }
```

And in your code:

```c
out_data->my_version = MYPLUGIN_VERSION;
```

This guarantees the PiPL and runtime values always agree.

### Comparing Versions

```c
static PF_Boolean
IsNewerThan(PFVersionInfo a, PFVersionInfo b)
{
    // Compare major, then minor, then bugfix
    A_long a_major = PF_Version_VERS(a);
    A_long b_major = PF_Version_VERS(b);
    if (a_major != b_major) return a_major > b_major;

    A_long a_minor = PF_Version_SUBVERS(a);
    A_long b_minor = PF_Version_SUBVERS(b);
    if (a_minor != b_minor) return a_minor > b_minor;

    A_long a_bug = PF_Version_BUGFIX(a);
    A_long b_bug = PF_Version_BUGFIX(b);
    if (a_bug != b_bug) return a_bug > b_bug;

    A_long a_stage = PF_Version_STAGE(a);
    A_long b_stage = PF_Version_STAGE(b);
    if (a_stage != b_stage) return a_stage > b_stage;

    return PF_Version_BUILD(a) > PF_Version_BUILD(b);
}
```

> **Warning:** Do not compare packed version integers directly with `>` or `<`. Because the major version is split across non-contiguous bit fields, a simple integer comparison does not produce correct ordering. Always extract the components and compare field by field as shown above.

## Pitfalls and Warnings

> **Pitfall: Version overflow.** The major version supports 0--127, minor and bugfix support 0--15, and build supports 0--511. If you exceed these ranges, the values are silently truncated by the bit mask. For example, `PF_VERSION(200, 0, 0, PF_Stage_RELEASE, 0)` will encode major version as `200 & 0x7F = 72` (low bits) split across the two fields, which gives 72 -- not 200.

> **Pitfall: PiPL/code mismatch.** If the `AE_Effect_Version` in your PiPL does not match `out_data->my_version` set in `PF_Cmd_GLOBAL_SETUP`, After Effects may fail to load the plugin or exhibit unexpected behavior. Always use a shared header for version constants.

> **Pitfall: Raw integer comparison.** The split major version encoding means that `PF_VERSION(8, 0, 0, PF_Stage_RELEASE, 0)` is NOT greater than `PF_VERSION(7, 0, 0, PF_Stage_RELEASE, 0)` when compared as plain integers, because version 8 has its high bit in the VERS_HIGH field at bit 26. Always use the extraction macros for comparisons.

> **Pitfall: Stage value confusion.** `PF_Stage_DEVELOP` is 0, not `PF_Stage_RELEASE`. A common mistake is using 0 thinking it means "release." Always use the named constants.

## See Also

- `AE_Effect.h` -- Version macro and constant definitions
- PiPL resource documentation for `AE_Effect_Version` property format
- `out_data->my_version` in `PF_Cmd_GLOBAL_SETUP`
