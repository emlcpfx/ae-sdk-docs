# Sequential Processing

> 1 Q&A · source: AE plugin dev community Discord

### How can I set up an AE plugin to asynchronously receive video frames from an external pipe or ML model?

AE's rendering pipeline is built around random frame access, not sequential processing. Set up a separate C++ thread (nothing to do with AE SDK) for the async pipe, and use a mutex to share data between the pipe thread and AE's render calls. Use AEGP_CauseIdleRoutinesToBeCalled() to trigger immediate communication. For sequential ML models requiring multiple frames: (1) Have an AEGP monitor the project and pre-fetch frames using AEGP_CheckoutOrRender_ItemFrame_AsyncManager, caching results to disk. (2) If the user jumps past cached results, stall the render until caching catches up. Consider the tradeoff between fast sequential pre-processing (like AE's camera tracker) vs slower random-access processing.

*Tags: `async`, `frame-caching`, `ml-model`, `pipe`, `sequential-processing`, `threading`*

---
