# Suites & Callbacks

SDK suites are collections of function pointers that provide access to AE functionality.

## How Suites Work

```cpp
// Acquire a suite
AEFX_SuiteScoper<PF_WorldSuite2> wsP = AEFX_SuiteScoper<PF_WorldSuite2>(
    in_data, kPFWorldSuite, kPFWorldSuiteVersion2, out_data);

// Use it
PF_PixelFormat format;
wsP->PF_GetPixelFormat(input_worldP, &format);
```

## Essential Suites

| Suite | What It Does |
|-------|-------------|
| `PF_WorldSuite2` | Pixel format detection, world creation |
| `PF_WorldTransformSuite1` | Compositing (`transfer_rect`), transforms |
| `PF_Iterate8/16/FloatSuite` | Per-pixel iteration |
| `PF_HandleSuite1` | Memory handle management |
| `PF_GPUDeviceSuite1` | GPU buffer management |
| `PF_EffectCustomUISuite` | Custom UI drawing |

## Suite Versioning

Suites are versioned (`kPFWorldSuiteVersion2`). Always check the SDK headers for the latest version. Requesting a version that doesn't exist in the running AE version will fail gracefully.

---

*21 documents in this section. Use **Search** (top of page) to find what you need.*
