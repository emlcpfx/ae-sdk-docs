# Memory & Threading

Memory management, caching, and multi-frame rendering (MFR) thread safety.

## Multi-Frame Rendering (MFR)

AE 2022+ renders multiple frames in parallel. Your plugin **must** be thread-safe.

**Key rules:**
- No global mutable state (use `sequence_data` or `PF_Handle` instead)
- `SEQUENCE_SETUP` and `SEQUENCE_RESETUP` are called per-thread
- `GLOBAL_SETUP` is called once — don't store per-instance state there
- Set `PF_OutFlag2_SUPPORTS_THREADED_RENDERING` (hex: `0x08000000`) to opt in

## Sequence Data

Per-instance persistent state that survives across frames:

- **Setup**: `PF_Cmd_SEQUENCE_SETUP` — allocate your data
- **Flatten**: `PF_Cmd_SEQUENCE_FLATTEN` — serialize for save/copy (must handle!)
- **Resetup**: `PF_Cmd_SEQUENCE_RESETUP` — unflatten on project reopen
- **Setdown**: `PF_Cmd_SEQUENCE_SETDOWN` — cleanup

## Caching

- **Compute Cache** — AE 2024+ lets plugins cache intermediate results across frames
- **Frame caching** — AE manages this; your plugin sees `PF_Cmd_SMART_RENDER` again on cache miss
- **`PF_OutFlag_NON_PARAM_VARY`** — set this if your output depends ONLY on parameters and current time

---

*50 documents in this section. Use **Search** (top of page) to find what you need.*
