# Plugin Architecture

How AE plugins are structured, loaded, and communicate with the host.

## Plugin Lifecycle

```
Load → PF_Cmd_GLOBAL_SETUP → PF_Cmd_PARAMS_SETUP → PF_Cmd_SEQUENCE_SETUP
                                                            ↓
                                              PF_Cmd_SMART_PRE_RENDER
                                                            ↓
                                              PF_Cmd_SMART_RENDER (per frame)
                                                            ↓
                                              PF_Cmd_SEQUENCE_SETDOWN
                                                            ↓
                                              PF_Cmd_GLOBAL_SETDOWN → Unload
```

## Essential Reading

- **[PiPL Resources](pipl.md)** — The resource that registers your plugin with AE. Get this wrong and your plugin won't load.
- **[Entry Points](entry-point.md)** — How AE calls your plugin via command selectors
- **[Plugin Structure](plugin-structure.md)** — The anatomy of an effect plugin

## Key Architecture Rules

- Your `main()` entry point receives a **command selector** — you switch on it to handle different lifecycle events
- **Global data** persists across all instances. **Sequence data** is per-instance. **Frame data** is per-render.
- Never allocate sequence data in `GLOBAL_SETUP` — it doesn't exist yet
- `SEQUENCE_RESETUP` is called when a project is reopened — your flat data needs to be reconstituted

---

*27 documents in this section. Use **Search** (top of page) to find what you need.*
