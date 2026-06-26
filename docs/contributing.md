# Contributing

This documentation is community-maintained. Contributions are welcome and encouraged.

## How to Contribute

### Fix an Error
1. Click the pencil icon on any page to edit it directly on GitHub
2. Submit a PR with your fix

### Add New Content
1. Fork the repository
2. Add your `.md` file to the appropriate `docs/` subdirectory:
   - `docs/getting-started/` — Beginner topics, project setup
   - `docs/architecture/` — Plugin structure, PiPL, entry points
   - `docs/parameters/` — Parameter types and UI
   - `docs/rendering/` — Render pipeline, effect worlds
   - `docs/effects/` — Color, compositing, alpha, masks
   - `docs/memory-threading/` — MFR, thread safety, caching
   - `docs/gpu/` — CUDA, OpenCL, Vulkan, WebGPU
   - `docs/aegp/` — AEGP API, scripting
   - `docs/suites/` — SDK suites and callbacks
   - `docs/coordinates/` — Matrix math, coordinate spaces
   - `docs/layers/` — Layers, compositions, track mattes
   - `docs/animation/` — Keyframes, time
   - `docs/platform/` — Build systems, debugging, deployment
   - `docs/licensing/` — Distribution and licensing
   - `docs/advanced/` — Performance, optimization, expert topics
   - `docs/guides/` — In-depth curated guides
3. Submit a PR

### Submit Q&A Pairs
If you have a question and answer about AE plugin development:
1. Add it to the appropriate topic page in `docs/qa/`
2. Use this format:

```markdown
## Your question here?

Your detailed answer.

```cpp
// Code example if applicable
```

*Tags: `tag1`, `tag2`, `tag3`*

---
```

## Content Guidelines

### Do
- Be technically accurate — test code before submitting
- Include code examples with proper error handling
- Use correct SDK function names and signatures
- Note which AE versions your information applies to
- Keep PII out of submissions (no names, emails, company names)

### Don't
- Don't guess at API names or function signatures
- Don't include proprietary plugin code
- Don't submit AI-generated content without verification
- Don't include personal information or credentials

## PiPL Hex Values

If your submission includes PiPL flag hex values, **double-check them against the SDK headers**.
Incorrect hex values are a common source of hard-to-debug plugin issues. Known correct values:

| Flag | Hex Value |
|------|-----------|
| `PF_OutFlag_DEEP_COLOR_AWARE` | `0x00400000` |
| `PF_OutFlag2_SUPPORTS_THREADED_RENDERING` | `0x08000000` |
| `PF_OutFlag2_SUPPORTS_GPU_RENDER_F32` | `0x02000000` |

## Building Locally

```bash
pip install -r requirements.txt
mkdocs serve
```

Then open `http://localhost:8000` to preview your changes.
