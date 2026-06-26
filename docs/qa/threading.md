# Q&A: threading

**111 entries** tagged with `threading`.

---

## How do you read sequence data in SmartRender since AE CC2021/2022?

Since CC2021/2022, in_data->seq_data is not correctly updated in render threads. You must use PF_EffectSequenceDataSuite1 with PF_GetConstSequenceData instead. The const handle must be locked with PF_LOCK_HANDLE before use. Note: According to Adobe engineers, PF_LOCK_HANDLE hasn't done anything since CS6 (it's a dummy function), but the code pattern of casting via the handle is still necessary for correct access.

```cpp
PF_ConstHandle const_seq;
AEFX_SuiteScoper<PF_EffectSequenceDataSuite1> seqdata_suite =
    AEFX_SuiteScoper<PF_EffectSequenceDataSuite1>(
        in_data,
        kPFEffectSequenceDataSuite,
        kPFEffectSequenceDataSuiteVersion1,
        out_data);
seqdata_suite->PF_GetConstSequenceData(in_data->effect_ref, &const_seq);
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Contributors: [**tlafo**](../contributors/tlafo/), [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2025-04-02 · Tags: `sequence-data`, `smart-render`, `mfr`, `pf-get-const-sequence-data`, `threading`*

---

## What is the AEGP memory leak issue on Mac during export with Compute Cache?

When using AEGP_MemorySuite to create memHandles during ComputeCache threads, the memory may not be properly freed during export on Mac (works fine on Windows). The used memory grows past the RAM limit. Using new/delete instead of AEGP memHandles avoids the leak. The issue is that memHandles allocated in one thread and freed in another may not actually release memory during the render thread. The AEGP tools report the memory as freed, but virtual memory keeps growing.

*Contributors: [**tlafo**](../contributors/tlafo/), [**dvb metareal**](../contributors/dvb-metareal/) · Source: adobe-plugin-devs · 2023-09-07 · Tags: `memory-leak`, `compute-cache`, `mac`, `export`, `aegp-memory-suite`, `threading`*

---

## Why is sequence data different between PF_Cmd_EVENT (UI thread) and SmartPreRender (render thread)?

Sequence data is separate on the render and UI threads by design. The render threads are free to render asynchronously without having their data change from another thread. Global data IS shared between render and UI threads, though it's not instance-specific. To share per-instance data: store data in global data keyed by instance identifiers (comp item ID + layer ID + effect index on layer), protected by a mutex. Comp item IDs don't change during a session. Layer IDs don't change on reorder but may change on copy/paste. Effect index changes when moved in the stack.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `sequence-data`, `threading`, `ui-thread`, `render-thread`, `global-data`, `mutex`*

---

## How can I set up an AE plugin to asynchronously receive video frames from an external pipe or ML model?

AE's rendering pipeline is built around random frame access, not sequential processing. Set up a separate C++ thread (nothing to do with AE SDK) for the async pipe, and use a mutex to share data between the pipe thread and AE's render calls. Use AEGP_CauseIdleRoutinesToBeCalled() to trigger immediate communication. For sequential ML models requiring multiple frames: (1) Have an AEGP monitor the project and pre-fetch frames using AEGP_CheckoutOrRender_ItemFrame_AsyncManager, caching results to disk. (2) If the user jumps past cached results, stall the render until caching catches up. Consider the tradeoff between fast sequential pre-processing (like AE's camera tracker) vs slower random-access processing.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `async`, `pipe`, `ml-model`, `threading`, `frame-caching`, `sequential-processing`*

---

## Why is PF_Cmd_SEQUENCE_FLATTEN called when an effect is first applied, and why are Sequence Resetup/Setdown called multiple times on different threads?

This is correct behavior when PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set. AE flattens sequence data so it can be copied (for render thread copies). Since AE 2015, rendering uses separate threads with separate project copies. Resetup may receive unflat data, so tag your data to indicate flat/unflat state. Use PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA to allow AE to get the flat copy without losing the original unflat one, reducing unnecessary resetup calls. During sequence setup/resetup, don't assume the plugin can find its own layer/comp - AE sometimes constructs the instance before associating it to a layer. For unique instance IDs, flag sequence data as 'requires checking' on resetup, then scan for duplicates when the instance can be located.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `sequence-data`, `flatten`, `resetup`, `threading`, `mfr`, `instance-id`*

---

## How can I force a re-render from a separate thread (e.g., when receiving WebSocket messages)?

It's only possible via idle_hook, which runs on the UI thread (30-50 times per second). Check your other thread for messages during idle events. Options to force re-render: (1) Change a hidden/invisible parameter value - this forces AE to re-render. (2) Call the command number to purge the cache. (3) Call AEGP_EffectCallGeneric to have the effect instance act on itself. You cannot set out_data->out_flags from a separate thread - it must happen on the main UI thread during an AE callback.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2022-01-01 · Tags: `force-rerender`, `idle-hook`, `websocket`, `threading`, `ui-thread`, `hidden-param`*

---

## Why is the plugin experiencing glitch behavior with MFR enabled when the Multithreaded flag is not set?

Different threads/CPUs are using different previous frames when MFR (Multi-Frame Rendering) is enabled, causing the glitch behavior. The issue resolves when MFR is turned off. The root cause appears to be that MFR is being invoked even though the Multithreaded flag is not explicitly set for the plugin.

*Tags: `mfr`, `threading`, `debugging`*

---

## Should error handling be done when checking out layers, and should check-in occur immediately after use?

Yes, error handling should be implemented for checkout operations. Check-in should be done immediately after getting the value. In a loop structure, you should checkout a frame, check for errors, perform the job if no error occurred, and then check-in the frame before moving to the next iteration. Each checked-out resource must be checked in, whether it was used or not.

```cpp
For X
     Checkout frame x
         If (no err) dojob
     Err2(check-in frame x)
