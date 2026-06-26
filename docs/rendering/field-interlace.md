# Field and Interlace Handling: FIEL_Public.h

## Overview

`FIEL_Public.h` defines a standard way to communicate interlace information within image file formats. Originally created by CoSA (the Company of Science and Art, which built After Effects before Adobe acquired it), this header dates back to AE 1.0 and has remained remarkably stable. The `FIEL_Label` structure is embedded in QuickTime movies as user data, in image files as a resource, and can be appended to ImageDescriptions.

Interlacing is a legacy video technique where each frame is divided into two fields -- one containing the even-numbered scan lines and one containing the odd-numbered scan lines. These fields are captured and displayed at different times, creating smoother motion on CRT displays at the cost of visible "combing" artifacts on progressive displays. While modern production is predominantly progressive, plugin developers must still handle interlaced footage correctly because:

1. Legacy footage and archives remain in active use
2. Broadcast delivery standards in some regions still require interlaced output
3. AE must correctly interpret field order to de-interlace or re-interlace footage

---

## The FIEL_Label Structure

```c
#define FIEL_Label_VERSION   1
#define FIEL_Tag             'FIEL'   // Use as user data and resource type
#define FIEL_ResID           128      // Resource ID for FIEL resource in files

#pragma pack(push, CoSAalign, 2)
typedef struct {
    uint32_t    signature;    // Always FIEL_Tag ('FIEL')
    int16_t     version;      // FIEL_Label_VERSION (currently 1)
    FIEL_Type   type;         // Frame type (progressive, interlaced, etc.)
    FIEL_Order  order;        // Field order (upper or lower first)
    uint32_t    reserved;     // Reserved for future use
} FIEL_Label;
#pragma pack(pop, CoSAalign)
```

The structure is packed to 2-byte alignment and has a verified size of 18 bytes (enforced by `AE_STRUCT_SIZE_ASSERT` in internal builds).

### Field Layout

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 4 bytes | `signature` | Always `'FIEL'` (0x4649454C) |
| 4 | 2 bytes | `version` | Label version, currently 1 |
| 6 | 4 bytes | `type` | `FIEL_Type` enum value |
| 10 | 4 bytes | `order` | `FIEL_Order` enum value |
| 14 | 4 bytes | `reserved` | Must be 0 |

> **Pitfall:** Note the 2-byte packing. Without the `#pragma pack(push, CoSAalign, 2)` directive, compilers may insert padding after the `version` field, breaking the 18-byte layout and corrupting data read from files.

---

## FIEL_Type: Frame Rendering Types

```c
enum {
    FIEL_Type_FRAME_RENDERED   = 0,   // Progressive (FIEL_Order is irrelevant)
    FIEL_Type_INTERLACED       = 1,   // Full-height interlaced frame
    FIEL_Type_HALF_HEIGHT      = 2,   // Half-height field-rendered
    FIEL_Type_FIELD_DOUBLED    = 3,   // 60fps full-size field-doubled frames
    FIEL_Type_UNSPECIFIED      = 4    // Do not use!
};
typedef uint32_t FIEL_Type;
```

### Type Descriptions

#### FIEL_Type_FRAME_RENDERED (0) -- Progressive

The frame was rendered as a single, complete image at one point in time. There is no temporal difference between scan lines. The `FIEL_Order` field is irrelevant for this type.

This is the most common type in modern production. All computer-generated content, most digital cinema footage, and web video are frame-rendered.

#### FIEL_Type_INTERLACED (1) -- Interlaced Full-Height

The frame contains two temporally distinct fields woven together into a single full-height image. One field contains the even scan lines and the other contains the odd scan lines, captured at slightly different times (typically 1/60th of a second apart for 30fps interlaced video).

This is the standard format for broadcast television (NTSC, PAL) and DV footage. The `FIEL_Order` field specifies which field was captured first.

**Visual representation of an interlaced frame:**
```
Line 0: -------- Field A (captured at time T)
Line 1: ======== Field B (captured at time T + 1/60s)
Line 2: -------- Field A
Line 3: ======== Field B
Line 4: -------- Field A
Line 5: ======== Field B
  ...
```

#### FIEL_Type_HALF_HEIGHT (2) -- Half-Height Field

The frame contains only one field and is literally half the vertical resolution of the full frame. This is less common and represents a single field stored as a half-height image.

#### FIEL_Type_FIELD_DOUBLED (3) -- Field Doubled

Each field has been line-doubled to produce a full-height frame at field rate. For NTSC, this gives 60 full-size frames per second, where each frame is derived from a single field with every line duplicated to fill the gaps.

#### FIEL_Type_UNSPECIFIED (4) -- Do Not Use

Marked explicitly in the header as "do not use!" This value indicates the interlace information is unknown. AE may fall back to user-configured interpretation settings when encountering this value.

> **Warning:** Most applications only support `FIEL_Type_INTERLACED` for interlaced content and `FIEL_Type_FRAME_RENDERED` for progressive content. The header notes that the other formats are included for completeness, but storing field-rendered video in `HALF_HEIGHT` or `FIELD_DOUBLED` format may result in improper de-interlacing by most applications.

