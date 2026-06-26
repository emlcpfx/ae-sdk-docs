# Render Farm Strategy

Complete guide to deploying After Effects plugins on render farms with two proven approaches: runtime detection and dedicated render-only builds.

## The Render Farm Challenge

**Problem**: Traditional plugin licensing blocks render farms:
- Each render node needs a license (expensive)
- Network license checks slow rendering
- Licensing overhead on every frame

**Solution**: Make plugins "farm-friendly" while protecting creative workstations.

## Approach 1: PF_IsRenderEngine() Detection (Recommended)

### Overview

**Single plugin build** that behaves differently based on runtime environment:
- **GUI mode** (`AfterFX.exe`): Checks license, shows watermark if unlicensed
- **Render mode** (`aerender.exe`): Skips watermark, renders cleanly

### Implementation

```cpp
#ifdef ENABLE_GUMROAD_LICENSING  // Or any licensing system

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

### Behavior Matrix

| Environment | Has License? | Watermark? | Renders? |
|-------------|--------------|------------|----------|
| Workstation GUI | Yes | No | Yes |
| Workstation GUI | No | **Yes** | Yes (with watermark) |
| Render Farm | Yes | No | Yes |
| Render Farm | No | **No** | **Yes (clean!)** |

### Deployment

**Same file everywhere!** No special farm builds.

```batch
copy "build\Release_Gumroad\MyPlugin.aex" ^
     "C:\Program Files\Adobe\Common\Plug-ins\7.0\MediaCore\MyPlugin.aex"
```

## Approach 2: BUILD_RENDER_ONLY (Alternative)

### Overview

**Separate build** specifically for render farms:
- No GUI parameters (all hidden)
- No licensing code (compiled out)
- Renders using project file settings
- Different filename to avoid confusion

### CMake Configuration

```cmake
option(BUILD_RENDER_ONLY "Build render-only version" OFF)

if(BUILD_RENDER_ONLY)
    target_compile_definitions(YourPlugin PRIVATE BUILD_RENDER_ONLY=1)
    set_target_properties(YourPlugin PROPERTIES
        OUTPUT_NAME "YourPlugin_Render"
    )
endif()
```

### Code Changes

```cpp
static PF_Err ParamsSetup(...) {
    #ifdef BUILD_RENDER_ONLY
    PF_ADD_TOPIC("RENDER ONLY VERSION", RENDER_ONLY_TOPIC);
    #else
    // Normal parameters
    PF_ADD_FLOAT_SLIDERX("Edge Softness", ...);
    #endif
    return PF_Err_NONE;
}
```

### Deployment

**CRITICAL**: Cannot have both on same machine - AE will reject duplicate Match Names.

- **Workstations**: Full version only
- **Farm nodes**: Render-only version only

## Comparison: Which Approach?

### Use PF_IsRenderEngine() If:
- You want simplest deployment (one file)
- You prefer single codebase maintenance
- You want fastest development

### Use BUILD_RENDER_ONLY If:
- You want absolute protection (can't be abused)
- You prefer explicit farm vs. workstation distinction
- You want smallest possible farm binary

## Render Farm Software Integration

### Deadline (Thinkbox)
1. Install plugin on all render nodes
2. Submit After Effects job
3. Deadline calls `aerender.exe` on each node
4. Plugin detects `isRenderEngine = TRUE` -> no watermark

### Render Garden
1. Install plugin on farm machines
2. Upload After Effects project
3. Render Garden uses `aerender.exe`
4. Plugin automatically farm-friendly

### BG Renderer Max
1. Install plugin on workstation
2. BG Renderer Max renders in background
3. Uses same detection as `aerender.exe`
4. No watermark in background renders

## Testing

### For PF_IsRenderEngine() Approach

```batch
# Delete license
reg delete "HKEY_CURRENT_USER\Software\MyCompany\MyPlugin" /f

# Test GUI - watermark should appear
# Open AE, apply plugin

# Test render farm - NO watermark
"C:\Program Files\Adobe\Adobe After Effects 2024\Support Files\aerender.exe" ^
    -project "test.aep" ^
    -comp "Comp 1" ^
    -output "output.mov"
```

## Troubleshooting

### Issue: Watermark appears on render farm
**Cause**: PF_IsRenderEngine() not implemented or failing
**Solution**: Verify suite acquisition succeeds

### Issue: Cannot install both versions
**Cause**: Duplicate Match Names (expected behavior)
**Solution**: Choose ONE per machine

### Issue: Render-only version doesn't render
**Solution**: Verify `AE_Effect_Match_Name` is identical in both builds

## Best Practices

1. **Test both modes** - GUI and aerender
2. **Same Match Name** - Critical for render-only builds
3. **Clear documentation** - Tell users which version for which environment
4. **Farm testing** - Always test on actual farm before release
5. **No network checks on farms** - Keep farms fast and simple
6. **Provide both options** - Some users prefer render-only builds

## Next Steps

- [Integration Guide](07-integration-guide.md) - Step-by-step implementation
- [Reference](08-reference.md) - Quick lookup tables and troubleshooting
