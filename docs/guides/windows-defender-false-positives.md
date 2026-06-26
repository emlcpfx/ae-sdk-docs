# Windows Defender False Positive Solutions for After Effects Plugins

## Problem
Windows Defender flags unsigned After Effects plugins as "Trojan:Win32/Wacatac.C!ml" - a machine learning-based detection that triggers on behavioral patterns common in AE plugins.

## Why This Happens
1. **Unsigned Binary**: Plugin lacks digital signature verification
2. **DLL Injection Pattern**: AE plugins inject into After Effects process (looks suspicious)
3. **ML Detection**: The "!ml" suffix indicates algorithmic pattern matching, not signature-based detection

## Solution 1: Static Linking (Most Effective)

### Configure CMake for Static Runtime
Edit your CMakeLists.txt to use static runtime:
```cmake
set_property(TARGET YourPlugin PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
```

### Build with Static Linking
```batch
@echo off
echo Building with static linking...

REM Set to use static runtime (/MT instead of /MD)
set CL=/MT /O2 /GL /GS /DYNAMICBASE /NXCOMPAT

REM Build with static libraries
"C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\MSBuild.exe" ^
    "YourProject.sln" ^
    -p:Configuration=Release ^
    -p:Platform=x64 ^
    -p:RuntimeLibrary=MultiThreaded ^
    -p:CharacterSet=Unicode

echo Static build complete
```

### Why Static Linking Helps
- Eliminates suspicious dynamic import patterns
- Includes all dependencies directly in the binary
- Changes the overall binary fingerprint significantly
- Results in different ML signature that is less likely to trigger false positives

## Solution 2: Add Version Information

Create a resource file (.rc) with version info:

```rc
#include <windows.h>

1 RT_MANIFEST "YourPlugin.manifest"

VS_VERSION_INFO VERSIONINFO
FILEVERSION    1,0,0,0
PRODUCTVERSION 1,0,0,0
FILEFLAGSMASK  VS_FFI_FILEFLAGSMASK
FILEFLAGS      0x0L
FILEOS         VOS_NT_WINDOWS32
FILETYPE       VFT_DLL
FILESUBTYPE    VFT2_UNKNOWN
BEGIN
    BLOCK "StringFileInfo"
    BEGIN
        BLOCK "040904E4"
        BEGIN
            VALUE "CompanyName",      "Your Company"
            VALUE "FileDescription",  "Plugin Description"
            VALUE "FileVersion",      "1.0.0.0"
            VALUE "InternalName",     "PluginName"
            VALUE "LegalCopyright",   "Copyright (C) 2025"
            VALUE "OriginalFilename", "Plugin.aex"
            VALUE "ProductName",      "Your Product"
            VALUE "ProductVersion",   "1.0.0.0"
        END
    END
    BLOCK "VarFileInfo"
    BEGIN
        VALUE "Translation", 0x409, 1252
    END
END
```

## Solution 3: Add Application Manifest

Create a manifest file (.manifest):

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
  <assemblyIdentity
    version="1.0.0.0"
    processorArchitecture="amd64"
    name="YourPlugin.AfterEffects.Plugin"
    type="win32"
  />
  <description>Your Plugin Description</description>
  <trustInfo xmlns="urn:schemas-microsoft-com:asm.v3">
    <security>
      <requestedPrivileges>
        <requestedExecutionLevel level="asInvoker" uiAccess="false"/>
      </requestedPrivileges>
    </security>
  </trustInfo>
  <compatibility xmlns="urn:schemas-microsoft-com:compatibility.v1">
    <application>
      <!-- Windows 10 -->
      <supportedOS Id="{8e0f7a12-bfb3-4fe8-b9a5-48fd50a15a9a}"/>
      <!-- Windows 11 -->
      <supportedOS Id="{1f676c76-80e1-4239-95bb-83d0f6d0da78}"/>
    </application>
  </compatibility>
