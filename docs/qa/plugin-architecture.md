# Q&A: plugin-architecture

**16 entries** tagged with `plugin-architecture`.

---

## Is Lua or Python a better choice for embedding a scripting language inside an After Effects plugin?

Lua is a significantly better fit for embedding inside an AE plugin. It is very easy to compile in and has a small footprint. Python is much clunkier to embed because it requires installing libraries on the user's system (asking permission to install packages), which is intrusive. While there may be cleaner ways to embed Python, Lua is far simpler to integrate. Both require similarly tedious C binding work without some kind of automation or code-generation tooling. dvb metareal shipped 'omino lua' as a script-driven drawing plugin using this approach.

*Contributors: [**dvb metareal**](../contributors/dvb-metareal/), [**rowbyte**](../contributors/rowbyte/) · Source: adobe-plugin-devs · 2023-08-16 · Tags: `lua`, `python`, `scripting`, `embedding`, `c-bindings`, `plugin-architecture`*

---

## What are the new entry points in After Effects plugins and how do they relate to PIPL?

After Effects plugins now have two entry points: EffectMainExtra and PluginDataEntryFunction, which are defined via PF_REGISTER_EFFECT in the samples. These new entry points may eventually replace PIPL, though there is no official announcement. Currently, PIPL is still used by After Effects but not by Premiere Pro anymore, so you only need to write PIPL for AE. The new entry points potentially allow for removing PIPL/rsrc resources entirely, though the exact mechanism needs further investigation in the samples.

*Tags: `pipl`, `aegp`, `ae`, `plugin-architecture`*

---

## How do you specify which function is EffectMain when using the new PF_REGISTER_EFFECT entry points?

The exact mechanism for specifying the EffectMain function with the new entry points is not clearly documented. It should be checked in the official After Effects plugin samples, though the investigation into this approach is ongoing.

*Tags: `pipl`, `aegp`, `plugin-architecture`*

---

## Can a single After Effects plugin file contain multiple PiPL entries with different entry points?

Yes, a plugin can have multiple PiPL entries allowing different entry points and different plugins in one file. However, this functionality is not consistently supported throughout After Effects and is not supported at all in Premiere Pro, so it has not gained much traction in practice.

*Tags: `pipl`, `premiere`, `plugin-architecture`*

---

## What are the challenges with using third-party GPU debuggers for After Effects plugins?

GPU debuggers like gDEBugger, RenderDoc, and others frequently crash or fail when tracing After Effects. The core issue is that these debuggers are designed for standalone games with permanent instances, whereas plugin architectures within host applications like After Effects or Premiere Pro create a more complex environment. Additionally, debuggers may capture the host application's graphics API (DirectX) instead of the plugin's rendering API (OpenGL, Vulkan), making them ineffective for plugin-specific debugging.

*Tags: `gpu`, `debugging`, `opengl`, `vulkan`, `plugin-architecture`*

---

## Can you use smart pointers in After Effects global data objects?

No, you should not use smart pointers in global data objects. Instead, use new/malloc directly since After Effects does funky things with that pointer. You can do delete/free on the pointer during global setup/teardown, but it's optional.

*Tags: `memory`, `aegp`, `plugin-architecture`*

---

## Can multiple effects be registered within a single DLL for After Effects plugins?

Yes, according to Alex Bizeau from maxon, it is possible to call multiple register effect functions in one DLL. This approach could be used to create a bootstrapper similar to what was done for OFX, allowing a single DLL to load multiple effects and reduce code duplication across OFX, After Effects, and AVX plugins.

*Tags: `pipl`, `aegp`, `plugin-architecture`, `deployment`, `build`*

---

## Is PiPL still required for After Effects plugins, or can plugins be registered without it?

PiPL is still required for After Effects plugins. While the function PF_Register_effect_ext2 was added two versions ago (in AE 23) to allow registration with URL info, this does not eliminate the need for PiPL/rsrc files. Plugins without PiPL cannot be found by After Effects. The new entry point functionality has advantages, but PiPL is still mandatory, though some of its values can be overwritten in code. However, OFX (which is partly based on the AE SDK) successfully eliminated PiPL entirely, making it more flexible for multi-plugin deployment.

