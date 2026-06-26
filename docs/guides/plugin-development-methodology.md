# Plugin Development Methodology
## A Guide to Building After Effects and OFX Plugins with AI Assistance

This document outlines a proven methodology for developing high-quality After Effects (AE) and OpenFX (OFX) plugins using AI agents like Claude Code.

---

## Core Principles

### 1. Quality In = Quality Out
The quality of your planning documents directly determines the quality of your output. Before writing any code:
- Invest time in detailed planning
- Be specific about technical implementation details
- Document coordinate systems, data flows, and edge cases
- **Never skip the planning phase**

### 2. Feature-Driven Development
Think in terms of discrete, testable features:
- Break your plugin into 3-5 core features
- Each feature should be independently testable
- Build and test one feature at a time
- Document expected behavior for each feature

### 3. Test-Driven Workflow
For each feature:
1. Implement the feature
2. Write a test for it
3. Only proceed to the next feature if the test passes
4. If test fails, fix the current feature before moving forward

---

## The Planning Process

### Step 1: Initial Feature List
Start by listing the core features your plugin needs:

**Example for a Camera Projection Plugin:**
- Feature 1: Load and parse Alembic geometry file
- Feature 2: Extract camera data from Alembic
- Feature 3: Project vertices using camera matrix
- Feature 4: Handle masks and alpha compositing
- Feature 5: Render projected geometry with textures

### Step 2: Use the Ask User Question Tool
When working with Claude Code, invoke detailed planning:

```
Read this plan file (or create one based on my description).

Interview me in detail using the ask user question tool about:
- Technical implementation choices
- UI/UX decisions
- Data structure choices
- Edge cases and error handling
- Performance considerations
- Platform-specific concerns (Windows/Mac)

Do not start building until you understand every detail.
```

This will trigger a detailed interview that covers:
- **Coordinate Systems**: Which coordinate spaces are you working in? (clip space, screen space, UV space, layer space)
- **Data Structures**: How will you store vertex data, camera matrices, etc.?
- **Platform Specifics**: CUDA vs Metal? D3D vs OpenGL?
- **Buffer Management**: Who owns memory? When is it released?
- **Threading**: Single-threaded or multi-threaded rendering?

### Step 3: Document Critical Technical Details

#### For After Effects Plugins:
Create a section in your PRD covering:

```markdown
## AE-Specific Technical Requirements

### Coordinate System Handling
- **Composition Space**: Full comp dimensions (e.g., 3840x2160)
- **Layer Space**: Usually matches comp, where UV coords map
- **Output Buffer Space**: May be cropped when masks applied
  - `output_origin_x/y`: Where buffer (0,0) sits in layer space
  - Buffer dims: `output->width/height`

### Mask Handling Strategy
- **PreRender Phase**: Store mask bounds from `in_result.result_rect`
- **SmartRender Phase**:
  - Dual checkout: unmasked texture + masked alpha
  - Calculate mask position: `mask_pos = masked_rect - output_origin`
  - Render to layer space, convert to buffer space
  - Apply mask alpha at buffer-relative position

### SmartFX Pipeline
- PreRender: Request checkout layers, store GPU data
- SmartRender: Retrieve GPU data, perform rendering
- Always checkout both masked and unmasked versions
- Handle output buffer cropping properly

### GPU Renderer Interface
- Input texture format: RGBA32F or ARGB format?
- Output buffer format: Match AE's expected format
- Coordinate mapping: UV (0,0)->(1,1) maps to layer dimensions, not buffer
```

#### For OFX Plugins:
```markdown
## OFX-Specific Technical Requirements

### Plugin Identity
- Name: "YourPlugin Name"
- ID: `com.yourcompany.pluginid`
- Bundle: `YourPlugin.ofx.bundle`
- Context: Filter, Generator, Transition?

### Host Compatibility
- Target host: DaVinci Resolve, Nuke, etc.
- Required capabilities: OpenGL, CUDA, Metal?
- Supported bit depths: 8-bit, 16-bit, 32-bit float?

### Action Sequence
- Describe -> DescribeInContext -> CreateInstance
- BeginInstanceChanged -> InstanceChanged -> EndInstanceChanged
- Render

### Coordinate Systems
- Source image bounds
- Output image bounds
- RoI (Region of Interest) handling
- RoD (Region of Definition) calculation

### Parameter Setup
- List each parameter with exact type
- Default values, min/max ranges
- UI hints and organization
- Which params trigger re-render vs re-description
```

