# PiPL Tool: The Resource Compilation Pipeline

On Windows, After Effects plugins embed their metadata in a binary resource called a **PiPL** (Plug-In Property List). The PiPL tells After Effects what kind of plugin it is, its entry point, supported flags, match name, and more -- all before any code runs. The SDK provides `PiPLTool.exe` to convert human-readable `.r` resource files into Windows `.rc` resources that the linker embeds in the final `.aex` DLL.

---

## Why PiPL Compilation Is Needed

On macOS, the `.r` file format originates from Apple's Rez resource compiler, which can process PiPL definitions natively. Windows has no equivalent, so Adobe provides `PiPLTool.exe` as a bridge: it reads preprocessed PiPL data and emits a Windows resource script.

The tool is located at:

```
Examples/Resources/PiPLtool.exe
```

---

## The 3-Step Compilation Pipeline

Every SDK example project on Windows uses the same three-step custom build process for `.r` files:

### Step 1: Preprocess the .r File

```
cl /I "$(ProjectDir)..\..\..\Headers" /EP "..\%(Filename).r" > "$(IntDir)%(Filename).rr"
```

| Component | Meaning |
|---|---|
| `cl` | The MSVC C/C++ compiler |
| `/I "...Headers"` | Include path for SDK headers (`AEConfig.h`, `AE_EffectVers.h`, `AE_General.r`) |
| `/EP` | Preprocess only, output to stdout, no `#line` directives |
| `> ... .rr` | Redirect preprocessed output to an intermediate `.rr` file |

This step resolves all `#include`, `#ifdef`, and `#define` directives. The `.r` file includes `AEConfig.h` (which defines platform macros like `AE_OS_WIN` and `AE_PROC_INTELx64`) and `AE_EffectVers.h` (which defines `PF_PLUG_IN_VERSION` and `PF_PLUG_IN_SUBVERS`).

**What the .rr file looks like**: Pure text with all macros expanded. The `#ifdef AE_OS_WIN` blocks are resolved, so only the Windows code path entries (like `CodeWin64X86`) remain.

### Step 2: Run PiPLTool

```
"$(ProjectDir)..\..\..\Resources\PiPLTool" "$(IntDir)%(Filename).rr" "$(IntDir)%(Filename).rrc"
```

| Component | Meaning |
|---|---|
| `PiPLTool` | The Adobe-provided converter |
| Input: `.rr` | Preprocessed PiPL text |
| Output: `.rrc` | An intermediate file containing Windows RC macros and hex data |

PiPLTool parses the PiPL text format (the `resource 'PiPL' (16000) { ... }` syntax) and emits a `.rrc` file that contains `#define` statements and hex-encoded binary data suitable for inclusion in a Windows resource script.

### Step 3: Preprocess the .rrc into .rc

```
cl /D "MSWindows" /EP "$(IntDir)%(Filename).rrc" > "$(ProjectDir)%(Filename).rc"
```

| Component | Meaning |
|---|---|
| `/D "MSWindows"` | Defines the MSWindows preprocessor symbol |
| `/EP` | Preprocess only again |
| `> ... .rc` | Final Windows resource script |

This second preprocessing pass resolves any remaining conditional compilation in the `.rrc` output. The resulting `.rc` file is a standard Windows resource script that the resource compiler (`rc.exe`) can process during the normal build.

---

## The Complete Flow Diagram

```
  YourPluginPiPL.r          (Human-written PiPL source)
        |
        | cl /EP (preprocess, resolve #includes and #ifdefs)
        v
  YourPluginPiPL.rr         (Preprocessed text, macros resolved)
        |
        | PiPLTool.exe (parse PiPL syntax, emit binary hex data)
        v
  YourPluginPiPL.rrc        (Intermediate RC macros + hex data)
        |
        | cl /D "MSWindows" /EP (final preprocessing)
        v
  YourPluginPiPL.rc          (Standard Windows resource script)
        |
        | rc.exe (Windows resource compiler, invoked by MSBuild)
        v
  YourPluginPiPL.res         (Compiled binary resource)
        |
        | link.exe (embedded in final .aex DLL)
        v
  YourPlugin.aex             (Plugin with embedded PiPL resource)
```

