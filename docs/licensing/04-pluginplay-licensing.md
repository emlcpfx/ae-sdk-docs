# PluginPlay Licensing Integration

Guide to integrating PluginPlay's device-based licensing system for subscription-focused After Effects plugins.

## Overview

**PluginPlay** is a subscription-focused plugin marketplace with device-based licensing and a modern developer portal.

### Advantages
- Subscription-friendly (monthly/yearly billing)
- Modern developer dashboard with analytics
- Device-based activation (not machine-based)
- Built-in update system
- Good for SaaS business models

### Disadvantages
- Largest binary size (2.7 MB due to OpenSSL + cURL)
- **Must use `/MD` runtime** (not `/MT`)
- No floating licenses
- Requires backend product registration

### Best For
Subscription plugins, SaaS products, plugins targeting monthly revenue.

## Setup Requirements

### 1. Developer Account

1. Register at **pluginplay.com/developers**
2. Create product with `product_id` (lowercase, e.g., "myplugin")
3. Configure pricing (lifetime vs subscription)
4. Get API keys

### 2. SDK Files

PluginPlay provides:
```
PluginPlaySDK/
    include/
        pplic.h
        ae_helpers.h
        dialog.h
    lib/x64/
        libpplic_Release.lib
        libssl_static.lib
        libcrypto_static.lib
        libcurl_a.lib
```

**Critical**: All libs built with `/MD` runtime - your plugin MUST also use `/MD`.

### 3. CMake Configuration

```cmake
option(ENABLE_PLUGINPLAY_LICENSING "Enable PluginPlay licensing" OFF)

if(ENABLE_PLUGINPLAY_LICENSING)
    set(PLUGINPLAY_LIC_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/PluginPlaySDK")
    target_compile_definitions(YourPlugin PRIVATE ENABLE_PLUGINPLAY_LICENSING=1)
    include_directories(${PLUGINPLAY_LIC_ROOT}/include)

    if(WIN32)
        set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
        target_link_libraries(YourPlugin PRIVATE
            ${PLUGINPLAY_LIC_ROOT}/lib/x64/libpplic_Release.lib
            ${PLUGINPLAY_LIC_ROOT}/lib/x64/libssl_static.lib
            ${PLUGINPLAY_LIC_ROOT}/lib/x64/libcrypto_static.lib
            ${PLUGINPLAY_LIC_ROOT}/lib/x64/libcurl_a.lib
            ws2_32.lib crypt32.lib wldap32.lib normaliz.lib
        )
    endif()
endif()
```

## Configuration

### Product Configuration

```cpp
#ifdef ENABLE_PLUGINPLAY_LICENSING
#include "pplic.h"
#include "ae_helpers.h"
#include "dialog.h"

#define FX_NAME "MyPlugin"

namespace pplic {
    struct config {
        std::string product_id = "myplugin";
        std::string version = "1.0.0";
        bool activated = false;
        std::string plan;
    };

    static config lic_config;
}
#endif
```

### Device Limits

| Plan Type | Device Limit |
|-----------|--------------|
| Lifetime | 1 device |
| Subscription | 4 devices |

## Storage Location

### Windows
```
%APPDATA%\PluginPlay\myplugin.dat
```

### macOS
```
~/Library/Application Support/PluginPlay/myplugin.dat
```

## Integration

### Watermark Check

```cpp
#ifdef ENABLE_PLUGINPLAY_LICENSING
PF_Boolean isRenderEngine = FALSE;
PFAppSuite6* appSuiteP = NULL;
in_data->pica_basicP->AcquireSuite(kPFAppSuite, kPFAppSuiteVersion6, &appSuiteP);
if (appSuiteP) {
    appSuiteP->PF_IsRenderEngine(&isRenderEngine);
    in_data->pica_basicP->ReleaseSuite(kPFAppSuite, kPFAppSuiteVersion6);
}

if (!isRenderEngine) {
    if (!pplic::lic_config.activated) {
        // Apply watermark
    }
}
#endif
```

## Build Configuration

### Critical: /MD Runtime

**PluginPlay libraries built with `/MD`** - your plugin MUST match.

**Correct Build**:
```batch
cmake -B build -DENABLE_PLUGINPLAY_LICENSING=ON
msbuild build\MyPlugin.vcxproj -p:Configuration=Release -p:Platform=x64
# Do NOT add -p:RuntimeLibrary=MultiThreaded
```

### Linker Errors if Wrong Runtime

```
error LNK2019: unresolved external symbol __imp___stdio_common_vsscanf
```

**Fix**: Remove `/MT` flag, use default `/MD`.

## Testing

### Delete License (Testing)

**Windows**:
```batch
del "%APPDATA%\PluginPlay\myplugin.dat"
```

**macOS**:
```bash
rm ~/Library/Application\ Support/PluginPlay/myplugin.dat
```

## Troubleshooting

### Issue: Linker errors (unresolved symbols)
**Cause**: Using `/MT` runtime with `/MD` PluginPlay libs
**Solution**: Remove `-p:RuntimeLibrary=MultiThreaded`

### Issue: Windows Defender flags plugin
**Cause**: Cannot use `/MT` static linking with PluginPlay
**Solutions**: Code signing (recommended), submit false positive, user must add exception

## Comparison: Why PluginPlay?

Choose PluginPlay over Gumroad/AEScripts if:
- You want subscription revenue model
- You prefer device-based licensing
- Binary size doesn't matter

Choose Gumroad/AEScripts if:
- You need small binary size
- You want static linking (`/MT`)
- You need floating licenses (AEScripts only)

## Next Steps

- [Watermarking](05-watermarking.md) - Implement watermark system
- [Render Farm Strategy](06-render-farm-strategy.md) - Configure farm support
- [Reference](08-reference.md) - Quick comparison tables

---

**Register**: pluginplay.com/developers for product registration and SDK access.
