# Quick Reference

Fast lookup tables, file paths, product IDs, and troubleshooting guide for After Effects plugin licensing.

## Licensing Systems Comparison

| Feature | Gumroad | AEScripts | PluginPlay | No Licensing |
|---------|---------|-----------|------------|--------------|
| **Binary Size** | 355 KB | 693 KB | 2.7 MB | 316 KB |
| **Implementation** | Header-only | Compiled lib | Compiled lib | N/A |
| **External Deps** | WinHTTP only | Multiple libs | OpenSSL+cURL | None |
| **Runtime** | `/MT` | `/MT` | `/MD` | `/MT` |
| **Machine Limit** | 2 (fixed) | Configurable | 1-4 (by plan) | N/A |
| **Network Check** | Every 48h | On activation | On activation | Never |
| **Offline Grace** | 7 days | Varies | 7 days | Forever |
| **Floating Licenses** | No | Yes | No | N/A |
| **Subscriptions** | No | Yes | Yes | N/A |
| **Render-Only** | No | Yes | No | N/A |
| **Setup Complexity** | Low | Medium | Medium | N/A |
| **Best For** | Direct sales | Marketplace | Subscriptions | Free/Demo |
| **Commission** | 3.5% + 30c | 30-40% | ~30% | N/A |

## Build Outputs

| Build Variant | Size | Output Directory | Filename |
|---------------|------|------------------|----------|
| No Licensing | 316 KB | `Release_NoLic` | `YourPlugin.aex` |
| Gumroad | 355 KB | `Release_Gumroad` | `YourPlugin.aex` |
| AEScripts | 693 KB | `Release_AEScripts` | `YourPlugin.aex` |
| PluginPlay | 2.7 MB | `Release_PluginPlay` | `YourPlugin.aex` |
| Render-Only | 316 KB | `Release_RenderOnly` | `YourPlugin_Render.aex` |

## File Paths

### Installation Paths

**Windows**:
```
C:\Program Files\Adobe\Common\Plug-ins\7.0\MediaCore\YourPlugin.aex
```

**macOS**:
```
/Library/Application Support/Adobe/Common/Plug-ins/7.0/MediaCore/YourPlugin.plugin
```

### License Storage Locations

| System | Windows | macOS |
|--------|---------|-------|
| **Gumroad** | `HKEY_CURRENT_USER\Software\MyCompany\MyPlugin` | `~/Library/Preferences/com.mycompany.myplugin.plist` |
| **AEScripts** | `%APPDATA%\AESCRIPTS\MyPlugin\MyPlugin.lic` | `~/Library/Application Support/AESCRIPTS/MyPlugin/MyPlugin.lic` |
| **PluginPlay** | `%APPDATA%\PluginPlay\myplugin.dat` | `~/Library/Application Support/PluginPlay/myplugin.dat` |

## Product IDs

### How to Get IDs

**Gumroad**:
1. Product dashboard -> Edit product
2. License Keys -> Expand module
3. Copy `product_id` from embed code

**AEScripts**:
1. Email contact@aescripts.com
2. Provide plugin details
3. Receive product ID and private number

**PluginPlay**:
1. Register at pluginplay.com/developers
2. Create product
3. Choose `product_id` (lowercase, no spaces)

## Build Commands Reference

### CMake Configuration

```bash
# No licensing
cmake -B build -DENABLE_GUMROAD_LICENSING=OFF

# Gumroad
cmake -B build -DENABLE_GUMROAD_LICENSING=ON

# AEScripts
cmake -B build -DENABLE_AESCRIPTS_LICENSING=ON

# PluginPlay
cmake -B build -DENABLE_PLUGINPLAY_LICENSING=ON

# Render-only
cmake -B build -DBUILD_RENDER_ONLY=ON
```

### MSBuild Commands

```batch
# Standard build
msbuild build\YourPlugin.vcxproj -p:Configuration=Release -p:Platform=x64

# Static runtime (/MT)
msbuild build\YourPlugin.vcxproj ^
    -p:Configuration=Release ^
    -p:Platform=x64 ^
    -p:RuntimeLibrary=MultiThreaded

# Dynamic runtime (/MD) - required for PluginPlay
msbuild build\YourPlugin.vcxproj ^
    -p:Configuration=Release ^
    -p:Platform=x64
```

## Testing Commands

### Delete Licenses

**Gumroad (Windows)**:
```batch
reg delete "HKEY_CURRENT_USER\Software\MyCompany\MyPlugin" /f
```

**Gumroad (macOS)**:
```bash
defaults delete com.mycompany.myplugin
```

