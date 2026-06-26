# Async Render

> 1 Q&A · source: AE plugin dev community Discord

### Is AEGP_RenderAndCheckoutLayerFrame_Async safe to use for rendering multiple frames?

There is a known bug with the async version of RenderAndCheckout related to memory releasing. The issue was discussed between several developers and the AE team. All developers reverted back to the synchronous version, doing their best to render one frame at a time on each idle cycle to avoid making the UI laggy. The frame receipt cannot be properly checked in when using the async version.

*Tags: `async-render`, `bug`, `frame-checkout`, `memory-leak`, `render-suite`*

---
