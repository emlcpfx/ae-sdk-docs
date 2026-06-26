# Q&A: plugin-communication

**7 entries** tagged with `plugin-communication`.

---

## Is there a way for a CEP panel to communicate with a C++ plugin?

There is no direct communication mechanism between CEP and C++ plugins. However, indirect communication is possible through other means such as using ExecuteScript to trigger commands.

*Tags: `cep`, `scripting`, `plugin-communication`*

---

## Can a C++ plugin detect if a CEP panel is open or bring it into focus?

A C++ plugin cannot directly check if a CEP panel is open or bring it into focus through standard plugin APIs.

*Tags: `cep`, `ui`, `plugin-communication`*

---

## What is a reliable method to pass data from an effect plugin to a text layer?

An AEGP plugin using an idle hook is the best approach. Each frame, push string results into a queue, and periodically process the queue in the idle hook. This requires two binaries (AEGP plugin and FX plugin) communicating through a C interface. Alternatively, you can encode data into dead pixels in your render and use sampleImage expressions to retrieve those pixels and set source text, though this has caveats with render format reliability.

*Tags: `aegp`, `text-layer`, `plugin-communication`, `render-loop`, `idle-hook`*

---

## How can you pass data from an AEGP plugin to an effects plugin in After Effects?

You need to create two separate binaries: an AEGP plugin and an FX plugin. Communicate between them using a C interface that pushes the string into a queue. This allows data to be shared across the two plugin types.

*Tags: `aegp`, `plugin-architecture`, `plugin-communication`, `c-interface`*

---

## How can a C++ plugin communicate with a CEP panel in After Effects?

C++ plug-ins and CEP panels can communicate using Vulcan (CSXS Events). However, there is no official Vulcan implementation in the C++ SDK. As a workaround, developers have used JSX scripts with AEGP_ExecuteScript to dispatch Vulcan events from plugin to plugin, and invisible checkboxes in the plugin UI to catch messages from CEP back to the plugin.

*Tags: `cep`, `aegp`, `scripting`, `plugin-communication`*

---

## How can you save and restore arbitrary data from a 3rd party plugin's property without using presets?

You can use the AEGP (After Effects General Plugin) approach: have an effect plugin communicate with an AEGP via a special suite (see "checkout" and "sweetie" samples). When the effect needs to apply changes, it sends data to the AEGP and stores it there without executing changes immediately. Let the effect finish execution and return. Then, during the AEGP's idle_hook call, check for messages from the effect and execute the changes after the effect is no longer in mid-call. This avoids crashes from modifying the scene while an effect is operating on it. Alternatively, you can bundle a preset file within your plugin and apply it via applyPreset() with a file path, avoiding the need to keep presets in Adobe's preset directory.

```cpp
var thePreset = File("C:/Program Files/Adobe/Adobe After Effects CS5.5/Support Files/Presets/Behaviors/wigglerama.ffx");
theLayer.applyPreset(thePreset);
```

*Tags: `aegp`, `arb-data`, `params`, `plugin-communication`*

---

## How can you pass platform-specific data from an effect plugin to an AEGP if PF_GET_PLATFORM_DATA() is unavailable in AEGPs?

Since AEGPs do not receive a valid in_data object and cannot use PF_GET_PLATFORM_DATA(), have the effect plugin obtain the platform-specific data it needs and pass that data directly to the AEGP when it sends it a message. The AEGP can then use this pre-obtained data without needing to call PF_GET_PLATFORM_DATA() itself. Alternatively, the effect can store the data in hidden properties or session data that the AEGP can query.

*Tags: `aegp`, `cross-platform`, `plugin-communication`*

---
