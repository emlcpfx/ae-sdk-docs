# AE Plugin SDK Community Docs

Community-maintained documentation for After Effects C++ plugin development.

**772 reference docs** | **1522 Q&A pairs** | **92 topic pages**

---

## Quick Navigation

| Section | Description |
|---------|-------------|
| [Getting Started](getting-started/) | First plugin, project setup, build systems |
| [Plugin Architecture](architecture/) | Entry points, PiPL, lifecycle, commands |
| [Parameters](parameters/) | Param types, UI controls, dynamic streams |
| [Rendering](rendering/) | Smart render, effect worlds, output rects |
| [Effects & Processing](effects/) | Color, compositing, alpha, masks |
| [Memory & Threading](memory-threading/) | MFR, thread safety, caching, sequence data |
| [GPU Rendering](gpu/) | CUDA, OpenCL, Vulkan, WebGPU, Metal, DirectX |
| [AEGP & Scripting](aegp/) | AEGP API, ExtendScript, expressions |
| [Suites & Callbacks](suites/) | SDK suites, iterate, event handling |
| [Coordinates & Transforms](coordinates/) | Matrix math, coordinate spaces, 3D |
| [Layers & Compositions](layers/) | Layer checkout, precomps, track mattes |
| [Animation & Time](animation/) | Keyframes, time, markers |
| [Platform & Build](platform/) | Windows, macOS, debugging, deployment |
| [Licensing & Distribution](licensing/) | Gumroad, aescripts, watermarking |
| [Advanced Topics](advanced/) | Performance, optimization, expert techniques |
| [Curated Guides](guides/) | In-depth verified guides on specific topics |
| [Q&A Reference](qa/) | 1522 community Q&A pairs by topic |

---

## For LLM / AI Code Assistants

This documentation is designed to be consumed by both humans and LLMs.
The raw markdown source is available in the [GitHub repo](https://github.com/emlcpfx/ae-sdk-docs).

To point your AI coding assistant at this knowledge base:

**Claude Code** — add to your project's `CLAUDE.md`:
```
AE SDK reference docs are at: [GitHub repo](https://github.com/emlcpfx/ae-sdk-docs)
Clone locally and read docs/ for comprehensive AE plugin development knowledge.
Private Repo, requires an invitiation.
```

**Cursor / Codex** — add the repo as a docs source or clone locally.

---

## Contributing

This is a community wiki. If you find errors, have solutions to share, or want to
add documentation:

1. Fork the repo
2. Edit or add `.md` files in the appropriate `docs/` subdirectory
3. Submit a PR

See the [Contributing Guide](contributing.md) for details.

---

*Built from real-world AE plugin development experience. All content has been reviewed
for accuracy, with PII removed and factual errors corrected (especially PiPL hex flag values).*
