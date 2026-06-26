# Idle Hook

> 15 Q&As · source: AE plugin dev community Discord

### How can I invoke AEGP functionality from ExtendScript (JSX) without creating a menu entry?

There is no direct way to invoke an AEGP from JavaScript without aid from a C external object. The simplest approach is to leave a flag in the JavaScript global scope and have the AEGP check for that flag on idle_hook calls (which happen 20-50 times per second). It's not immediate and synchronous, but it works easily. An alternative is using a C external object (ExternalObject in ExtendScript), though it's more complex to set up.

*Tags: `aegp`, `extendscript`, `external-object`, `idle-hook`, `interop`*

---

### How can I create a progress bar or continuously animated element in the ECW (Effect Control Window)?

ECW UI elements get idle calls only when the cursor is within their borders. For updates regardless of cursor position, register an idle_hook and trigger a redraw using PF_RefreshAllWindows() during that event. It's a brute-force approach but gets the job done. For softer redraws when using native ECW idle calls, consider PF_InvalidateRect or PF_UpdateParamUI instead.

*Tags: `animation`, `custom-ui`, `ecw`, `idle-hook`, `progress-bar`, `refresh`*

---

### How can I detect when an effect is removed from a layer?

There's no direct API for detection. PF_Cmd_SEQUENCE_SETDOWN is called when AE destroys an instance, but only when the deletion is out of the undo stack (100 steps later) or on project close/purge. Approaches: (1) Scan the project during idle_hook, catalog effect instances, and detect when one disappears. Not immediate. (2) Monitor 'cut' and 'delete' via command hooks, but this is unreliable - redo operations don't trigger menu commands. Important considerations: 'cut' puts the effect in limbo (can be pasted back or undone). For the idle_hook approach, note that in_data is not available in IdleHook - you need to acquire suites differently, using the SPBasicSuite passed during AEGP initialization.

*Tags: `command-hook`, `effect-removal`, `idle-hook`, `sequence-setdown`, `undo-stack`*

---

### How can I make the idle_hook get called more frequently for near-real-time communication?

Use AEGP_CauseIdleRoutinesToBeCalled() to trigger more frequent idle calls. Setting *max_sleepPL directly doesn't reliably work as it gets reset to its default value. Note that on macOS the idle hook caps at around 30 calls per second, while on Windows it can reach 60.

*Tags: `cause-idle`, `frame-rate`, `idle-hook`, `max-sleep`, `real-time`*

---

### How can I detect when an AE project is saved, from a plugin?

There's no reliable pre-save notification. Two approaches: (1) Use command_hook to catch save events, but it's not 100% bulletproof (sometimes saves happen without triggering the hook). (2) Implement an idle_hook and check if the project dirty flag changes from dirty to clean - if so, the user has saved. For effect plugins specifically, use sequence data with PF_OutFlag_SEQUENCE_DATA_NEEDS_FLATTENING, which causes AE to call your effect with PF_Cmd_SEQUENCE_FLATTEN on save.

*Tags: `command-hook`, `dirty-flag`, `idle-hook`, `project-save`, `sequence-flatten`*

---

### How can I force a re-render from a separate thread (e.g., when receiving WebSocket messages)?

It's only possible via idle_hook, which runs on the UI thread (30-50 times per second). Check your other thread for messages during idle events. Options to force re-render: (1) Change a hidden/invisible parameter value - this forces AE to re-render. (2) Call the command number to purge the cache. (3) Call AEGP_EffectCallGeneric to have the effect instance act on itself. You cannot set out_data->out_flags from a separate thread - it must happen on the main UI thread during an AE callback.

*Tags: `force-rerender`, `hidden-param`, `idle-hook`, `threading`, `ui-thread`, `websocket`*

---

### How do you display a progress banner (like 'Computing: X%') that updates during SmartRender in After Effects?

The technique involves using a hidden parameter to trigger re-renders. When the user presses a button, code changes a hidden parameter which triggers AE to call SmartRender again. Each SmartRender call checks the current state and draws the appropriate banner. Adobe's internal 3D Camera Tracker uses PF_Cmd_RESERVED3 (an undocumented idle command selector that AE calls repeatedly) to check background work status and sets PF_OutFlag_FORCE_RERENDER to trigger SmartRender calls. Parameter values can only be changed during PF_Cmd_USER_CHANGED_PARAM and PF_Cmd_EVENT. An alternative is to use AEGP_RegisterIdleHook with hidden parameter changes, though you don't have access to PF_OutData (and thus PF_OutFlag_FORCE_RERENDER) in an idle hook. In Premiere Pro, PF_OutFlag_FORCE_RERENDER doesn't work, so the hidden parameter trick is the recommended approach.

*Tags: `force-rerender`, `hidden-parameter`, `idle-hook`, `pf-cmd-reserved3`, `progress-bar`, `re-render`, `smartrender`*

---

### How can you trigger frame invalidation from a background worker thread without direct AEGP access?

