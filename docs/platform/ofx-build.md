# Building Against the OpenFX SDK

> Compiling an OFX plugin means linking the OpenFX C++ Support library, laying
> out a `.ofx.bundle` the host can discover, and -- on macOS with Command Line
> Tools only -- working around an empty libc++ include directory. This page
> covers the SDK directory layout, CMake linking, and bundle structure for any
> OFX plugin.

For the plugin code itself see
[The OpenFX C++ Plugin Model](ofx-plugin-structure.md).

## OpenFX SDK directory layout

A built OpenFX SDK has a consistent internal layout. Note that the distribution
archives commonly **extract into an `OpenFX/` subdirectory** -- point your build
at that inner directory, not the extraction root:

```
OpenFX/
  include/openfx/          <- core C headers (ofxCore.h, ofxImageEffect.h, ...)
  include/openfx/Support/  <- C++ support headers (ofxsImageEffect.h, ...)
  lib/libOfxSupport.a      <- macOS static support library
  lib/libOfxSupport.lib    <- Windows static support library
```

If `ofxsImageEffect.h` "cannot be found," the usual cause is pointing the SDK
variable at the extraction root instead of the `OpenFX/` subdirectory inside it.

## CMake: include paths, find_library, and linking

```cmake
# Caller supplies -DOFX_SDK_DIR=/path/to/OpenFX
if(NOT OFX_SDK_DIR)
    message(FATAL_ERROR "Set -DOFX_SDK_DIR to the OpenFX SDK root (the dir that "
                        "contains include/openfx and lib).")
endif()

target_include_directories(${PLUGIN_TARGET} PRIVATE
    "${OFX_SDK_DIR}/include/openfx"
    "${OFX_SDK_DIR}/include/openfx/Support")

# Locate the prebuilt support library per platform:
if(APPLE)
    find_library(OFX_SUPPORT_LIB OfxSupport
                 PATHS "${OFX_SDK_DIR}/lib" NO_DEFAULT_PATH)
elseif(WIN32)
    find_library(OFX_SUPPORT_LIB libOfxSupport
                 PATHS "${OFX_SDK_DIR}/lib" NO_DEFAULT_PATH)
endif()

if(NOT OFX_SUPPORT_LIB)
    message(FATAL_ERROR "OpenFX support library not found under ${OFX_SDK_DIR}/lib")
endif()

target_link_libraries(${PLUGIN_TARGET} PRIVATE ${OFX_SUPPORT_LIB})
```

The library *name* differs by platform -- `OfxSupport` (resolving
`libOfxSupport.a`) on macOS, `libOfxSupport` (`libOfxSupport.lib`) on Windows --
so the `find_library` call must be branched. `NO_DEFAULT_PATH` keeps it from
accidentally picking up an unrelated system library.

## The .ofx.bundle structure

An OFX plugin ships as a bundle directory named `<Plugin>.ofx.bundle`. The host
loads the binary from a platform-specific subfolder. The binary file is named
with a `.ofx` extension (it is a normal shared library renamed).

### macOS

```
MyPlugin.ofx.bundle/
  Contents/
    Info.plist                 <- required for Resolve discovery
    MacOS/
      MyPlugin.ofx             <- the universal (arm64 + x86_64) binary
    Resources/                 <- optional icons / resources
```

```cmake
if(APPLE)
    set_target_properties(${PLUGIN_TARGET} PROPERTIES
        BUNDLE TRUE
        BUNDLE_EXTENSION "ofx.bundle"
        OUTPUT_NAME "MyPlugin.ofx"
        MACOSX_BUNDLE_INFO_PLIST "${CMAKE_CURRENT_SOURCE_DIR}/Info.plist.in")
endif()
```

`Contents/Info.plist` is **required** for DaVinci Resolve to discover the
plugin; without it the bundle silently fails to load (Nuke is more lenient).
Generate it as part of the build.

### Windows

```
MyPlugin.ofx.bundle/
  Contents/
    Win64/
      MyPlugin.ofx             <- the x64 binary (a renamed .dll)
    Resources/
```

```cmake
if(WIN32)
    set_target_properties(${PLUGIN_TARGET} PROPERTIES
        OUTPUT_NAME "MyPlugin"
        SUFFIX ".ofx")
    install(FILES "$<TARGET_FILE:${PLUGIN_TARGET}>"
            DESTINATION
              "$ENV{COMMONPROGRAMFILES}/OFX/Plugins/MyPlugin.ofx.bundle/Contents/Win64")
endif()
```

### Install locations

| Platform | System OFX plugin directory |
|----------|-----------------------------|
| macOS    | `/Library/OFX/Plugins/` |
| Windows  | `C:\Program Files\Common Files\OFX\Plugins\` |

## macOS Command Line Tools: empty c++/v1 headers

A frequent macOS-only build break with **Command Line Tools installed but not
the full Xcode** is the C++ standard library headers (`<chrono>`, `<vector>`,
`<memory>`, ...) failing to resolve. The CLT ships an *empty* stub at
`usr/include/c++/v1/`, so the compiler finds the directory but no headers.

Fix: add the real libc++ include directory from the active SDK
(`xcrun --show-sdk-path`) to the include path:

```cmake
if(APPLE)
    execute_process(
        COMMAND xcrun --show-sdk-path
        OUTPUT_VARIABLE _OSX_SDK_PATH
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET)
    if(EXISTS "${_OSX_SDK_PATH}/usr/include/c++/v1")
        include_directories(SYSTEM "${_OSX_SDK_PATH}/usr/include/c++/v1")
    endif()
endif()
```

Installing the full Xcode also resolves it, but the SDK-path include keeps CI
runners with CLT-only images building.

## Universal binaries (macOS)

A single CMake build can already produce a fat (arm64 + x86_64) binary when
`CMAKE_OSX_ARCHITECTURES` lists both -- do **not** build the two architectures
separately and `lipo` them together, which fails with a "same architectures"
error because each output is already universal. For the AE-side details of
single-build universal output (and the PiPL caveat for AE plugins) see
[Universal Binary](universal-binary.md). The OFX bundle has no PiPL, so once the
binary is fat you only need to sign the bundle.

## Checklist

- [ ] `OFX_SDK_DIR` points at the inner `OpenFX/` directory (contains
      `include/openfx` and `lib`)
- [ ] Include both `include/openfx` and `include/openfx/Support`
- [ ] `find_library` branched: `OfxSupport` (mac) vs `libOfxSupport` (win)
- [ ] Output binary named `*.ofx`, placed in `Contents/MacOS` (mac) or
      `Contents/Win64` (win)
- [ ] `Contents/Info.plist` generated for macOS bundles (Resolve requires it)
- [ ] macOS: add SDK `c++/v1` include dir if building with Command Line Tools
- [ ] macOS: one fat build, not separate-arch + lipo

## See also

- [The OpenFX C++ Plugin Model](ofx-plugin-structure.md)
- [OFX Resolve Pitfalls](ofx-resolve-pitfalls.md)
- [Universal Binary](universal-binary.md)
- [Code Signing](code-signing.md) and [Notarization](notarization.md) for
  distributing the signed bundle.

*Tags: `ofx`, `openfx`, `build`, `cmake`, `bundle`, `macos`, `windows`, `universal-binary`*