End of loop
```

*Tags: `layer-checkout`, `aegp`, `error-handling`, `render-loop`, `threading`*

---

## Is it normal to receive thousands of calls to PF_Arbitrary_DISPOSE_FUNC on After Effects shutdown?

Yes, this is normal behavior. The high number of disposal calls on shutdown is likely due to MFR (Multi-Frame Rendering) duplicating arbitrary parameters for each thread, causing many copy and dispose cycles as parameters change and threads are cleaned up during shutdown.

*Tags: `mfr`, `arb-data`, `threading`, `memory`*

---

## How can I properly detect if a layer can be checked out in the userChangedParam callback without getting infinitely locked during effect duplication or closing?

Instead of attempting layer checkout in userChangedParam during shutdown or duplication, check the validity of layer data by examining width/height and Lrect values. Invalid layers will have random, oversized (>100000), or negative values. However, a more reliable approach is to avoid layer checkout during these callback scenarios, as the layer data may be unavailable or invalid during effect duplication and shutdown sequences.

*Tags: `layer-checkout`, `params`, `threading`*

---

## Can we call PF_PROGRESS and PF_ABORT from another thread started in the render function?

It should be possible, but you may need atomic operations or mutexes to work on all threads at once. However, there are challenges: the progress bar is only rendered at the bottom of the UI, and multiple threads may report progress out of order since they complete asynchronously. A better approach is to bake the computation in the backend (like in userchangedParam), update an arb param during the update, store the result in sequence data, and apply it (similar to warp stabilizer).

*Tags: `threading`, `render-loop`, `arb-data`, `ui`*

---

## How can we report progress while a blocking rendering computation is happening in the main thread?

The standard PF_PROGRESS approach won't work if the main thread is blocked. Instead, consider baking the computation in the backend (like in userchangedParam), updating an arb param during processing, storing the result in sequence data, and applying it in the render function (similar to how warp stabilizer works). This avoids blocking the main thread and allows proper progress reporting.

*Tags: `render-loop`, `params`, `arb-data`, `threading`*

---

## How should you handle thread safety when using global variables in a plugin?

Use mutexes to protect access to shared global variables. A mutex locks all threads from accessing the protected variable simultaneously - only one thread can write while holding the mutex. Atomic bools can also work but may cause errors in some cases. Initialize global variables only once since After Effects runs on two threads (UI and render) since CC 2014, and you need consistent values across threads.

```cpp
static int stuff;
// In code modifying the variable:
mtx.lock();
stuff = new_value;
mtx.unlock();
```

*Tags: `threading`, `memory`*

---

## How can you trigger frame invalidation from a background worker thread without direct AEGP access?

Use AEGP_ExecuteScript from IdleHook to run an After Effects script that modifies a composition property (like background color) or a hidden parameter value. This forces After Effects to re-evaluate which frames need invalidation. Be aware that this fails if modal dialogs are open.

*Tags: `aegp`, `scripting`, `threading`, `idle-hook`*

---

## Why is there a memory leak when using AEGP_memorysuite on Mac but not Windows during export?

The memory leak appears to be platform-specific (Mac only) and related to how AEGP_memorysuite handles memory operations. The issue is less severe with the mem_QUIET flag. Using standard C++ memory allocation (new/delete) instead of AEGP_memorysuite resolves the leak, and replacing memcpy with strncpy partially solves the problem (from mem pointer to buffer works, but not in the reverse direction). This suggests the issue may be related to how memcpy interacts with AEGP's memory management on macOS, particularly when memory is freed from a different thread than where it was allocated.

*Tags: `memory`, `macos`, `aegp`, `threading`, `debugging`*

---

## How should memory be freed differently on Mac versus Windows?

Memory should be freed later in another thread on Mac but not on Windows, due to platform-specific threading and memory management differences.

*Tags: `memory`, `macos`, `windows`, `threading`*

---

## Is there a limitation with AEGP_memsuite when allocating and freeing memory during the render thread?

The user observed that memory allocated using aegp_memHandle during the compute cache thread is not being freed when the freemem function is called, even though instruments show no leaks. This suggests there may be specific limitations or requirements for using AEGP_memsuite during the render thread that differ from other threads.

*Tags: `memory`, `aegp`, `render-loop`, `compute-cache`, `threading`, `debugging`*

---

## Why does a plugin act weird on Media Encoder and render without the effect applied when queued from After Effects?

The issue is likely related to static global variables in the plugin or elements defined in globalData or during global initialization thread. These can cause problems across different Media Encoder versions.

*Tags: `premiere`, `memory`, `threading`, `debugging`*

---

## What are the advantages of compute cache compared to arbdata for storing pointers to heap objects that are flattened/unflattened?

Compute cache can be written and read from any thread (render or UI thread) using std::atomic for thread safety. Unlike arbdata, compute cache is not stored with the project. Compute cache is designed to get a value per frame, whereas arbdata requires a global context where you define arrays or vectors to store data for different frames. Compute cache replaced sequence data after MFR was introduced since sequence data can no longer be used.

*Tags: `compute-cache`, `arb-data`, `sequence-data`, `threading`, `memory`, `mfr`*

---

## What structure should be used to pass in_data and out_data to compute cache functions?

Define a struct access_cache_data that contains PF_InData* and PF_OutData* pointers to pass the required suite information to the cache. Create a new object and copy only the needed parts, then delete it after computing to avoid crashes and maintain independence from the render thread.

```cpp
struct access_cache_data {
    PF_InData* in_data;
    PF_OutData* out_data;
}
```

*Tags: `compute-cache`, `aegp`, `threading`, `memory`*

---

## Are there certain suites that shouldn't be called from Death Hook?

Yes, there are restrictions. The Utility Suite should not be called from Death Hook as it can cause unhandled exceptions when trying to use functions like ReportInfo.

*Tags: `aegp`, `debugging`, `threading`*

---

## Is it possible for one effect to render from another effect during smartrender?

Direct effect-to-effect rendering during smartrender is not possible because After Effects will not call smartrender on other layers while a current smartrender is in progress—it waits until the current smartrender is done. Workarounds include: (1) using AEGP outside of render calls but this has poor UI; (2) calling EffectCallGeneric on another plugin if that plugin implements a passthrough code for SmartRender; (3) using AEGP_RenderAndCheckoutLayerFrame_Async which works on any thread but can be buggy; (4) delegating AEGP calls to the UI thread for the next call, though smartrender happens on the render thread and cannot directly call UI thread methods.

*Tags: `smartrender`, `render-loop`, `aegp`, `threading`*

---

## What is AEGP_RenderAndCheckoutLayerFrame_Async and can it be called from smart_render command?

AEGP_RenderAndCheckoutLayerFrame_Async is an asynchronous layer rendering and checkout function that can work on any thread, making it potentially usable from smartrender command, though it is known to be buggy.

*Tags: `smartrender`, `aegp`, `layer-checkout`, `threading`*

---

## How can a plugin maintain a unique ID per instance when duplicating, and why does sequence data become out of sync between the UI thread and render threads?

Sequence data is the appropriate mechanism for storing per-instance unique IDs. When duplicating, catch the duplication event and assign a new unique ID to the sequence data. However, sequence data may become out of sync on render threads even though it updates correctly in UpdateUI and UserChangeParams. Two potential solutions: (1) Check if the sequence is flattened in the project, as this can force render threads to use the correct sequence value, and (2) Use PF_GetConstSequenceData from the PF_EffectSequenceDataSuite1 suite instead of directly accessing in_data->seq_data, as the latter has not been updated correctly since CC 2021/2022.

*Tags: `sequence-data`, `render-loop`, `threading`, `params`*

---

## What does 'connected instance to instance' mean in the context of multiple plugin instances?

It refers to having multiple instances of the plugin on the same or different layers communicating with each other.

*Tags: `aegp`, `params`, `threading`*

---

## Is using f32 (32-bit float) data type safe and well-supported in After Effects plugin development?

Yes, f32 support is safe and essential for After Effects plugins, particularly those integrating 3D renderers. This has been validated through practical experience with production plugins like AtomKraft for AE and other Rust-based AE integrations that have run in production for extended periods.

*Tags: `build`, `deployment`, `gpu`, `threading`*

---

## Do you need to lock the handle when calling GetConstSequenceData for sequence data?

Yes, you need to lock the handle even when reading only. Use PF_LOCK_HANDLE(const_seq) to lock the handle before accessing it, and unlock it after use. This is necessary in multi-frame render (MFR) scenarios with multiple plugin applications, and the lack of locking can cause nullptr returns and crashes. While Adobe documentation doesn't clearly document this requirement, it appears to be essential for thread safety.

```cpp
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
```

*Tags: `mfr`, `sequence-data`, `threading`, `memory`, `aegp`, `debugging`*

---

## What causes the generic exception handler in Adobe suites manager to appear, and why is it more frequent in newer AE versions?

The generic exception handler/catch-all in the Adobe suites manager appears when double freeing or releasing a suite pointer. It has existed since CS2 or CS3 days. It appears more often in newer AE versions likely due to MFR (Multi-Frame Rendering) with global suite handles or suite pointers being shared between threads.

*Tags: `mfr`, `threading`, `memory`, `aegp`*

---

## How can a C++ plugin update a text layer every frame after rendering completes?

This is not cleanly possible due to AE's project locking during render operations. Possible workarounds include: (1) writing data to a compute cache or sequence data and reading it from the UI thread via the aegp_idle hook, (2) using AEGP_SetText from the UI thread instead of the render thread, (3) rendering text yourself in the render thread using an external library, or (4) saving to disk and having a ScriptUI panel with a timer read the data. The aegp_idle hook approach may not work without a GUI or in MediaEncoder. Setting text from the render thread causes a locked project error.

*Tags: `aegp`, `render-loop`, `threading`, `ui`, `sequence-data`, `compute-cache`*

---

## Would the aegp_idle hook approach work when rendering through MediaEncoder or without a GUI?

No, the aegp_idle hook approach would not work in MediaEncoder or without a GUI, since the idle hook requires an active UI thread to execute.

*Tags: `aegp`, `threading`, `ui`, `deployment`*

---

## What is the best way to pass data from an AEGP plugin to an effects plugin?

Use an AEGP plugin with an idle hook to process a queue. Every frame, push your string result into a queue, and the idle hook processes the queue occasionally. The AEGP plugin and FX plugin are separate binaries, so you'll need a C interface to push strings into the queue.

*Tags: `aegp`, `threading`, `render-loop`, `memory`*

---

## What are the risks of encoding data into dead pixels and retrieving them with sampleImage?

The solution is dependent on render format which can be tricky. CPU and GPU have different byte orders in After Effects (ARGB vs RGBA), and GPU formats like CUDA may use BGRA. Any swizzeling or color format conversion could mangle the data. Additionally, after threading changes in AE, sampleImage returns [0,0,0,0] if the background render thread hasn't finished rendering yet.

*Tags: `gpu`, `memory`, `render-loop`, `threading`, `cuda`*

---

## Does AEGP_RenderAndCheckoutLayerFrame perform the expensive rendering immediately or defer it until AEGP_GetReceiptWorld is called?

AEGP_RenderAndCheckoutLayerFrame performs the expensive render operation immediately and returns an AEGP_FrameReceiptH. You can then call AEGP_GetReceiptGuid() on the receipt to get a GUID that can be mixed into GuidMixInPtr() for dependency tracking. There is both a synchronous and asynchronous version available; the async version could potentially be called during SmartPreRender and awaited during the render call/thread.

*Tags: `aegp`, `smart-render`, `layer-checkout`, `threading`*

---

## What is the recommended approach for multithreading in Premiere plugins when the iterate suite doesn't work?

If the Iterate8Suite doesn't work reliably in Premiere (particularly with certain colorspace configurations), you should implement your own multithreading code instead of relying on the suite. Standard approaches include using std::parallel on Windows and equivalent threading libraries for Mac (like Grand Central Dispatch or pthreads). The iterate suite should theoretically work for 8-bit ARGB, but if you encounter issues, custom multithreading implementations are a stable fallback.

*Tags: `premiere`, `threading`, `multithreading`, `performance`, `bgra`, `macos`, `windows`*

---

## Why does a plugin fail with 'Not able to acquire AEFX Suite' error when playing in Premiere?

The issue may occur during render/playback if the plugin calls AEFX Suite in a shared thread or has incorrect host application detection logic. Check if there is conditional code that calls AEFX Suite only when the host is NOT Premiere (e.g., `if (app_id != 'PrMr') => call AEFX_suite`), but the host detection returns an incorrect value in Premiere. This can cause the plugin to attempt AEFX Suite calls in Premiere where they are not available. Premiere can also have unexpected behavior with certain suites, such as colorspace-related ones.

*Tags: `premiere`, `aegp`, `threading`, `debugging`*

---

## Why is a plugin exhibiting glitch behavior with Multi-Frame Rendering (MFR) even though the Multithreaded flag is not set?

MFR glitches can occur when different threads/CPUs are using different previous frames, causing inconsistent rendering behavior. Even without the Multithreaded flag explicitly set, the plugin may still attempt MFR, which can lead to these issues. Disabling MFR resolves the problem. The root cause appears to be improper frame state management across threads during multi-frame rendering.

*Tags: `mfr`, `threading`, `render-loop`, `debugging`*

---

## What is the behavior of After Effects regarding multi-frame rendering (MFR) and thread safety in plugins?

Even if plugins don't specify they are MFR safe, After Effects still calls them from different threads and requests frames out of sequence (for example, frame 5, then frame 3, then frame 6). The only way to prevent this non-sequential, multi-threaded behavior is to disable MFR completely.

*Tags: `mfr`, `threading`, `render-loop`*

---

## Why does Multi-Frame Rendering (MFR) start with multiple threads but reduce to a single thread on longer compositions?

James Whiffin observed that MFR often initiates rendering with 6-10 threads but consistently drops to single-threaded operation when compositions are sufficiently long. This behavior persists even when custom effects are disabled and only first-party Adobe effects are present, suggesting it is a characteristic of MFR itself rather than third-party plugin behavior.

*Tags: `mfr`, `threading`, `render-loop`, `performance`*

---

## Is it normal to receive thousands of PF_Arbitrary_DISPOSE_FUNC calls on After Effects shutdown?

Yes, this is normal behavior. The high number of disposal calls during shutdown is likely due to Multi-Frame Rendering (MFR) duplicating arbitrary parameters for each thread, combined with parameter changes triggering dispose/copy cycles.

*Tags: `arb-data`, `mfr`, `shutdown`, `threading`, `memory`*

---

## How can I safely detect if a layer is available for checkout in a userChangedParam callback?

A workaround is to validate layer data by checking width/height and Lrect values—if they are random, over 100,000, or negative, the layer is not properly available for checkout. Note that Param[0].u.ld.data will be empty if checkout hasn't occurred in that thread. However, this is not a fully reliable solution, and the proper approach may require additional checks during effect duplication and shutdown phases.

*Tags: `layer-checkout`, `params`, `arb-data`, `shutdown`, `threading`*

---

## Can you call PF_PROGRESS and PF_ABORT from a separate thread spawned during the render function?

No, you cannot call PF_PROGRESS and PF_ABORT from another thread. These functions must be called from the main thread. If you need to report progress during a blocking computation, you should consider alternative approaches such as baking the compute in a backend operation (like in userchangedParam), updating an arbitrary parameter during the update phase, storing results in sequence data, and applying them later (similar to how warp stabilizer works). This avoids blocking the main thread and allows proper progress reporting through the UI.

*Tags: `threading`, `render-loop`, `params`, `arb-data`*

---

## How should global variables be used in multi-threaded plugin development?

Define global variables at the top of the main .cpp file, initialized only once, to ensure the same value is accessed across both UI and render threads (which have run separately since CC 2014). Use atomic bools for thread-safe access, though mutexes may be needed in some cases. Avoid using static variables with MFR (Multi-Frame Rendering), but non-MFR plugins should be fine. Pass pointers to these globals carefully, as passing pointers (e.g., bool*) can cause crashes—use value types (e.g., bool) instead.

```cpp
static bool global_variable;
// In PF_cmd or prerender:
if(extra->cb->GuidMixInPtr) {
    extra->cb->GuidMixInPtr(in_data->effect_ref, sizeof(bool), reinterpret_cast<void*>(&global_variable));
}
```

*Tags: `memory`, `threading`, `smart-render`*

---

## What is the Adobe ae-plugin-thread-safety repository?

Adobe's ae-plugin-thread-safety repository at https://github.com/adobe/ae-plugin-thread-safety contains examples and best practices for handling thread-safe operations in After Effects plugins, including atomic variable usage and mutex patterns.

*Tags: `threading`, `open-source`, `reference`, `tool`*

---

## How can you trigger frame invalidation from external background threads in After Effects plugins?

GuidMixInPtr can be problematic for triggering invalidation from background threads when using global or sequence data variables. A reliable workaround is to change a composition property (like background color) via AEGP calls or scripting, which forces After Effects to check if frames need invalidation. Another approach is to use a hidden parameter that gets modified through AEGP or scripting calls, though this has limitations when modal dialogs are open. The method must ultimately signal to After Effects through a mechanism it actively monitors.

*Tags: `smart-render`, `threading`, `sequence-data`, `params`, `aegp`*

---

## How do you force After Effects to re-check whether frames need invalidating when using sequence data?

When sequence data isn't flattened, After Effects may not refresh frames as expected even when the data updates properly (verified via breakpoints). A workaround is to give AE a "kick" by changing composition background color or toggling a parameter, which forces it to check for invalidation. Alternatively, see the Adobe Community discussion on forcing rendering on events in separate threads: https://community.adobe.com/t5/after-effects-discussions/force-rendering-on-an-event-in-a-separate-thread-coming-back-to-the-main-ui-thread-on-an-event/m-p/13628197

*Tags: `sequence-data`, `render-loop`, `debugging`, `threading`*

---

## Why is there a memory leak when using AEGP_memorysuite on macOS but not Windows during export?

A user reported a memory leak when using AEGP_memorysuite to allocate, lock, memcpy to/from, and unlock memory handles during export (not preview render). The leak was macOS-specific and did not occur on Windows. Testing with different flags (mem_NONE, mem_CLEAR, mem_QUIET) showed the leak was less severe with mem_QUIET. Replacing the AEGP memory handle with a temporary pointer using new/delete eliminated the leak. Additionally, replacing memcpy with strncpy solved the issue when copying from the mem pointer to a buffer, but not in the opposite direction. This suggests a potential platform-specific issue with how AEGP_memorysuite handles memory during the export pipeline, possibly related to threading or memory alignment differences between macOS and Windows.

*Tags: `memory`, `aegp`, `macos`, `windows`, `debugging`, `threading`*

---

## How can you track and debug memory allocation and deallocation issues in After Effects plugins?

Wrap memory allocation and deallocation in custom logging wrapper functions to get a definitive record of what happened. Additionally, create a base class (like ObjectBase) that all C++ classes descend from to keep track of live instances of each type, which helps identify memory leaks and allocation mismatches. This approach also makes it easy to report current counts of allocations versus frees.

*Tags: `memory`, `debugging`, `threading`, `macos`, `windows`*

---

## Is it safe to allocate memory in one thread and free it in another thread in After Effects plugins?

It is generally not recommended to allocate memory in one thread and free it in another thread in After Effects plugins. AE makes few threading guarantees beyond pairing SETUP and SETDOWN calls. Even using mutex locks does not reliably solve cross-thread allocation/deallocation issues, as the plugin framework does not provide clear guarantees about thread lifecycle and frequency.

*Tags: `threading`, `memory`, `aegp`, `debugging`*

---

## How can you display a warning alert in an After Effects plugin without blocking the render thread?

Create a static global boolean variable that is modified by a mutex in the render thread, then check and print the warning if the boolean is true in another thread such as pFupdate_param_UI. This approach prevents blocking the render thread while still allowing warnings to be displayed to the user.

```cpp
static bool g_warning_flag = false;
static std::mutex g_warning_mutex;

