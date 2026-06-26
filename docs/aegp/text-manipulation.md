# Text Layer Manipulation: The Text_Twiddler Example

## Overview

After Effects text layers expose their content through the AEGP stream system. Unlike simple numeric or color properties, text data lives in a specialized `AEGP_TextDocumentH` handle accessed via the `AEGP_TextDocumentSuite`. Modifying text requires a specific workflow: obtain the stream, extract the text document value, modify it, and write it back.

The **Text_Twiddler** SDK example (`Examples/AEGP/Text_Twiddler/`) demonstrates the complete pattern for programmatically reading and writing text layer content, including proper stream management, undo grouping, and context-sensitive menu enabling.

---

## Core Suites for Text Manipulation

### AEGP_TextDocumentSuite1

This minimal suite (frozen in AE 6.5 era) provides direct access to the text content of a text document handle:

| Function | Purpose |
|----------|---------|
| `AEGP_GetNewText` | Retrieves the text content as a UTF-16 Unicode string in a memory handle |
| `AEGP_SetText` | Sets the text content from a UTF-16 Unicode array |

#### AEGP_GetNewText

```cpp
SPAPI A_Err AEGP_GetNewText(
    AEGP_PluginID       aegp_plugin_id,    // >> your plugin ID
    AEGP_TextDocumentH  text_documentH,    // >> from stream value
    AEGP_MemHandle      *unicodePH);       // << UTF-16, null-terminated
                                           //    must be disposed with
                                           //    AEGP_FreeMemHandle
```

The returned memory handle contains a null-terminated array of `A_u_short` (UTF-16) characters. You must dispose of it with `AEGP_FreeMemHandle` when done.

#### AEGP_SetText

```cpp
SPAPI A_Err AEGP_SetText(
    AEGP_TextDocumentH  text_documentH,    // >> from stream value
    const A_u_short     *unicodePS,        // >> UTF-16 character array
    A_long              lengthL);          // >> number of characters
                                           //    (NOT bytes)
```

> **Important:** The `lengthL` parameter is the number of **characters** (UTF-16 code units), not the byte count. For ASCII text, this equals the string length. For characters outside the Basic Multilingual Plane, surrogate pairs count as two units.

### AEGP_StreamSuite2

Used to access the `SOURCE_TEXT` stream on a text layer and read/write stream values:

| Function | Purpose |
|----------|---------|
| `AEGP_GetNewLayerStream` | Get a stream reference for a specific layer property |
| `AEGP_GetStreamType` | Verify the stream is `AEGP_StreamType_TEXT_DOCUMENT` |
| `AEGP_GetNewStreamValue` | Read the current value at a given time |
| `AEGP_SetStreamValue` | Write a modified value back to the stream |
| `AEGP_DisposeStreamValue` | Free a stream value obtained from `GetNewStreamValue` |
| `AEGP_DisposeStream` | Free a stream reference |

### AEGP_DynamicStreamSuite2

| Function | Purpose |
|----------|---------|
| `AEGP_GetNewStreamRefForLayer` | Get the root stream reference for a layer (used for dynamic stream traversal) |

### AEGP_LayerSuite5

| Function | Purpose |
|----------|---------|
| `AEGP_GetActiveLayer` | Get the currently selected layer |
| `AEGP_GetLayerObjectType` | Determine if a layer is `AEGP_ObjectType_TEXT` |
| `AEGP_GetLayerCurrentTime` | Get the current time position of a layer |

---

## The Text Stream Access Pattern

Accessing text layer content follows a specific multi-step pattern. This is the most important concept to understand:

```
Layer Handle
    |
    v
AEGP_GetNewLayerStream(AEGP_LayerStream_SOURCE_TEXT)
    |
    v
AEGP_StreamRefH (text_streamH)
    |
    v
AEGP_GetNewStreamValue(text_streamH, time)
    |
    v
AEGP_StreamValue.val.text_documentH
    |
    v
AEGP_GetNewText() / AEGP_SetText()
    |
    v
AEGP_SetStreamValue()       <-- writes changes back
    |
    v
AEGP_DisposeStreamValue()   <-- cleanup
AEGP_DisposeStream()        <-- cleanup
```

The text content is **not** a direct property you can read and write like a number. It is nested inside a stream value structure, which itself comes from a stream reference, which comes from a layer. Every level of this hierarchy requires proper acquisition and disposal.

---

## How the Text_Twiddler Example Works

### Menu Registration

Text_Twiddler inserts its command at the **top** of the Layer menu:

```cpp
ERR(suites.CommandSuite1()->AEGP_InsertMenuCommand(
    S_Text_Twiddler_cmd,
    "Text Twiddler (select one text layer)",    // Initial label
    AEGP_Menu_LAYER,
    AEGP_MENU_INSERT_AT_TOP));
```

### Context-Sensitive Menu Enabling

The `UpdateMenuHook` demonstrates a best practice: dynamically changing the menu item text based on context, and only enabling it when exactly one text layer is selected.

```cpp
static A_Err UpdateMenuHook(...)
{
    AEGP_LayerH    layerH = NULL;
    AEGP_ObjectType type  = AEGP_ObjectType_NONE;

    ERR(suites.LayerSuite5()->AEGP_GetActiveLayer(&layerH));

    if (layerH) {
        ERR(suites.LayerSuite5()->AEGP_GetLayerObjectType(layerH, &type));
        if (type == AEGP_ObjectType_TEXT) {
            // Text layer selected -- enable and show active name
            ERR(suites.CommandSuite1()->AEGP_SetMenuCommandName(
                S_Text_Twiddler_cmd, "Text Twiddler"));
            ERR(suites.CommandSuite1()->AEGP_EnableCommand(
                S_Text_Twiddler_cmd));
        } else {
            // Non-text layer selected -- disable
            ERR(suites.CommandSuite1()->AEGP_DisableCommand(
                S_Text_Twiddler_cmd));
        }
    } else {
        // No layer selected -- show instructional text
        ERR(suites.CommandSuite1()->AEGP_SetMenuCommandName(
            S_Text_Twiddler_cmd,
            "Text Twiddler (select one text layer)"));
    }
    return err;
}
```

Key patterns:
- Uses `AEGP_SetMenuCommandName` to dynamically update the label -- the menu shows "Text Twiddler" when a text layer is selected and "Text Twiddler (select one text layer)" when no layer is selected.
- Uses `AEGP_GetLayerObjectType` to check for `AEGP_ObjectType_TEXT` specifically.
- `AEGP_GetActiveLayer` returns `NULL` when zero or multiple layers are selected.

### Command Handler: Modifying Text Content

The heart of Text_Twiddler demonstrates the full text modification workflow:

```cpp
static A_Err CommandHook(...)
{
    if (command == S_Text_Twiddler_cmd) {
        try {
            // 1. Start an undo group
            ERR(suites.UtilitySuite3()->AEGP_StartUndoGroup("Text Twiddler"));

            // 2. Clear the value struct before first use
            AEGP_StreamValue val;
            AEFX_CLR_STRUCT(val);

            // 3. Get the active text layer
            ERR(suites.LayerSuite5()->AEGP_GetActiveLayer(&layerH));

            // 4. Get a stream ref for the layer (dynamic stream root)
            ERR(suites.DynamicStreamSuite2()->AEGP_GetNewStreamRefForLayer(
                S_my_id, layerH, &new_streamH));

            // 5. Get the SOURCE_TEXT stream specifically
            ERR(suites.StreamSuite2()->AEGP_GetNewLayerStream(
                S_my_id, layerH,
                AEGP_LayerStream_SOURCE_TEXT,
                &text_streamH));

            // 6. Verify stream type
            ERR(suites.StreamSuite2()->AEGP_GetStreamType(
                text_streamH, &stream_type));

            if (stream_type == AEGP_StreamType_TEXT_DOCUMENT) {
                // 7. Get current time
                ERR(suites.LayerSuite5()->AEGP_GetLayerCurrentTime(
                    layerH, AEGP_LTimeMode_LayerTime, &timeT));

                // 8. Get the stream value at current time
                ERR(suites.StreamSuite2()->AEGP_GetNewStreamValue(
                    S_my_id, text_streamH,
                    AEGP_LTimeMode_LayerTime, &timeT,
                    TRUE,       // pre-expression value
                    &val));

                // 9. Modify the text content (set to "BBB")
                const A_u_short unicode[] = {0x0042, 0x0042, 0x0042};
                A_long lengthL = sizeof(unicode) / sizeof(A_u_short);

                ERR(suites.TextDocumentSuite1()->AEGP_SetText(
                    val.val.text_documentH, unicode, lengthL));

                // 10. Write the modified value back to the stream
                ERR(suites.StreamSuite2()->AEGP_SetStreamValue(
                    S_my_id, text_streamH, &val));

                *handledPB = TRUE;
            }

            // 11. Cleanup (always runs via ERR2)
            ERR2(suites.StreamSuite2()->AEGP_DisposeStreamValue(&val));
            if (new_streamH)
                ERR2(suites.StreamSuite2()->AEGP_DisposeStream(new_streamH));
            if (text_streamH)
                ERR2(suites.StreamSuite2()->AEGP_DisposeStream(text_streamH));

            // 12. End undo group
            ERR2(suites.UtilitySuite3()->AEGP_EndUndoGroup());

        } catch (A_Err &thrown_err) {
            err = thrown_err;
        }
    }
    return err;
}
```