---

## Intermediate File Formats

### .r File (Input)

The `.r` file uses a syntax derived from the classic Mac Rez format:

```c
#include "AEConfig.h"
#include "AE_EffectVers.h"

#ifndef AE_OS_WIN
    #include "AE_General.r"
#endif

resource 'PiPL' (16000) {
    {
        Kind { AEEffect },
        Name { "My Effect" },
        Category { "My Category" },
#ifdef AE_OS_WIN
    #ifdef AE_PROC_INTELx64
        CodeWin64X86 {"EffectMain"},
    #endif
#else
    #ifdef AE_OS_MAC
        CodeMacIntel64 {"EffectMain"},
        CodeMacARM64 {"EffectMain"},
    #endif
#endif
        AE_PiPL_Version { 2, 0 },
        AE_Effect_Spec_Version { PF_PLUG_IN_VERSION, PF_PLUG_IN_SUBVERS },
        AE_Effect_Version { 1081345 },
        AE_Effect_Info_Flags { 0 },
        AE_Effect_Global_OutFlags { 0x2000000 },
        AE_Effect_Global_OutFlags_2 { 0x8000000 },
        AE_Effect_Match_Name { "ADBE My Effect" },
        AE_Reserved_Info { 0 },
        AE_Effect_Support_URL { "https://example.com" }
    }
};
```

### .rr File (After First Preprocessing)

All `#ifdef`/`#define` directives are resolved. On a 64-bit Windows build, only the `CodeWin64X86` entry remains:

```
resource 'PiPL' (16000) {
    {
        Kind { AEEffect },
        Name { "My Effect" },
        Category { "My Category" },
        CodeWin64X86 {"EffectMain"},
        AE_PiPL_Version { 2, 0 },
        AE_Effect_Spec_Version { 13, 28 },
        AE_Effect_Version { 1081345 },
        ...
    }
};
```

### .rrc File (PiPLTool Output)

This file contains Windows RC directives and raw hex data. It typically defines a custom resource type and includes binary PiPL data encoded as comma-separated hex bytes.

### .rc File (Final Resource Script)

A standard Windows resource file that `rc.exe` compiles into a `.res` binary. This is what Visual Studio's build system consumes.

---

## Setting Up Custom Build Steps in Visual Studio

The custom build step is defined on the `.r` file item in the `.vcxproj`:

```xml
<ItemGroup>
  <CustomBuild Include="..\MyPluginPiPL.r">
    <Message>Compiling the PiPL</Message>
    <Command>
cl /I "$(ProjectDir)..\..\..\Headers" /EP ".."\\"%(Filename).r" > "$(IntDir)"\\"%(Filename).rr"
"$(ProjectDir)..\..\..\Resources\PiPLTool" "$(IntDir)%(Filename).rr" "$(IntDir)%(Filename).rrc"
cl /D "MSWindows" /EP $(IntDir)%(Filename).rrc > "$(ProjectDir)"\\"%(Filename)".rc
    </Command>
    <Outputs>$(ProjectDir)%(Filename).rc;%(Outputs)</Outputs>
  </CustomBuild>
</ItemGroup>
```

The `.rc` output file must also be listed in the project's `ResourceCompile` items:

```xml
<ItemGroup>
  <ResourceCompile Include="MyPluginPiPL.rc" />
</ItemGroup>
```

> **Pitfall**: The `Outputs` field must list the `.rc` file so MSBuild knows when to re-run the custom build step. If this is missing, the PiPL will not be regenerated when the `.r` file changes.

---

## CMake Integration

For CMake-based projects, you can integrate PiPL compilation with a custom command:

```cmake
# Paths
set(AE_SDK_ROOT "path/to/AfterEffectsSDK/Examples")
set(PIPL_TOOL "${AE_SDK_ROOT}/Resources/PiPLtool.exe")
set(AE_HEADERS "${AE_SDK_ROOT}/Headers")

function(add_pipl_resource TARGET PIPL_R_FILE)
    get_filename_component(PIPL_NAME ${PIPL_R_FILE} NAME_WE)

    set(RR_FILE  "${CMAKE_CURRENT_BINARY_DIR}/${PIPL_NAME}.rr")
    set(RRC_FILE "${CMAKE_CURRENT_BINARY_DIR}/${PIPL_NAME}.rrc")
    set(RC_FILE  "${CMAKE_CURRENT_BINARY_DIR}/${PIPL_NAME}.rc")

    # Step 1: Preprocess .r to .rr
    add_custom_command(
        OUTPUT ${RR_FILE}
        COMMAND cl /I "${AE_HEADERS}" /EP "${PIPL_R_FILE}" > "${RR_FILE}"
        DEPENDS ${PIPL_R_FILE}
        COMMENT "PiPL Step 1: Preprocessing ${PIPL_NAME}.r"
        VERBATIM
    )

    # Step 2: PiPLTool .rr to .rrc
    add_custom_command(
        OUTPUT ${RRC_FILE}
        COMMAND "${PIPL_TOOL}" "${RR_FILE}" "${RRC_FILE}"
        DEPENDS ${RR_FILE}
        COMMENT "PiPL Step 2: Running PiPLTool on ${PIPL_NAME}.rr"
        VERBATIM
    )

    # Step 3: Preprocess .rrc to .rc
    add_custom_command(
        OUTPUT ${RC_FILE}
        COMMAND cl /D "MSWindows" /EP "${RRC_FILE}" > "${RC_FILE}"
        DEPENDS ${RRC_FILE}
        COMMENT "PiPL Step 3: Generating ${PIPL_NAME}.rc"
        VERBATIM
    )

    # Add the .rc to the target's sources
    target_sources(${TARGET} PRIVATE ${RC_FILE})
endfunction()

# Usage:
add_library(MyPlugin MODULE MyPlugin.cpp)
add_pipl_resource(MyPlugin "${CMAKE_CURRENT_SOURCE_DIR}/MyPluginPiPL.r")
```

> **Note**: The `cl` invocations require that the MSVC environment is set up (e.g., via `vcvarsall.bat` or the VS Developer Command Prompt). CMake's MSVC generator handles this automatically when building from within Visual Studio or with `cmake --build`.

---

## Common Errors and How to Fix Them

### "cl is not recognized as an internal or external command"

**Cause**: The MSVC compiler is not in the PATH. The custom build step runs in a shell that may not have the Visual Studio environment variables set.

**Fix**: Build from within Visual Studio, or run `vcvarsall.bat x64` before building from the command line.

### PiPLTool produces empty or garbled output

**Cause**: The `.rr` file contains unexpected content, often because the include paths are wrong and SDK headers were not found during preprocessing.

**Fix**: Check the include path in Step 1. Verify that `AEConfig.h` and the other headers resolve correctly. Inspect the `.rr` file manually to see if macros like `PF_PLUG_IN_VERSION` were expanded.

### "resource 'PiPL' (16000) not found" or link errors

**Cause**: The `.rc` file was not generated or was not included in the build.

**Fix**: Ensure:
1. The custom build step runs (check the Build Output window)
2. The `.rc` file is listed in the `ResourceCompile` item group
3. The resource ID (16000) matches what AE expects

### Stale PiPL after changes

**Cause**: MSBuild does not detect that the `.r` file changed because the `Outputs` property is missing or incorrect.

**Fix**: Verify the `<Outputs>` element points to the correct `.rc` path. Clean and rebuild. Delete the intermediate `.rr`, `.rrc`, and `.rc` files manually if needed.

### "AE_General.r: No such file or directory" on Windows

**Cause**: The `.r` file includes `AE_General.r` unconditionally.

**Fix**: Wrap the include in a platform check (all SDK examples do this):

```c
#ifndef AE_OS_WIN
    #include "AE_General.r"
#endif
```

`AE_General.r` is only needed on macOS where the Rez compiler processes it directly. On Windows, `PiPLTool` handles the conversion.

### PiPL flags don't match code

**Cause**: The hex values in the `.r` file's `AE_Effect_Global_OutFlags` do not match the flags set in `GlobalSetup`.

