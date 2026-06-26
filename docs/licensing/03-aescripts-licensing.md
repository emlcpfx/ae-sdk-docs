# AEScripts Licensing Integration

Guide to integrating the aescripts.com licensing framework into After Effects plugins.

## Overview

**AEScripts** is the largest After Effects plugin marketplace. Their licensing system provides professional features including floating licenses, subscriptions, and render-only licenses.

### Advantages
- Full-featured (floating, subscriptions, render-only, trials)
- Professional marketplace with large customer base
- Automatic updates via aescripts+ Manager
- Built-in analytics and activation tracking
- Educational and beta license support

### Disadvantages
- 693 KB binary size (larger than Gumroad)
- Requires developer account approval
- Marketplace takes 30-40% commission
- Must use their compiled library (closed source)

### Best For
Established plugins, professional tools, plugins needing floating licenses or subscriptions.

## Setup Requirements

### 1. Developer Account

Contact: **contact@aescripts.com**

Provide:
- Plugin name and description
- Pricing structure
- License types needed (SUL, FLT, SUB, etc.)

You'll receive:
- Product ID (e.g., `YOUR_PRODUCT_CODE`)
- Private number (e.g., `YOUR_PRODUCT_NUMBER`)
- Licensing library files

### 2. Library Files

AEScripts provides:
```
aescriptsLicensing/
    include/
        aescriptsLicensing.h
        aescriptsLicensing_AdobeHelpers.h
        TMsgDlg.h
    lib/
        Windows/VS2022/
            aescriptsLicensing_MT_Release.lib
            aescriptsLicensing_MD_Release.lib
        macOS/
            libaescriptsLicensing.a
```

**Important**: Use `_MT_Release.lib` for static linking.

### 3. CMake Configuration

```cmake
option(ENABLE_AESCRIPTS_LICENSING "Enable aescripts licensing" OFF)

if(ENABLE_AESCRIPTS_LICENSING)
    set(AESCRIPTS_LIC_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/aescriptsLicensing")
    target_compile_definitions(YourPlugin PRIVATE ENABLE_AESCRIPTS_LICENSING=1)
    include_directories(${AESCRIPTS_LIC_ROOT}/include)

    if(WIN32)
        target_link_libraries(YourPlugin PRIVATE
            ${AESCRIPTS_LIC_ROOT}/lib/Windows/VS2022/aescriptsLicensing_MT_Release.lib
            ws2_32.lib iphlpapi.lib winhttp.lib crypt32.lib bcrypt.lib
        )
    elseif(APPLE)
        target_link_libraries(YourPlugin PRIVATE
            ${AESCRIPTS_LIC_ROOT}/lib/macOS/libaescriptsLicensing.a
        )
    endif()
endif()
```

## Configuration

### Product Configuration

```cpp
#ifdef ENABLE_AESCRIPTS_LICENSING
#include "aescriptsLicensing.h"
#include "aescriptsLicensing_AdobeHelpers.h"

#define LIC_PRODUCT_NAME "MyPlugin"
#define LIC_PRIVATE_NUM YOUR_PRODUCT_NUMBER
#define LIC_PRODUCT_ID "YOUR_PRODUCT_CODE"
#define LIC_FILENAME "MyPlugin"

namespace aescripts {
    static aescriptsLicenseData licenseData;
}
#endif
```

### License Types

| Define | License Type | Description |
|--------|--------------|-------------|
| `LIC_SUL` | Single/Multi-User | 1-N fixed seats |
| `LIC_FLT` | Floating | Network license server |
| `LIC_SUB` | Subscription | Time-limited |
| `LIC_FSB` | Floating Subscription | Network + subscription |
| `LIC_REN` | Render-Only | No GUI, render farms only |
| `LIC_BTA` | Beta | Time-limited testing |
| `LIC_EDU` | Educational | Discounted for students |

## Storage Locations

### Windows
```
%APPDATA%\AESCRIPTS\MyPlugin\MyPlugin.lic
```

### macOS
```
~/Library/Application Support/AESCRIPTS/MyPlugin/MyPlugin.lic
```

## Integration

### Watermark Check

```cpp
#ifdef ENABLE_AESCRIPTS_LICENSING
PF_Boolean isRenderEngine = FALSE;
PFAppSuite6* appSuiteP = NULL;
in_data->pica_basicP->AcquireSuite(kPFAppSuite, kPFAppSuiteVersion6, &appSuiteP);
if (appSuiteP) {
    appSuiteP->PF_IsRenderEngine(&isRenderEngine);
    in_data->pica_basicP->ReleaseSuite(kPFAppSuite, kPFAppSuiteVersion6);
}

if (!isRenderEngine) {
    if (!aescripts::licenseData.registered ||
        aescripts::licenseData.overused == 1 ||
        !aescripts::licenseData.renderOK) {
        // Apply watermark
    }
}
#endif
```

## Floating Licenses

### Server Setup

1. Install **aescripts License Server** on network machine
2. Configure firewall (default port: 27000)
3. Add licenses via server GUI

### Overuse Detection

```cpp
if (aescripts::licenseData.overused == 1) {
    // Too many concurrent users - apply watermark
}
```

## Build Configuration

```batch
cmake -B build -DENABLE_AESCRIPTS_LICENSING=ON
msbuild build\MyPlugin.vcxproj ^
    -p:Configuration=Release ^
    -p:Platform=x64 ^
    -p:RuntimeLibrary=MultiThreaded
```

## Testing

### Delete License (Testing)

**Windows**:
```batch
del "%APPDATA%\AESCRIPTS\MyPlugin\MyPlugin.lic"
```

**macOS**:
```bash
rm ~/Library/Application\ Support/AESCRIPTS/MyPlugin/MyPlugin.lic
```

## Next Steps

- [Watermarking](05-watermarking.md) - Implement watermark system
- [Render Farm Strategy](06-render-farm-strategy.md) - Configure farm support
- [Reference](08-reference.md) - Quick lookup tables

---

**Contact**: contact@aescripts.com for developer account and library access.