// In render thread:
{
  std::lock_guard<std::mutex> lock(g_warning_mutex);
  g_warning_flag = true;
}

// In pFupdate_param_UI thread:
if (g_warning_flag) {
  // Display warning alert
  std::lock_guard<std::mutex> lock(g_warning_mutex);
  g_warning_flag = false;
}
```

*Tags: `threading`, `render-loop`, `ui`, `debugging`*

---

## Why does a plugin sometimes render without the effect applied when queued to Media Encoder from After Effects?

Static global variables in the plugin and elements defined in globalData or during global initialization thread can cause issues with Media Encoder compatibility. These should be reviewed and potentially refactored to avoid state persistence issues when the plugin runs in ME's different execution context.

*Tags: `media-encoder`, `debugging`, `memory`, `threading`*

---

## What are the advantages of compute cache compared to arbdata for storing pointers to heap data?

Compute cache offers several advantages over arbdata: (1) it can be written/read from any thread (render or UI) with optional thread-safety using std::atomic, (2) it's designed per-frame rather than requiring a global context like sequence or arb data, (3) it provides functions to specify whether to wait if another thread is computing the same key. However, arbdata gets saved with the project while compute cache does not. Compute cache replaces sequence data now that MFR is introduced and sequence data can no longer be used.

*Tags: `compute-cache`, `arb-data`, `threading`, `sequence-data`, `mfr`, `memory`*

---

## How do you properly access the compute cache in After Effects plugins and avoid crashes?

When accessing compute cache data, you must create a pointer to access_cache_data and extract the SPBasicSuite from in_data->pica_basicP. To avoid crashes and maintain independence from the render thread, create a new object and copy only the parts you need, then delete it after computing. This prevents thread safety issues and ensures the cache operates independently.

```cpp
access_cache_data* accessCacheData = static_cast<access_cache_data*>(optionsP);
if ( !accessCacheData->in_data ) {return A_Err_ALLOC;}
SPBasicSuite* bsuite;
bsuite = accessCacheData->in_data->pica_basicP;
if (!bsuite){return A_Err_ALLOC; }

