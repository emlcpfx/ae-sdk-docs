# After Effects Plugin Licensing & Build Documentation

Comprehensive guide to building Windows After Effects plugins with integrated licensing, watermarking, and render farm support.

## Overview

This documentation suite covers a complete licensing and build system for After Effects plugins. The system supports multiple licensing providers, sophisticated watermarking, and render farm deployment strategies -- all from a single codebase.

## What's Covered

### Building & Deployment
- **Windows Build Process** - CMake, MSBuild, build scripts
- **Multi-Version Builds** - Generate 5 plugin variants from one codebase
- **Static vs Dynamic Linking** - Runtime library configuration per licensing system

### Licensing Systems
- **Gumroad** - Header-only, 2-machine limit, 48h revalidation
- **AEScripts** - Full-featured with floating licenses, subscriptions, render-only
- **PluginPlay** - Device-based activation, lifetime/subscription models
- **No Licensing** - Free/demo builds without restrictions

### Watermarking
- **Visual Design** - Subtle diagonal stripe pattern (15% opacity)
- **Multi-Bit-Depth** - 8-bit, 16-bit, 32-bit float support
- **Conditional Application** - Different triggers per licensing system
- **Render Farm Skip** - Automatic watermark bypass in headless mode

### Render Farm Strategy
- **PF_IsRenderEngine()** - Runtime detection for command-line rendering
- **BUILD_RENDER_ONLY** - Alternative special farm-only build
- **Deployment Patterns** - Workstation vs. render node configurations

## Quick Start

### For Plugin Developers

Want to add licensing to your own plugin? Start here:

1. Read [Integration Guide](07-integration-guide.md) for step-by-step instructions
2. Choose your licensing system:
   - [Gumroad](02-gumroad-licensing.md) - Simple, direct sales
   - [AEScripts](03-aescripts-licensing.md) - Professional marketplace
   - [PluginPlay](04-pluginplay-licensing.md) - Subscription-focused
3. Set up your [Windows build](01-windows-building.md)
4. Implement [watermarking](05-watermarking.md)
5. Configure [render farm support](06-render-farm-strategy.md)

### For Build Engineers

Setting up automated builds:

```batch
# Build all 5 versions at once
build_all_versions.bat

# Output:
# - build/Release_NoLic/MyPlugin.aex (316 KB)
# - build/Release_Gumroad/MyPlugin.aex (355 KB)
# - build/Release_AEScripts/MyPlugin.aex (693 KB)
# - build/Release_PluginPlay/MyPlugin.aex (2.7 MB)
# - build/Release_RenderOnly/MyPlugin_Render.aex (316 KB)
```

See [Windows Building](01-windows-building.md) for details.

### For Render Farm Admins

Two deployment strategies:

**Approach 1: Standard Build with Runtime Detection**
- Install any licensing build (Gumroad/AEScripts/PluginPlay)
- Plugin automatically detects `aerender.exe` and skips watermark
- See [Render Farm Strategy](06-render-farm-strategy.md)

**Approach 2: Dedicated Render-Only Build**
- Use special `BUILD_RENDER_ONLY` version
- No licensing code, no GUI parameters
- See [Render Farm Strategy](06-render-farm-strategy.md)

## Documentation Structure

| Document | Description |
|----------|-------------|
| **[01-windows-building.md](01-windows-building.md)** | Complete Windows build process, CMake, MSBuild, scripts |
| **[02-gumroad-licensing.md](02-gumroad-licensing.md)** | Gumroad header-only implementation (simplest) |
| **[03-aescripts-licensing.md](03-aescripts-licensing.md)** | AEScripts licensing library integration (most features) |
| **[04-pluginplay-licensing.md](04-pluginplay-licensing.md)** | PluginPlay device-based licensing (subscription-friendly) |
| **[05-watermarking.md](05-watermarking.md)** | Watermark system implementation and bit-depth support |
| **[06-render-farm-strategy.md](06-render-farm-strategy.md)** | Two approaches to render farm deployment |
| **[07-integration-guide.md](07-integration-guide.md)** | Step-by-step: Add licensing to your plugin |
| **[08-reference.md](08-reference.md)** | Quick reference tables, paths, product IDs, troubleshooting |

## Key Features

### Single Codebase, Multiple Builds
```cpp
#ifdef ENABLE_GUMROAD_LICENSING
    // Gumroad-specific code
#elif defined(ENABLE_AESCRIPTS_LICENSING)
    // AEScripts-specific code
#elif defined(ENABLE_PLUGINPLAY_LICENSING)
    // PluginPlay-specific code
#else
    // No licensing (free version)
#endif
```

All licensing code is **compile-time optional** via preprocessor flags. Zero overhead when disabled.

### Intelligent Watermarking

Watermark appears ONLY when needed:
- **Unlicensed in GUI** -> Watermark appears
- **Licensed in GUI** -> No watermark
- **Unlicensed in aerender** -> No watermark (render farm friendly!)
- **Licensed in aerender** -> No watermark

See [Watermarking](05-watermarking.md) for implementation details.

### Render Farm Compatibility

```cpp
// Detect headless render mode
PF_Boolean isRenderEngine = FALSE;
PFAppSuite6* appSuiteP = NULL;
in_data->pica_basicP->AcquireSuite(kPFAppSuite, kPFAppSuiteVersion6, &appSuiteP);
if (appSuiteP) {
    appSuiteP->PF_IsRenderEngine(&isRenderEngine);
    in_data->pica_basicP->ReleaseSuite(kPFAppSuite, kPFAppSuiteVersion6);
}

// Skip watermark in render farms
if (!isRenderEngine) {
    // Apply watermark only in GUI mode
}
```

No special licensing required for render farms. See [Render Farm Strategy](06-render-farm-strategy.md).

