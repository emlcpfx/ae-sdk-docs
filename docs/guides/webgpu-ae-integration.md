# WebGPU Integration for After Effects Plugins

This document describes how to integrate WebGPU compute shaders into an After Effects plugin using wgpu-native.

> **Note:** The WebGPU C API shown here is specific to wgpu-native (Dawn's C API differs slightly).

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Project Setup](#project-setup)
3. [CMake Configuration](#cmake-configuration)
4. [WebGPU Renderer Class Structure](#webgpu-renderer-class-structure)
5. [Thread Safety and Worker Thread Pattern](#thread-safety-and-worker-thread-pattern)
6. [WebGPU Initialization](#webgpu-initialization)
7. [Compute Shader Development](#compute-shader-development)
8. [Pipeline and Bind Group Creation](#pipeline-and-bind-group-creation)
9. [GPU Buffer Management](#gpu-buffer-management)
10. [Data Transfer Between AE and GPU](#data-transfer-between-ae-and-gpu)
11. [Integration with AE Plugin Code](#integration-with-ae-plugin-code)
12. [CPU Fallback Implementation](#cpu-fallback-implementation)
13. [Debugging Tips](#debugging-tips)
14. [Common Pitfalls](#common-pitfalls)

---

## Architecture Overview

The WebGPU integration uses a **worker thread pattern** to safely handle GPU operations from within After Effects. This is critical because:

1. **AE's render thread** calls your plugin's render function
2. **WebGPU async operations** require event processing
3. **MFR (Multi-Frame Rendering)** means multiple threads may call render simultaneously

```
+-------------------------------------------------------------+
|                    After Effects                             |
|  +------------------+    +------------------+                |
|  |  Render Thread   |    |  Render Thread   |  (MFR)        |
|  |     (frame 1)    |    |     (frame 2)    |               |
|  +--------+---------+    +--------+---------+               |
|           |                       |                          |
|           +-----------+-----------+                          |
|                       |                                      |
|                       v                                      |
|           +-----------------------+                          |
|           |  WebGPU Renderer      | (Single Instance)        |
|           |  - Thread-safe API    |                          |
|           +-----------+-----------+                          |
|                       |                                      |
|                       v                                      |
|           +-----------------------+                          |
|           |  Worker Thread        | (Owns all GPU resources) |
|           |  - WebGPU Device      |                          |
|           |  - Pipelines          |                          |
|           |  - Textures           |                          |
|           +-----------------------+                          |
+-------------------------------------------------------------+
```

---

## Project Setup

### Directory Structure

```
your_plugin/
+-- CMakeLists.txt
+-- eliemichel_webgpu/          # WebGPU-distribution submodule
|   +-- CMakeLists.txt
|   +-- wgpu-native/
|   +-- dawn/
+-- src/
|   +-- main/
|   |   +-- YourPlugin.cpp      # Main AE plugin code
|   +-- webgpu/
|       +-- WebGPURenderer.h
|       +-- WebGPURenderer.cpp
|       +-- shaders/
|           +-- your_shader.wgsl
+-- resources/
    +-- YourPluginPiPL.r
```

### Getting WebGPU-distribution

Clone the Elie Michel WebGPU-distribution:

```bash
git clone https://github.com/eliemichel/WebGPU-distribution.git eliemichel_webgpu
```

This provides both **wgpu-native** (Rust-based) and **Dawn** (Google's implementation) backends.

---

## CMake Configuration

### Main CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(YourAEPlugin)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# After Effects SDK
set(AESDK_ROOT "path/to/AfterEffectsSDK/Examples" CACHE PATH "After Effects SDK path")
include_directories(${AESDK_ROOT}/Headers)
include_directories(${AESDK_ROOT}/Headers/SP)
include_directories(${AESDK_ROOT}/Headers/Win)
include_directories(${AESDK_ROOT}/Util)

# Threading
find_package(Threads REQUIRED)

# ==============================================================================
# WebGPU Configuration
# ==============================================================================
option(USE_WEBGPU "Enable WebGPU acceleration" ON)

if(USE_WEBGPU)
    message(STATUS "WebGPU acceleration ENABLED")

    # WebGPU backend options: WGPU or DAWN
    set(WEBGPU_BACKEND "WGPU" CACHE STRING "WebGPU backend")

    # IMPORTANT: Use STATIC linking for AE plugins
    # This embeds the WebGPU library into your .aex
    set(WEBGPU_LINK_TYPE "STATIC" CACHE STRING "WebGPU linking")
    set(WEBGPU_BUILD_FROM_SOURCE OFF CACHE BOOL "Build WebGPU from source")

    # Add WebGPU-distribution
    add_subdirectory("eliemichel_webgpu" webgpu-dist)

    include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src/webgpu)
endif()

# ==============================================================================
# Plugin Target
# ==============================================================================
set(PLUGIN_SOURCES
    src/main/YourPlugin.cpp
    ${AESDK_ROOT}/Util/AEGP_SuiteHandler.cpp
    ${AESDK_ROOT}/Util/MissingSuiteError.cpp
)

if(USE_WEBGPU)
    list(APPEND PLUGIN_SOURCES
        src/webgpu/WebGPURenderer.cpp
    )
endif()

add_library(YourPlugin SHARED ${PLUGIN_SOURCES})

# Preprocessor definitions
target_compile_definitions(YourPlugin PRIVATE
    WIN32
    _WINDOWS
    NDEBUG
    _USRDLL
    PF_DEEP_COLOR_AWARE=1
    MSWindows=1
    UNICODE
    _UNICODE
)

if(USE_WEBGPU)
    target_compile_definitions(YourPlugin PRIVATE
        HAS_WEBGPU=1
        WEBGPU_BACKEND_WGPU=1
    )
endif()

# Link libraries
if(USE_WEBGPU)
    target_link_libraries(YourPlugin
        webgpu                      # From WebGPU-distribution
        ${CMAKE_THREAD_LIBS_INIT}
    )
else()
    target_link_libraries(YourPlugin
        ${CMAKE_THREAD_LIBS_INIT}
    )
endif()

# Output as .aex
set_target_properties(YourPlugin PROPERTIES
    SUFFIX ".aex"
    PREFIX ""
    OUTPUT_NAME "YourPlugin"
)

# MSVC compiler flags
if(MSVC)
    target_compile_options(YourPlugin PRIVATE
        /W3          # Warning level 3
        /O2          # Optimize for speed
        /Oi          # Enable intrinsic functions
        /MT          # Static runtime (/MT is recommended for AE plugins to avoid
                     # CRT version conflicts, but /MD also works.)
        /EHsc        # Exception handling
    )
endif()

# PiPL resource compilation (standard AE SDK pattern)
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/YourPluginPiPL.rc
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/resources/YourPluginPiPL.r ${CMAKE_CURRENT_BINARY_DIR}/
    COMMAND cl /I "${AESDK_ROOT}/Headers" /I "${AESDK_ROOT}/Headers/SP" /I "${AESDK_ROOT}/Util" /EP "${CMAKE_CURRENT_BINARY_DIR}/YourPluginPiPL.r" > "${CMAKE_CURRENT_BINARY_DIR}/YourPluginPiPL.rr"
    COMMAND ${CMAKE_COMMAND} -E chdir ${CMAKE_CURRENT_BINARY_DIR} "${AESDK_ROOT}/Resources/PiPLtool.exe" "YourPluginPiPL.rr" "YourPluginPiPL.rrc"
    COMMAND cl /D "MSWindows" /EP "${CMAKE_CURRENT_BINARY_DIR}/YourPluginPiPL.rrc" > "${CMAKE_CURRENT_BINARY_DIR}/YourPluginPiPL.rc"
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/resources/YourPluginPiPL.r
    COMMENT "Compiling PiPL resource"
)

target_sources(YourPlugin PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/YourPluginPiPL.rc)
```

### Build Commands

```bash
# Configure with WebGPU
cmake -B build_gpu -DUSE_WEBGPU=ON

# Build
cmake --build build_gpu --config Release
```

---

## WebGPU Renderer Class Structure

### Header File (WebGPURenderer.h)

```cpp
#pragma once

#include <vector>
#include <webgpu.h>
#include <memory>
#include <string>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <atomic>

// Your effect parameters
struct YourEffectParams {
    float parameter1;
    float parameter2;
    int iterations;
    // ... other parameters
};

// Double-buffered result for thread-safe communication
struct WorkerResult {
    std::vector<uint8_t> pixelData;
    int width = 0;
    int height = 0;
    int channels = 0;
    bool ready = false;
    std::mutex mutex;
};

// Worker thread that owns all WebGPU resources
class GPUWorkerThread {
public:
    GPUWorkerThread();
    ~GPUWorkerThread();

    // Thread lifecycle
    bool Start();
    void Stop();
    bool IsRunning() const { return m_running.load(); }

    // Thread-safe API for AE render threads
    bool SubmitRenderWork(
        const unsigned char* inputBuffer,
        const YourEffectParams& params,
        int width, int height, int channels
    );

    bool IsResultReady();
    bool GetResult(unsigned char* outputBuffer, int width, int height, int channels);

private:
    // Worker thread main loop
    void WorkerThreadMain();

    // WebGPU operations (run ONLY on worker thread)
    bool InitializeWebGPU();
    bool CreateShaders();
    bool CreateComputePipelines();
    bool CreateBuffers(int width, int height);
    bool ProcessRenderWork();
    bool ReadGPUBuffer();
    void CleanupWebGPU();

    // Thread synchronization
    std::thread m_workerThread;
    std::atomic<bool> m_running{false};
    std::atomic<bool> m_shutdownRequested{false};

    // Work queue
    struct WorkItem {
        std::vector<uint8_t> inputData;
        YourEffectParams params;
        int width, height, channels;
        bool valid = false;
    };

    WorkItem m_currentWork;
    std::mutex m_workMutex;
    std::condition_variable m_workCondition;
    std::atomic<bool> m_hasWork{false};

    // Double-buffered results
    WorkerResult m_results[2];
    std::atomic<int> m_activeResult{0};
    std::atomic<int> m_workingResult{1};

    // WebGPU resources (owned exclusively by worker thread)
    WGPUInstance m_instance = nullptr;
    WGPUAdapter m_adapter = nullptr;
    WGPUDevice m_device = nullptr;
    WGPUQueue m_queue = nullptr;

    // Compute pipelines
    WGPUComputePipeline m_yourPipeline = nullptr;
    WGPUShaderModule m_yourShader = nullptr;

    // Textures and buffers
    WGPUTexture m_inputTexture = nullptr;
    WGPUTexture m_outputTexture = nullptr;
    WGPUTextureView m_inputTextureView = nullptr;
    WGPUTextureView m_outputTextureView = nullptr;
    WGPUBuffer m_paramsBuffer = nullptr;
    WGPUBuffer m_stagingBuffer = nullptr;

    // Bind group layouts
    WGPUBindGroupLayout m_bindGroupLayout = nullptr;

    size_t m_currentWidth = 0;
    size_t m_currentHeight = 0;
};

// Main renderer class - provides thread-safe API to AE
class WebGPURenderer {
public:
    WebGPURenderer();
    ~WebGPURenderer();

    bool Initialize();
    void Cleanup();

    bool Render(
        const unsigned char* inputBuffer,
        unsigned char* outputBuffer,
        int width, int height, int channels,
        const YourEffectParams& params
    );

    bool IsInitialized() const { return m_initialized; }

private:
    std::unique_ptr<GPUWorkerThread> m_workerThread;
    bool m_initialized = false;
    bool m_fallbackMode = false;
};
```

---

## Thread Safety and Worker Thread Pattern

The worker thread pattern is essential because:

1. **WebGPU requires consistent thread context** - Device creation and usage should happen on the same thread
2. **AE's MFR can call from multiple threads** - Your render function may be called from different threads
3. **Async operations need event processing** - `wgpuInstanceProcessEvents()` must be called regularly

### Implementation Pattern

```cpp
// Worker thread main loop
void GPUWorkerThread::WorkerThreadMain() {
    DebugLog("Worker thread started");

    // Initialize WebGPU on THIS thread
    if (!InitializeWebGPU()) {
        DebugLog("Failed to initialize WebGPU");
        m_running = false;
        return;
    }

    // Main work loop
    while (!m_shutdownRequested.load()) {
        std::unique_lock<std::mutex> lock(m_workMutex);

        // Wait for work or shutdown signal
        m_workCondition.wait(lock, [this] {
            return m_hasWork.load() || m_shutdownRequested.load();
        });

        if (m_shutdownRequested.load()) {
            break;
        }

        if (m_hasWork.load() && m_currentWork.valid) {
            lock.unlock();  // Release lock while processing

            if (ProcessRenderWork()) {
                DebugLog("Work processed successfully");
            }

            m_hasWork = false;
        }
    }

    // Cleanup WebGPU on the SAME thread that created it
    CleanupWebGPU();
    DebugLog("Worker thread exiting");
}

// Thread-safe work submission (called from AE render threads)
bool GPUWorkerThread::SubmitRenderWork(
    const unsigned char* inputBuffer,
    const YourEffectParams& params,
    int width, int height, int channels
) {
    if (!m_running.load()) {
        return false;
    }

    std::lock_guard<std::mutex> lock(m_workMutex);

    // Copy input data (thread-safe)
    size_t dataSize = width * height * channels;
    m_currentWork.inputData.resize(dataSize);
    memcpy(m_currentWork.inputData.data(), inputBuffer, dataSize);
    m_currentWork.params = params;
    m_currentWork.width = width;
    m_currentWork.height = height;
    m_currentWork.channels = channels;
    m_currentWork.valid = true;

    m_hasWork = true;
    m_workCondition.notify_one();

    return true;
}
```

---

## WebGPU Initialization

### Device and Queue Creation

```cpp
bool GPUWorkerThread::InitializeWebGPU() {
    DebugLog("Initializing WebGPU");

    // Step 1: Create WebGPU instance
    WGPUInstanceDescriptor instanceDesc = {};
    instanceDesc.nextInChain = nullptr;

    m_instance = wgpuCreateInstance(&instanceDesc);
    if (!m_instance) {
        DebugLog("Failed to create WebGPU instance");
        return false;
    }

    // Step 2: Request adapter (GPU)
    WGPURequestAdapterOptions adapterOptions = {};
    adapterOptions.powerPreference = WGPUPowerPreference_HighPerformance;
    adapterOptions.backendType = WGPUBackendType_D3D12;  // Use D3D12 on Windows

    struct AdapterData {
        std::atomic<bool> received{false};
        WGPUAdapter adapter = nullptr;
    } adapterData;

    WGPURequestAdapterCallbackInfo callbackInfo = {};
    callbackInfo.mode = WGPUCallbackMode_WaitAnyOnly;
    callbackInfo.callback = [](WGPURequestAdapterStatus status,
                               WGPUAdapter adapter,
                               WGPUStringView message,
                               void* userdata1,
                               void* userdata2) {
        auto& data = *static_cast<AdapterData*>(userdata1);
        if (status == WGPURequestAdapterStatus_Success) {
            data.adapter = adapter;
        }
        data.received.store(true);
    };
    callbackInfo.userdata1 = &adapterData;
    callbackInfo.userdata2 = nullptr;

    wgpuInstanceRequestAdapter(m_instance, &adapterOptions, callbackInfo);

    // Poll for adapter (safe in worker thread)
    int timeout = 1000;
    while (!adapterData.received.load() && timeout > 0) {
        wgpuInstanceProcessEvents(m_instance);
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
        timeout--;
    }

    m_adapter = adapterData.adapter;
    if (!m_adapter) {
        DebugLog("Failed to get WebGPU adapter");
        return false;
    }

    // Step 3: Create device
    struct DeviceData {
        std::atomic<bool> received{false};
        WGPUDevice device = nullptr;
    } deviceData;

    WGPUDeviceDescriptor deviceDesc = {};
    deviceDesc.label = {"MyPlugin Device", WGPU_STRLEN};
    deviceDesc.requiredFeatureCount = 0;
    deviceDesc.requiredFeatures = nullptr;
    deviceDesc.defaultQueue.label = {"Main Queue", WGPU_STRLEN};

    WGPURequestDeviceCallbackInfo deviceCallbackInfo = {};
    deviceCallbackInfo.mode = WGPUCallbackMode_WaitAnyOnly;
    deviceCallbackInfo.callback = [](WGPURequestDeviceStatus status,
                                     WGPUDevice device,
                                     WGPUStringView message,
                                     void* userdata1,
                                     void* userdata2) {
        auto& data = *static_cast<DeviceData*>(userdata1);
        if (status == WGPURequestDeviceStatus_Success) {
            data.device = device;
        }
        data.received.store(true);
    };
    deviceCallbackInfo.userdata1 = &deviceData;
    deviceCallbackInfo.userdata2 = nullptr;

    wgpuAdapterRequestDevice(m_adapter, &deviceDesc, deviceCallbackInfo);

    timeout = 1000;
    while (!deviceData.received.load() && timeout > 0) {
        wgpuInstanceProcessEvents(m_instance);
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
        timeout--;
    }

    m_device = deviceData.device;
    if (!m_device) {
        DebugLog("Failed to create WebGPU device");
        return false;
    }

    // Step 4: Get queue
    m_queue = wgpuDeviceGetQueue(m_device);
    if (!m_queue) {
        DebugLog("Failed to get WebGPU queue");
        return false;
    }

    // Step 5: Create shaders and pipelines
    if (!CreateShaders()) {
        return false;
    }

    if (!CreateComputePipelines()) {
        return false;
    }

    DebugLog("WebGPU initialization completed!");
    return true;
}
```

---

## Compute Shader Development

### WGSL Shader Structure

Compute shaders in WebGPU use WGSL (WebGPU Shading Language). Here's a template:

```wgsl
// shader.wgsl

// Uniform buffer for parameters
struct Params {
    parameter1: f32,
    parameter2: f32,
    padding1: f32,     // Align to 16 bytes
    padding2: f32,
}

// Bindings
@group(0) @binding(0) var input_texture: texture_2d<f32>;
@group(0) @binding(1) var output_texture: texture_storage_2d<rgba32float, write>;
@group(0) @binding(2) var<uniform> params: Params;

// Compute shader entry point
@compute @workgroup_size(8, 8, 1)
fn main(@builtin(global_invocation_id) global_id: vec3<u32>) {
    let dimensions = textureDimensions(input_texture);
    let coords = vec2<i32>(global_id.xy);

    // Bounds check (important!)
    if (coords.x >= i32(dimensions.x) || coords.y >= i32(dimensions.y)) {
        return;
    }

    // Read input
    let input_color = textureLoad(input_texture, coords, 0);

    // Your processing here
    var output_color = input_color;
    output_color.r = input_color.r * params.parameter1;
    output_color.g = input_color.g * params.parameter2;
    // ... etc

    // Write output
    textureStore(output_texture, coords, output_color);
}
```

### Workgroup Size Considerations

```wgsl
// Use 8x8 for general compatibility
@compute @workgroup_size(8, 8, 1)

// For horizontal blur with shared memory (advanced):
// @compute @workgroup_size(256, 1, 1)

// For vertical blur:
// @compute @workgroup_size(1, 256, 1)
```

**Important**: Avoid shared memory (`var<workgroup>`) on initial implementations - it can cause driver crashes on some GPUs.

### Embedding Shaders in C++

For reliability, embed shaders directly in your C++ code:

```cpp
bool GPUWorkerThread::CreateShaders() {
    const char* shaderSource = R"(
// Your shader - parameters
struct Params {
    parameter1: f32,
    parameter2: f32,
    padding1: f32,
    padding2: f32,
}

@group(0) @binding(0) var input_texture: texture_2d<f32>;
@group(0) @binding(1) var output_texture: texture_storage_2d<rgba32float, write>;
@group(0) @binding(2) var<uniform> params: Params;

@compute @workgroup_size(8, 8, 1)
fn main(@builtin(global_invocation_id) global_id: vec3<u32>) {
    let dims = textureDimensions(input_texture);
    let coords = vec2<i32>(global_id.xy);

    if (coords.x >= i32(dims.x) || coords.y >= i32(dims.y)) {
        return;
    }

    let color = textureLoad(input_texture, coords, 0);

    // Process...
    var result = color;
    result.r = color.r * params.parameter1;

    textureStore(output_texture, coords, result);
}
)";

    WGPUShaderModuleDescriptor shaderDesc = {};
    shaderDesc.label = {"Your Shader", WGPU_STRLEN};

    WGPUShaderSourceWGSL wgslDesc = {};
    wgslDesc.chain.sType = WGPUSType_ShaderSourceWGSL;
    wgslDesc.code = {shaderSource, WGPU_STRLEN};
    shaderDesc.nextInChain = &wgslDesc.chain;

    m_yourShader = wgpuDeviceCreateShaderModule(m_device, &shaderDesc);

    return m_yourShader != nullptr;
}
```

---

## Pipeline and Bind Group Creation

### Creating a Compute Pipeline

```cpp
bool GPUWorkerThread::CreateComputePipelines() {
    // Define bind group layout
    WGPUBindGroupLayoutEntry bindingLayouts[3] = {};

    // Binding 0: Input texture (read)
    bindingLayouts[0].binding = 0;
    bindingLayouts[0].visibility = WGPUShaderStage_Compute;
    bindingLayouts[0].texture.sampleType = WGPUTextureSampleType_Float;
    bindingLayouts[0].texture.viewDimension = WGPUTextureViewDimension_2D;
    bindingLayouts[0].texture.multisampled = false;

    // Binding 1: Output texture (write)
    bindingLayouts[1].binding = 1;
    bindingLayouts[1].visibility = WGPUShaderStage_Compute;
    bindingLayouts[1].storageTexture.access = WGPUStorageTextureAccess_WriteOnly;
    bindingLayouts[1].storageTexture.format = WGPUTextureFormat_RGBA32Float;
    bindingLayouts[1].storageTexture.viewDimension = WGPUTextureViewDimension_2D;

    // Binding 2: Parameters uniform buffer
    bindingLayouts[2].binding = 2;
    bindingLayouts[2].visibility = WGPUShaderStage_Compute;
    bindingLayouts[2].buffer.type = WGPUBufferBindingType_Uniform;
    bindingLayouts[2].buffer.minBindingSize = 16;  // 4 floats

    WGPUBindGroupLayoutDescriptor layoutDesc = {};
    layoutDesc.label = {"Bind Group Layout", WGPU_STRLEN};
    layoutDesc.entryCount = 3;
    layoutDesc.entries = bindingLayouts;

    m_bindGroupLayout = wgpuDeviceCreateBindGroupLayout(m_device, &layoutDesc);
    if (!m_bindGroupLayout) {
        return false;
    }

    // Create pipeline layout
    WGPUPipelineLayoutDescriptor pipelineLayoutDesc = {};
    pipelineLayoutDesc.label = {"Pipeline Layout", WGPU_STRLEN};
    pipelineLayoutDesc.bindGroupLayoutCount = 1;
    pipelineLayoutDesc.bindGroupLayouts = &m_bindGroupLayout;

    WGPUPipelineLayout pipelineLayout = wgpuDeviceCreatePipelineLayout(m_device, &pipelineLayoutDesc);

    // Create compute pipeline
    WGPUComputePipelineDescriptor pipelineDesc = {};
    pipelineDesc.label = {"Your Pipeline", WGPU_STRLEN};
    pipelineDesc.layout = pipelineLayout;
    pipelineDesc.compute.module = m_yourShader;
    pipelineDesc.compute.entryPoint = {"main", WGPU_STRLEN};

    m_yourPipeline = wgpuDeviceCreateComputePipeline(m_device, &pipelineDesc);

    wgpuPipelineLayoutRelease(pipelineLayout);  // Can release after pipeline creation

    return m_yourPipeline != nullptr;
}
```

---

## GPU Buffer Management

### Creating Textures

```cpp
bool GPUWorkerThread::CreateBuffers(int width, int height) {
    // Clean up if size changed
    if (m_currentWidth != width || m_currentHeight != height) {
        if (m_inputTexture) wgpuTextureRelease(m_inputTexture);
        if (m_outputTexture) wgpuTextureRelease(m_outputTexture);
        if (m_inputTextureView) wgpuTextureViewRelease(m_inputTextureView);
        if (m_outputTextureView) wgpuTextureViewRelease(m_outputTextureView);

        m_inputTexture = nullptr;
        m_outputTexture = nullptr;
        m_inputTextureView = nullptr;
        m_outputTextureView = nullptr;

        m_currentWidth = width;
        m_currentHeight = height;
    }

    // Skip if already created
    if (m_inputTexture && m_outputTexture) {
        return true;
    }

    // Input texture descriptor
    WGPUTextureDescriptor textureDesc = {};
    textureDesc.dimension = WGPUTextureDimension_2D;
    textureDesc.size.width = width;
    textureDesc.size.height = height;
    textureDesc.size.depthOrArrayLayers = 1;
    textureDesc.mipLevelCount = 1;
    textureDesc.sampleCount = 1;
    textureDesc.format = WGPUTextureFormat_RGBA32Float;  // For 32-bit float
    textureDesc.usage = WGPUTextureUsage_TextureBinding |
                        WGPUTextureUsage_CopyDst |
                        WGPUTextureUsage_CopySrc;

    textureDesc.label = {"Input Texture", WGPU_STRLEN};
    m_inputTexture = wgpuDeviceCreateTexture(m_device, &textureDesc);
    if (!m_inputTexture) return false;

    // Output texture (needs StorageBinding for compute writes)
    textureDesc.label = {"Output Texture", WGPU_STRLEN};
    textureDesc.usage = WGPUTextureUsage_StorageBinding | WGPUTextureUsage_CopySrc;
    m_outputTexture = wgpuDeviceCreateTexture(m_device, &textureDesc);
    if (!m_outputTexture) return false;

    // Create texture views
    WGPUTextureViewDescriptor viewDesc = {};
    viewDesc.format = WGPUTextureFormat_RGBA32Float;
    viewDesc.dimension = WGPUTextureViewDimension_2D;
    viewDesc.baseMipLevel = 0;
    viewDesc.mipLevelCount = 1;
    viewDesc.baseArrayLayer = 0;
    viewDesc.arrayLayerCount = 1;

    viewDesc.label = {"Input Texture View", WGPU_STRLEN};
    m_inputTextureView = wgpuTextureCreateView(m_inputTexture, &viewDesc);

    viewDesc.label = {"Output Texture View", WGPU_STRLEN};
    m_outputTextureView = wgpuTextureCreateView(m_outputTexture, &viewDesc);

    // Create staging buffer for GPU->CPU readback
    size_t bufferSize = width * height * 16;  // RGBA32Float = 16 bytes/pixel
    WGPUBufferDescriptor bufferDesc = {};
    bufferDesc.label = {"Staging Buffer", WGPU_STRLEN};
    bufferDesc.size = bufferSize;
    bufferDesc.usage = WGPUBufferUsage_MapRead | WGPUBufferUsage_CopyDst;
    bufferDesc.mappedAtCreation = false;

    m_stagingBuffer = wgpuDeviceCreateBuffer(m_device, &bufferDesc);

    // Create parameters uniform buffer
    WGPUBufferDescriptor paramsBufferDesc = {};
    paramsBufferDesc.label = {"Parameters Buffer", WGPU_STRLEN};
    paramsBufferDesc.size = 16;  // 4 floats
    paramsBufferDesc.usage = WGPUBufferUsage_Uniform | WGPUBufferUsage_CopyDst;
    paramsBufferDesc.mappedAtCreation = false;

    m_paramsBuffer = wgpuDeviceCreateBuffer(m_device, &paramsBufferDesc);

    return true;
}
```

---

## Data Transfer Between AE and GPU

### Uploading Data to GPU

```cpp
bool GPUWorkerThread::ProcessRenderWork() {
    int width = m_currentWork.width;
    int height = m_currentWork.height;

    if (!CreateBuffers(width, height)) {
        return false;
    }

    // Upload input texture data
    WGPUTexelCopyTextureInfo destination = {};
    destination.texture = m_inputTexture;
    destination.mipLevel = 0;
    destination.origin = {0, 0, 0};
    destination.aspect = WGPUTextureAspect_All;

    WGPUTexelCopyBufferLayout dataLayout = {};
    dataLayout.offset = 0;
    dataLayout.bytesPerRow = width * 16;  // RGBA32Float = 16 bytes/pixel
    dataLayout.rowsPerImage = height;

    WGPUExtent3D writeSize = {(uint32_t)width, (uint32_t)height, 1};

    wgpuQueueWriteTexture(m_queue, &destination,
                          m_currentWork.inputData.data(),
                          width * height * 16,
                          &dataLayout, &writeSize);

    // Upload parameters
    struct {
        float parameter1;
        float parameter2;
        float padding[2];
    } params = {
        m_currentWork.params.parameter1,
        m_currentWork.params.parameter2,
        {0, 0}
    };
    wgpuQueueWriteBuffer(m_queue, m_paramsBuffer, 0, &params, sizeof(params));

    // Execute compute pass
    // ... (see next section)

    // Read back results
    return ReadGPUBuffer();
}
```

### Reading Data Back from GPU

```cpp
bool GPUWorkerThread::ReadGPUBuffer() {
    int width = m_currentWork.width;
    int height = m_currentWork.height;

    // Copy output texture to staging buffer
    WGPUCommandEncoderDescriptor encoderDesc = {};
    WGPUCommandEncoder encoder = wgpuDeviceCreateCommandEncoder(m_device, &encoderDesc);

    WGPUTexelCopyTextureInfo source = {};
    source.texture = m_outputTexture;
    source.mipLevel = 0;
    source.origin = {0, 0, 0};
    source.aspect = WGPUTextureAspect_All;

    WGPUTexelCopyBufferInfo destination = {};
    destination.buffer = m_stagingBuffer;
    destination.layout.offset = 0;
    destination.layout.bytesPerRow = width * 16;
    destination.layout.rowsPerImage = height;

    WGPUExtent3D copySize = {(uint32_t)width, (uint32_t)height, 1};
    wgpuCommandEncoderCopyTextureToBuffer(encoder, &source, &destination, &copySize);

    WGPUCommandBuffer cmdBuffer = wgpuCommandEncoderFinish(encoder, nullptr);
    wgpuQueueSubmit(m_queue, 1, &cmdBuffer);
    wgpuCommandEncoderRelease(encoder);
    wgpuCommandBufferRelease(cmdBuffer);

    // Map staging buffer for CPU read
    struct MapData {
        std::atomic<bool> done{false};
        WGPUMapAsyncStatus status;
    } mapData;

    WGPUBufferMapCallbackInfo mapCallback = {};
    mapCallback.mode = WGPUCallbackMode_WaitAnyOnly;
    mapCallback.callback = [](WGPUMapAsyncStatus status,
                              WGPUStringView message,
                              void* userdata1,
                              void* userdata2) {
        auto& data = *static_cast<MapData*>(userdata1);
        data.status = status;
        data.done.store(true);
    };
    mapCallback.userdata1 = &mapData;

    wgpuBufferMapAsync(m_stagingBuffer, WGPUMapMode_Read, 0,
                       width * height * 16, mapCallback);

    // Wait for mapping to complete
    int timeout = 5000;
    while (!mapData.done.load() && timeout > 0) {
        wgpuInstanceProcessEvents(m_instance);
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
        timeout--;
    }

    if (mapData.status != WGPUMapAsyncStatus_Success) {
        return false;
    }

    // Copy data to result buffer
    const void* mappedData = wgpuBufferGetConstMappedRange(m_stagingBuffer, 0, width * height * 16);

    int workingIdx = m_workingResult.load();
    {
        std::lock_guard<std::mutex> lock(m_results[workingIdx].mutex);
        m_results[workingIdx].width = width;
        m_results[workingIdx].height = height;
        m_results[workingIdx].channels = 16;  // RGBA32Float
        m_results[workingIdx].pixelData.resize(width * height * 16);
        memcpy(m_results[workingIdx].pixelData.data(), mappedData, width * height * 16);
        m_results[workingIdx].ready = true;
    }

    wgpuBufferUnmap(m_stagingBuffer);

    // Swap buffers
    m_activeResult.store(workingIdx);
    m_workingResult.store(1 - workingIdx);

    return true;
}
```

---

## Integration with AE Plugin Code

### AE Plugin Main File

```cpp
// YourPlugin.cpp

#ifdef HAS_WEBGPU
  #include "WebGPURenderer.h"
  static std::unique_ptr<WebGPURenderer> g_gpuRenderer = nullptr;
  static std::mutex g_gpuMutex;
#endif

// ... AE SDK includes ...

static PF_Err Render(
    PF_InData *in_data,
    PF_OutData *out_data,
    PF_ParamDef *params[],
    PF_LayerDef *output)
{
    PF_Err err = PF_Err_NONE;

    // Get parameters from AE
    YourEffectParams effectParams;
    effectParams.parameter1 = params[PARAM_1]->u.fs_d.value;
    effectParams.parameter2 = params[PARAM_2]->u.fs_d.value;
    effectParams.iterations = params[PARAM_ITERATIONS]->u.sd.value;

    // Determine bit depth
    AEFX_SuiteScoper<PF_WorldSuite2> wsP = ...;
    PF_PixelFormat format;
    wsP->PF_GetPixelFormat(&params[PARAM_INPUT]->u.ld, &format);

    // Only use GPU for 32-bit float
    bool useGPU = false;
#ifdef HAS_WEBGPU
    if (format == PF_PixelFormat_ARGB128) {  // 32-bit float
        std::lock_guard<std::mutex> lock(g_gpuMutex);

        // Initialize GPU renderer on first use
        if (!g_gpuRenderer) {
            g_gpuRenderer = std::make_unique<WebGPURenderer>();
            if (!g_gpuRenderer->Initialize()) {
                g_gpuRenderer.reset();
            }
        }

        if (g_gpuRenderer && g_gpuRenderer->IsInitialized()) {
            useGPU = true;
        }
    }
#endif

    if (useGPU) {
#ifdef HAS_WEBGPU
        err = RenderWithGPU(in_data, out_data, params, output, effectParams);
#endif
    } else {
        err = RenderWithCPU(in_data, out_data, params, output, effectParams);
    }

    return err;
}

#ifdef HAS_WEBGPU
static PF_Err RenderWithGPU(
    PF_InData *in_data,
    PF_OutData *out_data,
    PF_ParamDef *params[],
    PF_LayerDef *output,
    const YourEffectParams& effectParams)
{
    PF_Err err = PF_Err_NONE;

    PF_LayerDef *input = &params[PARAM_INPUT]->u.ld;
    int width = input->width;
    int height = input->height;
    int rowbytes = input->rowbytes;

    // Convert AE pixel data to flat buffer
    // AE uses premultiplied ARGB, convert to RGBA for GPU
    std::vector<float> inputBuffer(width * height * 4);

    for (int y = 0; y < height; y++) {
        PF_PixelFloat* row = (PF_PixelFloat*)((char*)input->data + y * rowbytes);
        float* dst = inputBuffer.data() + y * width * 4;

        for (int x = 0; x < width; x++) {
            // AE format: ARGB premultiplied
            // GPU format: RGBA
            dst[x * 4 + 0] = row[x].red;
            dst[x * 4 + 1] = row[x].green;
            dst[x * 4 + 2] = row[x].blue;
            dst[x * 4 + 3] = row[x].alpha;
        }
    }

    // Process on GPU
    std::vector<float> outputBuffer(width * height * 4);

    if (!g_gpuRenderer->Render(
        (unsigned char*)inputBuffer.data(),
        (unsigned char*)outputBuffer.data(),
        width, height, 16,  // 16 bytes per pixel for RGBA32Float
        effectParams))
    {
        // Fallback to CPU if GPU fails
        return RenderWithCPU(in_data, out_data, params, output, effectParams);
    }

    // Copy results back to AE
    for (int y = 0; y < height; y++) {
        PF_PixelFloat* row = (PF_PixelFloat*)((char*)output->data + y * output->rowbytes);
        float* src = outputBuffer.data() + y * width * 4;

        for (int x = 0; x < width; x++) {
            row[x].red   = src[x * 4 + 0];
            row[x].green = src[x * 4 + 1];
            row[x].blue  = src[x * 4 + 2];
            row[x].alpha = src[x * 4 + 3];
        }
    }

    return err;
}
#endif
```

---

## CPU Fallback Implementation

Always implement a CPU fallback for:

1. GPUs that don't support WebGPU
2. 8-bit and 16-bit color modes (if you only implemented GPU for 32-bit)
3. Debugging comparison

```cpp
bool WebGPURenderer::Render(
    const unsigned char* inputBuffer,
    unsigned char* outputBuffer,
    int width, int height, int channels,
    const YourEffectParams& params)
{
    if (!m_initialized) {
        return false;
    }

    // Fallback mode - just copy
    if (m_fallbackMode) {
        memcpy(outputBuffer, inputBuffer, width * height * channels);
        return true;
    }

    if (!m_workerThread) {
        return false;
    }

    // Submit to GPU worker thread
    if (!m_workerThread->SubmitRenderWork(inputBuffer, params, width, height, channels)) {
        return false;
    }

    // Wait for result (with timeout)
    int timeout = 5000;  // 5 seconds
    while (!m_workerThread->IsResultReady() && timeout > 0) {
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
        timeout--;
    }

    if (timeout <= 0) {
        DebugLog("GPU render timeout");
        return false;
    }

    return m_workerThread->GetResult(outputBuffer, width, height, channels);
}
```

---

## Debugging Tips

### Debug Output on Windows

```cpp
#ifdef _WIN32
#include <Windows.h>
#define DebugLog(msg) OutputDebugStringA((std::string("MyPlugin: ") + msg + "\n").c_str())
#else
#include <iostream>
#define DebugLog(msg) std::cout << "MyPlugin: " << msg << std::endl
#endif
```

Use **DebugView** (from Sysinternals) to see OutputDebugString messages.

### Common Issues

1. **Plugin doesn't load**: Check PiPL resource, ensure /MT linking
2. **Black output**: Check texture formats match, verify buffer sizes
3. **Crash on shader creation**: Test shader in separate app first
4. **Timeout waiting for GPU**: Ensure event processing loop runs

---

## Common Pitfalls

### 1. Buffer Size Mismatches

```cpp
// WRONG - using wrong byte count
wgpuQueueWriteTexture(..., data, width * height * 4, ...);  // 4 = RGBA8

// CORRECT for RGBA32Float
wgpuQueueWriteTexture(..., data, width * height * 16, ...);  // 16 = RGBA32Float
```

### 2. Forgetting Bounds Checks in Shaders

```wgsl
// REQUIRED - prevent out-of-bounds access
if (coords.x >= i32(dims.x) || coords.y >= i32(dims.y)) {
    return;
}
```

### 3. Wrong Workgroup Dispatch Count

```cpp
// Workgroup size is 8x8, so divide and round up
uint32_t workgroupsX = (width + 7) / 8;
uint32_t workgroupsY = (height + 7) / 8;
wgpuComputePassEncoderDispatchWorkgroups(computePass, workgroupsX, workgroupsY, 1);
```

### 4. Not Releasing Resources

```cpp
// Always release resources you create
wgpuBindGroupRelease(bindGroup);
wgpuComputePassEncoderRelease(computePass);
wgpuCommandEncoderRelease(encoder);
wgpuCommandBufferRelease(cmdBuffer);
```

### 5. Static Linking Requirements

For AE plugins, /MT is recommended for static linking to avoid CRT version conflicts, but /MD also works. Ensure `WEBGPU_LINK_TYPE` is set to `STATIC` in CMake.

---

## Summary

To integrate WebGPU into an AE plugin:

1. **Use the worker thread pattern** - All WebGPU operations happen on a dedicated thread
2. **Use static linking** - Set `WEBGPU_LINK_TYPE=STATIC` and compile with `/MT`
3. **Embed shaders in code** - More reliable than file loading
4. **Use 8x8 workgroups** - Most compatible across GPUs
5. **Implement CPU fallback** - For unsupported GPUs or bit depths
6. **Handle data conversion** - AE uses ARGB premultiplied, convert as needed

The overhead of data transfer CPU<->GPU is worthwhile for complex per-pixel operations that would otherwise require O(n) or O(n^2) kernel passes on the CPU.