---

## FIEL_Order: Field Dominance

```c
enum {
    FIEL_Order_UPPER_FIRST = 0,   // Upper field (even lines) is temporally first
    FIEL_Order_LOWER_FIRST = 1    // Lower field (odd lines) is temporally first
};
typedef uint32_t FIEL_Order;
```

Field order (also called "field dominance") specifies which field was captured or should be displayed first in time.

### Upper Field First (FIEL_Order_UPPER_FIRST = 0)

The upper field (even-numbered scan lines, starting from line 0) was captured first. This is also called "even field first" or "field 1 dominant."

Common with:
- DV and HDV formats
- Most modern HD interlaced formats (1080i)
- Some professional broadcast equipment

### Lower Field First (FIEL_Order_LOWER_FIRST = 1)

The lower field (odd-numbered scan lines) was captured first. This is also called "odd field first" or "field 2 dominant."

Common with:
- Older analog capture cards
- Some SD broadcast formats
- Legacy NTSC equipment

> **Pitfall:** Getting field order wrong produces a distinctive "judder" or "stutter" in motion. Objects moving smoothly will appear to jump back and forth by one field's time offset. If you see this artifact, the field order is inverted.

### When FIEL_Order Matters

The order field is only meaningful when `type` is one of the field-rendered types:
- `FIEL_Type_INTERLACED` -- Order determines which interlaced field is temporally first
- `FIEL_Type_HALF_HEIGHT` -- Order determines which field the first sample contains
- `FIEL_Type_FIELD_DOUBLED` -- Order determines which field was the source

For `FIEL_Type_FRAME_RENDERED`, the order field is irrelevant and should be ignored.

---

## File Format Storage

The `FIEL_Label` is stored differently depending on the file format:

### QuickTime Movies
Stored as user data of type `'FIEL'`. Only the first `FIEL_Label` user data item (index 1) is honored. If a sequence is composed of multiple files, only the first frame's label may be used.

### Image/Animation Files
Stored as a `'FIEL'` resource with ID 128 (`FIEL_ResID`).

### ImageDescription Appended
If the creator cannot add user data or resources, the `FIEL_Label` can be appended to the end of a QuickTime `ImageDescription`. This is a fallback mechanism.

---

## Historical Context

The header preserves important version history:

- **AE 1.0/1.1 (CoSA era):** Used version 0 with the lowercase tag `'Fiel'` (not `'FIEL'`). The structure had a `short version` followed by an `int32_t type` where 0 = not field rendered, 1 = upper field first, 2 = lower field first. All field-rendered output was interlaced.

- **AE 1.2+ (version 1):** Introduced the current `FIEL_Label` structure with explicit type and order fields, plus the uppercase `'FIEL'` tag.

Backward compatibility: if you encounter the lowercase `'Fiel'` tag or version 0, apply the legacy interpretation.

---

## Detecting and Handling Interlaced Footage in a Plugin

### Reading FIEL Information from Footage Items (AEGP Approach)

AEGP plugins can query footage interpretation settings through the `AEGP_FootageSuite`:

```c
AEGP_FootageSuite5 *footSuite = NULL;
pica->AcquireSuite(kAEGPFootageSuite, kAEGPFootageSuiteVersion5, (const void **)&footSuite);

AEGP_FootageInterp interp;
footSuite->AEGP_GetFootageInterpretation(footage_itemH, FALSE, &interp);
// interp contains field and pulldown information
```

### In an Effect Plugin: Checking Field Rendering State

Effect plugins receive field information through `PF_InData`:

```c
PF_Err MyRender(PF_InData *in_data, /* ... */) {
    // Check if we are rendering a single field
    PF_Field field = in_data->field;

    // PF_Field values:
    // PF_Field_FRAME   - Rendering full frame (progressive or final composited)
    // PF_Field_UPPER   - Rendering upper field only
    // PF_Field_LOWER   - Rendering lower field only

    if (field == PF_Field_FRAME) {
        // Full progressive frame -- standard rendering
        RenderFullFrame(in_data, output);
    } else {
        // Field rendering -- only half the lines are valid
        RenderField(in_data, output, field);
    }

    return PF_Err_NONE;
}
```

### Effect Rendering Considerations for Interlaced Content

When AE renders an interlaced composition, it may call your effect twice per frame -- once for each field. Each call renders only the relevant field's scan lines. Your effect must handle this correctly.

#### Temporal Effects

Effects that reference other points in time (echo, motion blur, time remapping) must be especially careful. Each field represents a different point in time. When checking out layers at other times for a field render, the time values account for the field offset.

#### Spatial Effects

Effects that process neighboring pixels (blur, sharpen, edge detect) face a subtlety: in interlaced mode, adjacent lines belong to different fields captured at different times. A naive vertical blur on interlaced footage will blend pixels from different time points, creating ghosting.

**Correct approach for spatial filters on interlaced content:**