*Tags: `pipl`, `aegp`, `registration`, `plugin-architecture`*

---

## What are the advantages of writing host and plugin format-independent code for multiple platforms?

Writing host and plugin format-independent code frees up significant resources by allowing the processing code to be written once and only occasionally requiring wrapper project updates. This approach has proven effective across multiple platforms including After Effects, Premiere Pro, OFX, Nuke, and Frei0r. For example, when porting the GMIC plugin suite (containing ~2000 plugins) to OFX, all plugins fit in a single file, whereas for AE/Premiere Pro, 2000 tiny individual loader plugins had to be created. The OFX SDK has also benefited from this approach by eliminating detours and inconveniences present in the AE SDK, such as PiPL requirements.

*Tags: `cross-platform`, `ofx`, `plugin-architecture`, `aegp`, `premiere`*

---

## How can you pass data from an AEGP plugin to an effects plugin in After Effects?

You need to create two separate binaries: an AEGP plugin and an FX plugin. Communicate between them using a C interface that pushes the string into a queue. This allows data to be shared across the two plugin types.

*Tags: `aegp`, `plugin-architecture`, `plugin-communication`, `c-interface`*

---

## Is there a way to embed a CEP panel directly inside a plugin rather than as a separate extension?

No, After Effects requires CEP panels to be placed in designated CEP folders. However, two partial solutions exist: (1) Place a shortcut in the CEP folder pointing to your effect's folder to organize code elsewhere, or (2) Create an empty CEP that calls a function like 'FetchPanelCode()' defined in After Effects, which returns an HTML string that the panel can then evaluate dynamically.

*Tags: `cep`, `ui`, `plugin-architecture`*

---

## How can you keep UI and render threads in sync when automatically applying multiple effects in CC2015+?

When using Sequence Setup to automatically add effects to a layer, the UI and render copies of the project can become out of sync in CC2015+, causing verification failures and missing data effects. The solution is to use an AEGP (After Effects General Plug-in) with an idle hook instead of modifying the layer during Sequence Setup. The effect can set a flag when it's applied, and the AEGP's idle hook will then process the pending operation to add the additional effects. The AEGP can be bundled in the same binary as the effect plugins, but the AEGP's PiPL must be listed first in the bundle.

*Tags: `aegp`, `pipl`, `ui`, `smart-render`, `threading`, `plugin-architecture`*

---

## Is there a reference in the SDK documentation about bundling AEGP plugins with effect plugins?

According to the SDK guide, if you want to add an AEGP plug-in to the same binary as an effect plugin, the PiPL of the AEGP must be the first one in the resource list. This allows you to avoid creating a separate installer while still gaining access to AEGP features like idle hooks for proper effect synchronization.

*Tags: `aegp`, `pipl`, `sdk`, `plugin-architecture`, `reference`*

---

## Should arbitrary parameter data be stored in a single struct for all arb params or separate structs per parameter?

You can use either approach, but a single generalized arb handler that is reusable across multiple arb params is more efficient. Store the data in AE rather than in the handler class itself. During global setup, create and store the handlers in the global data structure, then pass pointers to them during param setup. Keep handlers locked throughout the session using placed new allocation. The key is that except for the 'create new' function, most handler functions remain the same across different arb types.

```cpp
// C++ style approach with reusable handler
// Store handler pointer in arb definition
// Handler class with virtual methods for each arb type
class ArbHandler {
public:
  virtual PF_Err CreateDefault(PF_Handle *arbH) = 0;
  virtual PF_Err Handle(PF_Cmd cmd, PF_Handle arbH) = 0;
};

// During global setup:
ArbHandler *handler = new ArbHandler();
ArbHandler **handlerH = (ArbHandler**)AE_Allocate(sizeof(ArbHandler*));
*handlerH = handler;
// Lock handle and store in global data
```

*Tags: `arb-data`, `params`, `plugin-architecture`, `c++`, `aegp`*

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
