# Doclick

> 1 Q&A · source: AE plugin dev community Discord

### Why can't I access parameter input layer pixel data in the DoClick function?

The incoming image data is only available during the render call, not in DoClick. However, there are several workarounds: (1) Store input/output pixels in sequence data during render for later access, though cached frames won't update; (2) Use AEGP_GetReceiptWorld() to fetch pixels of any video item at any time, though rendering composites can be slow; (3) In CS4 and earlier, use the UI drawing mechanism during the draw event; (4) Best practice: store click location and flags in sequence data during DoClick, then trigger a re-render to process pixels during the render call.

*Tags: `aegp`, `doclick`, `params`, `pixel`, `render-loop`, `sequence-data`*

---