## System Requirements

### Development Environment
- **OS**: Windows 10/11 (64-bit)
- **Compiler**: Visual Studio 2022 with C++ workload
- **Build System**: CMake 3.20+, MSBuild 17+
- **SDK**: After Effects SDK 2024 or later

### Licensing Libraries
- **Gumroad**: No external libraries (header-only)
- **AEScripts**: `aescriptsLicensing_MT_Release.lib` (contact contact@aescripts.com)
- **PluginPlay**: `libpplic_Release.lib` + OpenSSL + cURL (from PluginPlay SDK)

## Licensing System Comparison

| Feature | Gumroad | AEScripts | PluginPlay | No Licensing |
|---------|---------|-----------|------------|--------------|
| **Binary Size** | 355 KB | 693 KB | 2.7 MB | 316 KB |
| **Implementation** | Header-only | Compiled lib | Compiled lib | N/A |
| **External Deps** | WinHTTP only | Multiple libs | OpenSSL+cURL | None |
| **Machine Limit** | 2 fixed | Configurable | 1 (lifetime) | N/A |
| **Network Check** | Every 48h | On activation | On activation | Never |
| **Offline Grace** | 7 days | Varies | Varies | Forever |
| **Floating Licenses** | No | Yes | No | N/A |
| **Subscriptions** | No | Yes | Yes | N/A |
| **Render-Only Licenses** | No | Yes | No | N/A |
| **Best For** | Direct sales | Marketplace | Subscriptions | Free/Demo |

See [Reference](08-reference.md) for detailed comparison.

## Common Tasks

### Build a Single Version
```batch
# Gumroad build (most common)
cmake -B build -DENABLE_GUMROAD_LICENSING=ON
cmake --build build --config Release
```

### Build All Versions
```batch
# Builds all 5 variants
build_all_versions.bat
```

### Install to After Effects
```batch
copy "build\Release_Gumroad\YourPlugin.aex" ^
     "C:\Program Files\Adobe\Common\Plug-ins\7.0\MediaCore\"
```

### Delete License (for testing)
```batch
# Gumroad
reg delete "HKEY_CURRENT_USER\Software\MyCompany\MyPlugin" /f

# AEScripts
del "%APPDATA%\AESCRIPTS\MyPlugin\MyPlugin.lic"

# PluginPlay
del "%APPDATA%\PluginPlay\myplugin.dat"
```

## File Paths Reference

### Source Code
```
D:\Work\Coding\MyPlugin\
    CMakeLists.txt                          # Build configuration
    src\myplugin\MyPlugin.cpp               # Main plugin
    include\licensing\
        GumroadLicense.h                    # Gumroad (header-only)
        aescriptsLicensing.h                # AEScripts API
        (PluginPlay headers from SDK)
    resources\MyPluginPiPL.r                # Plugin info
    build scripts:
        build_all_versions.bat
        build_gumroad_only.bat
        build_render_only.bat
```

### License Storage

**Gumroad**:
- Windows: `HKEY_CURRENT_USER\Software\MyCompany\MyPlugin`
- macOS: `~/Library/Preferences/com.mycompany.myplugin.plist`

**AEScripts**:
- Windows: `%APPDATA%\AESCRIPTS\MyPlugin\MyPlugin.lic`
- macOS: `~/Library/Application Support/AESCRIPTS/MyPlugin/MyPlugin.lic`

**PluginPlay**:
- Windows: `%APPDATA%\PluginPlay\myplugin.dat`
- macOS: `~/Library/Application Support/PluginPlay/myplugin.dat`

## Troubleshooting

### Plugin won't load in After Effects
- Check Match Name is unique: `ADBE YourPlugin`
- Verify PiPL resource compiled correctly
- Look for duplicate Match Names (can't have both full + render-only)

### Windows Defender quarantines plugin
- Use static linking (`/MT` flag)
- Submit false positive to Microsoft

### License doesn't persist
- Check registry/preferences permissions
- Windows: Run AE as Administrator once
- macOS: Check `~/Library/Preferences` is writable

### Watermark appears on render farm
- Verify `PF_IsRenderEngine()` detection is implemented
- OR use `BUILD_RENDER_ONLY` version instead
- Check `isRenderEngine` value in debugger

### Build fails with "unresolved external symbol"
- **PluginPlay only**: Must use `/MD` runtime
- Remove `-p:RuntimeLibrary=MultiThreaded` from PluginPlay build
- Other systems: Verify library paths in CMakeLists.txt

See [Reference](08-reference.md) for complete troubleshooting guide.

## Best Practices

1. **Develop with No Licensing first** - Focus on plugin functionality
2. **Test all bit depths** - 8-bit, 16-bit, 32-bit float
3. **Test offline behavior** - Disconnect network, verify grace period
4. **Test render farms** - Use `aerender.exe` to verify watermark skip
5. **Test machine limits** - Activate on 3rd machine (should fail for Gumroad)
6. **Use static linking** - Avoids Windows Defender false positives
7. **Version Match Names** - Keep Match Name consistent across updates

## Support & Contact

### Licensing System Providers

- **Gumroad**: https://gumroad.com (self-service)
- **AEScripts**: contact@aescripts.com (requires developer account)
- **PluginPlay**: pluginplay.com/developers (registration required)

### After Effects SDK

- **Adobe**: After Effects SDK download at developer.adobe.com
- **Forums**: community.adobe.com/aftereffects

## Next Steps

1. **New to plugin development?** Start with [Windows Building](01-windows-building.md)
2. **Adding licensing?** Jump to [Integration Guide](07-integration-guide.md)
3. **Setting up render farms?** Read [Render Farm Strategy](06-render-farm-strategy.md)
4. **Need quick answers?** Check [Reference](08-reference.md)
