# Gumroad Licensing Integration

Complete guide to integrating Gumroad's licensing system into After Effects plugins using a header-only implementation.

## Overview

### Why Gumroad?

**Advantages**:
- Header-only - No external library dependencies
- Cross-platform - Same code for Windows/macOS
- Lightweight - Only 355 KB binary size
- Simple API - Single REST endpoint
- Direct sales - Keep 90%+ of revenue (3.5% + 30c fee)
- Instant setup - Enable License Keys in product settings

**Disadvantages**:
- Fixed 2-machine limit - Cannot be changed per customer
- No deactivation - Users can't move licenses themselves
- Basic features - No floating licenses, subscriptions, or render-only

**Best For**: Simple direct sales, indie developers, plugins with straightforward licensing needs.

## Architecture

### Header-Only Design

```cpp
// GumroadLicense.h - Single file, no external libs
namespace Gumroad {
    const int MAX_MACHINE_ACTIVATIONS = 2;  // Hard-coded limit

    struct LicenseData {
        bool valid;
        bool registered;
        bool renderOK;
        std::string email;
        std::string license_key;
        std::string product_id;
        std::string error_message;
        int uses;
        bool refunded;
        bool disputed;
        bool chargebacked;
        long long last_check_time;
    };

    bool ValidateLicense(const std::string& product_id,
                        LicenseData& data,
                        bool increment_uses);
}
```

### Platform-Specific Implementation

**Windows**: Uses `WinHTTP` API (built into Windows)
```cpp
#ifdef _WIN32
#include <windows.h>
#include <winhttp.h>
#pragma comment(lib, "winhttp.lib")
#endif
```

**macOS**: Uses `libcurl` (system-provided)
```cpp
#ifdef __APPLE__
#include <curl/curl.h>
#endif
```

## Setup Requirements

### 1. Gumroad Product Configuration

1. Go to your product page on Gumroad
2. Click "Edit product"
3. Scroll to "License Key Module"
4. Toggle "Generate license keys" -> **ON**
5. Expand the module to see your `product_id`

### 2. CMake Configuration

```cmake
option(ENABLE_GUMROAD_LICENSING "Enable Gumroad licensing" OFF)

if(ENABLE_GUMROAD_LICENSING)
    target_compile_definitions(YourPlugin PRIVATE ENABLE_GUMROAD_LICENSING=1)
    include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include/licensing)

    if(WIN32)
        target_link_libraries(YourPlugin PRIVATE winhttp.lib)
    endif()

    if(APPLE)
        find_package(CURL REQUIRED)
        target_link_libraries(YourPlugin PRIVATE
            CURL::libcurl
            "-framework CoreFoundation"
        )
    endif()
endif()
```

## Implementation

### Step 1: Add Header and Configuration

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

### Step 2: Add "License" Button

In `GlobalSetup()`:

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

### Step 3: Apply Watermark

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

## API Communication

### Endpoint

```
POST https://api.gumroad.com/v2/licenses/verify
```

### Request Parameters

```
product_id=YOUR_PRODUCT_ID
license_key=XXXX-XXXX-XXXX-XXXX
increment_uses_count=true
```

### Response Format

**Success**:
```json
{
  "success": true,
  "uses": 2,
  "purchase": {
    "email": "user@example.com",
    "refunded": false,
    "disputed": false,
    "chargebacked": false,
    "license_key": "XXXX-XXXX-XXXX-XXXX"
  }
}
```

## Storage Locations

### Windows: Registry

**Path**: `HKEY_CURRENT_USER\Software\MyCompany\MyPlugin`

**Values**:
- `LicenseKey` (REG_SZ): User's license key
- `LastCheck` (REG_QWORD): Unix timestamp of last API check

```cpp
static std::wstring GetRegistryPath() {
    return L"Software\\MyCompany\\MyPlugin";
}
```

**Testing**:
```batch
reg query "HKEY_CURRENT_USER\Software\MyCompany\MyPlugin" /v LicenseKey
reg delete "HKEY_CURRENT_USER\Software\MyCompany\MyPlugin" /f
```

### macOS: Preferences

**Path**: `~/Library/Preferences/com.mycompany.myplugin.plist`

```cpp
CFStringRef appID = CFSTR("com.mycompany.myplugin");
```

**Testing**:
```bash
defaults read com.mycompany.myplugin LicenseKey
defaults delete com.mycompany.myplugin
```

## Machine Limit Enforcement

### Hard-Coded 2-Machine Limit

```cpp
namespace Gumroad {
    const int MAX_MACHINE_ACTIVATIONS = 2;
}
```

### No Deactivation Support

Gumroad does NOT provide an API to decrement the `uses` counter.

**Workaround for Users**:
1. Contact seller (you)
2. Seller manually resets license via Gumroad dashboard
3. User can activate on new machine

## Network Validation

### Revalidation Timing

- **48-Hour Revalidation Interval**: Check with API every 48 hours
- **7-Day Offline Grace Period**: Allow offline use for up to 7 days

### Validation Scenarios

| Scenario | Revalidation Needed? | Network Available? | Result |
|----------|---------------------|-------------------|--------|
| First launch | Yes | Yes | Validate online, store timestamp |
| Within 48h | No | N/A | Use cached status |
| After 48h | Yes | Yes | Revalidate online |
| After 48h, within 7 days | Yes | No | Use cached status (offline grace) |
| After 7+ days | Yes | No | Block rendering (must connect) |

## Testing

### Test Checklist

- [ ] Fresh activation: Install on new machine, enter license key
- [ ] License persistence: Close/reopen After Effects, verify still activated
- [ ] Invalid key: Try fake license key, verify error message
- [ ] Offline mode: Disconnect network, verify works within 7 days
- [ ] Machine limit: Activate on 3rd machine, verify rejection
- [ ] Refunded license: Refund purchase, verify blocks rendering
- [ ] Render farm: Use `aerender.exe`, verify no watermark
- [ ] Watermark appearance: Unlicensed in GUI, verify watermark appears

## Best Practices

1. **Use read-only checks** for revalidation (`increment_uses=false`)
2. **Handle network errors gracefully** - allow offline grace period
3. **Clear error messages** - tell users exactly what's wrong
4. **Test machine limits** - ensure 3rd activation properly blocked
5. **Document refund policy** - users need to know about 2-machine limit
6. **Provide reset instructions** - explain how to move license to new machine

## Next Steps

- [Watermarking](05-watermarking.md) - Implement watermark system
- [Render Farm Strategy](06-render-farm-strategy.md) - Configure farm support
- [Integration Guide](07-integration-guide.md) - Complete integration walkthrough
