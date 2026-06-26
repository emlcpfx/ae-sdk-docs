# Backwards Compatible

> 1 Q&A · source: AE plugin dev community Discord

### How can you set a layer to be a trackmatte using AEGPSetLayerTransferMode with backwards compatibility before AE 23?

James Whiffin reported that AEGPSetLayerTransferMode only works to set trackmatte mode if the layer is already a trackmatte, but won't convert a non-trackmatte layer to one. He notes that AE 23+ has a new method for this purpose, but for backwards compatibility with earlier versions, the standard AEGP_SetLayerTransferMode approach has this limitation. The code structure involves creating an AEGP_LayerTransferMode struct with the desired transfer mode and track matte type, then calling the LayerSuite7 function.

```cpp
AEGP_LayerTransferMode mode = {};
mode.mode = PF_Xfer_IN_FRONT;
mode.track_matte = AEGP_TrackMatte_ALPHA;
mode.flags = 0;
ERR(suites.LayerSuite7()->AEGP_SetLayerTransferMode(layerH, &mode));
```

*Tags: `aegp`, `backwards-compatible`, `layer-checkout`*

---