### Step 4: Create PRD.md

Your PRD should follow this structure:

```markdown
# Plugin Name - Product Requirements Document

## Overview
[2-3 sentence description of what the plugin does]

## Core Features
1. **Feature Name**
   - Description: [What it does]
   - Input: [What data it needs]
   - Output: [What it produces]
   - Test Criteria: [How to verify it works]

2. **Feature Name**
   - ...

## Technical Architecture

### Coordinate Systems
[Detailed explanation of all coordinate spaces used]

### Data Flow
[How data moves through the plugin]
```
[Input] -> [Processing Steps] -> [Output]
```

### Critical Calculations
[Key algorithms, formulas, matrix math]

### Platform-Specific Details
- Windows: [Specific requirements]
- Mac: [Specific requirements]

### Error Handling
- Invalid input data
- Missing files
- Unsupported formats
- Out of memory conditions

## UI/UX Specifications

### Parameters
| Name | Type | Default | Min | Max | Description |
|------|------|---------|-----|-----|-------------|
| ... | ... | ... | ... | ... | ... |

### Layout
[How parameters are organized in the UI]

## Testing Strategy

### Per-Feature Tests
- Feature 1: [Specific test case]
- Feature 2: [Specific test case]

### Integration Tests
- [How features work together]

### Edge Cases
- Empty geometry
- Invalid camera data
- Extreme parameter values

## Known Constraints
- [Technical limitations]
- [Performance considerations]
- [Host-specific quirks]
```

---

## Development Workflow

### Without Automation (Learn First)

**For beginners: Build 1-2 plugins this way before using automation.**

1. **Feature 1: Plan**
   ```
   Let's implement Feature 1: [feature name]

   According to the PRD, this feature should:
   - [Requirement 1]
   - [Requirement 2]

   Please implement this feature.
   ```

2. **Feature 1: Test**
   ```
   Write a test to verify Feature 1 works correctly.
   The test should check:
   - [Expected behavior 1]
   - [Expected behavior 2]

   Run the test and show me the results.
   ```

3. **Feature 1: Verify**
   - If test passes -> Move to Feature 2
   - If test fails -> Fix Feature 1, retest

4. **Repeat** for each feature

### With Automation (RAPH Loop)

**Only use after you've successfully built at least one plugin manually.**

Requirements:
- Excellent PRD.md (from detailed planning)
- Progress.txt (for tracking)
- Test suite configuration

The RAPH loop will:
1. Read PRD.md feature list
2. Implement Feature N
3. Write test for Feature N
4. Run test
5. If pass -> Document in Progress.txt, move to Feature N+1
6. If fail -> Fix Feature N, re-test
7. Repeat until all features complete

**Warning**: If your PRD is incomplete or vague, RAPH will waste tokens building the wrong thing.

---

## Critical Technical Patterns

### After Effects: Mask Handling Pattern

