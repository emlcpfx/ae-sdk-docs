# Async

> 4 Q&As · source: AE plugin dev community Discord

### How can I set up an AE plugin to asynchronously receive video frames from an external pipe or ML model?

AE's rendering pipeline is built around random frame access, not sequential processing. Set up a separate C++ thread (nothing to do with AE SDK) for the async pipe, and use a mutex to share data between the pipe thread and AE's render calls. Use AEGP_CauseIdleRoutinesToBeCalled() to trigger immediate communication. For sequential ML models requiring multiple frames: (1) Have an AEGP monitor the project and pre-fetch frames using AEGP_CheckoutOrRender_ItemFrame_AsyncManager, caching results to disk. (2) If the user jumps past cached results, stall the render until caching catches up. Consider the tradeoff between fast sequential pre-processing (like AE's camera tracker) vs slower random-access processing.

*Tags: `async`, `frame-caching`, `ml-model`, `pipe`, `sequential-processing`, `threading`*

---

### What are the memory safety considerations when using AEGP_RenderAndCheckoutLayerFrame_Async?

According to community discussion with Adobe's AE team, there is a known memory release problem when using the async version of renderAndCheckout. As a result, developers have reverted to using the synchronous version instead, rendering one frame at a time on each idle cycle to avoid UI lag while properly managing memory.

*Tags: `aegp`, `async`, `memory`, `render-loop`, `threading`*

---

### How can you safely implement async layer frame rendering in an AEGP plugin?

Create an AsyncManager class that wraps AEGP_RenderAndCheckoutLayerFrame_Async with proper callback handling. The async call is thread-safe as long as only one thread calls it at a time. Use a static callback function that receives the request via refcon, retrieves the world from the frame receipt, and invokes a user-provided callback function. Store the callback as a heap-allocated std::function pointer passed through the refcon parameter, then delete it after use to prevent memory leaks.

```cpp
class AsyncRenderManager {
public:
  void renderAsync(LayerRenderOptionsPtr optionsH, std::function<void(WorldPtr)> callbackF) {
    auto callbackPtr = new std::function<void(WorldPtr)>(callbackF);
    auto ret = std::async(std::launch::async, [optionsH, callbackPtr]() {
      RenderSuite().renderAndCheckoutLayerFrameAsync(optionsH, callback,
        reinterpret_cast<AEGP_AsyncFrameRequestRefcon>(callbackPtr));
    });
  }
  
  static A_Err callback(AEGP_AsyncRequestId request_id, A_Boolean was_canceled, A_Err error,
    AEGP_FrameReceiptH receiptH, AEGP_AsyncFrameRequestRefcon refconP0) {
    auto callbackPtr = reinterpret_cast<std::function<void(WorldPtr)> *>(refconP0);
    if (callbackPtr && *callbackPtr) {
      FrameReceiptPtr ptr = std::make_shared<AEGP_FrameReceiptH>(receiptH);
      auto world = RenderSuite().getReceiptWorld(ptr);
      (*callbackPtr)(world);
      delete callbackPtr;
    }
    return error;
  }
};
```

*Tags: `aegp`, `async`, `memory`, `render-loop`, `threading`*

---

### How can I setup an After Effects plugin to asynchronously receive video frames from an external pipe?

You can setup a separate C++ thread (independent of the AE SDK) to handle reading/writing data to a pre-allocated memory buffer. Use a mutex to safely access this memory from both the async pipe thread and AE's render calls. To trigger the async pipe thread to process data promptly, use AEGP_CauseIdleRoutinesToBeCalled(). However, reconciling AE's random frame access model with sequential async pipe operations is challenging—you may need to either stall the render until required frames are available or cache frames to a file and fetch them on demand. For fetching source frames without blocking the UI, use AEGP_CheckoutOrRender_ItemFrame_AsyncManager or AEGP_CheckoutOrRender_LayerFrame_AsyncManager.

*Tags: `aegp`, `async`, `caching`, `memory`, `threading`*

---
