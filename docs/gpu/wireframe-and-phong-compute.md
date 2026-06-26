# Wireframe and Phong Compute Shaders, End to End

A worked, copy-pasteable example of rendering a 3D mesh into an AE effect's
output buffer using **Vulkan compute shaders only** — no graphics pipeline,
no render pass, no framebuffer. Covers:

- The Phong shader (smooth-shaded, software z-buffered).
- The wireframe shader (Bresenham, thick, alpha-composited).
- The C++ host side: descriptor layouts, pipelines, dispatch, barriers,
  and the data flow from AE camera + lights into shader-ready buffers.
- A combined "Phong + wireframe over top" mode in a single command buffer.

This page is the concrete companion to
[Triangle Rasterization in Compute Shaders](../rendering/triangle-rasterization-compute.md),
which covers the general pattern and theory. Read that first if any of the
mechanics here look unfamiliar — atomic depth, fill barriers, the
one-thread-per-triangle topology.

The reference plugin is HEADCASE, an AE plugin that rasterizes an animated
face mesh and overlays it on plate footage. The same shaders work for any
mesh-rendering effect: contour overlays, on-set 3D guides, custom 3D
volumetric outlines, projected templates.

## Architecture overview

```
AE plugin (CPU side)
  ├─ Read AE camera (AEGP_GetEffectCamera + AEGP_GetLayerToWorldXform)
  ├─ Read AE lights  (AEGP_LightSuite + AEGP_StreamSuite)
  ├─ Project 3D verts → 2D screen pixels + depth
  ├─ Compute per-vertex normals (area-weighted)
  └─ Build descriptor data ─────► VulkanRenderer::RenderMesh()
                                       │
                                       ▼
Vulkan compute (GPU side)
  ┌─ vkCmdFillBuffer  → reset depth SSBO to 0xFFFFFFFF
  │   pipeline barrier (transfer→compute)
  ├─ Phong dispatch    → atomic z-buffered Blinn-Phong shading
  │   pipeline barrier (compute→compute)         [only for combined mode]
  └─ Wireframe dispatch → Bresenham edges over top
```

Single command buffer, single output storage image, single descriptor pool.

## Data structures (must match shader and host bit-for-bit)

`std430` for SSBOs, `std140` for UBOs. The C++ structs below mirror the
GLSL declarations exactly:

```cpp
// Wireframe vertex — 2D screen position only (std430, 8 B)
struct WireVertex {
    float x, y;
};

// Phong vertex — screen pos + depth + world-space normal (std430, 24 B)
struct PhongVertex {
    float x, y, z;
    float nx, ny, nz;
};

// Triangle indices (std430, 16 B — padding keeps std430 alignment honest)
struct Triangle {
    unsigned int v0, v1, v2, padding;
};

// Wireframe UBO (std140, 32 B)
struct WireframeUBO {
    int   outputWidth, outputHeight, numTriangles, thickness;
    float colorR, colorG, colorB, colorA;
};

// Phong UBO (std140, 64 B — must be 16-byte aligned)
struct PhongUBO {
    int   outputWidth, outputHeight, numTriangles, pad1;
    float colorR, colorG, colorB, colorA;
    float lightDirX, lightDirY, lightDirZ;
    float ambient;
    float diffuseK, specularK, shininess;
    float pad2;
};
```

The `pad1` / `pad2` slots in `PhongUBO` enforce the `std140` 16-byte vec4
alignment rule between adjacent member groups. Don't remove them; the
shader's offsets depend on them.

## The Phong shader