---

## Unicode Text Handling

Text in AE is stored as UTF-16. The Text_Twiddler example constructs Unicode directly using hex code points:

```cpp
const A_u_short unicode[] = {0x0042, 0x0042, 0x0042};  // "BBB"
A_long lengthL = sizeof(unicode) / sizeof(A_u_short);   // 3 characters
```

### Converting Platform Strings to UTF-16

For real-world use, you need to convert from your native string encoding:

**Windows (wchar_t is already UTF-16):**
```cpp
const wchar_t *text = L"Hello World";
A_long len = (A_long)wcslen(text);
ERR(suites.TextDocumentSuite1()->AEGP_SetText(
    val.val.text_documentH,
    (const A_u_short *)text,
    len));
```

**Cross-platform (from UTF-8):**
```cpp
// Use MultiByteToWideChar on Windows, CFStringCreateWithCString on macOS,
// or a cross-platform library like ICU
```

### Reading Text Back

```cpp
AEGP_MemHandle unicodeH = NULL;
ERR(suites.TextDocumentSuite1()->AEGP_GetNewText(
    S_my_id, val.val.text_documentH, &unicodeH));

if (unicodeH) {
    A_u_short *textP = NULL;
    ERR(suites.MemorySuite1()->AEGP_LockMemHandle(unicodeH, (void **)&textP));
    // textP is now a null-terminated UTF-16 string
    // ... process the text ...
    ERR(suites.MemorySuite1()->AEGP_UnlockMemHandle(unicodeH));
    ERR(suites.MemorySuite1()->AEGP_FreeMemHandle(unicodeH));
}
```

---

## Stream Value Lifecycle

The `AEGP_StreamValue` union contains different value types depending on the stream. For text streams, the relevant field is `val.text_documentH`:

```
AEGP_StreamValue
  |
  +-- val.text_documentH    (AEGP_TextDocumentH)  -- for TEXT_DOCUMENT streams
  +-- val.one_d             (A_FpLong)            -- for ONE_D streams
  +-- val.two_d             (A_FpLong[2])         -- for TWO_D streams
  +-- val.color             (PF_PixelFloat)       -- for COLOR streams
  +-- ...etc
```

> **Critical:** Always call `AEFX_CLR_STRUCT(val)` before first use. The stream value union contains pointers that must be zero-initialized to avoid crashes during disposal.

---

## AEGP_LayerStream_SOURCE_TEXT

The `SOURCE_TEXT` stream is the entry point for all text content access. Other text-related properties can be accessed through the dynamic stream system, but the actual text content (the characters, font, size, etc. as a compound document) is accessed through this stream.

### Stream Type Verification

Always verify the stream type before accessing text-specific data:

```cpp
AEGP_StreamType stream_type;
ERR(suites.StreamSuite2()->AEGP_GetStreamType(text_streamH, &stream_type));
if (stream_type == AEGP_StreamType_TEXT_DOCUMENT) {
    // Safe to access val.text_documentH
}
```

---

## Undo Group Pattern

Text_Twiddler wraps all modifications in an undo group, which is mandatory for undoable AEGP operations:

```cpp
ERR(suites.UtilitySuite3()->AEGP_StartUndoGroup("Text Twiddler"));
// ... all modifications ...
ERR2(suites.UtilitySuite3()->AEGP_EndUndoGroup());
```