struct access_cache_data {
    PF_InData* in_data;
    PF_OutData* out_data;
};
```

*Tags: `compute-cache`, `memory`, `threading`, `aegp`, `debugging`*

---

## What suites are unsafe to call from the Death Hook in After Effects plugins?

There are restrictions on which suites can be safely called from the Death Hook. The Utility Suite, specifically ReportInfo, has been reported to cause unhandled exceptions when called from Death Hook. This suggests that certain suites with external dependencies or state management are not safe to invoke during plugin shutdown, and developers should be cautious about suite usage in Death Hook contexts.

*Tags: `aegp`, `debugging`, `memory`, `threading`*

---

## Can one effect render from another effect during smartrender in After Effects?

During smartrender, it is not straightforward to have one effect call another effect's render. Smartrender happens on the render thread, and AE will not call smartrender on other layers if a current smartrender is already in progress—it waits until the current smartrender completes. One possible approach mentioned is to use EffectCallGeneric with a custom passthrough code if the other plugin implements it, but this has significant limitations. AEGP calls cannot be made from the render thread directly; they must be delegated to the UI thread. The async render and checkout functions (AEGP_RenderAndCheckoutLayerFrame_Async) might work from any thread but are noted as buggy. Ultimately, the questioner settled on manually implementing effects internally rather than calling other plugins dynamically.

*Tags: `smart-render`, `aegp`, `threading`, `render-loop`*

---

## Why does After Effects throw an 'effect attempting to modify a locked project' error when adjusting effect parameters during smartrender?

During a smartrender command, the project becomes locked to prevent modifications. Attempting to use AEGP calls to adjust effect parameters while smartrender is in progress results in this error because AEGP operations that modify the project cannot execute on the render thread. Such modifications must be deferred to the UI thread after the smartrender completes.

*Tags: `smart-render`, `aegp`, `threading`, `params`*

---

## Is it possible to force After Effects to render frames sequentially in a plugin?

No, there is no way to force After Effects to give you frames sequentially. The plugin must be designed to handle out-of-order frame rendering or implement its own logic to manage sequential processing if required.

*Tags: `render-loop`, `threading`, `aegp`*

---

## Should you use cv::mixChannels with memcpy or the IterateSuite for channel swizzling performance?

You should benchmark both approaches. Using cv::mixChannels followed by memcpy may be slower than using After Effects' IterateSuite (which provides one thread per row) and performing channel swizzling manually during the copy operation from matrix to layer. The IterateSuite approach can provide better performance by combining the channel conversion and copy in a single operation per row.

*Tags: `threading`, `render-loop`, `optimization`, `aegp`*

---

## How can you ensure sequence data stays synchronized across render threads when duplicating a plugin instance?

When duplicating a plugin and assigning a new unique ID to sequence data, the sequence data may become out of sync on render threads even though UpdateUI and UserChangeParams receive the correct values. Two potential solutions: (1) Check if the sequence is flattened in the project, as flattening can force render threads to get the correct sequence value. (2) Use PF_GetConstSequenceData from the PF_EffectSequenceDataSuite1 suite instead of relying on in_data->seq_data directly, as the latter is not updated correctly since CC2021/2022.

*Tags: `sequence-data`, `threading`, `render-loop`, `params`, `debugging`*

---

## When should you lock a handle when reading sequence data in After Effects plugins?

When calling GetConstSequenceData(), you should lock the handle before casting and unlock after use, even for read-only access. The lock syntax is: seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq); followed by unlock. This is particularly important in multi-frame rendering (MFR) scenarios with multiple plugin applications, as failure to lock can cause nullptr returns and threading-related crashes. The Adobe documentation does not clearly document this requirement, but it is necessary to prevent data corruption across threads.

```cpp
seqP = (unflat_seqData*)PF_LOCK_HANDLE(const_seq);
// use seqP
PF_UNLOCK_HANDLE(const_seq);
```

*Tags: `sequence-data`, `mfr`, `memory`, `threading`, `aegp`, `debugging`*

---

## What causes the generic exception handler/catch-all error in Adobe After Effects, and why does it appear more frequently in newer versions?

The generic exception handler/catch-all in the Adobe suites manager is triggered by issues like double freeing or releasing a suite pointer. This error has existed since CS2 or CS3 days. It appears more often in newer AE versions due to Multi-Frame Rendering (MFR) with global suite handles or suite pointers being shared between threads, which increases the likelihood of memory management conflicts.

*Tags: `mfr`, `threading`, `aegp`, `memory`*

---

## How can you unhover/remove highlight from a custom UI button when the cursor leaves the parameter UI area?

Use a debounce pattern with a timer to detect when the mouse has left the button area. Create a debounce function that wraps a callback (like unhover_after_2s) with a delay. Call this debounced callback every time a mouse-over event occurs on the button. The callback should trigger an updateUI on the main thread, which forces After Effects to re-render the custom UI and remove the hover state. This prevents the button from staying highlighted when the cursor moves outside the UI area.

```cpp
template <typename Func, typename... Args>
std::function<void(Args...)> debounce(Func func, std::chrono::milliseconds delay) {
    std::shared_ptr<std::chrono::steady_clock::time_point> lastCallTime =
        std::make_shared<std::chrono::steady_clock::time_point>();
    return [=](Args... args) {
        *lastCallTime = std::chrono::steady_clock::now();
        std::thread([=]() {
            std::this_thread::sleep_for(delay);
            if (std::chrono::steady_clock::now() - *lastCallTime >= delay) {
                func(args...);
            }
        }).detach();
    };
}
// Usage: auto mouse_over_button = debounce(unhover_after_2s, std::chrono::milliseconds(2000));
```

*Tags: `ui`, `custom-ui`, `threading`, `params`*

---

## How can you implement debouncing for callback functions in C++?

Alex Bizeau from maxon shared a template function that implements debouncing using std::chrono for timing and std::thread for delayed execution. The debounce function wraps a callback and returns a new function that only executes the original callback after a specified delay has passed without additional calls. It uses a shared_ptr to track the last call time and spawns a detached thread that checks if enough time has elapsed before invoking the original function.

```cpp
#include <chrono> #include <functional> #include <thread>  template <typename Func, typename... Args> std::function<void(Args...)> debounce(Func func, std::chrono::milliseconds delay) {     std::shared_ptr<std::chrono::steady_clock::time_point> lastCallTime =         std::make_shared<std::chrono::steady_clock::time_point>();      return [=](Args... args) {         *lastCallTime = std::chrono::steady_clock::now();         std::thread([=]() {             std::this_thread::sleep_for(delay);             if (std::chrono::steady_clock::now() - *lastCallTime >= delay) {                 func(args...);             }         }).detach();     }; }
```

*Tags: `threading`, `performance`, `reference`*

---

## Can an After Effects effect plugin compute heavy calculations and update parameter values from the render function?

This appears to be an unsolved use case. The developer wants to perform computationally expensive operations in an effect plugin's render function and have those results update a parameter slider, but current After Effects plugin architecture doesn't provide a straightforward mechanism for plugins to push parameter updates back to the host during rendering. Parameters are typically read during render, not written to.

*Tags: `params`, `render-loop`, `aegp`, `threading`*

---

## How can a C++ plugin update a text layer every frame during rendering without locking the project?

This is a fundamental limitation in After Effects: calling AEGP_SetText from the render thread causes the project to lock. Several workarounds exist: (1) Write data to sequence data or compute cache from the render thread, then read and apply it from the UI thread using the aegp_idle hook, which is called multiple times per second; (2) Use your own text rendering engine in the render thread with an external library; (3) Write to disk and have a ScriptUI panel with a timer read it back (though this is unreliable). However, none of these work reliably in MediaEncoder or headless rendering without a GUI. The fundamental issue is that modifying a text layer from render could create infinite loops, which is why AE locks the project.

*Tags: `aegp`, `render-loop`, `threading`, `ui`, `sequence-data`, `compute-cache`*

---

## What are the byte order and format considerations when encoding data into pixels for sampleImage retrieval?

sampleImage always returns RGBA format regardless of the internal render format or CPU vs GPU byte order differences (ARGB vs RGBA vs BGRA in CUDA). To avoid byte order issues, encode each color channel separately and limit it to one byte per color. After AE's threading changes, sampleImage returns [0,0,0,0] if the background render thread hasn't finished rendering the frame, so timing is critical.

*Tags: `sampleImage`, `pixel-encoding`, `byte-order`, `threading`, `rgba`*

---

## Can you use the Iterate8Suite in Premiere Pro plugins, and what are the alternatives for multithreading?

The Iterate8Suite has issues in Premiere Pro and doesn't work reliably with multithreading. The suite should work in 8-bit ARGB colorspace (SuiteV2 since 2022), but for BGRA you must use the special BGRA suite. If you need multithreading, implement your own using standard libraries like std::parallel for cross-platform compatibility.

*Tags: `premiere`, `threading`, `aegp`, `multithreading`*

---

## Why does my plugin report 'Not able to acquire AEFX Suite' when playing in Premiere?

This error typically occurs during playback/render in Premiere and may be caused by calling AEFX Suite functions in a shared thread or in code paths that execute in Premiere context. Check if you have conditional logic like `if (app_id != 'PrMr') => call AEFX_suite` where the app_id detection may be returning an incorrect value in Premiere, causing AEFX Suite calls to execute when they shouldn't. Additionally, Premiere can have quirks with certain suites (similar to known colorspace suite issues), so verify your host application detection logic and ensure AEFX Suite calls are not made from Premiere contexts.

*Tags: `premiere`, `aegp`, `threading`, `debugging`*

---

## Does the number of available threads returned by AEGP_GetNumThreads change during an After Effects session?

To the best of knowledge, the number of available threads does not change throughout an AE session. The decision on using PF_Iterations_ONCE_PER_PROCESSOR is an optimization decision that depends on your algorithm. For image processing, PF_Iterations_ONCE_PER_PROCESSOR is rarely optimal because some threads finish their chunk before others and cannot help with remaining work on other threads. Using n iterations instead assures minimal loss of available CPU power for most image processing algorithms.

*Tags: `threading`, `render-loop`, `aegp`, `optimization`*

---

## Why are global variables considered bad practice in plugin development?

Global variables are discouraged for several reasons: (1) They reduce code readability because function behavior depends on data outside their parameters; (2) They can cause CPU cache inefficiency as the processor must fetch data from different memory segments; (3) In multi-threaded scenarios, they require mutexes which impose performance penalties and make code non-thread-safe; (4) They are generally considered poor programming practice. However, they are widely used in practice, especially for constant data that is set once and never changes.

*Tags: `threading`, `memory`, `debugging`*

---

## What are the memory safety considerations when using AEGP_RenderAndCheckoutLayerFrame_Async?

According to community discussion with Adobe's AE team, there is a known memory release problem when using the async version of renderAndCheckout. As a result, developers have reverted to using the synchronous version instead, rendering one frame at a time on each idle cycle to avoid UI lag while properly managing memory.

*Tags: `aegp`, `memory`, `async`, `render-loop`, `threading`*

---

## How can you safely implement async layer frame rendering in an AEGP plugin?

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

*Tags: `aegp`, `threading`, `async`, `memory`, `render-loop`*

---

## Can iterate_generic be used to parallelize multiple transform_world calls across threads?

No, iterate_generic should not be used to call transform_world from utility threads. The iterate_generic suite uses utility threads that are separate from the main render threads, and AE API interaction is only safe from main threads. transform_world is internally multithreaded but is not designed to be called from non-main utility threads, which can result in bugs or crashes. Additionally, acquiring suites repeatedly in the iteration callback causes significant overhead. Instead, consider writing a custom transform implementation optimized for multi-threaded execution.

```cpp
suites.Iterate8Suite1()->iterate_generic(
  numThreads,
  &xform_data,
  MT_Xform
);

