# Q&A: cep

**20 entries** tagged with `cep`.

---

## What is plugplug.DLL and how can you use it from C++ plugins?

plugplug.DLL is the gateway to communicate with CEP (Common Extensibility Platform) from C++. It allows sending events between C++ plugins and CEP panels. You can reverse-engineer the DLL's exports using depends.exe and reference the Illustrator SDK sample as a guide (it's not 1:1 but close enough). A GitHub repo with a working implementation is available: https://github.com/Trentonom0r3/AE-SDK-CEP-UTILS

*Contributors: [**tlafo**](../contributors/tlafo/), [**spigon**](../contributors/spigon/) · Source: adobe-plugin-devs · 2024-05-02 · Tags: `plugplug`, `cep`, `inter-process-communication`, `dll`*

---

## How can I trigger a plugin refresh in Premiere when CEP updates hidden parameters, since Premiere doesn't treat script interactions as user actions like AE does?

There is no known workaround for this. Premiere does not consider script interactions as user actions, so PF_UserChangedParam won't be triggered from CEP. A practical workaround is to use external dependencies watching for file changes (though it's not very reactive) or add a button parameter to force a manual refresh.

*Contributors: [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2022-05-31 · Tags: `premiere`, `cep`, `script-interaction`, `parameter-update`, `workaround`*

---

## What is UXP for Premiere Pro and how does it relate to CEP panel migration?

UXP (Unified Extensibility Platform) for Premiere Pro entered public beta in December 2024. It is the successor to CEP for building panels and extensions. Initially it is a web-based development environment, but a hybrid UXP+C++ approach is planned. For C++ computational tasks without the hybrid mode, you would need a service with IPC. The Premiere UXP beta is currently limited in functionality but expected to expand. Plugin developers with existing CEP panels should begin evaluating migration. Resources include the Adobe Creative Cloud Developer forums and the Hyperbrew blog (hyperbrew.co/blog/premiere-pro-uxp-beta). Bolt UXP (hyperbrew.co/resources/bolt-uxp/) is a tool to help with UXP development.

*Contributors: [**Erin F.**](../contributors/erin-f/), [**Justin**](../contributors/justin/), [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2024-12-06 · Tags: `premiere`, `uxp`, `cep`, `panel`, `migration`, `hybrid-cpp`, `beta`*

---

## How can a C++ plugin display toast notifications next to the mouse cursor in After Effects?

This can be achieved with a C++ plugin working alongside CEP extensions. Scott Black's Control Groups extension uses this technique to display toast-like notifications that appear alongside the mouse cursor. This is a more heavy-handed solution if your only goal is toast notifications, but useful if you already have a C++ plugin component.

*Contributors: [**Scott Black**](../contributors/scott-black/) · Source: aescripts discord · 2025-12-05 · Tags: `toast-notification`, `ui`, `cep`, `cpp-plugin`, `mouse-cursor`*

---

## Is there a way for a CEP panel to communicate with a C++ plugin?

There is no direct communication mechanism between CEP and C++ plugins. However, indirect communication is possible through other means such as using ExecuteScript to trigger commands.

*Tags: `cep`, `scripting`, `plugin-communication`*

---

## Can a CEP panel change parameter values of a C++ plugin?

CEP cannot directly change plugin parameter values. Parameter changes must be made through the plugin's own UI or other indirect mechanisms.

*Tags: `cep`, `params`, `ui`*

---

## Can a C++ plugin detect if a CEP panel is open or bring it into focus?

A C++ plugin cannot directly check if a CEP panel is open or bring it into focus through standard plugin APIs.

*Tags: `cep`, `ui`, `plugin-communication`*

---

## How can a C++ plugin open a CEP panel?

A C++ plugin can open a CEP panel by using ExecuteScript to find the Command ID of the extension by name, then executing that command through the execute command suite.

*Tags: `cep`, `scripting`, `ui`*

---

## How can C++ plugin code communicate with a CEP panel?

C++ can open a CEP by executing scripting that calls the CEP. For CEP to plugin communication, store values to transfer in JavaScript memory using global variables. For large data sizes that are expensive in memory, use a temp file instead. Alternatively, set a hidden checkbox that is activated by script to catch the value. Another solution is to run a hidden panel as a server that receives requests from the plugin and adjusts values in CEP.

*Tags: `cep`, `scripting`, `params`, `memory`*

---

## How can C++ check if a CEP panel is opened?

C++ can check if a CEP is opened using the TCP protocol or by using a hidden checkbox that the CEP activates via scripting.

*Tags: `cep`, `scripting`, `debugging`*

---

## How can you trigger parameter updates in Premiere from CEP script interactions when PF_UserChangedParam doesn't work?

When working with AE/Premiere plugins updated by CEP backend data, using a hidden parameter updated via script that calls PF_UserChangedParam works well in After Effects, but Premiere does not recognize script interactions as user actions. Alternative approaches include monitoring external file dependencies for reactivity (though this can be slow), or implementing a manual button to force refresh as a workaround.

*Tags: `premiere`, `cep`, `params`, `scripting`, `cross-platform`*

---

## How long will the CEP engine remain available in Premiere Pro before the transition to UXP is required?

This question was asked but not answered in the conversation. Erin F. shared information about UXP for Premiere Pro entering public beta and encouraged developers with CEP panels to communicate their migration needs to the Premiere Pro team, but did not provide a specific timeline for CEP deprecation.

*Tags: `premiere`, `cep`, `uxp`, `migration`*

---

## What is the status of UXP support for Premiere Pro and where can I learn more about it?

UXP for Premiere Pro is now in public beta. Developers with existing CEP panels should review the announcement and communicate their migration requirements to the Premiere Pro team. More information is available at the Creative Cloud Developer Forums: https://forums.creativeclouddeveloper.com/t/uxp-now-available-in-premiere-pro-beta/8795

*Tags: `premiere`, `uxp`, `cep`, `deployment`, `migration`*

---

## Is there a way to communicate between a CEP panel and a C++ plugin in After Effects?

Direct communication between CEP and C++ plugins is not possible. However, indirect communication can be achieved: a C++ plugin can open a CEP panel by using the ExecuteScript to find the Command ID (menu) by the extension name, then executing that command through the command suite. CEP cannot directly change plugin parameter values, and the plugin cannot directly check if a CEP panel is open or bring it into focus.

*Tags: `cep`, `c++`, `scripting`, `ui`, `aegp`*

---

## How can C++ plugins communicate with CEP panels to exchange parameter values?

There are several approaches: (1) CEP can change parameter values via scripting, and C++ can open a CEP by executing a script that calls it; (2) C++ can check if CEP is open using TCP protocol or a hidden checkbox that CEP activates via scripting; (3) Use a hidden panel running as a server that receives requests from the plugin and adjusts values in CEP; (4) For CEP-to-plugin communication, store values in JavaScript memory using global variables, or use temporary files if memory size is a concern; (5) In AE-only scenarios, set a hidden checkbox activated by script to catch the value.

*Tags: `cep`, `scripting`, `params`, `ui`, `communication`*

---

## How can you trigger parameter updates in Premiere Pro from a CEP script when PF_userChangedParam doesn't work?

In After Effects, using a hidden parameter updated by a script calling PF_userChangedParam works well for triggering updates. However, Premiere Pro does not consider script interactions as user actions, so this approach is not reactive. Alternative solutions include: monitoring external file dependencies for changes (though this is less reactive), or setting up a button to force manual refresh of the plugin parameters.

*Tags: `premiere`, `scripting`, `params`, `cep`*

---

## How long will the CEP engine remain available in Premiere Pro before it's deprecated?

The conversation mentions that UXP for Premiere Pro is now in public beta and encourages CEP panel developers to migrate, but no specific timeline for CEP deprecation was provided in the response.

*Tags: `premiere`, `cep`, `uxp`, `deployment`, `cross-platform`*

---

## How can a C++ plugin communicate with a CEP panel in After Effects?

C++ plug-ins and CEP panels can communicate using Vulcan (CSXS Events). However, there is no official Vulcan implementation in the C++ SDK. As a workaround, developers have used JSX scripts with AEGP_ExecuteScript to dispatch Vulcan events from plugin to plugin, and invisible checkboxes in the plugin UI to catch messages from CEP back to the plugin.

*Tags: `cep`, `aegp`, `scripting`, `plugin-communication`*

---

## Is there a way to embed a CEP panel directly inside a plugin rather than as a separate extension?

No, After Effects requires CEP panels to be placed in designated CEP folders. However, two partial solutions exist: (1) Place a shortcut in the CEP folder pointing to your effect's folder to organize code elsewhere, or (2) Create an empty CEP that calls a function like 'FetchPanelCode()' defined in After Effects, which returns an HTML string that the panel can then evaluate dynamically.

*Tags: `cep`, `ui`, `plugin-architecture`*

---

## How can an AEGP communicate with a CEP/ScriptUI panel to pass data back and forth?

You can use AEGP_ExecuteScript() to send data from the AEGP to your JavaScript panel, and retrieve data from the JavaScript side via the returned value. For layer selection detection, you can use comp.selectedLayers and comp.selectedProperties from JavaScript with a scheduled task to check for selection changes at intervals. Alternatively, use idle_hook in the AEGP to check periodically and execute scripts via AEGP_ExecuteScript(). For deeper integration, there is a JavaScript SDK that allows you to build an AEGP that gets called to execute custom JavaScript APIs.

*Tags: `aegp`, `scripting`, `ui`, `cep`*

---