```glsl
#version 450

// One thread per triangle.  Vertices already in 2D screen-pixel space.
layout(local_size_x = 256) in;

// Output texture (input was copied here before dispatch)
layout(binding = 0, rgba32f) uniform image2D outputTexture;

struct PhongVertex {
    float x, y, z;
    float nx, ny, nz;
};
layout(std430, binding = 1) readonly buffer VertexBuffer {
    PhongVertex vertices[];
};

struct Triangle { uint v0, v1, v2, padding; };
layout(std430, binding = 2) readonly buffer TriangleBuffer {
    Triangle triangles[];
};

layout(std140, binding = 3) uniform PhongUBO {
    int   outputWidth, outputHeight, numTriangles, pad1;
    float colorR, colorG, colorB, colorA;
    float lightDirX, lightDirY, lightDirZ, ambient;
    float diffuseK, specularK, shininess, pad2;
} ubo;

layout(std430, binding = 4) buffer DepthBuffer {
    uint depthBuf[];
};

void main() {
    uint triIdx = gl_GlobalInvocationID.x;
    if (triIdx >= ubo.numTriangles) return;

    Triangle tri = triangles[triIdx];
    PhongVertex v0 = vertices[tri.v0];
    PhongVertex v1 = vertices[tri.v1];
    PhongVertex v2 = vertices[tri.v2];

    // Screen-space bounding box, clipped to output
    int minX = max(0, int(floor(min(min(v0.x, v1.x), v2.x))));
    int maxX = min(ubo.outputWidth  - 1, int(ceil(max(max(v0.x, v1.x), v2.x))));
    int minY = max(0, int(floor(min(min(v0.y, v1.y), v2.y))));
    int maxY = min(ubo.outputHeight - 1, int(ceil(max(max(v0.y, v1.y), v2.y))));
    if (maxX < minX || maxY < minY) return;
    if ((maxX - minX) > 2000 || (maxY - minY) > 2000) return;   // safety cap

    // Signed area = 2× triangle area; sign tells winding
    float area = (v1.x - v0.x)*(v2.y - v0.y) - (v2.x - v0.x)*(v1.y - v0.y);
    if (abs(area) < 1e-6) return;
    float invArea = 1.0 / area;

    vec3 lightDir = normalize(vec3(ubo.lightDirX, ubo.lightDirY, ubo.lightDirZ));

    for (int py = minY; py <= maxY; py++) {
        for (int px = minX; px <= maxX; px++) {
            float fpx = float(px) + 0.5;
            float fpy = float(py) + 0.5;

            // Edge-function barycentrics
            float w0 = ((v1.x - fpx)*(v2.y - fpy) - (v2.x - fpx)*(v1.y - fpy)) * invArea;
            float w1 = ((v2.x - fpx)*(v0.y - fpy) - (v0.x - fpx)*(v2.y - fpy)) * invArea;
            float w2 = 1.0 - w0 - w1;
            if (w0 < 0.0 || w1 < 0.0 || w2 < 0.0) continue;

            // Interpolate depth, atomic depth-test
            float z = w0*v0.z + w1*v1.z + w2*v2.z;
            uint zUint = floatBitsToUint(z + 100.0);      // +100 = positivity bias
            uint pixIdx = uint(py) * uint(ubo.outputWidth) + uint(px);
            uint oldZ = atomicMin(depthBuf[pixIdx], zUint);
            if (zUint > oldZ) continue;

            // Smooth normal at this pixel
            vec3 n = normalize(
                w0*vec3(v0.nx,v0.ny,v0.nz) +
                w1*vec3(v1.nx,v1.ny,v1.nz) +
                w2*vec3(v2.nx,v2.ny,v2.nz));

            // Blinn-Phong, two-sided
            float NdotL = abs(dot(n, -lightDir));
            vec3  H     = normalize(-lightDir + vec3(0.0, 0.0, 1.0));   // view along -Z
            float NdotH = abs(dot(n, H));
            float spec  = pow(NdotH, ubo.shininess);
            float intensity = min(1.0, ubo.ambient + ubo.diffuseK*NdotL + ubo.specularK*spec);

            vec3 litColor = vec3(ubo.colorR, ubo.colorG, ubo.colorB) * intensity;
            float alpha = ubo.colorA;
            vec4 existing = imageLoad(outputTexture, ivec2(px, py));
            vec4 blended = vec4(mix(existing.rgb, litColor, alpha),
                                max(existing.a, alpha));
            imageStore(outputTexture, ivec2(px, py), blended);
        }
    }
}
```

Notable implementation decisions:

- **`abs(dot)` for two-sided shading.** A mesh in AE's coordinate handedness
  is often inverted relative to your authoring software; the absolute
  value means the user doesn't have to manually flip normals to get a lit
  surface. The cost is that backface culling is no longer free — see
  [Triangle Rasterization: Backface culling](../rendering/triangle-rasterization-compute.md#backface-culling).
- **View vector `(0, 0, 1)`.** The shader assumes view along −Z. If your
  camera math hands you a different convention, transform the light
  direction on the CPU before upload rather than parameterising the view
  vector in the UBO.
- **Alpha composite on `imageLoad` → `imageStore`.** The output image is
  bound as `STORAGE_IMAGE` (read/write); the rasterizer composites
  directly. No second target needed.

## The wireframe shader

```glsl
#version 450
layout(local_size_x = 256) in;

layout(binding = 0, rgba32f) uniform image2D outputTexture;

struct Vertex { float x, y; };
layout(std430, binding = 1) readonly buffer VertexBuffer { Vertex vertices[]; };

struct Triangle { uint v0, v1, v2, padding; };
layout(std430, binding = 2) readonly buffer TriangleBuffer { Triangle triangles[]; };

layout(std140, binding = 3) uniform UniformBuffer {
    int   outputWidth, outputHeight, numTriangles, thickness;
    float colorR, colorG, colorB, colorA;
} ubo;

// Bresenham with thickness + alpha composite + iteration cap
void drawLine(int x0, int y0, int x1, int y1) {
    int dx = abs(x1 - x0), dy = abs(y1 - y0);
    int sx = x0 < x1 ? 1 : -1;
    int sy = y0 < y1 ? 1 : -1;
    int err = dx - dy;
    int half_thick = ubo.thickness / 2;
    vec4 wire_color = vec4(ubo.colorR, ubo.colorG, ubo.colorB, ubo.colorA);

    for (int iter = 0; iter < 16000; iter++) {    // hard cap — see safety section
        for (int ty = -half_thick; ty <= half_thick; ty++) {
            for (int tx = -half_thick; tx <= half_thick; tx++) {
                int px = x0 + tx, py = y0 + ty;
                if (px >= 0 && px < ubo.outputWidth && py >= 0 && py < ubo.outputHeight) {
                    vec4 existing = imageLoad(outputTexture, ivec2(px, py));
                    vec4 blended  = mix(existing, vec4(wire_color.rgb, existing.a), wire_color.a);
                    imageStore(outputTexture, ivec2(px, py), blended);
                }
            }
        }
        if (x0 == x1 && y0 == y1) break;
        int e2 = 2 * err;
        if (e2 > -dy) { err -= dy; x0 += sx; }
        if (e2 <  dx) { err += dx; y0 += sy; }
    }
}

void main() {
    uint triIdx = gl_GlobalInvocationID.x;
    if (triIdx >= ubo.numTriangles) return;

    Triangle tri = triangles[triIdx];
    Vertex v0 = vertices[tri.v0], v1 = vertices[tri.v1], v2 = vertices[tri.v2];

    // Cull entirely-offscreen triangles
    float minX = min(min(v0.x,v1.x),v2.x), maxX = max(max(v0.x,v1.x),v2.x);
    float minY = min(min(v0.y,v1.y),v2.y), maxY = max(max(v0.y,v1.y),v2.y);
    if (maxX < 0 || minX >= ubo.outputWidth || maxY < 0 || minY >= ubo.outputHeight) return;
    if (maxX - minX > 2000.0 || maxY - minY > 2000.0) return;   // safety cap

    drawLine(int(v0.x), int(v0.y), int(v1.x), int(v1.y));
    drawLine(int(v1.x), int(v1.y), int(v2.x), int(v2.y));
    drawLine(int(v2.x), int(v2.y), int(v0.x), int(v0.y));
}
```

The wireframe shader is intentionally **depth-test-free**: a wireframe over
shaded geometry is supposed to be visible regardless of occlusion. If you
want depth-occluded wireframes, give it the same depth SSBO binding as
Phong and gate writes with `if (zUint > depthBuf[pixIdx]) continue;`.

## Shader compilation and embedding

Compile GLSL to SPIR-V at build time, then convert SPIR-V to a C++ header
that lives in your plugin binary. No runtime file dependency.

```batch
glslc phong.comp     -o phong.spv
glslc wireframe.comp -o wireframe.spv
python spv_to_header.py phong.spv     phong_spv     src/phong_spv.h
python spv_to_header.py wireframe.spv wireframe_spv src/wireframe_spv.h
```

The header-generator is ~30 lines of Python:

```python
import sys, os
spv_path, array_name, header_path = sys.argv[1], sys.argv[2], sys.argv[3]
with open(spv_path, "rb") as f: data = f.read()
with open(header_path, "w") as f:
    f.write(f"#pragma once\n\n")
    f.write(f"static const unsigned char {array_name}[] = {{\n")
    for i, byte in enumerate(data):
        if i % 16 == 0: f.write("    ")
        f.write(f"0x{byte:02x},")
        f.write("\n" if i % 16 == 15 else " ")
    f.write("\n};\nstatic const unsigned int {0}_len = {1};\n".format(array_name, len(data)))
```

Load at plugin startup:

```cpp
#include "phong_spv.h"

VkShaderModuleCreateInfo info = { VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO };
info.codeSize = phong_spv_len;
info.pCode    = reinterpret_cast<const uint32_t*>(phong_spv);
vkCreateShaderModule(device, &info, nullptr, &m_phongShader);
```

## Descriptor-set layouts

### Phong — 5 bindings

```cpp
VkDescriptorSetLayoutBinding b[5] = {};
b[0].binding = 0; b[0].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_IMAGE;   // output
b[1].binding = 1; b[1].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;  // vertices
b[2].binding = 2; b[2].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;  // triangles
b[3].binding = 3; b[3].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;  // UBO
b[4].binding = 4; b[4].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;  // depth
for (auto& x : b) { x.descriptorCount = 1; x.stageFlags = VK_SHADER_STAGE_COMPUTE_BIT; }

VkDescriptorSetLayoutCreateInfo layoutInfo = { VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO };
layoutInfo.bindingCount = 5;
layoutInfo.pBindings    = b;
vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &phongDescriptorSetLayout);
```

### Wireframe — 4 bindings

Same as Phong without the depth SSBO. Identical pipeline-layout
construction.

## Two-pixel-format pipelines (8-bit / 32-bit float)

AE delivers 8-bit (`PF_PixelFormat_ARGB32`) or 32-bit float
(`PF_PixelFormat_ARGB128`) buffers. Compile the shader twice — once with
`layout(binding = 0, rgba32f)`, once with `layout(binding = 0, rgba8)` —
and create two pipelines sharing one pipeline layout:

```cpp
VkComputePipelineCreateInfo pi = { VK_STRUCTURE_TYPE_COMPUTE_PIPELINE_CREATE_INFO };
pi.stage.sType  = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
pi.stage.stage  = VK_SHADER_STAGE_COMPUTE_BIT;
pi.stage.pName  = "main";
pi.layout       = m_phongPipelineLayout;

pi.stage.module = m_phongShader;        // rgba32f variant
vkCreateComputePipelines(device, VK_NULL_HANDLE, 1, &pi, nullptr, &m_phongPipeline);

pi.stage.module = m_phongShader8bit;    // rgba8 variant
vkCreateComputePipelines(device, VK_NULL_HANDLE, 1, &pi, nullptr, &m_phongPipeline8bit);
```

At dispatch time pick by pixel format:

```cpp
bool is_float = (output->rowbytes / output->width) >= 16;
VkPipeline pipe = is_float ? m_phongPipeline : m_phongPipeline8bit;
VkFormat   fmt  = is_float ? VK_FORMAT_R32G32B32A32_SFLOAT : VK_FORMAT_R8G8B8A8_UNORM;
```

## The CPU side: feeding the shaders

Two pieces are common to every dispatch: **per-vertex normals** (for
Phong) and **vertex packing**.

### Per-vertex normals, area-weighted

Sum each face's geometric normal (the cross product, magnitude = 2 × face
area) into each of its three vertices, then normalise:

```cpp
std::vector<float> normals(numVerts * 3, 0.0f);
for (int f = 0; f < numFaces; f++) {
    int i0 = faces[f*3 + 0], i1 = faces[f*3 + 1], i2 = faces[f*3 + 2];

    float e1x = verts_3d[i1*3]   - verts_3d[i0*3];
    float e1y = verts_3d[i1*3+1] - verts_3d[i0*3+1];
    float e1z = verts_3d[i1*3+2] - verts_3d[i0*3+2];
    float e2x = verts_3d[i2*3]   - verts_3d[i0*3];
    float e2y = verts_3d[i2*3+1] - verts_3d[i0*3+1];
    float e2z = verts_3d[i2*3+2] - verts_3d[i0*3+2];

    float nx = e1y*e2z - e1z*e2y;     // |cross| = 2 × area, so this naturally area-weights
    float ny = e1z*e2x - e1x*e2z;
    float nz = e1x*e2y - e1y*e2x;

    normals[i0*3]+=nx; normals[i0*3+1]+=ny; normals[i0*3+2]+=nz;
    normals[i1*3]+=nx; normals[i1*3+1]+=ny; normals[i1*3+2]+=nz;
    normals[i2*3]+=nx; normals[i2*3+1]+=ny; normals[i2*3+2]+=nz;
}
for (int v = 0; v < numVerts; v++) {
    float len = std::sqrt(normals[v*3]*normals[v*3]
                        + normals[v*3+1]*normals[v*3+1]
                        + normals[v*3+2]*normals[v*3+2]);
    if (len > 1e-8f) { normals[v*3]/=len; normals[v*3+1]/=len; normals[v*3+2]/=len; }
}
```

