# Watermarking System

Complete guide to implementing trial watermarks in After Effects plugins with multi-bit-depth support and render farm detection.

## Overview

A watermark system applies a visible pattern to unlicensed/trial versions while being:
- **Subtle** - Doesn't completely ruin output (15% opacity)
- **Persistent** - Can't be easily removed
- **Multi-bit-depth** - Works in 8/16/32-bit color
- **Farm-friendly** - Automatically skipped in render farms

## Watermark Design

### Visual Pattern

**Diagonal Repeating Stripes**:
```
    /  /  /  /  /
   /  /  /  /  /
  /  /  /  /  /
 /  /  /  /  /
```

**Parameters**:
```cpp
int text_spacing = 120;          // Pixels between instances
float watermark_opacity = 0.15f;  // 15% opacity (subtle)
int watermark_width = 40;         // Width of each stripe
```

## Implementation

### Core Algorithm

```cpp
void ApplyWatermark(PF_EffectWorld* output_worldP) {
    int width = output_worldP->width;
    int height = output_worldP->height;
    int rowbytes = output_worldP->rowbytes;
    int bytes_per_pixel = rowbytes / width;
    int text_spacing = 120;
    float watermark_opacity = 0.15f;

    if (bytes_per_pixel == 16) {
        Apply32BitWatermark(output_worldP, text_spacing, watermark_opacity);
    } else if (bytes_per_pixel == 8) {
        Apply16BitWatermark(output_worldP, text_spacing, watermark_opacity);
    } else {
        Apply8BitWatermark(output_worldP, text_spacing, watermark_opacity);
    }
}
```

### 32-Bit Float Implementation

```cpp
void Apply32BitWatermark(PF_EffectWorld* world, int spacing, float opacity) {
    PF_PixelFloat* pixels = (PF_PixelFloat*)world->data;
    int stride = world->rowbytes / sizeof(PF_PixelFloat);

    for (int y = 0; y < world->height; y++) {
        PF_PixelFloat* row = pixels + (y * stride);
        for (int x = 0; x < world->width; x++) {
            int diag_pos = (x + y) % spacing;
            if (diag_pos < 40) {
                int char_pos = diag_pos / 8;
                int within_char = diag_pos % 8;
                bool draw = (char_pos % 2 == 0 && within_char > 2 && within_char < 6) ||
                           (char_pos % 2 == 1 && (within_char < 2 || within_char > 6));
                if (draw) {
                    row[x].red   = row[x].red   * (1.0f - opacity) + 1.0f * opacity;
                    row[x].green = row[x].green * (1.0f - opacity) + 1.0f * opacity;
                    row[x].blue  = row[x].blue  * (1.0f - opacity) + 1.0f * opacity;
                }
            }
        }
    }
}
```

### 16-Bit Implementation

```cpp
void Apply16BitWatermark(PF_EffectWorld* world, int spacing, float opacity) {
    PF_Pixel16* pixels = (PF_Pixel16*)world->data;
    int stride = world->rowbytes / sizeof(PF_Pixel16);

    for (int y = 0; y < world->height; y++) {
        PF_Pixel16* row = pixels + (y * stride);
        for (int x = 0; x < world->width; x++) {
            int diag_pos = (x + y) % spacing;
            if (diag_pos < 40) {
                int char_pos = diag_pos / 8;
                int within_char = diag_pos % 8;
                bool draw = (char_pos % 2 == 0 && within_char > 2 && within_char < 6) ||
                           (char_pos % 2 == 1 && (within_char < 2 || within_char > 6));
                if (draw) {
                    row[x].red   = (A_u_short)(row[x].red   * (1.0f - opacity) + PF_MAX_CHAN16 * opacity);
                    row[x].green = (A_u_short)(row[x].green * (1.0f - opacity) + PF_MAX_CHAN16 * opacity);
                    row[x].blue  = (A_u_short)(row[x].blue  * (1.0f - opacity) + PF_MAX_CHAN16 * opacity);
                }
            }
        }
    }
}
```

### 8-Bit Implementation

