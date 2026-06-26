# Triangle Rasterization in Compute Shaders

When an After Effects plugin needs to draw 3D geometry into a 2D output buffer
— wireframes, shaded meshes, projected guides, contour overlays — there are
three rasterization paths to choose from:

1. **AE's host renderer.** Not available to effects: AE only renders scene
   layers, not arbitrary geometry from inside an effect.
2. **Vulkan graphics pipeline** (vertex shader → fixed-function rasterizer →
   fragment shader, with a `VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL`
   depth attachment). The fastest path, but requires a graphics-capable
   queue, render passes, framebuffers, and image transitions to/from your
   pixel buffer.
3. **Vulkan compute pipeline** with a software rasterizer. Slower per pixel,
   but uses one queue, one descriptor set, one storage image, and ships in
   any Vulkan profile that supports compute (i.e. all of them). The pattern
   composites cleanly with the rest of an AE effect's pixel work, which is
   typically also compute.

This page covers the **compute-shader software-rasterization pattern**: the
one-thread-per-triangle topology, software z-buffering via atomic operations
on a `uint` SSBO, the memory-barrier ordering that's required, the safety
caps that keep a single bad vertex from hanging the GPU, and what's left on
the table for a more performant implementation.

The reference implementation in this guide is HEADCASE's Phong + wireframe
compute shaders; the same pattern is widely applicable to any AE plugin that
needs to rasterize its own geometry from inside a compute dispatch.

## Why compute, not graphics

The graphics pipeline is faster and gives you free hardware depth testing.
You should pick it if any of these are true:

- You need MSAA — the compute approach is single-sample only.
- You have thousands of triangles per dispatch and each one covers many
  pixels — graphics is heavily optimised for that regime.
- You need texture sampling with mipmaps, anisotropic filtering, or sampler
  border modes — compute storage images don't have a sampler attached.

You should pick compute if any of these are true:

- You're already running compute for the rest of the effect (denoise, blur,
  composite) and want a single command-buffer flow.
- You have low triangle counts (hundreds to low thousands) or thin
  primitives (lines, dots, sparse meshes) where graphics-pipeline overhead
  dominates.
- You want to share a single output image between non-rasterization passes
  and the rasterizer without round-tripping through render passes and image
  layout transitions.
- You're writing 32-bit float (`R32G32B32A32_SFLOAT`) and want
  `imageLoad`/`imageStore` semantics rather than blending hardware.

For an AE effect plugin, the third bullet is usually the deciding one.

## One thread per triangle

The compute rasterizer assigns **one workgroup invocation to each
triangle**, not one to each pixel. Each thread reads its triangle's three
vertices, computes the screen-space bounding box, and iterates every pixel
inside that box, doing an inside-test via edge-function barycentrics.

```glsl
layout(local_size_x = 256, local_size_y = 1, local_size_z = 1) in;

void main() {
    uint triIdx = gl_GlobalInvocationID.x;
    if (triIdx >= ubo.numTriangles) return;

    Triangle tri = triangles[triIdx];
    Vertex v0 = vertices[tri.v0];
    Vertex v1 = vertices[tri.v1];
    Vertex v2 = vertices[tri.v2];

    int minX = max(0, int(floor(min(min(v0.x, v1.x), v2.x))));
    int maxX = min(width  - 1, int(ceil(max(max(v0.x, v1.x), v2.x))));
    int minY = max(0, int(floor(min(min(v0.y, v1.y), v2.y))));
    int maxY = min(height - 1, int(ceil(max(max(v0.y, v1.y), v2.y))));
    if (maxX < minX || maxY < minY) return;

    float area = (v1.x - v0.x)*(v2.y - v0.y) - (v2.x - v0.x)*(v1.y - v0.y);
    if (abs(area) < 1e-6) return;
    float invArea = 1.0 / area;

    for (int py = minY; py <= maxY; py++) {
        for (int px = minX; px <= maxX; px++) {
            float fpx = float(px) + 0.5;
            float fpy = float(py) + 0.5;
            float w0 = ((v1.x - fpx)*(v2.y - fpy) - (v2.x - fpx)*(v1.y - fpy)) * invArea;
            float w1 = ((v2.x - fpx)*(v0.y - fpy) - (v0.x - fpx)*(v2.y - fpy)) * invArea;
            float w2 = 1.0 - w0 - w1;
            if (w0 < 0.0 || w1 < 0.0 || w2 < 0.0) continue;
            // ... interpolate attributes, depth-test, shade, write ...
        }
    }
}
```

Dispatch is `ceil(numTriangles / 256)` workgroups of 256 threads each:

```cpp
vkCmdDispatch(cmdBuf, (numTriangles + 255) / 256, 1, 1);
```