```c
PF_Err RenderField(PF_InData *in_data, PF_EffectWorld *output, PF_Field field) {
    // Option 1: Process only the lines belonging to the current field
    // Upper field = lines 0, 2, 4, ...
    // Lower field = lines 1, 3, 5, ...
    A_long start_line = (field == PF_Field_UPPER) ? 0 : 1;

    for (A_long y = start_line; y < output->height; y += 2) {
        // Process line y, reading only from lines of the SAME field
        // (y-2, y, y+2) not (y-1, y, y+1)
        ProcessScanLine(in_data, output, y, /*field_step=*/2);
    }

    return PF_Err_NONE;
}
```

#### Motion Estimation

Motion vectors calculated on interlaced footage must account for the temporal offset between fields. A motion vector that is correct for a full frame may be wrong for a single field, since the fields represent different moments in time.

---

## Creating FIEL Labels

### Writing a Progressive Label

```c
FIEL_Label MakeProgressiveLabel(void) {
    FIEL_Label label;
    memset(&label, 0, sizeof(label));
    label.signature = FIEL_Tag;           // 'FIEL'
    label.version   = FIEL_Label_VERSION; // 1
    label.type      = FIEL_Type_FRAME_RENDERED;
    label.order     = FIEL_Order_UPPER_FIRST; // Irrelevant for progressive
    label.reserved  = 0;
    return label;
}
```

### Writing an Interlaced Label

```c
FIEL_Label MakeInterlacedLabel(FIEL_Order fieldOrder) {
    FIEL_Label label;
    memset(&label, 0, sizeof(label));
    label.signature = FIEL_Tag;
    label.version   = FIEL_Label_VERSION;
    label.type      = FIEL_Type_INTERLACED;
    label.order     = fieldOrder;
    label.reserved  = 0;
    return label;
}

// Upper field first (most HD formats):
FIEL_Label hdLabel = MakeInterlacedLabel(FIEL_Order_UPPER_FIRST);

// Lower field first (some legacy SD formats):
FIEL_Label sdLabel = MakeInterlacedLabel(FIEL_Order_LOWER_FIRST);
```

### Validating a FIEL Label

```c
bool ValidateFIELLabel(const FIEL_Label *label) {
    if (!label) return false;

    // Check signature
    if (label->signature != FIEL_Tag) return false;

    // Check version
    if (label->version < 0 || label->version > FIEL_Label_VERSION) return false;

    // Check type range
    if (label->type > FIEL_Type_FIELD_DOUBLED) {
        // FIEL_Type_UNSPECIFIED (4) is explicitly "do not use"
        return false;
    }

    // Check order range
    if (label->order > FIEL_Order_LOWER_FIRST) return false;

    return true;
}
```

---

## Common Pitfalls

### 1. Ignoring Field Order

The single most common interlace bug is ignoring field order entirely. When AE tells your effect it is rendering `PF_Field_UPPER` or `PF_Field_LOWER`, your temporal calculations (time remapping, motion blur, frame blending) must account for the sub-frame time offset of that field.

### 2. Structure Packing

The `FIEL_Label` uses `#pragma pack(push, CoSAalign, 2)`. If you copy this structure definition into your own code without the pragma, the compiler will add padding and the structure will not be 18 bytes. This breaks file I/O.

### 3. Confusing Upper/Lower with Even/Odd

"Upper field" refers to the field starting at scan line 0 (the topmost line of the image). In most conventions, this is the "even" field (lines 0, 2, 4...). "Lower field" starts at scan line 1 (lines 1, 3, 5...). Some documentation uses "odd/even" while others use "upper/lower" -- they refer to the same thing, but the mapping can be confusing.

### 4. Vertical Filtering Across Fields

Applying a vertical convolution kernel across adjacent lines of an interlaced frame mixes pixels from different points in time, producing ghosting artifacts. When processing interlaced content, either:
- Process each field independently (skip every other line)
- De-interlace first, process, then re-interlace

### 5. Legacy Version 0 Labels

If your AEIO plugin reads footage formats that may contain CoSA-era `'Fiel'` (lowercase 'i') labels, you need to handle the version 0 format. In version 0, the type field directly encodes both type and order: 0 = not field rendered, 1 = upper first, 2 = lower first.

### 6. Assuming Progressive

Modern footage is usually progressive, but assuming all footage is progressive is a bug. Always check the field state in `in_data->field` during rendering, and handle interlaced footage gracefully even if your effect does not specifically optimize for it. At minimum, return correct results -- even if the quality is suboptimal for interlaced content.

---

## Summary Table

| Property | Value | Meaning |
|----------|-------|---------|
| Tag | `'FIEL'` | Resource and user data type identifier |
| Resource ID | 128 | Standard resource ID in image files |
| Version | 1 | Current label version |
| Structure size | 18 bytes | With 2-byte packing |
| Progressive | `type=0, order=ignored` | Full frame, no temporal split |
| Interlaced upper first | `type=1, order=0` | HD standard, DV |
| Interlaced lower first | `type=1, order=1` | Some legacy SD |
| Half height | `type=2` | Single field, half resolution |
| Field doubled | `type=3` | Single field stretched to full height |
| Unspecified | `type=4` | Do not use |