```cpp
// 1. PreRender - Store mask info
PF_Err PreRender(PF_InData* in_data, PF_OutData* out_data, PF_PreRenderExtra* extra) {
    GPUPreRenderData* gpu_data = new GPUPreRenderData();

    // Store mask bounds in LAYER space
    gpu_data->masked_rect_left = in_result.result_rect.left;
    gpu_data->masked_rect_top = in_result.result_rect.top;
    gpu_data->masked_rect_right = in_result.result_rect.right;
    gpu_data->masked_rect_bottom = in_result.result_rect.bottom;

    extra->output->pre_render_data = gpu_data;
    return PF_Err_NONE;
}

// 2. SmartRender - Use mask info
PF_Err SmartRender(PF_InData* in_data, PF_OutData* out_data, PF_SmartRenderExtra* extra) {
    // Retrieve mask data
    GPUPreRenderData* gpu_data = (GPUPreRenderData*)extra->input->pre_render_data;
    A_long masked_rect_left = gpu_data->masked_rect_left;
    A_long masked_rect_top = gpu_data->masked_rect_top;

    // Get output origin (where buffer sits in layer space)
    int output_origin_x = (int)in_data->output_origin_x;
    int output_origin_y = (int)in_data->output_origin_y;

    // Calculate mask position in BUFFER coordinates
    int mask_pos_x = masked_rect_left - output_origin_x;
    int mask_pos_y = masked_rect_top - output_origin_y;

    // Dual checkout
    PF_CheckoutResult unmasked_texture;
    PF_CHECKOUT_PARAM(...); // Full texture without mask

    PF_EffectWorld* masked_alpha = NULL;
    extra->cb->checkout_layer_pixels(...); // Cropped masked buffer

    // Render
    RenderToLayerSpace(output, unmasked_texture,
                       in_data->width, in_data->height,  // Full layer dims
                       output_origin_x, output_origin_y); // Offset

    // Apply mask at buffer-relative position
    ApplyMaskAlpha(output, masked_alpha, mask_pos_x, mask_pos_y);

    return PF_Err_NONE;
}

// 3. Render - Map coordinates correctly
void RenderToLayerSpace(PF_LayerDef* output, Texture* texture,
                        int layer_width, int layer_height,
                        int output_origin_x, int output_origin_y) {
    for (each vertex) {
        // Calculate in LAYER space (full comp dimensions)
        float layer_x = vertex.uv_x * layer_width;
        float layer_y = vertex.uv_y * layer_height;

        // Convert to BUFFER space (subtract origin)
        int buffer_x = (int)layer_x - output_origin_x;
        int buffer_y = (int)layer_y - output_origin_y;

        // Bounds check against buffer
        if (buffer_x >= 0 && buffer_x < output->width &&
            buffer_y >= 0 && buffer_y < output->height) {
            WritePixel(output, buffer_x, buffer_y, color);
        }
    }
}

// 4. Apply mask alpha at offset position
void ApplyMaskAlpha(PF_LayerDef* output, PF_LayerDef* mask,
                    int mask_pos_x, int mask_pos_y) {
    for (int y = 0; y < output->height; y++) {
        for (int x = 0; x < output->width; x++) {
            // Convert output coords to mask coords
            int mask_x = x - mask_pos_x;
            int mask_y = y - mask_pos_y;

            if (mask_x >= 0 && mask_x < mask->width &&
                mask_y >= 0 && mask_y < mask->height) {
                // Inside mask - apply alpha
                float alpha = GetAlpha(mask, mask_x, mask_y);
                MultiplyAlpha(output, x, y, alpha);
            } else {
                // Outside mask - transparent
                SetAlpha(output, x, y, 0.0f);
            }
        }
    }
}
```

### OFX: Basic Render Pattern

```cpp
// 1. Describe - Define plugin capabilities
void PluginFactory::describe(ImageEffectDescriptor& desc) {
    desc.setLabel("Plugin Name");
    desc.setPluginGrouping("YourCompany");
    desc.addSupportedContext(eContextFilter);
    desc.addSupportedBitDepth(eBitDepthFloat);
    desc.setSupportsTiles(false);
    desc.setRenderThreadSafety(eRenderFullySafe);
}

// 2. DescribeInContext - Define parameters
void PluginFactory::describeInContext(ImageEffectDescriptor& desc, ContextEnum context) {
    ClipDescriptor* srcClip = desc.defineClip(kOfxImageEffectSimpleSourceClipName);
    srcClip->addSupportedComponent(ePixelComponentRGBA);

    ClipDescriptor* dstClip = desc.defineClip(kOfxImageEffectOutputClipName);
    dstClip->addSupportedComponent(ePixelComponentRGBA);

    PageParamDescriptor* page = desc.definePageParam("Main");

    DoubleParamDescriptor* param = desc.defineDoubleParam("paramName");
    param->setLabel("Parameter Label");
    param->setDefault(1.0);
    param->setRange(0.0, 10.0);
    page->addChild(*param);
}

// 3. Render - Process image
void PluginInstance::render(const RenderArguments& args) {
    Image* src = _srcClip->fetchImage(args.time);
    Image* dst = _dstClip->fetchImage(args.time);

    OfxRectI renderWindow = args.renderWindow;

    for (int y = renderWindow.y1; y < renderWindow.y2; y++) {
        for (int x = renderWindow.x1; x < renderWindow.x2; x++) {
            float* srcPix = (float*)src->getPixelAddress(x, y);
            float* dstPix = (float*)dst->getPixelAddress(x, y);

            // Process pixel
            dstPix[0] = ProcessRed(srcPix[0]);
            dstPix[1] = ProcessGreen(srcPix[1]);
            dstPix[2] = ProcessBlue(srcPix[2]);
            dstPix[3] = srcPix[3]; // Alpha
        }
    }

    // Use the OFX support library's RAII image management or call
    // releaseImage() explicitly. Do not use raw delete on OFX images.
    src->releaseImage();
    dst->releaseImage();
}
```

