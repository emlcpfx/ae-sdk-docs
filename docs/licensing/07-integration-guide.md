# Integration Guide

Step-by-step guide to add Gumroad licensing to your After Effects plugin from scratch.

## Overview

This guide walks through adding Gumroad licensing (simplest system) to an existing After Effects plugin. The process takes ~2-3 hours for a typical plugin.

**Why Gumroad for this guide?**
- Header-only (no external libs)
- Smallest binary overhead (+39 KB)
- Cross-platform (Windows/macOS same code)
- Simple API (one REST endpoint)

For AEScripts or PluginPlay, follow similar steps but use their respective docs.

## Prerequisites

- Working After Effects plugin project
- CMake build system
- Gumroad account with product created
- License Keys enabled on Gumroad product

## Step 1: Get Gumroad Product ID

1. Log into Gumroad
2. Go to your product page
3. Click "Edit product"
4. Scroll to "License Key Module"
5. Toggle "Generate license keys" -> **ON**
6. Copy the `product_id` from embed code

## Step 2: Add Header File

```bash
mkdir -p include/licensing
cp path/to/GumroadLicense.h include/licensing/
```

## Step 3: Update CMakeLists.txt

```cmake
option(ENABLE_GUMROAD_LICENSING "Enable Gumroad licensing" OFF)

if(ENABLE_GUMROAD_LICENSING)
    target_compile_definitions(YourPlugin PRIVATE ENABLE_GUMROAD_LICENSING=1)
    include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include/licensing)

    if(WIN32)
        target_link_libraries(YourPlugin PRIVATE winhttp.lib)
    elseif(APPLE)
        find_package(CURL REQUIRED)
        target_link_libraries(YourPlugin PRIVATE
            CURL::libcurl
            "-framework CoreFoundation"
        )
    endif()

    set_target_properties(YourPlugin PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY_RELEASE "${CMAKE_BINARY_DIR}/Release_Gumroad"
    )
else()
    set_target_properties(YourPlugin PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY_RELEASE "${CMAKE_BINARY_DIR}/Release_NoLic"
    )
endif()
```

## Step 4: Update Plugin Source Code

### 4.1 Add Header and Configuration

```cpp
#ifdef ENABLE_GUMROAD_LICENSING
#include "GumroadLicense.h"

#define GUMROAD_PRODUCT_ID "YOUR_PRODUCT_ID"

namespace Gumroad {
    static LicenseData g_licenseData;
    static bool g_initialized = false;
}
#endif
```

### 4.2 Update GlobalSetup

```cpp
#ifdef ENABLE_GUMROAD_LICENSING
if (in_data->appl_id != 'PrMr') {
    PF_EffectUISuite1* effect_ui_suiteP = NULL;
    in_data->pica_basicP->AcquireSuite(kPFEffectUISuite, kPFEffectUISuiteVersion1,
                                      (const void**)&effect_ui_suiteP);
    if (effect_ui_suiteP) {
        effect_ui_suiteP->PF_SetOptionsButtonName(in_data->effect_ref, "License");
        in_data->pica_basicP->ReleaseSuite(kPFEffectUISuite, kPFEffectUISuiteVersion1);
    }
}
#endif
```

### 4.3 Update EffectMain

```cpp
#ifdef ENABLE_GUMROAD_LICENSING
if (cmd == PF_Cmd_GLOBAL_SETUP && !Gumroad::g_initialized) {
    Gumroad::ValidateLicense(GUMROAD_PRODUCT_ID, Gumroad::g_licenseData, false);
    Gumroad::g_initialized = true;
}
#endif

// In switch:
case PF_Cmd_DO_DIALOG:
    #ifdef ENABLE_GUMROAD_LICENSING
    #ifdef _WIN32
    DialogBoxParam(GetModuleHandle(NULL), MAKEINTRESOURCE(IDD_GUMROAD_LICENSE),
                   NULL, GumroadDialogProc, 0);
    #endif
    #endif
    break;
```

### 4.4 Add Watermark to Render Function

```cpp
#ifdef ENABLE_GUMROAD_LICENSING
PF_Boolean isRenderEngine = FALSE;
PFAppSuite6* appSuiteP = NULL;
in_data->pica_basicP->AcquireSuite(kPFAppSuite, kPFAppSuiteVersion6, &appSuiteP);
if (appSuiteP) {
    appSuiteP->PF_IsRenderEngine(&isRenderEngine);
    in_data->pica_basicP->ReleaseSuite(kPFAppSuite, kPFAppSuiteVersion6);
}

if (!isRenderEngine) {
    if (!Gumroad::g_licenseData.registered || !Gumroad::g_licenseData.renderOK) {
        ApplyWatermark(output_worldP);
    }
}
#endif
```

## Step 5: Build and Test

### Build Both Versions

```bash
# No licensing (free version)
cmake -B build -DENABLE_GUMROAD_LICENSING=OFF
cmake --build build --config Release

# With Gumroad licensing
cmake -B build -DENABLE_GUMROAD_LICENSING=ON
cmake --build build --config Release
```

### Test Scenarios

**Unlicensed behavior**:
1. Open After Effects
2. Apply plugin to layer
3. RAM preview -> watermark should appear

**License activation**:
1. Click "License" button
2. Enter license key from Gumroad
3. Click "Activate"
4. RAM preview -> no watermark

**Persistence**:
1. Close After Effects
2. Reopen -> no watermark (license remembered)

**Render farm**:
1. Delete license: `reg delete "HKEY_CURRENT_USER\Software\MyCompany\MyPlugin" /f`
2. Render via aerender -> NO watermark

## Troubleshooting

### Build fails with "GumroadLicense.h not found"
Check `include_directories()` in CMakeLists.txt

### Linker error: "unresolved external symbol WinHttpOpen"
Add `winhttp.lib` to `target_link_libraries()`

### "License" button doesn't appear
- Check `ENABLE_GUMROAD_LICENSING` is defined
- Verify not running in Premiere Pro (`appl_id == 'PrMr'`)

### License doesn't persist after restart
Run After Effects as Administrator once to create registry key.

## Next Steps

1. Test thoroughly - All scenarios above
2. Add macOS dialog - Similar to Windows but using Cocoa
3. Consider other systems:
   - [AEScripts](03-aescripts-licensing.md) for marketplace
   - [PluginPlay](04-pluginplay-licensing.md) for subscriptions

---

**Total time**: 2-3 hours for first integration, ~30 minutes for subsequent plugins.