</assembly>
```

## Solution 4: Security Compiler Flags

Add these to your build configuration to enable security features that also change the binary layout:

```xml
<ClCompile>
  <!-- Security features -->
  <ControlFlowGuard>Guard</ControlFlowGuard>
  <SpectreMitigation>Spectre</SpectreMitigation>
  <SDLCheck>true</SDLCheck>
  <GuardEHContMetadata>true</GuardEHContMetadata>

  <!-- Optimizations that affect binary layout -->
  <WholeProgramOptimization>true</WholeProgramOptimization>
  <InlineFunctionExpansion>AnySuitable</InlineFunctionExpansion>
  <FunctionLevelLinking>true</FunctionLevelLinking>
</ClCompile>

<Link>
  <!-- Security features -->
  <DataExecutionPrevention>true</DataExecutionPrevention>
  <RandomizedBaseAddress>true</RandomizedBaseAddress>
  <CETCompat>true</CETCompat>

  <!-- Changes binary layout -->
  <LinkTimeCodeGeneration>UseLinkTimeCodeGeneration</LinkTimeCodeGeneration>
</Link>
```

## For End Users

### Quick Fix: Whitelist the Plugin
PowerShell script for users to run as Administrator:

```powershell
# whitelist_plugin.ps1
$pluginPath = "C:\Program Files\Adobe\Common\Plug-ins\7.0\MediaCore\YourPlugin.aex"

Write-Host "Adding Windows Defender exclusion..." -ForegroundColor Green

try {
    Add-MpPreference -ExclusionPath $pluginPath -ErrorAction Stop
    Write-Host "Successfully added exclusion!" -ForegroundColor Green
} catch {
    Write-Host "Error: Please run as Administrator" -ForegroundColor Red
}
```

### Report False Positive to Microsoft
1. Visit: https://www.microsoft.com/wdsi/filesubmission
2. Select "Software developer"
3. Upload the .aex file
4. Provide details:
   - Software name: Your Plugin Name
   - Category: Video/Graphics Software
   - Description: After Effects plugin for [your functionality]

## Long-term Solution: Code Signing

### Option 1: Standard Code Signing Certificate
- Cost: $200-300/year
- Providers: Sectigo, DigiCert
- Reduces warnings but may still trigger SmartScreen initially

### Option 2: EV Code Signing Certificate
- Cost: $300-500/year
- Immediate SmartScreen bypass
- Hardware token required
- Best option for professional distribution

### How to Sign Your Plugin
```batch
signtool sign /f "certificate.pfx" /p "password" /t http://timestamp.sectigo.com "YourPlugin.aex"
```

## Build Process Summary

1. **Clean build directory**: `rm -rf build`
2. **Configure with CMake**: `cmake -B build -G "Visual Studio 17 2022" -A x64`
3. **Build with static linking**: Use `/MT` flag via MSBuild or CMake

## Testing Your Build

1. **Test on clean VM**: Verify plugin loads without warnings
2. **Submit to VirusTotal**: Check detection rate (expect some false positives initially)
3. **Test installation**: Ensure plugin works in After Effects

## Common Issues and Solutions

### Issue: Still Getting Flagged
- Try building on different machine (changes environmental factors)
- Change optimization levels (/O1 vs /O2)
- Enable additional security flags (Control Flow Guard, CET)

### Issue: Plugin Won't Load After Changes
- Verify PE headers are intact
- Ensure manifest is properly embedded
- Test without post-processing first

### Issue: Build Fails with Static Linking
- Remove MFC references (not needed for AE plugins)
- Ensure all dependencies support /MT flag
- Check for conflicting runtime library settings

## Summary

The most effective approach for avoiding false positives:
1. **Static linking** (/MT) fundamentally changes the binary structure
2. **Version information and manifest** add legitimacy metadata
3. **Security compiler flags** change code generation patterns
4. **Code signing** is the definitive long-term solution for professional distribution