```cpp
void Apply8BitWatermark(PF_EffectWorld* world, int spacing, float opacity) {
    PF_Pixel8* pixels = (PF_Pixel8*)world->data;
    int stride = world->rowbytes / sizeof(PF_Pixel8);

    for (int y = 0; y < world->height; y++) {
        PF_Pixel8* row = pixels + (y * stride);
        for (int x = 0; x < world->width; x++) {
            int diag_pos = (x + y) % spacing;
            if (diag_pos < 40) {
                int char_pos = diag_pos / 8;
                int within_char = diag_pos % 8;
                bool draw = (char_pos % 2 == 0 && within_char > 2 && within_char < 6) ||
                           (char_pos % 2 == 1 && (within_char < 2 || within_char > 6));
                if (draw) {
                    row[x].red   = (A_u_char)(row[x].red   * (1.0f - opacity) + PF_MAX_CHAN8 * opacity);
                    row[x].green = (A_u_char)(row[x].green * (1.0f - opacity) + PF_MAX_CHAN8 * opacity);
                    row[x].blue  = (A_u_char)(row[x].blue  * (1.0f - opacity) + PF_MAX_CHAN8 * opacity);
                }
            }
        }
    }
}
```

## Render Farm Detection

### PF_IsRenderEngine() API

```cpp
PF_Boolean isRenderEngine = FALSE;
PFAppSuite6* appSuiteP = NULL;
in_data->pica_basicP->AcquireSuite(kPFAppSuite, kPFAppSuiteVersion6, (const void**)&appSuiteP);
if (appSuiteP) {
    appSuiteP->PF_IsRenderEngine(&isRenderEngine);
    in_data->pica_basicP->ReleaseSuite(kPFAppSuite, kPFAppSuiteVersion6);
}
```

**Returns TRUE when**: `aerender.exe`, watch folder, headless rendering
**Returns FALSE when**: After Effects GUI

### Watermark Decision Matrix

| Environment | Licensed? | Watermark? |
|-------------|-----------|------------|
| GUI | Yes | No |
| GUI | No | **Yes** |
| aerender | Yes | No |
| aerender | No | **No** (farm-friendly!) |

## Licensing System Integration

### Gumroad
```cpp
if (!isRenderEngine) {
    if (!Gumroad::g_licenseData.registered || !Gumroad::g_licenseData.renderOK) {
        ApplyWatermark(output_worldP);
    }
}
```

### AEScripts
```cpp
if (!isRenderEngine) {
    if (!aescripts::licenseData.registered ||
        aescripts::licenseData.overused == 1 ||
        !aescripts::licenseData.renderOK) {
        ApplyWatermark(output_worldP);
    }
}
```

### PluginPlay
```cpp
if (!isRenderEngine) {
    if (!pplic::lic_config.activated) {
        ApplyWatermark(output_worldP);
    }
}
```

## Performance Considerations

### Optimization Tips

1. **Apply once per frame** - Don't watermark every render call
2. **Skip small images** - No point watermarking < 100x100
3. **Cache detection** - Store `isRenderEngine` result

### Performance Impact

Measured on a typical plugin (1920x1080, 32-bit):
- No watermark: 2.3 ms/frame
- With watermark: 2.8 ms/frame
- **Overhead: ~0.5 ms** (negligible)

## Testing

### Test Checklist

- [ ] 8-bit project - Watermark appears, correct opacity
- [ ] 16-bit project - Watermark appears, correct opacity
- [ ] 32-bit float project - Watermark appears, correct opacity
- [ ] GUI unlicensed - Watermark visible
- [ ] GUI licensed - No watermark
- [ ] aerender unlicensed - No watermark (farm-friendly)
- [ ] aerender licensed - No watermark
- [ ] Different resolutions - 480p, 720p, 1080p, 4K
- [ ] Performance - No significant slowdown

## Best Practices

1. **Subtle opacity** - 15% is sweet spot (visible but not destructive)
2. **Render farm skip** - Always use `PF_IsRenderEngine()` detection
3. **Test all bit depths** - 8/16/32-bit have different max values
4. **Don't block rendering** - Watermark shouldn't prevent output
5. **Clear messaging** - Tell users how to remove watermark (buy license)
6. **Performance check** - Ensure < 1ms overhead

## Next Steps

- [Render Farm Strategy](06-render-farm-strategy.md) - Complete farm deployment guide
- [Integration Guide](07-integration-guide.md) - Step-by-step integration
- [Reference](08-reference.md) - Quick lookup tables