Area-weighting (as opposed to uniform-weighting) gives smoother shading on
meshes with unequal triangle sizes — large triangles contribute their
normal more heavily, which is what you want for sparse-mesh smoothing.

### Pack the GPU buffer

```cpp
std::vector<PhongVertex> phong_verts(numVerts);
for (int v = 0; v < numVerts; v++) {
    phong_verts[v] = {
        verts_2d[v*2], verts_2d[v*2+1],         // screen-space pixels
        verts_3d[v*3+2],                         // depth (your z metric)
        normals[v*3], normals[v*3+1], normals[v*3+2]
    };
}

std::vector<WireVertex> wire_verts(numVerts);
for (int v = 0; v < numVerts; v++)
    wire_verts[v] = { verts_2d[v*2], verts_2d[v*2+1] };
```

`verts_2d` is in **output pixel coordinates** (0 ≤ x < outputWidth,
0 ≤ y < outputHeight). `verts_3d.z` is your depth metric — see
[AE Camera Pipeline](../coordinates/ae-camera-pipeline.md) for how to
get there from AE's row-vector matrices.

### Light direction from AE

Pull from [AE Lights](../effects/ae-lights.md) and convert to view space:

```cpp
// Take the first PARALLEL light from the comp (or fall back to a default)
float light_world[3];
GetFirstParallelLightDirection(in_data, time, light_world);   // your helper

// Transform world-space light direction into the same view space as the verts
TransformDirection(viewMatrix, light_world, light_view);

// Normalise — the UBO expects unit length
float len = std::sqrt(light_view[0]*light_view[0]
                    + light_view[1]*light_view[1]
                    + light_view[2]*light_view[2]);
if (len > 1e-8f) { light_view[0]/=len; light_view[1]/=len; light_view[2]/=len; }
```

If no scene light exists, a sensible default that highlights front-facing
surfaces from above-left is `{−0.2, −0.3, −1.0}` then normalised.

## The dispatch: combined Phong + wireframe

End-to-end command-buffer recording for the most-complex case (shaded
mesh with wireframe overlay):

