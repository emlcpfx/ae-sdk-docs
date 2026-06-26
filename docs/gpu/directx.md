# Directx

> 3 Q&As · source: AE plugin dev community Discord

### Is DirectX rendering already implemented in After Effects production, and has anyone made a plugin using it?

DirectX rendering was not widely available for plugins as of the conversation date. Adobe replaced the UI from OpenGL to DX12 in 2021, and the latest SDK has an official flag for it, but it's likely not yet ready to expose to plugins. It may be added for Windows on ARM device support, where DirectX is the first-party option. Adobe has not been supporting Vulkan, possibly because the version of AFX that passes DX12 handles to plugins hasn't been released yet.

*Tags: `apple-silicon`, `directx`, `gpu`, `sdk`, `windows`*

---

### How can DirectX compute be enabled for debugging in After Effects?

DirectX compute can be enabled by setting the 'forcedirectxcompute' flag in the console panel within AE settings. However, a full computer restart is required for the setting to take effect; restarting AE alone is not sufficient. Once properly configured, it becomes possible to debug DirectX for future productions.

*Tags: `debugging`, `directx`, `gpu`, `windows`*

---

### Is DirectX rendering currently implemented in After Effects production, and how can it be enabled for plugin development?

DirectX rendering support was added to the After Effects SDK with an official flag, though it appears to be in early stages. Adobe replaced the UI from OpenGL to DX12 in 2021. To enable DirectX compute for debugging and testing, use the 'forcedxcompute' setting in the After Effects console panel under settings, but note that a full computer restart (not just AE restart) is required for the setting to take effect. The feature is likely being developed primarily for Windows on ARM device support, as those devices have first-party support for DX12. Adobe has not yet officially supported Vulkan exposure to plugins.

*Tags: `apple-silicon`, `debugging`, `directx`, `gpu`, `windows`*

---
