# Vulkan Cross-Plugin Crash Fix (NVIDIA Driver)

**Severity:** Crash (Access Violation) during MFR rendering
**Status:** Fixed with cross-plugin named mutex

## Symptom

After Effects crashes during MFR (Multi-Frame Rendering) when multiple Vulkan plugins are in the same comp. Crash dump shows:

```
INVALID_POINTER_READ_c0000005_nvoglv64.dll!Unknown
nvoglv64!vkGetInstanceProcAddr+0x67f99: mov edi, [rbx+20h]  (rbx=0)
```

Stack trace shows one plugin triggering another via AE's render graph (e.g., PluginB checks out a layer that has PluginA applied):
```
nvoglv64!vkGetInstanceProcAddr+0x67f99   <- null deref inside NVIDIA driver
nvoglv64!vkGetInstanceProcAddr+0xafeca
PluginA!PluginDataEntryFunction2+0x3a55d  <- PluginA Vulkan render path
PluginA+0xca44
PluginA!EffectMain+0x446                  <- SmartRender dispatch
...
PluginB+0xe323                            <- PluginB checked out a layer
PluginB!EffectMain+0x6c9                  <- PluginB also rendering via Vulkan
```

Crash is deterministic (same Failure.Hash every time) but on different MFR threads.

## Root Cause

**The NVIDIA Vulkan driver has shared internal state across multiple `VkDevice` instances in the same process.** Each plugin creates its own independent `VkInstance` and `VkDevice`, which the Vulkan spec allows. However, when two plugins make concurrent Vulkan API calls (even on separate devices), the NVIDIA driver's internal bookkeeping gets corrupted, leading to a null pointer dereference.

This happens during MFR because:
1. AE renders frames on multiple threads simultaneously
2. PluginB's SmartRender uses Vulkan for its GPU path
3. PluginB checks out a layer via `PF_CHECKOUT_PARAM` / `FLTp_Node::CheckoutLayerRender`
4. That layer has PluginA applied, so AE renders PluginA on the same or different thread
5. PluginA's render function also uses Vulkan
6. Both plugins are now making Vulkan calls concurrently on separate `VkDevice` instances
7. NVIDIA driver's shared state is corrupted -> crash at `[rbx+0x20]` where `rbx=0`

### Why per-plugin mutexes didn't fix it

Each plugin already had internal mutexes (`m_renderMutex`, `m_queueMutex`, `m_descriptorPoolMutex`) to serialize Vulkan calls within that plugin. These prevent MFR threads from racing within the same plugin, but they do NOT prevent Plugin A's Vulkan calls from racing with Plugin B's Vulkan calls. The plugins are separate `.aex` files with no shared state.

### What was tried and failed

1. **Descriptor pool mutex on free** - Added `m_descriptorPoolMutex` around `vkFreeDescriptorSets` in cleanup. Did NOT fix the crash (correct fix but not the root cause).
2. **vkInvalidateMappedMemoryRanges ordering** - Fixed spec violation where invalidate was called before map. Did NOT fix the crash.
3. **Global per-plugin render mutex** - Serialized all Vulkan calls within a single plugin with a single `std::mutex`. Did NOT fix the crash (the conflict is cross-plugin, not intra-plugin).
4. **Comprehensive Vulkan error checking** - Added return value checks on `vkQueueSubmit`, `vkQueueWaitIdle`, `vkBindBufferMemory`, `vkBindImageMemory`, `vkBeginCommandBuffer`, `vkEndCommandBuffer`, plus a `m_deviceLost` flag. Good defensive coding but did NOT fix the crash.

## Fix: Cross-Plugin Named Mutex

All Vulkan plugins from the same vendor now acquire a **Windows named mutex** before any Vulkan render work. Since all plugins are loaded into the same `AfterFX.exe` process, the named mutex serializes Vulkan access across all of them.

### Implementation

Each plugin has a RAII lock class in its Vulkan renderer `.cpp` file:

```cpp
#ifdef _WIN32
class CrossPluginVulkanLock {
public:
    CrossPluginVulkanLock() : m_mutex(NULL) {
        m_mutex = CreateMutexA(NULL, FALSE, "Local\\MyCompany_VulkanGlobalMutex");
        if (m_mutex) WaitForSingleObject(m_mutex, INFINITE);
        // Note: Consider using a timeout instead of INFINITE to prevent
        // deadlocks if a plugin crashes while holding the mutex.
    }
    ~CrossPluginVulkanLock() {
        if (m_mutex) { ReleaseMutex(m_mutex); CloseHandle(m_mutex); }
    }
    CrossPluginVulkanLock(const CrossPluginVulkanLock&) = delete;
    CrossPluginVulkanLock& operator=(const CrossPluginVulkanLock&) = delete;
private:
    HANDLE m_mutex;
};
#endif
```

The lock is acquired at the entry of each plugin's main GPU render function:

```cpp
// At the top of every Vulkan render function:
CrossPluginVulkanLock crossPluginLock;  // Serializes across all plugins
std::lock_guard<std::mutex> renderLock(m_renderMutex);  // Serializes within this plugin
```

### Files Changed

Each plugin that uses Vulkan needs the `CrossPluginVulkanLock` added at the top of its GPU render functions. For example:

| Plugin | File | Function(s) |
|--------|------|-------------|
| **PluginA** | `src/gpu/VulkanRenderer.cpp` | All Vulkan render entry points |
| **PluginB** | `src/gpu/VulkanRenderer.cpp` | All Vulkan render entry points |
| **PluginC** | `src/gpu/VulkanBeauty.cpp` | `Process` |
| **PluginD** | `src/gpu/VulkanBeauty.cpp` | `ProcessAllZones` |

### Mutex Name

```
Local\MyCompany_VulkanGlobalMutex
```

`Local\` prefix means it's per-session (not global across terminal services sessions). All plugins from the same vendor MUST use this exact name.

### Performance Impact

- MFR CPU work (scene loading, matrix math, pixel format conversion) still runs in parallel across threads
- Only Vulkan GPU work is serialized, which was already limited by the single GPU
- The GPU was already the bottleneck, so serialization has minimal practical impact on render time

## Additional Fixes Applied

These were applied during the investigation and are good defensive improvements even though they weren't the root cause:

### 1. Descriptor pool mutex on free
All render function cleanup sections now lock `m_descriptorPoolMutex` before calling `vkFreeDescriptorSets`.

### 2. vkInvalidateMappedMemoryRanges ordering
`DownloadFromImage()` now calls `vkMapMemory` before `vkInvalidateMappedMemoryRanges` (was incorrectly reversed).

### 3. Comprehensive Vulkan error checking
- `vkQueueSubmit`, `vkQueueWaitIdle`, `vkEndCommandBuffer`, `vkBeginCommandBuffer` return values checked
- `vkBindBufferMemory`, `vkBindImageMemory` return values checked
- `m_deviceLost` atomic flag prevents cascade crashes after `VK_ERROR_DEVICE_LOST`
- All cleanup sections skip Vulkan destroy calls when device is lost

### 4. Crash trace logging
File-based logging (controlled by a `VULKAN_CRASH_TRACE` define) that records thread ID, timestamp, and render function entry/exit for crash investigation.

## Lessons Learned

1. **Multiple `VkDevice` instances in one process are NOT safe on NVIDIA.** The Vulkan spec allows it, but the NVIDIA driver (nvoglv64.dll v32.0.15.7700) has shared internal state that requires external synchronization. This is an NVIDIA driver bug, but we must work around it.

2. **Crash dumps can be misleading.** The crash appeared to be a race condition within a single plugin (consistent Failure.Hash, different threads). The real issue was cross-plugin contention that only manifested when the render graph triggered multiple Vulkan plugins.

3. **Named mutexes are the simplest cross-plugin synchronization.** Each plugin is a separate `.aex` DLL with no shared memory. Windows named mutexes allow synchronization without any shared code or DLL dependency.

4. **Any new plugin that uses Vulkan from the same vendor MUST acquire the named mutex before making Vulkan API calls.** Copy the `CrossPluginVulkanLock` class and add it at the top of the render function.