```cpp
// 1. Convert AE's ARGB to RGBA, upload to a storage image
ConvertARGB_to_RGBA(input_argb, w, h, rowbytes, rgbaBuffer.data(), is_float);
CreateImage(w, h, fmt, image, imageMem);
CreateImageView(image, fmt, imageView);
UploadToImage(image, w, h, rgbaBuffer.data(), imageBytes, fmt);

// 2. Upload shared triangle buffer
UploadToBuffer(triangles.data(),
               triangles.size() * sizeof(Triangle),
               VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
               triBuf, triMem);

// 3. Phong: vertices, UBO, depth SSBO, descriptor set
UploadToBuffer(phong_verts.data(),
               phong_verts.size() * sizeof(PhongVertex),
               VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
               phongVertBuf, phongVertMem);

PhongUBO pubo = { w, h, (int)triangles.size(), 0,
                  color[0], color[1], color[2], color[3],
                  light[0], light[1], light[2],
                  ambient, kd, ks, shininess, 0 };
UploadToBuffer(&pubo, sizeof(pubo), VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, phongUBO, phongUBOMem);

VkDeviceSize depthBytes = (VkDeviceSize)w * h * sizeof(uint32_t);
AllocateHostVisibleBuffer(depthBytes,
    VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT,
    depthBuf, depthMem);

AllocateDescriptorSet(phongDescriptorSetLayout, phongDescSet);
// vkUpdateDescriptorSets with 5 writes: image, phongVerts, tris, ubo, depth

// 4. Wireframe: vertices, UBO, descriptor set
UploadToBuffer(wire_verts.data(),
               wire_verts.size() * sizeof(WireVertex),
               VK_BUFFER_USAGE_STORAGE_BUFFER_BIT,
               wireVertBuf, wireVertMem);

WireframeUBO wubo = { w, h, (int)triangles.size(), thickness,
                      color[0], color[1], color[2], color[3] };
UploadToBuffer(&wubo, sizeof(wubo), VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, wireUBO, wireUBOMem);

AllocateDescriptorSet(wireframeDescriptorSetLayout, wireDescSet);
// vkUpdateDescriptorSets with 4 writes: image, wireVerts, tris, ubo

// 5. Record + submit
VkCommandBuffer cmdBuf;
BeginSingleTimeCommands(cmdBuf);

// Clear the depth SSBO to +infinity (uint max)
vkCmdFillBuffer(cmdBuf, depthBuf, 0, depthBytes, 0xFFFFFFFF);

VkBufferMemoryBarrier fillB = { VK_STRUCTURE_TYPE_BUFFER_MEMORY_BARRIER };
fillB.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
fillB.dstAccessMask = VK_ACCESS_SHADER_READ_BIT | VK_ACCESS_SHADER_WRITE_BIT;
fillB.srcQueueFamilyIndex = fillB.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
fillB.buffer = depthBuf; fillB.offset = 0; fillB.size = VK_WHOLE_SIZE;
vkCmdPipelineBarrier(cmdBuf,
    VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
    0, 0, nullptr, 1, &fillB, 0, nullptr);

// Phong pass
uint32_t groups = ((uint32_t)triangles.size() + 255) / 256;
vkCmdBindPipeline(cmdBuf, VK_PIPELINE_BIND_POINT_COMPUTE,
                  is_float ? m_phongPipeline : m_phongPipeline8bit);
vkCmdBindDescriptorSets(cmdBuf, VK_PIPELINE_BIND_POINT_COMPUTE,
                        m_phongPipelineLayout, 0, 1, &phongDescSet, 0, nullptr);
vkCmdDispatch(cmdBuf, groups, 1, 1);

// Compute → compute barrier on the shared output image
VkMemoryBarrier passB = { VK_STRUCTURE_TYPE_MEMORY_BARRIER };
passB.srcAccessMask = VK_ACCESS_SHADER_WRITE_BIT;
passB.dstAccessMask = VK_ACCESS_SHADER_READ_BIT | VK_ACCESS_SHADER_WRITE_BIT;
vkCmdPipelineBarrier(cmdBuf,
    VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT, VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
    0, 1, &passB, 0, nullptr, 0, nullptr);

// Wireframe pass
vkCmdBindPipeline(cmdBuf, VK_PIPELINE_BIND_POINT_COMPUTE,
                  is_float ? m_wireframePipeline : m_wireframePipeline8bit);
vkCmdBindDescriptorSets(cmdBuf, VK_PIPELINE_BIND_POINT_COMPUTE,
                        m_wireframePipelineLayout, 0, 1, &wireDescSet, 0, nullptr);
vkCmdDispatch(cmdBuf, groups, 1, 1);

EndSingleTimeCommands(cmdBuf);

// 6. Read back, convert RGBA → ARGB into the AE output buffer
DownloadFromImage(image, w, h, rgbaBuffer.data(), imageBytes, fmt);
ConvertRGBA_to_ARGB(rgbaBuffer.data(), w, h, output_argb, rowbytes, is_float);
```

