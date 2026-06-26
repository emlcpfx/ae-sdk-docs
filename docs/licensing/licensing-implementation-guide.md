# MyBlurPlugin - Gumroad Licensing Implementation Guide

## Completed Steps

### 1. Licensing Header
- **File**: `include/licensing/GumroadLicense.h`
- **Status**: Copied and customized
- **Changes**: Updated registry paths, user agents, and preferences domain

### 2. CMakeLists.txt
- **Status**: Updated with Gumroad licensing option
- **Changes**:
  - Added `ENABLE_GUMROAD_LICENSING` option
  - Conditional WinHTTP/cURL linking
  - Separate output directories for Gumroad vs No-Lic builds

### 3. Full Plugin Implementation
- **File**: `src/plugin/Plugin_Full.cpp`
- **Status**: Created (needs to replace Plugin.cpp)
- **Features**:
  - Gumroad license validation on plugin load
  - "License" button in Effect Controls
  - Windows license dialog with activation
  - Watermark functions (8/16/32-bit)
  - Render farm detection with PF_IsRenderEngine()
  - Conditional watermark application

## Manual Steps Required

### Step 1: Replace Plugin.cpp

```batch
REM Backup original
copy "src\plugin\Plugin.cpp" "src\plugin\Plugin_Original.cpp.bak"

REM Replace with licensed version
copy "src\plugin\Plugin_Full.cpp" "src\plugin\Plugin.cpp"
```

### Step 2: Add Gumroad Product ID

Edit `src/plugin/Plugin.cpp`:

Find line:
```cpp
#define GUMROAD_PRODUCT_ID "YOUR_PRODUCT_ID_HERE"
```

Replace with your actual Gumroad product ID (get from Gumroad product settings).

### Step 3: Create Windows Dialog Resource

Create `resources\License.rc`:

```rc
#include <windows.h>

IDD_GUMROAD_LICENSE DIALOGEX 0, 0, 300, 120
STYLE DS_SETFONT | DS_MODALFRAME | DS_FIXEDSYS | WS_POPUP | WS_CAPTION | WS_SYSMENU
CAPTION "MyBlurPlugin - License"
FONT 8, "MS Shell Dlg", 400, 0, 0x1
BEGIN
    LTEXT "Enter your license key from Gumroad:", IDC_STATIC, 10, 10, 280, 10
    EDITTEXT IDC_LICENSE_KEY, 10, 25, 280, 14, ES_AUTOHSCROLL
    LTEXT "After activating, restart After Effects for changes to take effect.", IDC_STATIC, 10, 50, 280, 20
    DEFPUSHBUTTON "Activate", IDOK, 180, 95, 50, 14
    PUSHBUTTON "Cancel", IDCANCEL, 240, 95, 50, 14
END
```

Add to CMakeLists.txt (Windows section):
```cmake
if(ENABLE_GUMROAD_LICENSING)
    target_sources(MyBlurPlugin PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/resources/License.rc)
endif()
```

### Step 4: Create Build Scripts

**`build_nolic.bat`**:
```batch
@echo off
echo Building MyBlurPlugin - No Licensing Version
echo.

set CMAKE_PATH="C:\Program Files\CMake\bin\cmake.exe"
set MSBUILD_PATH="C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\MSBuild.exe"

%CMAKE_PATH% -B build -DENABLE_GUMROAD_LICENSING=OFF

%MSBUILD_PATH% build\MyBlurPlugin.vcxproj ^
    -p:Configuration=Release ^
    -p:Platform=x64 ^
    -p:RuntimeLibrary=MultiThreaded

if %ERRORLEVEL% NEQ 0 (
    echo [ERROR] Build failed!
    pause
    exit /b 1
)

echo [SUCCESS] Output: build\Release_NoLic\MyBlurPlugin.aex
pause
```

**`build_gumroad.bat`**:
```batch
@echo off
echo Building MyBlurPlugin - Gumroad Version
echo.

set CMAKE_PATH="C:\Program Files\CMake\bin\cmake.exe"
set MSBUILD_PATH="C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\MSBuild.exe"

%CMAKE_PATH% -B build -DENABLE_GUMROAD_LICENSING=ON

%MSBUILD_PATH% build\MyBlurPlugin.vcxproj ^
    -p:Configuration=Release ^
    -p:Platform=x64 ^
    -p:RuntimeLibrary=MultiThreaded

if %ERRORLEVEL% NEQ 0 (
    echo [ERROR] Build failed!
    pause
    exit /b 1
)

echo [SUCCESS] Output: build\Release_Gumroad\MyBlurPlugin.aex
pause
```

## Testing

### Test No-Licensing Version

```batch
build_nolic.bat
copy "build\Release_NoLic\MyBlurPlugin.aex" ^
     "C:\Program Files\Adobe\Common\Plug-ins\7.0\MediaCore\"
# Apply to layer -> Should work with no restrictions
```

### Test Gumroad Version (Unlicensed)

```batch
build_gumroad.bat
reg delete "HKEY_CURRENT_USER\Software\MyCompany\MyBlurPlugin" /f
copy "build\Release_Gumroad\MyBlurPlugin.aex" ^
     "C:\Program Files\Adobe\Common\Plug-ins\7.0\MediaCore\"
# Apply to layer -> WATERMARK appears
```

### Test Render Farm (Unlicensed)

```batch
reg delete "HKEY_CURRENT_USER\Software\MyCompany\MyBlurPlugin" /f
"C:\Program Files\Adobe\Adobe After Effects 2024\Support Files\aerender.exe" ^
    -project "test.aep" -comp "Comp 1"
# Check output -> NO watermark (farm-friendly!)
```

## Build Comparison

| Version | Size | License Check | Watermark | Best For |
|---------|------|---------------|-----------|----------|
| **No Licensing** | ~320 KB | Never | Never | Free/demo distribution |
| **Gumroad** | ~360 KB | On load + every 48h | When unlicensed in GUI | Direct sales |

## Troubleshooting

### "License" button doesn't appear
- Check `ENABLE_GUMROAD_LICENSING=ON` in build
- Verify not running in Premiere Pro

### Watermark appears on render farm
- Check `PF_IsRenderEngine()` is implemented
- Add debug logging to verify detection

### License doesn't persist
- Run After Effects as Administrator once
- Check registry permissions

## References

- Full licensing documentation: see licensing/ directory
- Gumroad API: https://gumroad.com/api
