# Windows Build Process for After Effects Plugins

Complete guide to building After Effects plugins on Windows with CMake, MSBuild, and automated build scripts.

## Table of Contents

- [Build Environment](#build-environment)
- [Build Tools](#build-tools)
- [CMake Configuration](#cmake-configuration)
- [Build Scripts](#build-scripts)
- [Runtime Library Configuration](#runtime-library-configuration)
- [Windows Defender Bypass](#windows-defender-bypass)
- [Build Outputs](#build-outputs)
- [Troubleshooting](#troubleshooting)

## Build Environment

### Requirements

- **Operating System**: Windows 10/11 (64-bit)
- **Compiler**: Visual Studio 2022 (Community, Professional, or Enterprise)
- **Workloads**: Desktop development with C++
- **CMake**: 3.20 or later (bundled with VS2022)
- **MSBuild**: 17.0 or later (bundled with VS2022)
- **After Effects SDK**: 2024 or later

### SDK Path Configuration

The After Effects SDK must be accessible to CMake:

```cmake
# In CMakeLists.txt
set(AESDK_ROOT_DEFAULT "path/to/AfterEffectsSDK/Examples")
```

**To override**, set environment variable or CMake variable:
```batch
set AESDK_ROOT=C:\Path\To\AfterEffectsSDK\Examples
cmake -B build -DAESDK_ROOT="C:\Path\To\AfterEffectsSDK\Examples"
```

## Build Tools

### Tool Paths

Default Visual Studio 2022 paths:

```batch
# CMake
"C:\Program Files\Microsoft Visual Studio\2022\Community\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe"

# MSBuild
"C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\MSBuild.exe"

# Compiler (cl.exe)
"C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.44.35207\bin\Hostx64\x64\cl.exe"
```

### Command Line Setup

**Option 1: VS Developer Command Prompt** (Recommended)
```batch
# Start Menu -> Visual Studio 2022 -> Developer Command Prompt for VS 2022
# All tools automatically in PATH
cmake --version
msbuild -version
```

**Option 2: Use Full Paths** (in scripts)
```batch
set CMAKE_PATH="C:\Program Files\...\cmake.exe"
set MSBUILD_PATH="C:\Program Files\...\MSBuild.exe"
%CMAKE_PATH% -B build
%MSBUILD_PATH% build\YourPlugin.vcxproj
```

## CMake Configuration

### Basic Configuration

```batch
# Configure project
cmake -B build -S .

# Build (calls MSBuild internally)
cmake --build build --config Release
```

### Licensing Options

CMake options control which licensing system to build:

```cmake
option(ENABLE_AESCRIPTS_LICENSING "Enable aescripts licensing" OFF)
option(ENABLE_PLUGINPLAY_LICENSING "Enable PluginPlay licensing" OFF)
option(ENABLE_GUMROAD_LICENSING "Enable Gumroad licensing" OFF)
option(BUILD_RENDER_ONLY "Build render-only version" OFF)
```

**Rules**:
- Only **ONE** licensing flag can be ON at a time
- OR all OFF for free version
- `BUILD_RENDER_ONLY` is mutually exclusive with licensing flags

### Build Examples

**No Licensing (Free Version)**:
```batch
cmake -B build ^
    -DENABLE_AESCRIPTS_LICENSING=OFF ^
    -DENABLE_PLUGINPLAY_LICENSING=OFF ^
    -DENABLE_GUMROAD_LICENSING=OFF ^
    -DBUILD_RENDER_ONLY=OFF

cmake --build build --config Release
```

**Gumroad Licensing**:
```batch
cmake -B build ^
    -DENABLE_GUMROAD_LICENSING=ON ^
    -DENABLE_AESCRIPTS_LICENSING=OFF ^
    -DENABLE_PLUGINPLAY_LICENSING=OFF

cmake --build build --config Release
```

**Render-Only Version**:
```batch
cmake -B build ^
    -DBUILD_RENDER_ONLY=ON ^
    -DENABLE_AESCRIPTS_LICENSING=OFF ^
    -DENABLE_PLUGINPLAY_LICENSING=OFF ^
    -DENABLE_GUMROAD_LICENSING=OFF

cmake --build build --config Release
```

### Output Directory Control

CMake automatically generates output paths based on licensing flags:

```cmake
if(BUILD_RENDER_ONLY)
    set(LICENSE_SUFFIX "_RenderOnly")
    set(OUTPUT_FILENAME "MyPlugin_Render")
elseif(ENABLE_GUMROAD_LICENSING)
    set(LICENSE_SUFFIX "_Gumroad")
    set(OUTPUT_FILENAME "MyPlugin")
elseif(ENABLE_AESCRIPTS_LICENSING)
    set(LICENSE_SUFFIX "_AEScripts")
    set(OUTPUT_FILENAME "MyPlugin")
elseif(ENABLE_PLUGINPLAY_LICENSING)
    set(LICENSE_SUFFIX "_PluginPlay")
    set(OUTPUT_FILENAME "MyPlugin")
else()
    set(LICENSE_SUFFIX "_NoLic")
    set(OUTPUT_FILENAME "MyPlugin")
endif()

set_target_properties(MyPlugin PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY_RELEASE "${CMAKE_BINARY_DIR}/Release${LICENSE_SUFFIX}"
)
```

### Preprocessor Definitions

CMake sets preprocessor flags based on options:

```cmake
if(ENABLE_GUMROAD_LICENSING)
    target_compile_definitions(MyPlugin PRIVATE ENABLE_GUMROAD_LICENSING=1)
endif()
```

Used in code:
```cpp
#ifdef ENABLE_GUMROAD_LICENSING
    // Gumroad-specific code
#endif
```

### Library Linking

#### Gumroad (Minimal)
```cmake
if(WIN32)
    target_link_libraries(MyPlugin PRIVATE
        winhttp.lib  # HTTP client (system library)
    )
endif()
```

#### AEScripts (Multiple Libraries)
```cmake
target_link_libraries(MyPlugin PRIVATE
    ${AESCRIPTS_LIC_ROOT}/lib/Windows/VS2022/aescriptsLicensing_MT_Release.lib
    ws2_32.lib      # Windows Sockets
    iphlpapi.lib    # IP Helper API
    winhttp.lib     # HTTP client
    crypt32.lib     # Cryptography
    bcrypt.lib      # Modern crypto
)
```

#### PluginPlay (Largest)
```cmake
target_link_libraries(MyPlugin PRIVATE
    ${PLUGINPLAY_LIC_ROOT}/lib/x64/libpplic_Release.lib
    ${PLUGINPLAY_LIC_ROOT}/lib/x64/libssl_static.lib
    ${PLUGINPLAY_LIC_ROOT}/lib/x64/libcrypto_static.lib
    ${PLUGINPLAY_LIC_ROOT}/lib/x64/libcurl_a.lib
    ws2_32.lib crypt32.lib wldap32.lib normaliz.lib
)
```

## Build Scripts

Automated build scripts for all variants.

### build_all_versions.bat

Builds all 5 plugin variants in sequence:

```batch
@echo off
echo Building ALL Plugin Versions
echo.

set CMAKE_PATH="C:\Program Files\...\cmake.exe"
set MSBUILD_PATH="C:\Program Files\...\MSBuild.exe"
set BUILD_DIR=build

REM ========================================
REM [1/5] No Licensing
REM ========================================
del /Q "%BUILD_DIR%\CMakeCache.txt" 2>nul

%CMAKE_PATH% -B %BUILD_DIR% -S . ^
    -DENABLE_AESCRIPTS_LICENSING=OFF ^
    -DENABLE_PLUGINPLAY_LICENSING=OFF ^
    -DENABLE_GUMROAD_LICENSING=OFF ^
    -DCMAKE_BUILD_TYPE=Release

%MSBUILD_PATH% "%BUILD_DIR%\MyPlugin.vcxproj" ^
    -p:Configuration=Release ^
    -p:Platform=x64 ^
    -p:RuntimeLibrary=MultiThreaded

if %ERRORLEVEL% NEQ 0 (
    echo [ERROR] No Licensing build failed!
    pause
    exit /b 1
)

echo [SUCCESS] No Licensing version built

REM ... (AEScripts, PluginPlay, Gumroad, Render-Only follow same pattern)

echo.
echo ========================================
echo ALL BUILDS COMPLETE!
echo ========================================
pause
```

### Manual MSBuild Commands

**Standard Build**:
```batch
msbuild "build\MyPlugin.vcxproj" ^
    -p:Configuration=Release ^
    -p:Platform=x64
```

**With Static Runtime**:
```batch
msbuild "build\MyPlugin.vcxproj" ^
    -p:Configuration=Release ^
    -p:Platform=x64 ^
    -p:RuntimeLibrary=MultiThreaded
```

## Runtime Library Configuration

Critical distinction between licensing variants.

### /MT vs /MD

**`/MT` (MultiThreaded - Static Runtime)**:
- Links CRT statically into plugin
- No external DLL dependencies
- Larger binary size
- **Reduces Windows Defender false positives**

**`/MD` (MultiThreaded DLL - Dynamic Runtime)**:
- Links CRT dynamically (requires `vcruntime140.dll`)
- Smaller binary size
- **Required for PluginPlay** (their libs built with `/MD`)

### Runtime Configuration Per Build

| Build Variant | Runtime | MSBuild Flag | Reason |
|---------------|---------|--------------|--------|
| No Licensing | `/MT` | `-p:RuntimeLibrary=MultiThreaded` | Fewer false positives |
| Gumroad | `/MT` | `-p:RuntimeLibrary=MultiThreaded` | Fewer false positives |
| AEScripts | `/MT` | `-p:RuntimeLibrary=MultiThreaded` | Fewer false positives |
| PluginPlay | `/MD` | (none - CMake default) | **Must match** PluginPlay libs |
| Render-Only | `/MT` | `-p:RuntimeLibrary=MultiThreaded` | Simplest deployment |

### Why PluginPlay Requires /MD

PluginPlay provides precompiled `.lib` files built with `/MD`. Using `/MT` causes linker errors:

```
error LNK2019: unresolved external symbol __imp___stdio_common_vsscanf
error LNK2019: unresolved external symbol __imp___stdio_common_vsprintf
```

**Solution**: Do NOT add `-p:RuntimeLibrary=MultiThreaded` for PluginPlay builds.

## Windows Defender Bypass

Windows Defender and other AV systems frequently flag After Effects plugins as malware due to their DLL-like structure and dynamic loading behavior.

### The Problem

**False Positive Triggers**:
- Plugins are `.aex` files (renamed `.dll`)
- Loaded dynamically by After Effects
- Access system resources (registry, network)
- ML-based AV heuristics flag as "suspicious"

### Solution 1: Static Linking (Most Effective)

```batch
msbuild MyPlugin.vcxproj ^
    -p:Configuration=Release ^
    -p:Platform=x64 ^
    -p:RuntimeLibrary=MultiThreaded
```

**Effectiveness**: 90%+ success rate

### Solution 2: Security Compiler Flags

```batch
set CL=/O2 /GL /GS /DYNAMICBASE /NXCOMPAT
set LDFLAGS=/LTCG /OPT:REF /OPT:ICF /GUARD:CF
```

### Solution 3: Code Signing (Professional)

```batch
signtool sign /f "cert.pfx" /p "password" /t http://timestamp.digicert.com "MyPlugin.aex"
```

### Testing for False Positives

```batch
REM Check file with Windows Defender
"%ProgramFiles%\Windows Defender\MpCmdRun.exe" -Scan -ScanType 3 -File "MyPlugin.aex"
```

## Build Outputs

### File Sizes (Release, x64)

| Build Variant | Size | Runtime | Dependencies |
|---------------|------|---------|--------------|
| No Licensing | 316 KB | `/MT` | None |
| Gumroad | 355 KB | `/MT` | WinHTTP (system) |
| AEScripts | 693 KB | `/MT` | Multiple libs |
| PluginPlay | 2.7 MB | `/MD` | OpenSSL + cURL |
| Render-Only | 316 KB | `/MT` | None |

### Installation

```batch
REM Copy to After Effects plugin directory
copy /Y "build\Release_Gumroad\MyPlugin.aex" ^
     "C:\Program Files\Adobe\Common\Plug-ins\7.0\MediaCore\MyPlugin.aex"
```

**Common Plugin Paths**:
- **After Effects 2024**: `C:\Program Files\Adobe\Common\Plug-ins\7.0\MediaCore\`
- **User Plugins**: `C:\Program Files\Adobe\Adobe After Effects 2024\Support Files\Plug-ins\`

## Troubleshooting

### CMake can't find SDK

**Solution**:
```batch
set AESDK_ROOT=C:\Path\To\AfterEffectsSDK\Examples
cmake -B build -DAESDK_ROOT="C:\Path\To\AfterEffectsSDK\Examples"
```

### Linker errors with PluginPlay

**Cause**: Using `/MT` with PluginPlay (must use `/MD`)

**Solution**: Remove `-p:RuntimeLibrary=MultiThreaded` from PluginPlay build.

### Plugin won't load in After Effects

**Check PiPL Resource**:
```batch
"C:\Program Files\...\dumpbin.exe" /RESOURCES "MyPlugin.aex"
```

### Windows Defender quarantines plugin

1. Use static linking (`/MT`)
2. Add exclusion (temporary): `powershell -Command "Add-MpPreference -ExclusionPath 'build'"`
3. Submit false positive: https://www.microsoft.com/wdsi/filesubmission

## Best Practices

1. **Use Build Scripts** - Consistent, repeatable builds
2. **Clean CMake Cache** - Between different licensing builds
3. **Static Linking** - For reduced AV false positives (except PluginPlay)
4. **Test All Bit Depths** - 8-bit, 16-bit, 32-bit float
5. **Version Control** - Tag release builds in Git
6. **Archive Builds** - Keep `.aex` files for each release
7. **Test Installation** - Copy to AE and verify loading
8. **Check File Size** - Verify matches expected size

## Next Steps

- [Gumroad Licensing](02-gumroad-licensing.md) - Add Gumroad licensing
- [AEScripts Licensing](03-aescripts-licensing.md) - Add AEScripts licensing
- [PluginPlay Licensing](04-pluginplay-licensing.md) - Add PluginPlay licensing
- [Integration Guide](07-integration-guide.md) - Step-by-step licensing integration