For Phong-only or wireframe-only, drop the unneeded passes and the
compute-to-compute barrier between them. The depth fill + barrier is only
needed when Phong runs.

## CPU fallback

Vulkan can fail to initialise — driver missing, GPU lost, plugin running
on a machine without Vulkan-capable hardware. Keep a CPU rasterizer ready
as a fallback for at least one mode. The same edge-function topology
ports almost directly:

```cpp
// CPU equivalent of the Phong inner loop (per-triangle)
std::vector<uint32_t> zbuf(width * height, 0xFFFFFFFF);

for (size_t ti = 0; ti < tris.size(); ti++) {
    // ... same bounding box, area, invArea as the shader ...

    for (int py = minY; py <= maxY; py++) {
        for (int px = minX; px <= maxX; px++) {
            // ... same barycentric setup ...

            float z = w0*z0 + w1*z1 + w2*z2;
            uint32_t zUint;
            float zBiased = z + 100.0f;
            memcpy(&zUint, &zBiased, sizeof(uint32_t));   // bit-cast, no aliasing UB
            int pidx = py * width + px;
            if (zUint > zbuf[pidx]) continue;
            zbuf[pidx] = zUint;

            // ... interpolate, shade, alpha composite into ARGB buffer ...
        }
    }
}
```

The convention that keeps GPU and CPU paths swappable is **smaller z =
closer**, with the same `+100.0` positivity bias and the same uint
encoding. If you mix CPU and GPU passes in a single frame, this
convention is what makes the depth buffers compatible.

For the wireframe CPU fallback, the same Bresenham loop in C++ works
unchanged; you can drop the GLSL safety cap (CPU hangs are recoverable).

## What you'd change for a production-grade implementation

The shaders above prioritise being **simple, portable, and easy to
debug**. A few directions to push further once the basics work:

### Move shading to a graphics pipeline

By far the biggest single win available. A graphics pipeline with a real
depth attachment and a fragment shader replaces all of the atomic-uint
depth machinery with hardware Z, and replaces the per-pixel triangle
walk with hardware rasterization. The shading math is unchanged. Trade:
~200 lines of additional Vulkan setup (render pass, framebuffer, image
layout transitions) for an order-of-magnitude per-pixel speedup. Keep the
wireframe pass on compute — line primitives don't benefit from the
graphics pipeline.

### Atomic floats for the depth buffer

If `VK_KHR_shader_atomic_float` is available on the device:

```glsl
#extension GL_EXT_shader_atomic_float : require
layout(std430, binding = 4) buffer DepthBuffer { float depthBuf[]; };

float z = w0*v0.z + w1*v1.z + w2*v2.z;
float oldZ = atomicMin(depthBuf[pixIdx], z);
if (z > oldZ) continue;
```

Removes the `+100.0` bias hack and supports negative depths cleanly.
Initialise the buffer with `vkCmdFillBuffer` and `floatBitsToUint(+INF)`
instead of `0xFFFFFFFF`. Fall back to the bit-cast path if the extension
isn't supported.

### Depth prepass

For heavy shading (textured PBR, environment lookups, expensive
materials), run two dispatches:

1. Depth-only pass — every thread still walks the bounding box, but
   only updates the depth SSBO. No shading, no color writes.
2. Shading pass — reads the now-final depth, only shades where
   `zUint == depthBuf[pixIdx]`.

