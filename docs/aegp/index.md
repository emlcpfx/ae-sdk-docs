# AEGP & Scripting

The AEGP API gives plugins deep access to AE's internals — projects, comps, layers, keyframes, and scripting.

## AEGP vs Effect Plugins

- **Effect plugins** process pixels on a single layer
- **AEGP plugins** can manipulate the entire project — add layers, modify keyframes, run scripts

## Common Use Cases

- **Custom importers/exporters** (AEIO)
- **Panel plugins** with custom UI
- **Automation** — batch processing, project manipulation
- **Script bridge** — execute ExtendScript from C++ via `AEGP_ExecuteScript`
- **Idle hooks** — run background tasks when AE is idle

## Key Suites

- `AEGP_CompSuite` — Work with compositions
- `AEGP_LayerSuite` — Manipulate layers
- `AEGP_StreamSuite` — Read/write parameter streams
- `AEGP_KeyframeSuite` — Manage keyframes
- `AEGP_MemorySuite` — Memory allocation through AE

---

*32 documents in this section. Use **Search** (top of page) to find what you need.*
