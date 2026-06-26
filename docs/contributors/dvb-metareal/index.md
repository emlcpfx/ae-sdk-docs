# dvb metareal

**2 contributions** to AE SDK community knowledge.

Top topics: `memory-leak`, `compute-cache`, `mac`, `export`, `aegp-memory-suite`, `threading`, `lua`, `python`, `scripting`, `embedding`

---

## What is the AEGP memory leak issue on Mac during export with Compute Cache?

When using AEGP_MemorySuite to create memHandles during ComputeCache threads, the memory may not be properly freed during export on Mac (works fine on Windows). The used memory grows past the RAM limit. Using new/delete instead of AEGP memHandles avoids the leak. The issue is that memHandles allocated in one thread and freed in another may not actually release memory during the render thread. The AEGP tools report the memory as freed, but virtual memory keeps growing.

*Source: adobe-plugin-devs · 2023-09-07 · Tags: `memory-leak`, `compute-cache`, `mac`, `export`, `aegp-memory-suite`, `threading` · [View in Q&A](../qa/memory-leak/)*

---

## Is Lua or Python a better choice for embedding a scripting language inside an After Effects plugin?

Lua is a significantly better fit for embedding inside an AE plugin. It is very easy to compile in and has a small footprint. Python is much clunkier to embed because it requires installing libraries on the user's system (asking permission to install packages), which is intrusive. While there may be cleaner ways to embed Python, Lua is far simpler to integrate. Both require similarly tedious C binding work without some kind of automation or code-generation tooling. dvb metareal shipped 'omino lua' as a script-driven drawing plugin using this approach.

*Source: adobe-plugin-devs · 2023-08-16 · Tags: `lua`, `python`, `scripting`, `embedding`, `c-bindings`, `plugin-architecture` · [View in Q&A](../qa/lua/)*

---