Eliminates overdraw of expensive shading work at the cost of an extra
dispatch and a pipeline barrier. Worth it when shading is > ~5× the cost
of the depth test.

### SDF wireframe

Replace Bresenham with a signed-distance line test for the wireframe.
Gives free analytical antialiasing and removes the iteration cap:

```glsl
float distToSegment(vec2 p, vec2 a, vec2 b) {
    vec2 pa = p - a, ba = b - a;
    float h = clamp(dot(pa, ba) / dot(ba, ba), 0.0, 1.0);
    return length(pa - ba * h);
}

// Inside the triangle bbox, per pixel:
float d = min(min(distToSegment(p, v0xy, v1xy),
                  distToSegment(p, v1xy, v2xy)),
                  distToSegment(p, v2xy, v0xy));
float alpha = (1.0 - smoothstep(thickness - 0.5, thickness + 0.5, d)) * ubo.colorA;
if (alpha < 1e-3) continue;
```

Thick wireframes look like rounded segments rather than stepped squares,
and edges no longer alias as the camera moves.

### Tile binning for huge meshes

If you scale past ~50k triangles, the load imbalance from one
oversized triangle blocking a workgroup starts to dominate. Add a
binning prepass that sorts triangles into screen tiles, then dispatch
one workgroup per tile rasterizing only its bin. See
[Triangle Rasterization: Tile binning](../rendering/triangle-rasterization-compute.md#tile-binning-for-large-triangles).

### Indirect dispatch with culling

For animated meshes where many triangles are off-screen or back-facing,
run a culling compute pass that writes survivors to a compacted SSBO and
a triangle counter, then `vkCmdDispatchIndirect` against the counter.
Saves all the bounding-box work on rejected triangles.

### Per-vertex normals on the GPU

The CPU-side normal computation is fine for static meshes but a
bottleneck for heavy deformation (skinning, blendshapes). A precompute
compute pass that takes raw 3D vertices and outputs the PhongVertex
buffer keeps the CPU out of the per-frame critical path.

### MFR-safe descriptor pool

After Effects runs SmartFX renders across multiple threads
(MFR — multi-frame rendering). Your descriptor pool and command-pool
strategy needs to be either per-thread or mutex-guarded. The reference
implementation uses:

- One shared descriptor pool created with
  `VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT`, guarded by a
  mutex.
- Per-thread command pools, looked up by `std::this_thread::get_id()`.

See [Vulkan Architecture](../guides/vulkan-architecture.md) for the full
MFR-safe pattern.

### Push constants instead of UBO

For the small structs above (32 / 64 B), Vulkan push constants are
faster than a UBO upload — no buffer allocation, no descriptor update,
the data goes directly in the command stream. Limits: 128 B on most
desktop drivers, 256 B on newer hardware. Trivial to migrate; switch
the UBO declaration to:

```glsl
layout(push_constant) uniform PhongPC { ... } pc;
```

and replace `UploadToBuffer + vkUpdateDescriptorSets` with
`vkCmdPushConstants(cmdBuf, layout, VK_SHADER_STAGE_COMPUTE_BIT, 0, sizeof(pc), &pc)`.

## Related pages

- [Triangle Rasterization in Compute Shaders](../rendering/triangle-rasterization-compute.md)
  — the general pattern, with theory and alternative topologies.
- [AE Camera Pipeline](../coordinates/ae-camera-pipeline.md) — how to
  produce the screen-pixel-plus-depth verts the shader consumes.
- [AE Camera Matrix Conventions](../coordinates/ae-camera-matrix-conventions.md)
  — row-vector / column-vector clarifier; required before any
  camera-matrix code touches a shader.
- [AE Lights](../effects/ae-lights.md) — fetch and falloff math for the
  scene lights you'll feed into the Phong UBO.
- [Vulkan Architecture](../guides/vulkan-architecture.md) — plugin-local
  Vulkan instance, MFR-safe command and descriptor pools, delay-loaded
  `vulkan-1.dll` on Windows.
- [GPU Memory](gpu-memory.md) — buffer / image allocation, host-visible
  vs device-local, alignment requirements.
