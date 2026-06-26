# Persistent Data Storage: The Persisto Example

## Overview

AEGPs often need to store settings that survive across After Effects sessions -- plugin preferences, license state, cached configuration, user defaults. The `AEGP_PersistentDataSuite` provides a structured key-value storage system that persists in AE's preferences file on disk.

The **Persisto** SDK example (`Examples/AEGP/Persisto/`) demonstrates reading, writing, and checking for persistent data, including the notable ability to read and modify AE's own internal preferences.

---

## Core Concept: Blobs, Sections, and Keys

Persistent data is organized in a three-level hierarchy:

```
Blob (AEGP_PersistentBlobH)
  |
  +-- Section ("Main Pref Section", "MyPlugin", ...)
  |     |
  |     +-- Key ("Fuzziness")     -> Value (3.286)
  |     +-- Key ("Username")      -> Value ("Artist")
  |     +-- Key ("WindowPos")     -> Value (raw bytes)
  |
  +-- Section ("Another Section")
        |
        +-- Key ("SomeKey")       -> Value (42)
```

- **Blob** -- The top-level container. Currently, the only blob host is the **application** (AE's preferences). Future SDK versions may support project-level blobs.
- **Section** -- A named group of related keys, similar to an INI file section or a registry key.
- **Key** -- A named value within a section.

### Where Persistent Data Lives

The application blob is stored in AE's preferences file on disk:

- **Windows:** `%APPDATA%\Adobe\After Effects\<version>\`
- **macOS:** `~/Library/Preferences/Adobe/After Effects/<version>/`

You can retrieve the preferences directory path using `AEGP_GetPrefsDirectory`.

> **Important:** Persistent data writes are committed when AE saves its preferences (at quit, or when preferences are explicitly saved). If AE crashes, recent writes may be lost.

---

## AEGP_PersistentDataSuite (Versions 3 and 4)

### Version Differences

| Version | AE Version | Key Difference |
|---------|-----------|----------------|
| Suite 3 | AE 10.0 | `AEGP_GetApplicationBlob(blobPH)` -- no blob type parameter |
| Suite 4 | AE 12.0 | `AEGP_GetApplicationBlob(blob_type, blobPH)` -- adds `AEGP_PersistentType` parameter |

The Persisto example uses Suite 3. In modern code, prefer Suite 4 with the explicit blob type.

### Section and Key Management

| Function | Purpose |
|----------|---------|
| `AEGP_GetApplicationBlob` | Get the application preferences blob handle |
| `AEGP_GetNumSections` | Count sections in a blob |
| `AEGP_GetSectionKeyByIndex` | Get section name by index |
| `AEGP_DoesKeyExist` | Check if a specific section+key combination exists |
| `AEGP_GetNumKeys` | Count keys within a section |
| `AEGP_GetValueKeyByIndex` | Get key name by index |
| `AEGP_DeleteEntry` | Remove a key (no error if key not found) |
| `AEGP_GetPrefsDirectory` | Get the filesystem path to AE's preferences directory |

### Typed Getters

All getters follow the same pattern: if the key does not exist, the **default value is both written to the blob and returned**. This means getters can also act as initializers.

| Function | Data Type | Default Param |
|----------|-----------|---------------|
| `AEGP_GetLong` | `A_long` | `defaultL` |
| `AEGP_GetFpLong` | `A_FpLong` (double) | `defaultF` |
| `AEGP_GetString` | `A_char *` (C string) | `defaultZ0` (NULL = empty string) |
| `AEGP_GetTime` | `A_Time` | `defaultPT0` |
| `AEGP_GetARGB` | `PF_PixelFloat` | `defaultP0` |
| `AEGP_GetData` | Raw bytes | `defaultPV0` (NULL = all zeros) |
| `AEGP_GetDataHandle` | `AEGP_MemHandle` | `defaultH0` (NULL = no default) |

> **Critical behavior:** `AEGP_GetString` and other getters will **write the default value** to persistent storage if the key is not found. As the Persisto source comments: "Somewhat counter-intuitively, AEGP_GetString() will actually SET as well, if the key isn't found." This is by design -- it ensures consistent initialization.

### Typed Setters

| Function | Data Type |
|----------|-----------|
| `AEGP_SetLong` | `A_long` |
| `AEGP_SetFpLong` | `A_FpLong` (double) |
| `AEGP_SetString` | `A_char *` (C string) |
| `AEGP_SetTime` | `A_Time` |
| `AEGP_SetARGB` | `PF_PixelFloat` |
| `AEGP_SetData` | Raw bytes with explicit size |
| `AEGP_SetDataHandle` | `AEGP_MemHandle` (not adopted; you still own it) |

### Precision Note

From the SDK header comments:

> "FpLongs are stored with 6 decimal places of precision. There is no provision for specifying a different precision."

If you need higher precision, use `AEGP_SetData` with raw bytes.

---

## How the Persisto Example Works

### Entry Point

Persisto registers a menu command under the File menu, always enabled:

```cpp
ERR(suites.CommandSuite1()->AEGP_GetUniqueCommand(&S_persisto_cmd));
ERR(suites.CommandSuite1()->AEGP_InsertMenuCommand(
    S_persisto_cmd, "Persisto!",
    AEGP_Menu_FILE, AEGP_MENU_INSERT_SORTED));
```

The `UpdateMenuHook` unconditionally enables the command (as long as the command token is valid):

```cpp
static A_Err UpdateMenuHook(...)
{
    if (S_persisto_cmd) {
        err = suites.CommandSuite1()->AEGP_EnableCommand(S_persisto_cmd);
    }
    return err;
}
```

### Command Handler: Three Demonstrations

When the user clicks the menu item, Persisto demonstrates three capabilities:

#### 1. Reading and Writing AE's Own Preferences

Persisto shows how to access AE's built-in preference keys -- not just your own plugin's data:

```cpp
AEGP_PersistentBlobH blobH = NULL;
ERR(suites.PersistentDataSuite3()->AEGP_GetApplicationBlob(&blobH));

// Check if AE's alpha interpretation pref exists
A_Boolean askB = FALSE;
ERR(suites.PersistentDataSuite3()->AEGP_DoesKeyExist(
    blobH,
    "Main Pref Section",                    // AE's own section!
    "Pref_DEFAULT_UNLABELED_ALPHA",         // AE's own key!
    &askB));

if (!askB) {
    A_Boolean defaultB = FALSE;
    ERR(suites.PersistentDataSuite3()->AEGP_SetData(
        blobH,
        "Main Pref Section",
        "Pref_DEFAULT_UNLABELED_ALPHA",
        sizeof(A_char),
        &defaultB));
}
```

This temporarily suppresses AE's "interpret alpha" dialog when importing footage. The source comments remind you to **restore any settings you change behind the user's back**.

> **Warning:** Modifying AE's internal preferences is powerful but risky. Internal key names and their expected formats are undocumented and can change between AE versions. Always restore original values when done.

#### 2. Storing and Retrieving a Floating-Point Value

```cpp
A_FpLong valueF = 0.0;
A_Boolean found_my_stuffB = FALSE;

// Check existence first
ERR(suites.PersistentDataSuite3()->AEGP_DoesKeyExist(
    blobH,
    "Persisto",         // Plugin's own section
    "Fuzziness",        // Key name
    &found_my_stuffB));

// Get value (writes default 3.286 if key doesn't exist)
ERR(suites.PersistentDataSuite3()->AEGP_GetFpLong(
    blobH,
    "Persisto",
    "Fuzziness",
    3.286,              // DEFAULT_FUZZINESS -- used if key not found
    &valueF));

// Report what happened
if (found_my_stuffB && 3.286 == valueF) {
    // Key existed with expected value
    AEGP_ReportInfo(S_my_id, "Value already existed");
} else if (found_my_stuffB) {
    // Key existed but had a different value (user or another plugin changed it)
    AEGP_ReportInfo(S_my_id, "Different value found than expected!");
} else {
    // Key did not exist -- default was written and returned
    AEGP_ReportInfo(S_my_id, "New value added!");
}
```

This pattern is important: you can distinguish between "never set" and "set but changed" by checking existence before reading.

#### 3. Storing and Retrieving a String Value

```cpp
A_char bufferAC[AEGP_MAX_ABOUT_STRING_SIZE] = {'\0'};
A_u_long actual_buf_size = 0;

ERR(suites.PersistentDataSuite3()->AEGP_GetString(
    blobH,
    "Persisto",             // Section
    "Cliche Du Jour",       // Key
    "Default",              // Default value (written if key not found)
    AEGP_MAX_ABOUT_STRING_SIZE,
    bufferAC,               // Output buffer
    &actual_buf_size));     // Actual size needed (includes null terminator)
```

If "Cliche Du Jour" does not exist, `AEGP_GetString` writes "Default" to persistent storage and returns it. On subsequent invocations, it returns the previously stored value.

---

## Data Type Reference

| Type | Getter | Setter | Notes |
|------|--------|--------|-------|
| Integer | `AEGP_GetLong` | `AEGP_SetLong` | `A_long` (32-bit signed) |
| Float | `AEGP_GetFpLong` | `AEGP_SetFpLong` | `A_FpLong` (double), 6 decimal places |
| String | `AEGP_GetString` | `AEGP_SetString` | C strings (`A_char *`), null-terminated |
| Time | `AEGP_GetTime` | `AEGP_SetTime` | `A_Time` struct (value/scale) |
| Color | `AEGP_GetARGB` | `AEGP_SetARGB` | `PF_PixelFloat` (ARGB float) |
| Raw data | `AEGP_GetData` | `AEGP_SetData` | Arbitrary bytes with explicit size |
| Handle | `AEGP_GetDataHandle` | `AEGP_SetDataHandle` | `AEGP_MemHandle` (caller owns) |

---

## Common Patterns

### Plugin Preferences

Store user-configurable settings that persist across sessions:

```cpp
// Write preferences
ERR(suites.PersistentDataSuite4()->AEGP_GetApplicationBlob(
    AEGP_PersistentType_MACHINE_SPECIFIC, &blobH));

ERR(suites.PersistentDataSuite4()->AEGP_SetLong(
    blobH, "MyPlugin", "QualityLevel", 3));
ERR(suites.PersistentDataSuite4()->AEGP_SetString(
    blobH, "MyPlugin", "OutputPath", "C:\\renders\\"));
ERR(suites.PersistentDataSuite4()->AEGP_SetFpLong(
    blobH, "MyPlugin", "Threshold", 0.75));

// Read preferences (with defaults for first run)
A_long quality = 0;
ERR(suites.PersistentDataSuite4()->AEGP_GetLong(
    blobH, "MyPlugin", "QualityLevel", 2, &quality));
// Returns 2 on first run (default), 3 on subsequent runs
```

### License State Storage

```cpp
// Store activation status
ERR(suites.PersistentDataSuite4()->AEGP_SetLong(
    blobH, "MyPlugin_License", "Activated", 1));
ERR(suites.PersistentDataSuite4()->AEGP_SetString(
    blobH, "MyPlugin_License", "LicenseKey", key_string));

// Check on startup
A_long activated = 0;
ERR(suites.PersistentDataSuite4()->AEGP_GetLong(
    blobH, "MyPlugin_License", "Activated", 0, &activated));
```

> **Security note:** Persistent data is stored in a readable preferences file. Do not store sensitive data (license keys, passwords) in plain text. Use encryption or hashing before storing.

### Window Position / UI State

```cpp
// Save window position as raw data
A_Rect windowRect = {100, 100, 800, 600};
ERR(suites.PersistentDataSuite4()->AEGP_SetData(
    blobH, "MyPlugin", "WindowRect",
    sizeof(A_Rect), &windowRect));

// Restore on next launch
A_Rect savedRect;
ERR(suites.PersistentDataSuite4()->AEGP_GetData(
    blobH, "MyPlugin", "WindowRect",
    sizeof(A_Rect), NULL, &savedRect));
// NULL default means all zeros if not found
```

### Enumerating Stored Data

```cpp
A_long num_sections = 0;
ERR(suites.PersistentDataSuite4()->AEGP_GetNumSections(blobH, &num_sections));

for (A_long i = 0; i < num_sections; i++) {
    A_char section_key[256];
    ERR(suites.PersistentDataSuite4()->AEGP_GetSectionKeyByIndex(
        blobH, i, 256, section_key));

    A_long num_keys = 0;
    ERR(suites.PersistentDataSuite4()->AEGP_GetNumKeys(
        blobH, section_key, &num_keys));

    for (A_long j = 0; j < num_keys; j++) {
        A_char value_key[256];
        ERR(suites.PersistentDataSuite4()->AEGP_GetValueKeyByIndex(
            blobH, section_key, j, 256, value_key));
        // Now you have section_key and value_key
    }
}
```

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Assuming getters are read-only | Getters **write the default value** if the key does not exist. This is intentional but surprising. |
| Using section/key names that collide with AE's own prefs | Use a unique, plugin-specific section name (e.g., your plugin's match name) |
| Modifying AE's internal preferences without restoring them | Always save the original value, modify, do your work, then restore |
| Storing data that changes frequently | Persistent data is written to disk at quit; frequent writes add to quit time |
| Assuming `AEGP_GetString` buffer is large enough | Always pass `actual_buf_sizeLu0` to detect truncation. Pass NULL to get an error if the buffer is too small. |
| Not handling the case where data format changes between plugin versions | Include a version key in your section; migrate data if the version mismatches |
| Using `AEGP_SetDataHandle` and assuming the suite adopts the handle | The handle is **not adopted**. You still own it and must free it yourself. |
| Relying on persistent data for project-specific settings | The persistent blob is application-wide (preferences), not project-specific. For project data, use sequence data or flat data in effects. |

---

## Suite 3 vs Suite 4: GetApplicationBlob

The main API difference between versions:

**Suite 3 (used by Persisto):**
```cpp
AEGP_PersistentBlobH blobH = NULL;
ERR(suites.PersistentDataSuite3()->AEGP_GetApplicationBlob(&blobH));
```

**Suite 4 (recommended for modern code):**
```cpp
AEGP_PersistentBlobH blobH = NULL;
ERR(suites.PersistentDataSuite4()->AEGP_GetApplicationBlob(
    AEGP_PersistentType_MACHINE_SPECIFIC,   // or other blob type
    &blobH));
```

The `AEGP_PersistentType` parameter (added in AE 12.0) allows distinguishing between machine-specific and roaming preferences.

---

## Accessing AE's Internal Preferences

As Persisto demonstrates, you can read and write AE's own preference keys. Some known section/key combinations:

| Section | Key | Type | Purpose |
|---------|-----|------|---------|
| `"Main Pref Section"` | `"Pref_DEFAULT_UNLABELED_ALPHA"` | Data | Controls alpha interpretation dialog |

> **Warning:** These internal keys are undocumented, unsupported, and subject to change. Use at your own risk. Always test against your target AE version.

---

## Best Practices

1. **Use a unique section name** -- prefix with your company or plugin name to avoid collisions.
2. **Include a data version key** -- store a version number in your section so you can migrate data formats between plugin updates.
3. **Provide sensible defaults** -- the getter-with-default pattern makes first-run initialization automatic.
4. **Minimize stored data** -- preferences files should stay small. Store only what is necessary.
5. **Handle missing data gracefully** -- use `AEGP_DoesKeyExist` before reads when you need to distinguish "never set" from "set to default."
6. **Clean up on uninstall** -- if your plugin provides an uninstaller, use `AEGP_DeleteEntry` to remove your keys.

---

## Related Suites

| Suite | Relationship |
|-------|-------------|
| `AEGP_CommandSuite1` | Register menu commands (Persisto's UI entry point) |
| `AEGP_RegisterSuite5` | Register hook callbacks |
| `AEGP_UtilitySuite3` | `AEGP_ReportInfo` for user feedback |
| `AEGP_MemorySuite1` | Manage `AEGP_MemHandle` values used with `GetDataHandle`/`SetDataHandle` |
