# Vulkan Hybrid Rendering Architecture for After Effects Plugins

## Table of Contents
1. [Overview](#overview)
2. [Why Vulkan Instead of AE GPU API](#why-vulkan-instead-of-ae-gpu-api)
3. [Architecture Design](#architecture-design)
4. [Thread Safety](#thread-safety)
5. [Performance Optimization](#performance-optimization)
6. [CPU vs GPU Rendering Paths](#cpu-vs-gpu-rendering-paths)
7. [Building and Dependencies](#building-and-dependencies)
8. [Porting Guide](#porting-guide)
9. [Troubleshooting](#troubleshooting)

---

## Overview

This approach uses a **Vulkan Hybrid Rendering** architecture that bypasses After Effects' GPU API entirely. The architecture:

- Supports masks (via CPU checkout of unmasked source)
- Thread-safe (handles Play mode with multiple render threads)
- Fast (3.4x faster than pure CPU at 4K resolution)
- Full precision (maintains 32-bit float RGBA throughout pipeline)

### Key Performance (4K Resolution, 202 Triangles)
```
CPU Path:  ~700ms
GPU Path:  ~195ms (3.4x faster)

GPU Breakdown:
- Buffer Upload:   0.3ms   (0.2%)
- Input Upload:    68ms    (35%)
- Shader Execute:  53ms    (27%)
- Output Download: 75ms    (38%)
```

---

## Why Vulkan Instead of AE GPU API

### The Problem with AE's GPU API

After Effects provides a GPU API (`PF_GPUDeviceSuite`, `AEGP_GPUWorldSuite`), but it has a critical limitation:

**When a mask is present, you cannot get the unmasked source layer.**

> **Note:** The behavior of mask accessibility in GPU rendering may vary by SDK version. With SmartFX, you may be able to request unmasked input in PreRender.

```cpp
// AE GPU API limitation:
PF_CheckoutResult checkout;
ERR(suites.GPUDeviceSuite->Checkout(...));
// Returns masked layer - can't get unmasked source!
```

For effects that need to sample the unmasked input (like camera projection unwrap), this is a dealbreaker.

### The Hybrid Solution

```
1. CPU Checkout -> Get unmasked source via PF_CHECKOUT_PARAM
2. Upload to GPU -> Vulkan staging buffer (68ms)
3. Render on GPU -> Vulkan compute shader (53ms)
4. Download result -> Cached staging buffer (75ms)
5. Return to AE  -> CPU memory as expected
```

This maintains mask support while leveraging GPU acceleration.

---

## Architecture Design

### High-Level Flow

```
+-------------------------------------------------------------+
|                    After Effects                              |
|  PF_Cmd_SMART_PRE_RENDER -> PF_Cmd_SMART_RENDER              |
+------------------------+------------------------------------+
                         |
                         v
         +-------------------------------+
         |  MyPlugin.cpp                 |
         |  (Main plugin logic)          |
         +-----------+-------------------+
                     |
          +----------+----------+
          |                     |
      CPU Path              GPU Path
          |                     |
          v                     v
+-----------------+   +----------------------+
| GenericRenderer |   | VulkanHybridRenderer |
| (CPU triangle   |   | (Compute shader)     |
|  iteration)     |   |                      |
+-----------------+   +----------------------+
```

### Core Components

#### 1. VulkanHybridRenderer Class
**File:** `src/gpu/VulkanRenderer.cpp`

Manages Vulkan device, pipelines, and rendering operations.

```cpp
class VulkanHybridRenderer {
    // Vulkan device objects (owned by plugin, not AE)
    VkInstance m_instance;
    VkDevice m_device;
    VkQueue m_queue;

    // Per-thread resources for thread safety
    std::unordered_map<std::thread::id, PerThreadResources> m_threadResources;
    std::mutex m_queueMutex;
    std::mutex m_descriptorPoolMutex;

    // Initialization state (atomic for thread safety)
    std::atomic<bool> m_initialized;
};
```

#### 2. Compute Shaders
**File:** `shaders/my_shader.comp` (GLSL) -> `src/gpu/my_shader_spv.h` (SPIR-V)

Embedded as byte arrays in the plugin binary for zero external dependencies.

```glsl
#version 450
layout(local_size_x = 16, local_size_y = 16) in;

layout(binding = 0, rgba32f) uniform readonly image2D inputTexture;
layout(binding = 1, rgba32f) uniform writeonly image2D outputTexture;
layout(std430, binding = 2) readonly buffer VertexBuffer { Vertex vertices[]; };
layout(std430, binding = 3) readonly buffer TriangleBuffer { Triangle triangles[]; };
layout(std140, binding = 4) uniform UniformBuffer { /* matrices, params */ };

void main() {
    // Per-pixel triangle intersection and texture sampling
}
```

#### 3. Integration Points
**File:** `src/main/MyPlugin.cpp`

```cpp
// GPU render path in SmartRender
if (g_vulkanHybridRenderer && g_vulkanHybridRenderer->IsAvailable()) {
    // Load scene data
    LoadScene(params, scene);

    // Checkout unmasked input (CPU memory)
    PF_CHECKOUT_PARAM(..., &unmasked_input_param);

    // Render via Vulkan
    g_vulkanHybridRenderer->RenderHybridUnwrap(
        input_worldP,      // CPU input
        output_worldP,     // CPU output
        gpu_vertices,
        gpu_triangles,
        view_matrix,
        proj_matrix,
        render_params);
}
```

---

## Thread Safety

### The Problem

After Effects uses **multi-threaded rendering** during Play mode:
- Multiple frames rendered in parallel
- Same plugin instance called from different threads
- Shared Vulkan resources = **race conditions**

### The Solution: Per-Thread Command Pools + Fine-Grained Locking

#### 1. Per-Thread Command Pools

Vulkan command pools are **NOT thread-safe**. Solution: Each thread gets its own pool.

```cpp
// In VulkanRenderer.h:
struct PerThreadResources {
    VkCommandPool commandPool;  // Each thread has its own
};
std::unordered_map<std::thread::id, PerThreadResources> m_threadResources;
std::mutex m_threadResourcesMutex;  // Only for map access

// In VulkanRenderer.cpp:
VkCommandPool VulkanHybridRenderer::GetThreadCommandPool() {
    std::thread::id threadId = std::this_thread::get_id();

    // Fast path: check if pool exists
    {
        std::lock_guard<std::mutex> lock(m_threadResourcesMutex);
        auto it = m_threadResources.find(threadId);
        if (it != m_threadResources.end()) {
            return it->second.commandPool;
        }
    }

    // Slow path: create new pool for this thread
    VkCommandPool newPool = CreateCommandPool(...);
    {
        std::lock_guard<std::mutex> lock(m_threadResourcesMutex);
        m_threadResources[threadId].commandPool = newPool;
    }
    return newPool;
}
```

#### 2. Fine-Grained Locking Strategy

**Only lock during operations that MUST be serialized:**

```cpp
// BAD: Lock entire render (serializes everything)
PF_Err RenderHybridUnwrap(...) {
    std::lock_guard<std::mutex> lock(m_renderMutex);  // Blocks all threads!
    // ... entire render ...
}

// GOOD: Lock only critical sections
PF_Err RenderHybridUnwrap(...) {
    // No lock: Per-thread command pool
    VkCommandPool pool = GetThreadCommandPool();

    // No lock: Build command buffers in parallel
    VkCommandBuffer cmd = AllocateCommandBuffer(pool);
    RecordCommands(cmd);

    // Lock ONLY for queue submission (fast operation)
    {
        std::lock_guard<std::mutex> lock(m_queueMutex);
        vkQueueSubmit(m_queue, ...);
        vkQueueWaitIdle(m_queue);
    }
}
```

#### 3. Atomic Initialization Flag

Prevents race between render threads and shutdown:

```cpp
// In VulkanRenderer.h:
std::atomic<bool> m_initialized;  // Atomic prevents race with Shutdown()

// In render functions:
if (!m_initialized) {  // Safe: atomic load
    return PF_Err_UNRECOGNIZED_PARAM_TYPE;
}
```

### Critical Sections Requiring Locks

| Operation | Lock Required | Why |
|-----------|---------------|-----|
| Command buffer allocation | No | Per-thread pools |
| Command buffer recording | No | Independent command buffers |
| **Queue submission** | Yes | Vulkan spec requires serialization |
| **Descriptor allocation** | Yes | Shared descriptor pool |
| Device creation/destruction | Yes | Only happens once at init/shutdown |

---

## Performance Optimization

### 1. Cached Memory for Downloads (CRITICAL)

**The Problem:** Default Vulkan staging buffers use `HOST_COHERENT` memory:
```cpp
// Slow (uncached):
VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT
// Every CPU read goes directly to memory
// 460ms to download 127MB (276 MB/s)
```

**The Solution:** Use `HOST_CACHED` memory:
```cpp
// Fast (cached):
VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_CACHED_BIT
// CPU reads from cache
// Requires manual cache invalidation
// 75ms to download 127MB (1700 MB/s)
```

**Implementation:**
```cpp
// In DownloadFromImage():
PF_Err err = AllocateHostVisibleBuffer(
    imageSize,
    VK_BUFFER_USAGE_TRANSFER_DST_BIT,
    stagingBuffer,
    stagingMemory,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_CACHED_BIT);

// After GPU->staging copy, before CPU read:
VkMappedMemoryRange memRange = {};
memRange.sType = VK_STRUCTURE_TYPE_MAPPED_MEMORY_RANGE;
memRange.memory = stagingMemory;
memRange.offset = 0;
memRange.size = VK_WHOLE_SIZE;
vkInvalidateMappedMemoryRanges(m_device, 1, &memRange);  // Sync cache

// Now CPU reads are fast:
void* mapped;
vkMapMemory(m_device, stagingMemory, 0, imageSize, 0, &mapped);
memcpy(pixels, mapped, imageSize);  // Reads from cache
```

**Result:** Download time reduced from 460ms to 75ms (6x faster)

### 2. Optimal Image Tiling

Use `VK_IMAGE_TILING_OPTIMAL` for GPU-side images:
```cpp
VkImageCreateInfo imageInfo = {};
imageInfo.tiling = VK_IMAGE_TILING_OPTIMAL;  // Best for GPU compute
imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
imageInfo.usage = VK_IMAGE_USAGE_STORAGE_BIT | VK_IMAGE_USAGE_TRANSFER_DST_BIT;
```

LINEAR tiling allows direct CPU access but is slower for GPU operations.

### 3. Memory Property Selection

| Use Case | Memory Properties | When to Use |
|----------|-------------------|-------------|
| **Upload buffers** | `HOST_VISIBLE \| HOST_COHERENT` | CPU writes to GPU (one-way) |
| **Download buffers** | `HOST_VISIBLE \| HOST_CACHED` | GPU writes, CPU reads (must invalidate) |
| **GPU-only resources** | `DEVICE_LOCAL` | Images, vertex buffers (fastest GPU access) |

### 4. Persistent Resources vs Per-Frame

```cpp
// Create once, reuse across frames:
- VkDevice, VkQueue, VkCommandPool
- VkPipeline, VkDescriptorSetLayout, VkDescriptorPool
- Per-thread command pools

// Create per-frame, destroy after use:
- VkCommandBuffer (allocated from pool)
- VkDescriptorSet (allocated from pool)
- Staging buffers (vertex/triangle/uniform/image data)
```

### 5. Compute Workgroup Size

Match shader local size to GPU warp/wavefront size:
```glsl
layout(local_size_x = 16, local_size_y = 16) in;  // 256 threads per workgroup
```

Dispatch dimensions:
```cpp
uint32_t groupCountX = (output_width + 15) / 16;   // Round up
uint32_t groupCountY = (output_height + 15) / 16;
vkCmdDispatch(commandBuffer, groupCountX, groupCountY, 1);
```

---

## CPU vs GPU Rendering Paths

### When to Use Each Path

```cpp
// Decision logic in MyPlugin.cpp:
if (use_gpu && g_vulkanHybridRenderer && g_vulkanHybridRenderer->IsAvailable()) {
    // GPU Path: 3.4x faster, supports masks
    RenderGPU(...);
} else {
    // CPU Path: Fallback, more compatible
    RenderCPU(...);
}
```

### CPU Path (GenericRenderer)
**File:** `src/rendering/GenericRenderer.cpp`

**Advantages:**
- No GPU required
- Simpler code (standard C++)
- Easier to debug

**Disadvantages:**
- Slow (~700ms at 4K)
- Scales poorly with resolution

**Implementation:**
```cpp
// Per-pixel, per-triangle iteration
for (int y = 0; y < output_height; y++) {
    for (int x = 0; x < output_width; x++) {
        for (int t = 0; t < num_triangles; t++) {
            // Barycentric test, depth test, texture sample
        }
    }
}
```

### GPU Path (VulkanHybridRenderer)
**File:** `src/gpu/VulkanRenderer.cpp`

**Advantages:**
- Fast (~195ms at 4K)
- Scales well with resolution
- Parallelizes across GPU cores

**Disadvantages:**
- Requires Vulkan-capable GPU
- More complex code
- Data transfer overhead (upload 68ms + download 75ms)

**Implementation:**
```cpp
// Compute shader: 16x16 threads per workgroup, massively parallel
void main() {
    ivec2 pixelCoord = ivec2(gl_GlobalInvocationID.xy);
    // Each thread handles one pixel independently
}
```

### Data Flow Comparison

**CPU Path:**
```
Input (AE) -> [CPU Memory] -> GenericRenderer -> [CPU Memory] -> Output (AE)
             |                                                  |
             +---------------- All in CPU memory ---------------+
```

**GPU Path (Hybrid):**
```
Input (AE) -> [CPU Memory] -> Upload -> [GPU VRAM] -> Compute Shader -> [GPU VRAM] -> Download -> [CPU Memory] -> Output (AE)
             |                 68ms                    53ms                          75ms                        |
             +--------------------------------------- Hybrid Pipeline ---------------------------------------+
```

### Unified Scene Loading

Both paths use the same scene loader:
```cpp
struct SceneLoadParams {
    const char* scene_path;
    float time;
    int selected_geo_index;
    const char* selected_geo_path;
};

static bool LoadScene(const SceneLoadParams& params, LoadedScene& scene) {
    // Load mesh and camera (shared by both CPU and GPU)
    return scene.mesh_loaded && scene.camera_loaded;
}
```

This ensures **identical results** from CPU and GPU paths.

---

## Building and Dependencies

### Required Dependencies

#### 1. Vulkan SDK
**Download:** https://vulkan.lunarg.com/

**Version:** 1.2+ recommended

**What you need:**
- `vulkan.h` headers
- `vulkan-1.lib` import library (Windows)
- SPIR-V compiler (`glslangValidator` or `glslc`)

#### 2. CMake Configuration

**File:** `CMakeLists.txt`

```cmake
# Find Vulkan
find_package(Vulkan REQUIRED)

# Optional: CUDA (not required for Vulkan path)
option(ENABLE_CUDA "Enable CUDA GPU support" OFF)
option(ENABLE_VULKAN "Enable Vulkan GPU support" ON)

if(ENABLE_VULKAN)
    add_definitions(-DHAVE_VULKAN)

    # Link Vulkan
    target_link_libraries(${PROJECT_NAME}
        PRIVATE
        Vulkan::Vulkan
    )

    # Add Vulkan source files
    target_sources(${PROJECT_NAME} PRIVATE
        src/gpu/VulkanRenderer.cpp
        src/gpu/VulkanRenderer.h
    )
endif()
```

#### 3. Shader Compilation

**GLSL -> SPIR-V -> C++ Header:**

```bash
# Compile shader to SPIR-V binary
glslc shaders/my_shader.comp -o shaders/my_shader.spv

# Convert binary to C++ byte array
xxd -i shaders/my_shader.spv > src/gpu/my_shader_spv.h
```

**Result:** Embedded shader as C++ array:
```cpp
// File: src/gpu/my_shader_spv.h
unsigned char my_shader_spv[] = {
  0x03, 0x02, 0x23, 0x07, ...  // SPIR-V bytecode
};
unsigned int my_shader_spv_len = 27208;
```

### Static Linking Considerations

#### Vulkan-1.lib Linking

**Dynamic (Default):**
```cmake
target_link_libraries(${PROJECT_NAME} PRIVATE Vulkan::Vulkan)
# Links against vulkan-1.dll (must be present on user's system)
```

**Static Linking:**
Vulkan doesn't provide a static library. Instead, the loader (`vulkan-1.dll`) is a thin shim that dynamically loads the actual GPU driver.

**Best Practice:**
- Ship `vulkan-1.dll` with your plugin (redistribution allowed)
- Or require Vulkan SDK/drivers installed on user's system

#### After Effects SDK Linking

```cmake
# The AE SDK source files are compiled directly into your plugin.
# There is no separate AE SDK library to link.
target_sources(${PROJECT_NAME} PRIVATE
    ${AE_SDK_ROOT}/Util/AEGP_SuiteHandler.cpp
    ${AE_SDK_ROOT}/Util/MissingSuiteError.cpp
)
```

#### Full Static Build

To create a fully self-contained plugin:
```cmake
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded")  # Static CRT

target_link_libraries(${PROJECT_NAME} PRIVATE
    Vulkan::Vulkan          # Vulkan loader
)

# Embed shaders as byte arrays (no external files needed)
```

**Result:** Single `.aex` file with no external dependencies (except `vulkan-1.dll`).

### Platform-Specific Notes

#### Windows
- Use `/MT` (static CRT) or `/MD` (dynamic CRT) consistently
- Vulkan SDK installs to `C:\VulkanSDK\<version>`
- May need to add Vulkan bin dir to PATH for shader compiler

#### macOS
- Use `libvulkan.dylib` or MoltenVK for Metal translation
- Requires `VK_KHR_portability_subset` extension
- Different memory alignment requirements

---

## Porting Guide

### Step-by-Step: Adding Vulkan to Your Plugin

#### 1. Set Up Project Structure

```
your_plugin/
+-- src/
|   +-- gpu/
|   |   +-- VulkanRenderer.h        # Vulkan renderer header
|   |   +-- VulkanRenderer.cpp      # Vulkan renderer implementation
|   |   +-- your_shader_spv.h       # Your compiled shader
|   +-- main/
|       +-- YourPlugin.cpp          # Your plugin entry point
+-- shaders/
|   +-- your_shader.comp            # Your GLSL compute shader
+-- CMakeLists.txt
```

#### 2. Add Vulkan to CMake

```cmake
find_package(Vulkan REQUIRED)
add_definitions(-DHAVE_VULKAN)

target_sources(${PROJECT_NAME} PRIVATE
    src/gpu/VulkanRenderer.cpp
    src/gpu/VulkanRenderer.h
)

target_link_libraries(${PROJECT_NAME} PRIVATE Vulkan::Vulkan)
```

#### 3. Create Your Compute Shader

**File:** `shaders/your_shader.comp`

```glsl
#version 450
layout(local_size_x = 16, local_size_y = 16) in;

// Input/output images
layout(binding = 0, rgba32f) uniform readonly image2D inputImage;
layout(binding = 1, rgba32f) uniform writeonly image2D outputImage;

// Your parameters as uniform buffer
layout(std140, binding = 2) uniform Params {
    float someParam;
    int anotherParam;
} params;

void main() {
    ivec2 coord = ivec2(gl_GlobalInvocationID.xy);

    // Your per-pixel processing
    vec4 inputPixel = imageLoad(inputImage, coord);
    vec4 outputPixel = yourProcessing(inputPixel, params);
    imageStore(outputImage, coord, outputPixel);
}
```

**Compile it:**
```bash
glslc shaders/your_shader.comp -o shaders/your_shader.spv
xxd -i shaders/your_shader.spv > src/gpu/your_shader_spv.h
```

#### 4. Initialize Vulkan Renderer

**In GlobalSetup (PF_Cmd_GLOBAL_SETUP):**

```cpp
#ifdef HAVE_VULKAN
#include "gpu/VulkanRenderer.h"

VulkanHybridRenderer* g_vulkanRenderer = nullptr;

static PF_Err GlobalSetup(...) {
    // Initialize Vulkan
    g_vulkanRenderer = new VulkanHybridRenderer();
    PF_Err err = g_vulkanRenderer->Initialize();
    if (err != PF_Err_NONE) {
        delete g_vulkanRenderer;
        g_vulkanRenderer = nullptr;
        // Fall back to CPU-only mode
    }
    return PF_Err_NONE;
}
#endif
```

**In GlobalSetdown (PF_Cmd_GLOBAL_SETDOWN):**

```cpp
static PF_Err GlobalSetdown(...) {
    #ifdef HAVE_VULKAN
    // Shutdown Vulkan FIRST (before other cleanup)
    if (g_vulkanRenderer) {
        g_vulkanRenderer->Shutdown();
        delete g_vulkanRenderer;
        g_vulkanRenderer = nullptr;
    }
    #endif
    return PF_Err_NONE;
}
```

#### 5. Integrate into Render Function

**In SmartRender (PF_Cmd_SMART_RENDER):**

```cpp
static PF_Err SmartRender(...) {
    // 1. Checkout input layer (CPU memory)
    PF_LayerDef* input;
    PF_LayerDef* output;
    ERR(extra->cb->checkout_layer(..., &input));

    // 2. Choose GPU or CPU path
    #ifdef HAVE_VULKAN
    if (use_gpu && g_vulkanRenderer && g_vulkanRenderer->IsAvailable()) {
        // Prepare GPU data structures
        std::vector<YourVertexType> vertices;
        std::vector<YourParamType> params;

        // Call Vulkan renderer
        ERR(g_vulkanRenderer->RenderYourEffect(
            input,      // CPU input
            output,     // CPU output
            vertices,   // Your geometry
            params));   // Your parameters
    } else
    #endif
    {
        // CPU fallback
        RenderCPU(input, output, vertices, params);
    }

    return PF_Err_NONE;
}
```

#### 6. Customize VulkanRenderer for Your Effect

**Modify the render function to fit your needs:**

```cpp
// In VulkanRenderer.cpp:
PF_Err VulkanHybridRenderer::RenderYourEffect(
    PF_LayerDef* cpu_input,
    PF_LayerDef* cpu_output,
    const std::vector<YourData>& data,
    const YourParams& params)
{
    // 1. Upload input texture
    UploadToImage(cpu_input->data, ...);

    // 2. Upload your parameters as uniform buffer
    UniformData uniforms = { /* fill from params */ };
    UploadToBuffer(&uniforms, sizeof(uniforms), ...);

    // 3. Dispatch compute shader
    vkCmdBindPipeline(cmd, VK_PIPELINE_BIND_POINT_COMPUTE, m_yourPipeline);
    vkCmdDispatch(cmd, groupCountX, groupCountY, 1);

    // 4. Download result
    DownloadFromImage(outputImage, ..., cpu_output->data);

    return PF_Err_NONE;
}
```

#### 7. Add Benchmarking (Optional but Recommended)

```cpp
#include <chrono>

auto bench_start = std::chrono::high_resolution_clock::now();

// ... render operations ...

auto bench_end = std::chrono::high_resolution_clock::now();
auto total_ms = std::chrono::duration<double, std::milli>(bench_end - bench_start).count();

char bench_msg[256];
sprintf(bench_msg, "[BENCHMARK] YourEffect GPU: TOTAL=%.2fms | Res=%dx%d\n",
        total_ms, output->width, output->height);
DEBUG_LOG_BENCHMARK(bench_msg);
```

---

## Troubleshooting

### Common Issues

#### 1. Black/Transparent Output

**Symptoms:** GPU renders but output is completely black.

**Possible Causes:**
- Shader not finding any geometry
- Incorrect matrix transforms
- Wrong image layout transitions
- Descriptor bindings mismatch

**Debug:**
```glsl
// In shader, add debug visualization:
#define DEBUG_TRIANGLE_HITS 1

if (out_y == 10) {
    final_color = vec4(1.0, 0.0, 0.0, 1.0);  // Red stripe = shader running
}
```

#### 2. Crash During Play Mode

**Symptoms:** Plugin crashes when pressing Play in After Effects.

**Cause:** Race condition - multiple threads accessing shared Vulkan resources.

**Solution:** Ensure per-thread command pools and fine-grained locking:
```cpp
// Each thread gets its own command pool
VkCommandPool pool = GetThreadCommandPool();

// Lock only queue submission
{
    std::lock_guard<std::mutex> lock(m_queueMutex);
    vkQueueSubmit(...);
}
```

#### 3. Slow Download Performance

**Symptoms:** Download taking >400ms at 4K.

**Cause:** Using `HOST_COHERENT` memory (uncached).

**Solution:** Use `HOST_CACHED` with manual invalidation:
```cpp
// Allocate with cached memory
AllocateHostVisibleBuffer(...,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_CACHED_BIT);

// Before reading, invalidate cache
VkMappedMemoryRange range = { /* ... */ };
vkInvalidateMappedMemoryRanges(device, 1, &range);
```

#### 4. Vulkan Validation Errors

**Symptoms:** Console shows Vulkan validation layer errors.

**Enable validation layers for debugging:**
```cpp
VkInstanceCreateInfo createInfo = {};
const char* validationLayers[] = { "VK_LAYER_KHRONOS_validation" };
createInfo.enabledLayerCount = 1;
createInfo.ppEnabledLayerNames = validationLayers;
```

**Common errors:**
- `VUID-vkCmdDraw-None-02699`: Descriptor not updated -> Check `UpdateDescriptorSet`
- `VUID-VkImageMemoryBarrier-image-01207`: Invalid layout transition -> Check image layouts
- `VUID-vkAllocateMemory-pAllocateInfo-01713`: Out of memory -> Reduce resource allocation

#### 5. CPU and GPU Results Don't Match

**Symptoms:** GPU output looks different from CPU output.

**Possible Causes:**
- Floating-point precision differences
- Different matrix conventions (row-major vs column-major)
- Shader culling logic different from CPU

**Debug:**
```cpp
// Add comparison logging
LogUvMatchSample(cpu_result, gpu_result);  // Compare specific pixels

// Ensure matrices match
DEBUG_LOG_MATRIX(view_matrix);  // Check both paths use same data
```

#### 6. Plugin Won't Load / Initialize Fails

**Symptoms:** `Initialize()` returns error, Vulkan not available.

**Checklist:**
- Vulkan SDK installed?
- GPU supports Vulkan 1.2+?
- `vulkan-1.dll` present in system PATH?
- GPU drivers up to date?

**Fallback:**
```cpp
if (!g_vulkanRenderer->IsAvailable()) {
    // Gracefully fall back to CPU rendering
    use_gpu = false;
}
```

---

## Performance Benchmarking

### Built-In Benchmark System

**File:** `src/utils/DebugLog.h`

```cpp
#define DEBUG_GROUP_BENCHMARK   1  // Enable benchmark logging
```

**Output Format:**
```
[BENCHMARK] GPU Unwrap: TOTAL=195ms | Buffers=0.3ms | InputUpload=68ms | Shader=53ms | Download=75ms | Res=3840x2160 Tris=202
[BENCHMARK] CPU Unwrap: TOTAL=700ms | Res=3840x2160 Tris=202 SS=2x
```

### Adding Custom Benchmarks

```cpp
auto stage_start = std::chrono::high_resolution_clock::now();
// ... operation ...
auto stage_end = std::chrono::high_resolution_clock::now();
auto elapsed_ms = std::chrono::duration<double, std::milli>(stage_end - stage_start).count();

sprintf(msg, "[BENCHMARK] YourStage: %.2fms\n", elapsed_ms);
DEBUG_LOG_BENCHMARK(msg);
```

### Typical Performance Targets

| Resolution | CPU Target | GPU Target | Speedup |
|------------|------------|------------|---------|
| 1080p (1920x1080) | ~150ms | ~40ms | 3.75x |
| 4K (3840x2160) | ~700ms | ~195ms | 3.6x |
| 8K (7680x4320) | ~2800ms | ~750ms | 3.7x |

GPU speedup remains consistent across resolutions (3-4x) due to parallel processing.

---

## Future Optimizations

### Potential Improvements

1. **Async Transfer Queues**
   - Use dedicated transfer queue for uploads/downloads
   - Overlap with compute operations
   - Requires more complex synchronization

2. **Persistent Mapped Buffers**
   - Keep staging buffers mapped across frames
   - Avoid vkMapMemory/vkUnmapMemory overhead
   - ~5-10ms savings

3. **Resizable BAR Support**
   - Direct CPU access to device-local memory
   - Eliminates staging buffer copy
   - GPU must support `DEVICE_LOCAL | HOST_VISIBLE` memory

4. **Compute Pipeline Caching**
   - Save compiled pipelines to disk
   - Faster initialization on subsequent runs
   - Use `VkPipelineCache`

5. **Multi-GPU Support**
   - Distribute frames across multiple GPUs
   - Requires device group creation
   - Significant implementation complexity

---

## Conclusion

The Vulkan Hybrid Rendering architecture provides:

- **3.4x performance improvement** over CPU at 4K
- **Full mask support** (via CPU checkout)
- **Thread-safe** multi-threaded rendering
- **Production-ready** with comprehensive error handling

This approach is ideal for After Effects plugins that:
- Need GPU acceleration
- Must support masks
- Process high-resolution imagery
- Require custom compute operations

The key insight: **Bypass AE's GPU API entirely** when its limitations (no unmasked source with masks) prevent your use case. The hybrid CPU-to-GPU architecture maintains compatibility while delivering significant performance gains.

---

**Document Version:** 1.0
**Last Updated:** 2026-01-29