---

## Common Mistakes to Avoid

### 1. Wrong Coordinate Mapping
```cpp
// WRONG - Maps to buffer dimensions
float x = uv * output->width;

// CORRECT - Maps to layer, then offsets to buffer
float layer_x = uv * layer_width;
float buffer_x = layer_x - output_origin_x;
```

### 2. Mask Position in Wrong Space
```cpp
// WRONG - Uses layer-space coords in buffer
ApplyMask(output, mask, masked_rect_left, masked_rect_top);

// CORRECT - Converts to buffer-relative coords
int mask_pos_x = masked_rect_left - output_origin_x;
ApplyMask(output, mask, mask_pos_x, mask_pos_y);
```

### 3. Assuming Buffers Align
```cpp
// WRONG - Assumes 1:1 pixel mapping
output[y][x].alpha *= mask[y][x].alpha;

// CORRECT - Accounts for offset
int mask_y = y - mask_pos_y;
if (mask_y >= 0 && mask_y < mask->height) {
    output[y][x].alpha *= mask[mask_y][x].alpha;
}
```

### 4. Vague Planning
```markdown
WRONG:
"Add camera projection feature"

CORRECT:
"Feature: Camera Projection
- Load 4x4 camera matrix from Alembic at current time
- Transform each vertex position by camera matrix
- Divide by w component for perspective correction
- Map normalized device coords [-1,1] to UV [0,1]
- Sample input texture at UV coordinates
- Write to output buffer in layer space
- Test: Render checkerboard texture, verify no distortion"
```

---

## Tips and Tricks

### 1. Context Management
- Never use more than 50% of token context in a single session
- If context hits 100K tokens (for 200K limit), start fresh session
- Summarize completed features before starting new ones

### 2. Build Structure
Always build to proper bundle structure:
```
PluginName.ofx.bundle/
    Contents/
        Info.plist
        Win64/
            PluginName.ofx
```

### 3. Debug Output
For both AE and OFX plugins:
- Write to log file in known location (`C:\temp\plugin_debug.txt`)
- Use platform debug output (`OutputDebugStringA()` on Windows)
- Include timestamps and call stack context
- Log coordinate space conversions

### 4. When to Ask Questions
Ask Claude Code to explain:
- "What coordinate space is this value in?"
- "Walk me through the data flow for this feature"
- "What edge cases should I test?"
- "Why did this test fail?"

### 5. Pen and Paper
Before coding complex features:
- Sketch coordinate systems on paper
- Draw data flow diagrams
- Write out matrix multiplication by hand
- Visualize buffer layouts

---

## Summary Checklist

**Before Starting:**
- [ ] Created detailed PRD.md with all features listed
- [ ] Used Ask User Question tool for thorough planning
- [ ] Documented all coordinate systems and data structures
- [ ] Listed all edge cases and error conditions
- [ ] Specified UI/UX for every parameter
- [ ] Defined test criteria for each feature

**During Development:**
- [ ] Building one feature at a time
- [ ] Testing each feature before moving to next
- [ ] Documenting progress in progress.txt
- [ ] Monitoring context usage (< 50%)
- [ ] Verifying coordinate space conversions
- [ ] Handling platform-specific differences

**Before Using RAPH:**
- [ ] Built at least one plugin manually
- [ ] Understand the full development workflow
- [ ] Have comprehensive PRD.md
- [ ] Know how to write effective tests
- [ ] Can debug failures when they occur

---

## Final Thoughts

**The secret to great plugins isn't the AI tool you use -- it's the quality of your planning.**

Spend time thinking about:
- How should this feature *feel* to the user?
- What are the edge cases?
- How do coordinate systems interact?
- What happens when things go wrong?

The models are now good enough that if you give them a perfect plan, they'll give you a perfect implementation. But if you give them a vague plan, you'll waste time and effort debugging.

**Build with audacity. Plan with precision. Test relentlessly.**