**AEScripts (Windows)**:
```batch
del "%APPDATA%\AESCRIPTS\MyPlugin\MyPlugin.lic"
```

**PluginPlay (Windows)**:
```batch
del "%APPDATA%\PluginPlay\myplugin.dat"
```

### Test Render Farm

```batch
"C:\Program Files\Adobe\Adobe After Effects 2024\Support Files\aerender.exe" ^
    -project "C:\path\to\project.aep" ^
    -comp "Comp 1" ^
    -output "C:\output.mov"
```

## Troubleshooting Guide

### Build Issues

| Error | Cause | Solution |
|-------|-------|----------|
| `GumroadLicense.h not found` | Wrong include path | Check `include_directories()` |
| `unresolved external symbol WinHttpOpen` | Missing library | Add `winhttp.lib` |
| `unresolved external symbol __imp___stdio_common_vsscanf` | PluginPlay with `/MT` | Remove `-p:RuntimeLibrary=MultiThreaded` |
| `CMake Error: Could not find AESDK_ROOT` | SDK path not set | Set `AESDK_ROOT` variable |

### Runtime Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Plugin won't load | Duplicate Match Name | Remove one version |
| "License" button missing | Suite acquisition failed | Check `appl_id != 'PrMr'` |
| License doesn't persist | Registry write failed | Run AE as Administrator once |
| Watermark on farm | Detection not implemented | Add `PF_IsRenderEngine()` |
| Windows Defender flags | Using `/MD` runtime | Use `/MT` (except PluginPlay) |

## Code Snippets

### Detect Render Farm

```cpp
PF_Boolean isRenderEngine = FALSE;
PFAppSuite6* appSuiteP = NULL;
in_data->pica_basicP->AcquireSuite(kPFAppSuite, kPFAppSuiteVersion6, &appSuiteP);
if (appSuiteP) {
    appSuiteP->PF_IsRenderEngine(&isRenderEngine);
    in_data->pica_basicP->ReleaseSuite(kPFAppSuite, kPFAppSuiteVersion6);
}
```

### Add "License" Button

```cpp
PF_EffectUISuite1* effect_ui_suiteP = NULL;
in_data->pica_basicP->AcquireSuite(kPFEffectUISuite, kPFEffectUISuiteVersion1, &effect_ui_suiteP);
if (effect_ui_suiteP) {
    effect_ui_suiteP->PF_SetOptionsButtonName(in_data->effect_ref, "License");
    in_data->pica_basicP->ReleaseSuite(kPFEffectUISuite, kPFEffectUISuiteVersion1);
}
```

### Bit Depth Detection

```cpp
int bytes_per_pixel = output_worldP->rowbytes / output_worldP->width;
if (bytes_per_pixel == 16) {
    // 32-bit float
} else if (bytes_per_pixel == 8) {
    // 16-bit
} else {
    // 8-bit
}
```

## Preprocessor Flags

| Flag | Purpose | Set By |
|------|---------|--------|
| `ENABLE_GUMROAD_LICENSING` | Enable Gumroad licensing | CMake option |
| `ENABLE_AESCRIPTS_LICENSING` | Enable AEScripts licensing | CMake option |
| `ENABLE_PLUGINPLAY_LICENSING` | Enable PluginPlay licensing | CMake option |
| `BUILD_RENDER_ONLY` | Build render-only version | CMake option |
| `_WIN32` | Windows platform | Compiler |
| `__APPLE__` | macOS platform | Compiler |

## Testing Checklist

### Before Release

- [ ] Build all versions (no-lic, Gumroad, AEScripts, PluginPlay)
- [ ] Test 8-bit project
- [ ] Test 16-bit project
- [ ] Test 32-bit float project
- [ ] Test GUI unlicensed -> watermark appears
- [ ] Test GUI licensed -> no watermark
- [ ] Test aerender unlicensed -> no watermark (farm-friendly)
- [ ] Test license persistence (close/reopen AE)
- [ ] Test offline mode (within 7 days)
- [ ] Test machine limits (3rd activation blocked)
- [ ] Test Windows Defender (not quarantined)
- [ ] Test MFR (Multi-Frame Rendering)
- [ ] Test on clean machine

## Resources

### External Resources

**Licensing Providers**:
- Gumroad: https://gumroad.com
- AEScripts: https://aescripts.com (contact@aescripts.com)
- PluginPlay: https://pluginplay.com/developers

**After Effects SDK**:
- Download: https://developer.adobe.com
- Forums: https://community.adobe.com/aftereffects

**Tools**:
- Visual Studio: https://visualstudio.microsoft.com
- CMake: https://cmake.org