### Why not one thread per pixel?

Pixel-per-thread requires either:

- Iterating every triangle for every pixel (`O(pixels × tris)` — useless
  for non-trivial meshes), or
- An acceleration structure (BVH or tile-bin) built per-dispatch on the
  CPU or in a prepass — significant complexity.

Triangle-per-thread is `O(tris × average_AABB_area)` — typically much less
work than the pixel-per-thread cross product, and it parallelises trivially.
The downside is **load imbalance**: a single huge triangle blocks one
workgroup while everyone else finishes. For meshes with extreme triangle
size variance, see [§Tile binning](#tile-binning-for-large-triangles) below.

## Software z-buffer via `atomicMin` on `uint` SSBO

Compute shaders can't bind a depth attachment. The pattern instead is to
declare a `uint` SSBO sized `width × height`, treat each entry as a depth
slot, and use `atomicMin` to resolve races:

```glsl
layout(std430, binding = 4) buffer DepthBuffer {
    uint depthBuf[];
};

// inside the per-pixel loop:
float z = w0*v0.z + w1*v1.z + w2*v2.z;       // interpolated depth
uint zUint = floatBitsToUint(z + 100.0);     // bit-cast (see below)
uint pixIdx = uint(py) * uint(width) + uint(px);
uint oldZ  = atomicMin(depthBuf[pixIdx], zUint);
if (zUint > oldZ) continue;                  // lost the race — don't shade
// ... shade and write ...
```

**The `+ 100.0` bias.** `floatBitsToUint` preserves ordering for **positive**
floats only — the IEEE-754 sign bit inverts comparison for negatives. A
fixed positive bias guarantees every depth value in the working range
becomes a positive float before the cast. Pick the bias so that the
smallest expected depth (closest to camera) is still safely > 0. For a
camera at z=0 looking down −z with geometry typically at z ∈ [−50, 50],
`+100.0` works; for larger scenes scale accordingly.

**Why uint and not float?** Vulkan core requires `atomicMin` for integer
SSBO types. Atomic float operations exist as the optional extension
`VK_KHR_shader_atomic_float`, but it's not universally available, and on
older drivers it's slower than the bit-cast trick. The uint approach works
everywhere.

**The race between depth test and shade.** Two threads can both pass the
`zUint > oldZ` check if they're shading near-coplanar fragments — the
depth buffer ends up holding the closer of the two, but the color writes
race independently. In practice the resulting pixel-level z-fight is
invisible for opaque single-pass shading; for layered transparency or
sharp depth transitions you need a depth prepass (see
[§Depth prepass](#depth-prepass-for-shading-cost-reduction) below).

## The mandatory barriers

A compute rasterizer with a software z-buffer needs **two pipeline
barriers** that are easy to forget and produce silent corruption when
omitted.

### Clear-to-shader barrier

The depth SSBO must be reset to "infinity" (`0xFFFFFFFF`) every dispatch.
The clear is typically a `vkCmdFillBuffer`, which lives in the
`TRANSFER` stage; the rasterizer reads/writes the same buffer in the
`COMPUTE_SHADER` stage. A pipeline barrier between them is required, or
the shader can start reading before the fill commits:

```cpp
vkCmdFillBuffer(cmdBuf, depthBuf, 0, depthSize, 0xFFFFFFFF);

VkBufferMemoryBarrier b = { VK_STRUCTURE_TYPE_BUFFER_MEMORY_BARRIER };
b.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
b.dstAccessMask = VK_ACCESS_SHADER_READ_BIT | VK_ACCESS_SHADER_WRITE_BIT;
b.srcQueueFamilyIndex = b.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
b.buffer = depthBuf; b.offset = 0; b.size = VK_WHOLE_SIZE;

vkCmdPipelineBarrier(cmdBuf,
    VK_PIPELINE_STAGE_TRANSFER_BIT,
    VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
    0, 0, nullptr, 1, &b, 0, nullptr);
```

Symptom when missing: stale depth from prior frame leaks through; surfaces
intermittently punch through each other; the bug seems to "go away" on
faster GPUs because the fill happens to win the race.

### Pass-to-pass barrier

If you run a shaded pass and then a wireframe (or any second pass) writing
to the same output image, a compute → compute memory barrier is required
between them:

```cpp
VkMemoryBarrier m = { VK_STRUCTURE_TYPE_MEMORY_BARRIER };
m.srcAccessMask = VK_ACCESS_SHADER_WRITE_BIT;
m.dstAccessMask = VK_ACCESS_SHADER_READ_BIT | VK_ACCESS_SHADER_WRITE_BIT;
vkCmdPipelineBarrier(cmdBuf,
    VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
    VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
    0, 1, &m, 0, nullptr, 0, nullptr);
```

Symptom when missing: wireframe-over-shaded mode renders with the
wireframe partially under the shaded surface, randomly, on certain GPUs.

## Safety caps

A single garbage vertex — caused by a near-zero `w` in projection, a NaN
in a tracking input, or a degenerate animated keyframe — can produce a
bounding box of millions of pixels or a Bresenham line that loops forever.
On the GPU, both lock up the entire kernel until the OS resets the device.
Two cheap defensive caps prevent this:

**AABB cap.** After clipping the bbox to the output, reject any triangle
whose final bbox is larger than a sane upper bound:

```glsl
if ((maxX - minX) > 2000 || (maxY - minY) > 2000) return;
```

The threshold should be comfortably larger than your max plausible
on-screen triangle (≈ output diagonal) but small enough that a runaway
projection gets caught.

**Line-iteration cap.** Bresenham line drawing in compute should always
have a hard iteration limit:

```glsl
for (int iter = 0; iter < 16000; iter++) {
    // ... step ...
    if (x0 == x1 && y0 == y1) break;
}
```

Pick the cap above your max possible line length in pixels (≈ output
diagonal × 1.5 for safety). The cost of capping is zero in normal
operation; the cost of not capping is a GPU hang.

## Putting it together

Minimal dispatch shape for a single Phong-style pass:

```cpp
// 1. Begin a one-shot command buffer
BeginSingleTimeCommands(cmdBuf);

// 2. Clear the depth SSBO to +infinity (uint max)
vkCmdFillBuffer(cmdBuf, depthBuf, 0, depthSize, 0xFFFFFFFF);
vkCmdPipelineBarrier(cmdBuf,
    VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
    0, 0, nullptr, 1, &fillBarrier, 0, nullptr);

// 3. Bind the rasterizer pipeline + descriptor set
vkCmdBindPipeline(cmdBuf, VK_PIPELINE_BIND_POINT_COMPUTE, m_pipeline);
vkCmdBindDescriptorSets(cmdBuf, VK_PIPELINE_BIND_POINT_COMPUTE,
    m_pipelineLayout, 0, 1, &descSet, 0, nullptr);

// 4. One workgroup per 256 triangles
uint32_t groups = (numTris + 255) / 256;
vkCmdDispatch(cmdBuf, groups, 1, 1);

// 5. Submit
EndSingleTimeCommands(cmdBuf);
```

The descriptor set typically holds 4 or 5 bindings:

| Binding | Type | Purpose |
|---------|------|---------|
| 0 | `STORAGE_IMAGE` | output (read/write) |
| 1 | `STORAGE_BUFFER` | vertex SSBO (readonly) |
| 2 | `STORAGE_BUFFER` | triangle index SSBO (readonly) |
| 3 | `UNIFORM_BUFFER` | UBO (dimensions, color, shading params) |
| 4 | `STORAGE_BUFFER` | depth SSBO (read/write) — omit for pure wireframe |

`std430` and `std140` packing rules apply — the C++ structs and GLSL
structs must match exactly, including padding. See
[Vulkan buffer alignment](../guides/vulkan-architecture.md) for layout
details.

## Theoretical improvements

The pattern above prioritises **simplicity, portability, and integration
with non-rasterizer compute passes**. None of the following are necessary
for correctness, but several can yield significant speedups on production
workloads.

### Hardware rasterization for the shading pass

The single biggest win available, by a wide margin. Move the shaded
(Phong / textured) pass to a graphics pipeline with a real depth
attachment and a fragment shader. Keep the wireframe pass on compute so
you don't pay graphics-pipeline overhead for thin primitives, and use the
same storage image — graphics writes to it via an image-store fragment
shader, compute reads/writes it directly.

Trade-off: requires a graphics queue family, render-pass / framebuffer
management, and image-layout transitions between the two passes. About
200 lines of additional Vulkan setup, but the per-pixel cost drops by an
order of magnitude or more on modern hardware.

### Tile binning for large triangles

Replace one-thread-per-triangle with a two-pass approach: first pass bins
triangles into screen-space tiles (e.g. 16×16 or 32×32), second pass
dispatches one workgroup per tile and rasterizes only the triangles
binned into it. Eliminates load imbalance from oversized triangles and
makes the per-pixel work coherent within a workgroup, which can improve
texture-cache behaviour for textured variants.

Standard reference: cwfletcher / Laine & Karras "High-Performance
Software Rasterization on GPUs" (HPG 2011), specifically the "fine-grained
tiling" section. Adds significant complexity — only worth it for meshes
with > 50k triangles or pathological size distributions.

### Atomic floats

If `VK_KHR_shader_atomic_float` is available (most modern desktop drivers
since 2021), replace the bias-and-bit-cast trick with a direct
`atomicMin` on a `float` SSBO:

```glsl
#extension GL_EXT_shader_atomic_float : require
layout(std430, binding = 4) buffer DepthBuffer { float depthBuf[]; };

float z = w0*v0.z + w1*v1.z + w2*v2.z;
float oldZ = atomicMin(depthBuf[pixIdx], z);
if (z > oldZ) continue;
```

Cleaner, no bias to maintain, supports negative depth ranges natively. The
cost is a feature-availability check at device creation; on hardware that
doesn't support it, fall back to the bit-cast path.

### Depth prepass for shading-cost reduction

If the per-pixel shading cost is high (textured PBR, expensive procedural
materials, environment lookups), split the rasterizer into two dispatches:

1. **Depth-only pass.** Same per-pixel loop, but does only the atomic
   depth update — no shading, no color writes. Cheap.
2. **Shading pass.** Reads the now-final depth buffer, only shades the
   fragment if `zUint == depthBuf[pixIdx]` (i.e. it's the actual winner).

Eliminates overdraw of shading work. Costs an extra dispatch and a barrier
between them. Worth it when shading > ~5× the cost of depth-test alone.

### Conservative wireframe via signed-distance lines

Bresenham draws aliased lines at thickness > 1 as a stack of squares —
visually crude. Replace the per-pixel inside-test with a signed-distance
calculation against the line segment and shade with a smoothstep against
the desired thickness:

```glsl
float distToSegment(vec2 p, vec2 a, vec2 b) {
    vec2 pa = p - a, ba = b - a;
    float h = clamp(dot(pa, ba) / dot(ba, ba), 0.0, 1.0);
    return length(pa - ba * h);
}

// Per-pixel, inside the triangle bbox:
float d = min(min(distToSegment(p, v0, v1),
                  distToSegment(p, v1, v2)),
                  distToSegment(p, v2, v0));
float alpha = 1.0 - smoothstep(thickness - 0.5, thickness + 0.5, d);
```

Gives analytical antialiasing for free; thicker lines look like proper
rounded segments rather than stepped squares. Costs more arithmetic per
pixel but eliminates the Bresenham inner loop and the iteration cap.

### Indirect dispatch with frustum-cull compaction

For animated meshes where many triangles are off-screen or back-facing,
run a culling compute pass that writes surviving triangle indices to a
compacted SSBO and a counter, then issue `vkCmdDispatchIndirect` against
the counter. Saves all the per-pixel work from rejected triangles.
Implementation cost is one extra pipeline; the saving is proportional to
your cull ratio.

### Backface culling

If your geometry is closed and you don't need two-sided lighting:

```glsl
float area = (v1.x - v0.x)*(v2.y - v0.y) - (v2.x - v0.x)*(v1.y - v0.y);
if (area < 0.0) return;     // back-facing in this winding
```

Doubles throughput on closed meshes. Cannot be combined with `abs(dot)`
two-sided lighting — pick one.

### Per-vertex normals on the GPU

The reference implementation computes smooth per-vertex normals on the
CPU before each dispatch (sum of area-weighted face normals, then
renormalise). For meshes that don't deform per-frame this is fine; for
heavy deformation (skinning, blendshapes), move the normal computation
into a precompute dispatch so the CPU doesn't bottleneck the pipeline.

## When the pattern is the wrong choice

A few cases where you should reach for something else entirely:

- **Vector text or 2D shapes.** Use Skia, Pathfinder, or AE's own
  DrawbotSuite — they tessellate, rasterize, and antialias glyphs and
  paths far better than any per-triangle compute rasterizer.
- **Volumetrics / participating media.** Use ray-marching in compute, not
  triangle rasterization.
- **Heavy texturing with mipmaps.** Use the graphics pipeline. Storage
  images don't have samplers.
- **Output > ~8k pixels on a side.** The single-dispatch single-image
  model starts to thrash L2; consider tile-by-tile output buffers and
  multiple dispatches.

## Related pages

- [AE Camera Pipeline](../coordinates/ae-camera-pipeline.md) — how to get
  vertices from world space into the screen-pixel space the rasterizer
  consumes.
- [AE Camera Matrix Conventions](../coordinates/ae-camera-matrix-conventions.md)
  — the row-vector / column-vector gotchas you must clear before
  rasterizing anything.
- [AE Lights](../effects/ae-lights.md) — fetch and falloff math for the
  scene lights that drive shading inside the rasterizer.
- [Vulkan Architecture](../guides/vulkan-architecture.md) — instance,
  device, command pool, and descriptor-pool layout for a plugin-local
  Vulkan runtime.
- [Wireframe & Phong Compute Shaders](../gpu/wireframe-and-phong-compute.md)
  — a worked end-to-end example using this pattern.