> **Warning:** You must **always** call `AEGP_EndUndoGroup` if you called `AEGP_StartUndoGroup`, even if errors occurred. Failing to close an undo group will corrupt AE's undo stack. Use `ERR2` (which executes regardless of prior errors) for the `EndUndoGroup` call.

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Not calling `AEFX_CLR_STRUCT(val)` | Uninitialized stream values cause crashes in `AEGP_DisposeStreamValue` |
| Forgetting to call `AEGP_DisposeStreamValue` | Memory leak; the text document handle is not freed |
| Forgetting to call `AEGP_DisposeStream` | Memory leak; stream references must be explicitly freed |
| Not closing undo groups | Corrupts AE's undo system; always pair Start/End with ERR2 |
| Passing byte count instead of character count to `AEGP_SetText` | The `lengthL` parameter is characters (UTF-16 code units), not bytes |
| Assuming `AEGP_GetActiveLayer` returns a text layer | Always verify with `AEGP_GetLayerObjectType` == `AEGP_ObjectType_TEXT` |
| Setting text without calling `AEGP_SetStreamValue` afterward | `AEGP_SetText` modifies the document handle in memory, but the change is not committed to the timeline until `AEGP_SetStreamValue` is called |
| Accessing text at the wrong time mode | Use `AEGP_LTimeMode_LayerTime` for layer-relative time, `AEGP_LTimeMode_CompTime` for composition time |

---

## Advanced: Text Document Properties Beyond Content

The `AEGP_TextDocumentSuite1` only exposes Get/Set for the raw text string. To access font, size, color, tracking, and other text properties, you need to work through the **text animator** and **dynamic stream** systems, or use scripting. The SDK's text document suite is intentionally minimal -- it gets and sets the character content, not the styling.

For full text property control, consider:

1. **AEGP_DynamicStreamSuite** -- traverse the text layer's property groups to access individual animator properties
2. **AEGP_StreamSuite** -- read/write keyframed text property values
3. **ExtendScript / ScriptUI** -- `TextDocument` objects in scripting expose `font`, `fontSize`, `fillColor`, `strokeColor`, `tracking`, `leading`, `baselineShift`, and many more properties that are not directly exposed through the C++ text document suite

---

## Layer Object Types

When checking whether a layer is a text layer:

| Constant | Layer Type |
|----------|-----------|
| `AEGP_ObjectType_AV` | Audio/Video (footage, solid, etc.) |
| `AEGP_ObjectType_LIGHT` | Light layer |
| `AEGP_ObjectType_CAMERA` | Camera layer |
| `AEGP_ObjectType_TEXT` | Text layer |
| `AEGP_ObjectType_VECTOR` | Shape layer |

---

## Complete Minimal Example: Replace Text on Active Layer

```cpp
A_Err ReplaceTextOnActiveLayer(const A_u_short *newTextP, A_long charCount)
{
    A_Err err = A_Err_NONE, err2 = A_Err_NONE;
    AEGP_SuiteHandler suites(sP);
    AEGP_LayerH       layerH       = NULL;
    AEGP_StreamRefH   text_streamH = NULL;
    AEGP_StreamValue  val;

    AEFX_CLR_STRUCT(val);

    ERR(suites.UtilitySuite3()->AEGP_StartUndoGroup("Replace Text"));
    ERR(suites.LayerSuite5()->AEGP_GetActiveLayer(&layerH));

    if (!err && layerH) {
        A_Time timeT = {0, 1};
        ERR(suites.LayerSuite5()->AEGP_GetLayerCurrentTime(
            layerH, AEGP_LTimeMode_LayerTime, &timeT));

        ERR(suites.StreamSuite2()->AEGP_GetNewLayerStream(
            S_my_id, layerH, AEGP_LayerStream_SOURCE_TEXT, &text_streamH));

        ERR(suites.StreamSuite2()->AEGP_GetNewStreamValue(
            S_my_id, text_streamH, AEGP_LTimeMode_LayerTime,
            &timeT, TRUE, &val));

        ERR(suites.TextDocumentSuite1()->AEGP_SetText(
            val.val.text_documentH, newTextP, charCount));

        ERR(suites.StreamSuite2()->AEGP_SetStreamValue(
            S_my_id, text_streamH, &val));
    }

    ERR2(suites.StreamSuite2()->AEGP_DisposeStreamValue(&val));
    if (text_streamH)
        ERR2(suites.StreamSuite2()->AEGP_DisposeStream(text_streamH));
    ERR2(suites.UtilitySuite3()->AEGP_EndUndoGroup());

    return err;
}
```

---

## Related Suites

| Suite | Relationship |
|-------|-------------|
| `AEGP_StreamSuite2` | Core stream access for reading/writing the SOURCE_TEXT property |
| `AEGP_DynamicStreamSuite2` | Traversing dynamic property groups (text animators) |
| `AEGP_LayerSuite5` | Getting active layer, checking object type, getting current time |
| `AEGP_MemorySuite1` | Managing memory handles returned by `AEGP_GetNewText` |
| `AEGP_UtilitySuite3` | Undo group management |
| `AEGP_CommandSuite1` | Menu command registration and dynamic naming |