PF_Err MT_Xform(void* refcon, A_long threadInd, A_long iterNum, A_long iterTotal) {
  // Avoid calling transform_world or other AE suite functions from here
  // Pre-acquire suites in main thread instead
}
```

*Tags: `threading`, `render-loop`, `aegp`, `smartfx`, `performance`*

---

## Why is there no performance improvement when calling suite functions repeatedly inside iterate_generic callbacks?

Repeatedly acquiring suites inside iteration callbacks creates significant overhead that negates parallel execution benefits. Suite acquisition should be done once from the main thread before the iteration, and then pointers to the callbacks should be passed to the iteration function. Pre-acquiring suites can improve performance by orders of magnitude compared to acquiring them for every iteration.

*Tags: `threading`, `performance`, `caching`, `smartfx`, `optimization`*

---

## How can sequence data be safely shared and synchronized between PF_Cmd_EVENT and SmartPreRender calls?

Sequence data is intentionally separate between the render and UI threads by design, allowing render threads to execute asynchronously without data changes from other threads. To share data between them, use global data (shared across threads but not instance-specific) or store render instance data with an identifier such as comp item ID + layer ID + effect index on layer, then have the UI thread look up the correct data using a mutex. Note that arb/hidden params can only be written from the UI thread; render threads cannot modify project data.

*Tags: `sequence-data`, `threading`, `render-loop`, `aegp`, `arb-data`*

---

## How can I setup an After Effects plugin to asynchronously receive video frames from an external pipe?

You can setup a separate C++ thread (independent of the AE SDK) to handle reading/writing data to a pre-allocated memory buffer. Use a mutex to safely access this memory from both the async pipe thread and AE's render calls. To trigger the async pipe thread to process data promptly, use AEGP_CauseIdleRoutinesToBeCalled(). However, reconciling AE's random frame access model with sequential async pipe operations is challenging—you may need to either stall the render until required frames are available or cache frames to a file and fetch them on demand. For fetching source frames without blocking the UI, use AEGP_CheckoutOrRender_ItemFrame_AsyncManager or AEGP_CheckoutOrRender_LayerFrame_AsyncManager.

*Tags: `threading`, `async`, `aegp`, `memory`, `caching`*

---

## Why is PF_Cmd_SEQUENCE_FLATTEN sent when an effect is first applied, and why are SEQUENCE_RESETUP and SEQUENCE_SETDOWN called multiple times on different threads?

PF_Cmd_SEQUENCE_FLATTEN is sent when PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING is set during global setup. Flattening allows plugins to serialize complex data structures (containing pointers or other non-copyable elements) into a single copyable piece of memory. Multiple SEQUENCE_RESETUP calls occur because AE's rendering architecture separates UI and rendering threads, each maintaining their own copy of sequence data. The PF_OutFlag2_SUPPORTS_GET_FLATTENED_SEQUENCE_DATA flag allows returning a flattened copy without losing the original, reducing some resetup calls on the UI thread. The PF_OutFlag2_MUTABLE_RENDER_SEQUENCE_DATA_SLOWER flag accommodates plugins requiring separate sequence data per rendering thread. Multiple SEQUENCE_SETDOWN calls correspond to cleanup on different threads.

*Tags: `sequence-data`, `threading`, `aegp`, `mfr`*

---

## How can I store per-effect instance data and ensure unique instance IDs when effects are copied, duplicated, or loaded from disk?

There is no built-in mechanism for per-instance unique IDs. A practical workaround is to store an instance ID in sequence data and use global data to track instances. On SEQUENCE_RESETUP, flag the sequence data as 'requires checking the id', then scan the project (or layer) for conflicting IDs. If duplicates are found, deduce which is the original instance (by tracking where it was last seen) and assign a new ID to the copy. This approach requires careful state management across copy/paste, duplication, and file loading operations. Note that during SEQUENCE_RESETUP, you may receive either flattened or unflattened data, so tag your data structure to track its state.

*Tags: `sequence-data`, `arb-data`, `aegp`, `threading`*

---

## Should I use multithreading or MFR in my After Effects plugin, or can both be used together?

Both multithreading (MT) and Multi-Frame Rendering (MFR) can coexist in plugins. While they may seem incompatible at first, MFR and MT don't necessarily compete for CPU resources because not all steps in the rendering pipeline of a single frame are multi-threadable. MFR excels at handling non-multi-threadable steps, while the multi-threadable parts are often CPU cache-friendly and don't compete with MFR threads. The Adobe After Effects engineering team has extensively tested these scenarios and optimized the mechanism to handle a wide range of rendering scenarios, including plugins that aren't MFR-enabled and those relying on GPU rendering.

*Tags: `mfr`, `threading`, `render-loop`, `sdk`*

---

## How can I access AEGP suites within an IdleHook callback?

The IdleHook callback does not have direct access to in_data, so you cannot directly access suites through the in_data->pica_basicP pointer. You need to store a reference to the AEGP_SuiteHandler or pica_basicP pointer in a global or static variable during plugin initialization (in EffectMain or at plugin startup), and then retrieve it in your IdleHook callback to execute suite operations.

*Tags: `aegp`, `threading`, `sdk`*

---

## How can you force rendering from a separate thread in After Effects plugins?

You cannot directly set out_flags from a separate thread. Instead, use the idle hook, which fires 30-50 times per second on the main UI thread. From the idle hook, you can: (1) change an invisible parameter value to force re-render, (2) call command numbers to purge cache, or (3) use AEGP_EffectCallGeneric to have the effect instance act on itself. The idle hook is the recommended approach for syncing work from separate threads back to the main thread.

```cpp
// In idle hook (main thread)
// Change invisible param to trigger re-render
out_data->out_flags |= PF_OutFlag_REFRESH_UI | PF_OutFlag_FORCE_RERENDER;
```

*Tags: `threading`, `render-loop`, `aegp`, `ui`, `debugging`*

---

## How do you handle WebSocket messages in a separate thread while still triggering UI updates in After Effects?

Use an idle hook callback combined with a separate worker thread. The worker thread (e.g., using tinythread via easywsclient) handles WebSocket messages and stores state. The idle hook, which runs on the main UI thread at 30-50 Hz, polls for new messages and triggers parameter changes or cache purges. This pattern ensures all UI modifications happen on the main thread while external communication occurs asynchronously.

*Tags: `threading`, `ui`, `render-loop`, `cross-platform`*

---

## Why isn't sequence_data updated when modified in UpdateParameterUI and passed to SequenceResetup?

The issue is that UpdateParameterUI is not the correct place to modify sequence_data with FORCE_RERENDER. According to Adobe documentation, FORCE_RERENDER works during PF_Cmd_USER_CHANGED_PARAM and also in CLICK and DRAG events (if PF_Cmd_GET_FLATTENED_SEQUENCE_DATA is implemented). Modifications to sequence_data should be made in PF_Cmd_USER_CHANGED_PARAM instead of UpdateParameterUI to ensure the data is properly synchronized between UI and render threads.

*Tags: `sequence-data`, `aegp`, `threading`, `params`, `ui`*

---

## Why does AEGP_GetLayerToWorldXform return zeros when called from multiple threads using Iterate8Suite1?

Most AEGP functions are not thread-safe. AEGP_GetLayerToWorldXform should only be called from the main thread that After Effects called your plug-in with. Calling it from other threads will return zeros or cause internal verification failures. Only a few AEGP functions are documented as thread-safe.

*Tags: `aegp`, `threading`, `memory`, `sdk`*

---

## How should you handle frame interruption in After Effects plugins to avoid crashes when advancing frames quickly?

When a user interrupts frame processing (by advancing to another frame, clicking buttons, etc.), After Effects does not forcefully kill the rendering thread. Instead, the plugin must call PF_ABORT() during the render call (not pre-render) to check if AE wants processing to stop. If PF_ABORT returns true, you should return a PF_Interrupt_CANCEL error message to tell AE the output buffer should not be used. Crashes during interruption are often caused by re-entrancy issues—if your crash occurs when accessing shared data structures like matrices, ensure thread-safety by using mutexes or allocating structures on the stack rather than globally. Call PF_ABORT periodically during lengthy processing to maintain responsive interactivity while scrubbing or changing parameters.

```cpp
if (err = PF_ABORT(in_dataP)) {
  err = PF_Interrupt_CANCEL;
  return err;  // Must return after setting the error
}
```

*Tags: `threading`, `render-loop`, `memory`, `debugging`, `aegp`*

---

## Is it safe to start a background process from the UserChangedParam callback?

Yes, there are no issues starting background processes from UserChangedParam. In fact, it's the preferred location for such operations because it's user-triggered (rare), and it's called on the UI thread rather than the render thread, making it safer for UI-related work.

*Tags: `params`, `threading`, `ui`*

---

## How does sequence data get synchronized between the UI thread and render thread in Smart Rendering?

Sequence data synchronization between UI and render threads happens at specific times only: when a new instance is created and during specific UI events. Synchronization occurs when PF_Cmd_USER_CHANGED_PARAM is triggered, and in CLICK and DRAG events if PF_Cmd_GET_FLATTENED_SEQUENCE_DATA is implemented. The mechanism involves PF_Cmd_GET_FLATTENED_SEQUENCE_DATA and PF_Cmd_SEQUENCE_RESETUP for flattening and unflattening respectively. However, PreRendering and SmartRendering do not automatically trigger this flattening. For reliable data persistence, Adobe recommends using invisible arb parameters which sync on every change, or implementing a message-passing mechanism through global data structures accessible to both threads.

*Tags: `sequence-data`, `smart-render`, `arb-data`, `threading`, `render-loop`*

---

## What is the best way to persist custom objects from the UI to the rendering thread in After Effects plugins?

The recommended approach is to use invisible arb parameters, which create undo entries and provide immediate invalidation of cached frames on changes. Alternatively, if arb parameters don't fit your design, you can devise a message-passing mechanism by placing instructions in a global data structure common to both threads, with the render thread checking for messages. Direct use of sequence data flattening outside of USER_CHANGED_PARAM or UI click/drag events is not reliable and should be avoided, as it can lead to sync issues with SmartRendering and PreRendering.

*Tags: `arb-data`, `threading`, `sequence-data`, `smart-render`, `params`*

---

## Can I call a command selector like PF_Cmd_DO_DIALOG in the render function?

No, since CC2015 the render thread cannot perform operations that change the project. Instead, you should use alternative approaches: (1) Use an idle hook to send a message from the render thread to an AEGP, which then creates a window during the idle hook call when it's safe. (2) Use SEQUENCE_SETUP, which is called whenever a new instance of your effect is applied. (3) Use GLOBAL_SETUP, which is called once per session when the effect is first applied. These approaches allow you to open dialogs for user input (like username/password) without trying to trigger commands from the render thread.

*Tags: `render-loop`, `aegp`, `threading`, `ui`*

---

## Is it possible to import footage in the background using AEGP functions?

No, it is not possible to import footage in the background. AEGP functions must execute on the main thread only, so background import operations are not supported by the After Effects SDK.

*Tags: `aegp`, `threading`, `import`, `main-thread`*

---

## Are effect_ref and AEGP_EffectRefH the same thing and interchangeable?

No, effect_ref and AEGP_EffectRefH are not interchangeable. While they may have similar values because both denote RAM addresses, they hold different kinds of data. The effect_ref is used for very few things like GetNewEffectForEffect(), while AEGP_EffectRefH is used for a wide range of AEGP callbacks. Additionally, you cannot rely on the validity of either beyond the scope of a single AE call—after a call ends, AE may change its dataset and invalidate the references. To safely use AEGP functions from an idle hook, store your effect's index on the layer, layer ID, and comp item ID instead, then look up a fresh AEGP_EffectRefH during each idle hook call.

*Tags: `aegp`, `threading`, `params`, `memory`*

---

## How can I safely modify effect parameters from a custom thread in After Effects?

You cannot safely call most AEGP functions from custom threads—doing so will cause catastrophic crashes. The only exception is AEGP_CauseIdleRoutinesToBeCalled(), which is safe to call from any thread. To modify effect parameters, use an idle_hook instead. Store your effect's index on the layer, layer ID, and comp item ID (not AEGP_EffectRefH), then during the idle hook call, look up the comp, layer, and effect to obtain a fresh AEGP_EffectRefH for use in that single call. This ensures references remain valid and all operations occur on AE's main thread.

*Tags: `aegp`, `threading`, `idle-hook`, `params`*

---

## What does the max_sleepPL argument in an idle hook function represent?

The A_long *max_sleepPL argument in an idle hook function represents the maximum sleep time in units of 1/60 seconds. It functions as an upper bound ('wait no longer than X * 1/60 seconds') rather than a fixed duration, providing a way to control idle hook execution frequency without requiring longer waits.

*Tags: `aegp`, `render-loop`, `threading`*

---

## Should I use the Iterate Suite with many iterations containing inner loops, or fewer iterations with larger inner loops?

When using the Iterate Suite with multithreading, the best approach depends on how well tasks can be split between cores rather than mathematical efficiency. If you have 4-75 outer iterations with 10,000 inner loop iterations, fewer large iterations would have less overhead. However, the optimal strategy is to break the work into smaller tasks that can run in parallel across available threads. With variable inner loop sizes (10-10,000), consider distributing tasks evenly across cores to avoid situations where some threads finish early and wait for others. Testing different configurations is recommended to find the optimal thread count for your specific workstation.

*Tags: `threading`, `iterate-suite`, `performance`, `multithreading`, `optimization`*

---

## What is the performance impact of acquiring and releasing the sample suite on every pixel during iteration?

Acquiring and releasing the sample suite on every pixel significantly weighs down performance. For optimal results when you need to sample many adjacent pixels, it's recommended to access the input and output buffers directly and use iterate_generic instead, which allows each thread to handle a single buffer line and eliminates the overhead of repeated suite acquisition/release operations.

*Tags: `ccu`, `performance`, `memory`, `threading`, `optimization`*

---

## How can I pass data from a PopupDialog function to the Render function in an After Effects plugin?

There are two recommended approaches: (1) Store the message in global_data along with an effect identifier (comp itemID, layerID, and effect index), so the render thread can check if its matching identifier has a message waiting. (2) Write the data to an arbitrary parameter using SetStreamValue(), which gets synchronized immediately and reliably between threads. The arb param approach also allows undo/redo functionality for the user.

*Tags: `sequence-data`, `threading`, `aegp`, `arb-data`*

---

## Why is sequence_data different between PF_Cmd_RENDER and PF_Cmd_USER_CHANGED_PARAM in After Effects CC 2015 and later?

In After Effects CC 2015 and later, the render thread and UI thread have separate sequence_data handles that only synchronize at specific occasions. This is a design change from earlier versions like CS5.5. To force synchronization, set PF_OutFlag_FORCE_RERENDER during UserChangedParam, which forces a flatten call on the UI thread and an unflatten on the render thread, syncing the render thread's sequence data with the UI data.

*Tags: `sequence-data`, `threading`, `params`, `render-loop`, `ui`*

---

## How can data be shared between render and UI threads if sequence_data synchronization is not reliable?

Global data is shared between threads and can be used as an alternative solution for render-to-UI-thread communication when sequence_data synchronization is not reliable. However, relying on updating UI sequence_data from the render function is not supported by design in CC 2015 and later versions.

*Tags: `threading`, `memory`, `render-loop`, `ui`*

---

## Is there a flag to disable multithreading support for an After Effects plugin?

No, there is no flag to disable multithreading for plugins. All plugins are expected to handle multi-threaded rendering. If your render code is not thread-safe, you could use a mutex at the top of your rendering function to allow only one thread at a time, but this would severely cripple performance. The recommended approach is to make your plugin's render function multi-threadable instead.

*Tags: `threading`, `render-loop`, `aegp`*

---

## How should plugins handle non-thread-safe code in the render function?

While there is no way to disable multithreading entirely, if your code is not thread-safe, you can use a mutex at the top of your rendering function to serialize access and allow only one thread at a time to execute the critical section. However, this approach significantly reduces performance benefits of multithreading. The better solution is to refactor your plugin to make the render function properly thread-safe and multi-threadable.

*Tags: `threading`, `render-loop`, `memory`*

---

## How can you exchange data between an effect plugin and an IdleHook callback function?

While an IdleHook can reside in an effect plugin, the callback doesn't have direct access to the effect's in_data structure or parameters. The recommended approach is to allocate and lock a shared piece of RAM during global setup, store it in the global data structure, and pass it as the IdleRefCon to AEGP_RegisterIdleHook. AE will then pass this shared memory to the hook handling function on each idle call. Since this RAM is shared between all instances and threads, use a mutex to prevent race conditions. During idle calls, you can read/write parameter values using AEGP_GetNewStreamValue(), passing the instance reference data through the shared RAM.

*Tags: `aegp`, `memory`, `threading`, `params`*

---

## How can I debug iterate_generic functions properly in Visual Studio without erratic step-over behavior?

When debugging iterate_generic functions in Visual Studio, hit the breakpoint for the first time, then disable the breakpoint before stepping line by line. This prevents the debugger from jumping between threads (BEE WorkQueue and system DLLs) and allows normal line-by-line stepping.

*Tags: `debugging`, `iterate-generic`, `threading`, `windows`*

---

## How can you keep UI and render threads in sync when automatically applying multiple effects in CC2015+?

When using Sequence Setup to automatically add effects to a layer, the UI and render copies of the project can become out of sync in CC2015+, causing verification failures and missing data effects. The solution is to use an AEGP (After Effects General Plug-in) with an idle hook instead of modifying the layer during Sequence Setup. The effect can set a flag when it's applied, and the AEGP's idle hook will then process the pending operation to add the additional effects. The AEGP can be bundled in the same binary as the effect plugins, but the AEGP's PiPL must be listed first in the bundle.

*Tags: `aegp`, `pipl`, `ui`, `smart-render`, `threading`, `plugin-architecture`*

---

## How can I access output width and height values in a UI event handler?

The out_data.width and out_data.height values are only available during rendering and will be 0 in UI event handlers. To access these values in a DragHandle or DoClick event, you need to cache the desired values in sequence_data during the render call, then retrieve them during the event call. Note that in After Effects 13.5+, sequence_data cannot be changed from the render thread—only changes from the UI thread persist to the project and render thread.

*Tags: `ui`, `render-loop`, `sequence-data`, `threading`, `params`*

---

## How can I share data between the render thread and UI event handlers?

Since After Effects 13.5, sequence_data modified in the render thread cannot be accessed in UI threads. Two workarounds are: (1) add a pointer to sequence_data that is shared between threads (requiring logic to distinguish live pointers from saved invalid ones), or (2) store data in the global_data handle which is shared between threads and instances (requiring logic to identify which data belongs to which instance). The recommended approach is caching values in sequence_data during render and reading them back in event handlers from the UI thread.

*Tags: `sequence-data`, `threading`, `arb-data`, `ui`, `render-loop`*

---

## How can I safely use threads in an After Effects plugin while calling AEGP suite functions?

Threading is possible in After Effects plugins, but AEGP suite calls must only be made from the main thread. After Effects behaves unpredictably if suite calls are made from other threads, and the application can become unstable even after the worker thread has completed. Use threads only for background computation; marshal all AE API calls back to the main thread. Boost threads have been confirmed to work reliably when this constraint is followed.

```cpp
static void myFunc(){
  for(int i = 0; i<100; i++){
    // Do computation here, not AE API calls
  }
}
// Call AEGP suites only from main thread
ERR(suites.LayerSuite8()->AEGP_GetActiveLayer(&current_layerH));
```

*Tags: `threading`, `aegp`, `plugin-development`, `mainthread`*

---

## How can I set time-variant stream values for every frame in an AEGP plugin without creating keyframes at each frame?

According to shachar carmi, expressions are the only way to modify parameter values between keyframes without creating keyframes at each frame. Using AEGP_SetLayerStreamValue() or the Keyframe Suite will automatically create a keyframe at the current time if the stream is already keyframed. On AE 13.5+, only the UI thread can safely change parameter values. The SetStreamValue() function sets the value at the current item time, but if keyframes already exist on that stream, it will create or modify a keyframe at that time.

*Tags: `aegp`, `params`, `keyframe`, `threading`*

---

## How can I allow users to abort a long-running keyframe baking process triggered by PF_Cmd_USER_CHANGED_PARAM?

Instead of blocking the entire baking process in a single PF_Cmd_USER_CHANGED_PARAM call, implement incremental processing: get a clock reading at the beginning and process keyframe changes in ~1 second chunks. Let the AEGP handle continuation during idle time. This allows users to interrupt for interaction, and processing resumes immediately after. This approach enables user interaction while the baking process continues in segments rather than blocking on a single long operation.

*Tags: `params`, `aegp`, `ui`, `threading`*

---

## How can I run an idle hook function only once and then stop it from being called?

There is no built-in unregister function to stop receiving idle hook calls after AEGP_RegisterIdleHook(). Instead, you should set a flag in your plug-in that bypasses the idle hook code after the first run. For example, use a boolean SWITCH variable that you set to FALSE after the first execution, so subsequent idle hook calls skip the processing logic.

```cpp
BOOL SWITCH = TRUE;
static A_Err IdleHook(
  AEGP_GlobalRefcon plugin_refconP,
  AEGP_IdleRefcon refconP,
  A_long *max_sleepPL)
{
  *max_sleepPL = 500;
  A_Err err = A_Err_NONE;
  if (SWITCH) {
    // do something...
    SWITCH = FALSE;
  }
  return err;
}
```

*Tags: `aegp`, `idle-hook`, `plugin-architecture`, `threading`*

---

## Can I call AEGP functions from an external thread, and what is the correct pattern for using idle hooks with external threads?

You can call any AE function from an external thread, but only when AE is ready for it. The correct pattern is: (1) register the idle hook and launch the external thread from the entry point function, (2) have your external thread store its data in a global scope structure, (3) in the idle hook function, check if the global structure is filled and ready; if so, execute processing code; if not, check again next time, (4) set a skip idle hook flag to true when done and kill the external thread. Alternatively, make a synchronized call to your server from within the idle hook and avoid the external thread complexity.

*Tags: `aegp`, `threading`, `idle-hook`, `synchronization`, `plugin-architecture`*

---

## How can I implement a custom pixel iteration function similar to PF_Iterate8Suite1::iterate?

To implement a custom iteration function, start by examining the 'shifter' sample which implements one of the iteration functions. For direct pixel data access without the iteration suite, consult the 'CCU' sample, particularly its render function. You can also use the iterate_generic() function to obtain threading services without additional functionality. For optimal performance, thread rows rather than individual pixels to reduce CPU overhead per pixel.

*Tags: `mfr`, `render-loop`, `threading`, `reference`, `pixel-iteration`*

---

## Does host_lock_handle provide thread safety and is it recursive?

host_lock_handle does NOT provide thread safety—it only prevents AE from moving the memory chunk. Regarding recursiveness (multiple locks on the same handle), the behavior is unclear. Calling host_lock_handle twice followed by a single host_unlock_handle may or may not leave the handle locked, depending on whether the locking mechanism is reference-counted. It's safest to maintain a 1:1 ratio of lock to unlock calls per handle.

*Tags: `memory`, `threading`, `aegp`*

---