**Fix**: The PiPL flags and the code-set flags must be identical. AE reads the PiPL first (before loading the plugin) for certain decisions. Common values:

| Flag | Hex Value |
|---|---|
| `PF_OutFlag_DEEP_COLOR_AWARE` | `0x2000000` |
| `PF_OutFlag_PIX_INDEPENDENT` | `0x40` |
| `PF_OutFlag_USE_OUTPUT_EXTENT` | `0x400` |
| `PF_OutFlag2_SUPPORTS_THREADED_RENDERING` | `0x8000000` |
| `PF_OutFlag2_SUPPORTS_GPU_RENDER_F32` | `0x200000` |

Compute the combined value by OR-ing the flags you need. For example, `PIX_INDEPENDENT | USE_OUTPUT_EXTENT` = `0x40 | 0x400` = `0x440` = `1088` decimal.

---

## AE_General.r: The PiPL Type Definition

The file `Examples/Resources/AE_General.r` defines the binary layout of the PiPL resource type. It tells the Rez compiler (macOS) how to encode each property. Key property types defined:

| Property Name | Key Code | Purpose |
|---|---|---|
| `Kind` | `'kind'` | Plugin type (AEEffect, AEGP, etc.) |
| `Name` | `'name'` | Display name |
| `Category` | `'catg'` | Menu category |
| `Version` | `'vers'` | Plugin version |
| `CodeWin64X86` | `'8664'` | Windows x64 entry point |
| `CodeMacIntel64` | `'mi64'` | macOS Intel 64 entry point |
| `CodeMacARM64` | `'ma64'` | macOS ARM64 entry point |
| `AE_PiPL_Version` | `'ePVR'` | PiPL format version |
| `AE_Effect_Spec_Version` | `'eSVR'` | API version |
| `AE_Effect_Version` | `'eVER'` | Effect version |
| `AE_Effect_Info_Flags` | `'eINF'` | Info flags |
| `AE_Effect_Global_OutFlags` | `'eGLO'` | Global out_flags |
| `AE_Effect_Global_OutFlags_2` | `'eGL2'` | Global out_flags2 |
| `AE_Effect_Match_Name` | `'eMNA'` | Unique match name |
| `AE_Reserved_Info` | `'aeFL'` | Reserved |
| `AE_Effect_Support_URL` | `'eURL'` | Support URL |

---

## AEGP Plugin PiPLs

AEGP plugins (panels, command hooks, artisans) use a simpler PiPL:

```c
resource 'PiPL' (16000) {
    {
        Kind { AEGP },
        Name { "My AEGP Plugin" },
        Category { "General Plugin" },
        Version { 196608 },
#ifdef AE_OS_WIN
    #ifdef AE_PROC_INTELx64
        CodeWin64X86 {"EntryPointFunc"},
    #endif
#else
    #ifdef AE_OS_MAC
        CodeMacIntel64 {"EntryPointFunc"},
        CodeMacARM64 {"EntryPointFunc"},
    #endif
#endif
    }
};
```

AEGP PiPLs do not include `AE_Effect_Global_OutFlags`, `AE_Effect_Match_Name`, or other effect-specific properties.

---

## Debugging PiPL Issues

1. **Inspect intermediate files**: Look at the `.rr` file to verify preprocessing worked. Look at the `.rrc` and `.rc` files to verify PiPLTool produced valid output.

2. **Check resource with a hex editor**: After building, use a PE resource viewer (such as Resource Hacker) on the `.aex` file to inspect the embedded PiPL resource. Verify the binary data matches your expectations.

3. **Enable verbose build output**: In Visual Studio, set Tools > Options > Projects and Solutions > Build and Run > MSBuild output verbosity to "Detailed" to see the exact commands being run.

4. **Check the resource ID**: PiPLs must use resource ID 16000. Using a different ID will cause AE to not find the PiPL.

5. **Verify the entry point name**: The string in `CodeWin64X86 {"EffectMain"}` must exactly match the exported function name in your DLL. Check with `dumpbin /exports YourPlugin.aex`.