Use AEGP_ExecuteScript from IdleHook to run an After Effects script that modifies a composition property (like background color) or a hidden parameter value. This forces After Effects to re-evaluate which frames need invalidation. Be aware that this fails if modal dialogs are open.

*Tags: `aegp`, `idle-hook`, `scripting`, `threading`*

---

### What is a reliable method to pass data from an effect plugin to a text layer?

An AEGP plugin using an idle hook is the best approach. Each frame, push string results into a queue, and periodically process the queue in the idle hook. This requires two binaries (AEGP plugin and FX plugin) communicating through a C interface. Alternatively, you can encode data into dead pixels in your render and use sampleImage expressions to retrieve those pixels and set source text, though this has caveats with render format reliability.

*Tags: `aegp`, `idle-hook`, `plugin-communication`, `render-loop`, `text-layer`*

---

### How do I make the idle hook get called more frequently to support higher frame rate precomp playback?

Use the AEGP_CauseIdleRoutinesToBeCalled function instead of trying to manually set max_sleepPL. This will cause idle routines to be called more frequently. Note that on macOS the call rate appears to be capped at around 30 calls per second, while on Windows it can reach 60 calls per second.

*Tags: `aegp`, `cross-platform`, `idle-hook`, `macos`, `windows`*

---

### How can I move a layer from one composition to another or create a precomposition from a layer using AEGP calls?

You can trigger the precompose command just like any other After Effects command through AEGP. However, if you attempt to execute this during a PF_Cmd_USER_CHANGED_PARAM callback, it will crash because precomposing invalidates the effect from which you're currently operating. To avoid this crash, you need to call the precompose command from an AEGP idle hook instead, which executes the operation outside of the current effect's execution context.

*Tags: `aegp`, `callback`, `idle-hook`, `plugin-crash`, `precomp`*

---

### How can I safely modify effect parameters from a custom thread in After Effects?

You cannot safely call most AEGP functions from custom threads—doing so will cause catastrophic crashes. The only exception is AEGP_CauseIdleRoutinesToBeCalled(), which is safe to call from any thread. To modify effect parameters, use an idle_hook instead. Store your effect's index on the layer, layer ID, and comp item ID (not AEGP_EffectRefH), then during the idle hook call, look up the comp, layer, and effect to obtain a fresh AEGP_EffectRefH for use in that single call. This ensures references remain valid and all operations occur on AE's main thread.

*Tags: `aegp`, `idle-hook`, `params`, `threading`*

---

### How can I run an idle hook function only once and then stop it from being called?

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

### Can I call AEGP functions from an external thread, and what is the correct pattern for using idle hooks with external threads?

You can call any AE function from an external thread, but only when AE is ready for it. The correct pattern is: (1) register the idle hook and launch the external thread from the entry point function, (2) have your external thread store its data in a global scope structure, (3) in the idle hook function, check if the global structure is filled and ready; if so, execute processing code; if not, check again next time, (4) set a skip idle hook flag to true when done and kill the external thread. Alternatively, make a synchronized call to your server from within the idle hook and avoid the external thread complexity.

*Tags: `aegp`, `idle-hook`, `plugin-architecture`, `synchronization`, `threading`*

---

### How do you properly implement inter-effect communication using AEGP_EffectCallGeneric in After Effects plugins?

When calling an effect from an AEGP (like Sweetie calling Checkout), you must handle timing carefully. The key issue is that you cannot call an effect back immediately while it's still processing its original call to your AEGP. Instead, use idle_hook: when the effect calls your AEGP, set a flag and wait for the next idle_hook call, then initiate AEGP_EffectCallGeneric when the effect is guaranteed to be idle. Alternatively, respond immediately via the same custom suite that did the calling, passing data for the calling effect to process without needing effectCallGeneric. Do not use AEGP_GetNewEffectForEffect with a passed effect_ref—instead, use AEGP_GetLayerEffectByIndex to look up the effect on the target layer. For permanent effect tracking across the project lifetime, store the comp itemID and layer layerID rather than relying on EffectRefH which can become invalid.

```cpp
// From AEGP: wait for idle hook before calling effect back
if (AEFX_AcquireSuite(in_data, out_data, kDuckSuite1, kDuckSuiteVersion1, "Couldn't load suite.", (void**)&dsP)) {
  PF_STRCPY(out_data->return_msg, "No Duck Suite!");
} else {
  if (dsP) {
    dsP->Quack(2);
  }
  AEFX_ReleaseSuite(in_data, out_data, kDuckSuite1, kDuckSuiteVersion1, "Couldn't release suite.");
}

// From Effect: pass in_data to AEGP via custom suite for callback
static SPAPI A_Err CalledFromEffect(void* ptr) {
  PF_InData* pIndata = static_cast<PF_InData*>(ptr);
  // Use AEGP_GetLayerEffectByIndex instead of AEGP_GetNewEffectForEffect
}
```

*Tags: `aegp`, `effect-ref`, `idle-hook`, `inter-plugin`, `reference`, `timing`*

---
