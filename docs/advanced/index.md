# Advanced Topics

Performance optimization, error handling, compatibility, and expert-level techniques.

## Performance

- **Iterate suites** are faster than manual pixel loops (AE can parallelize them)
- **SmartFX** avoids unnecessary pixel copies
- **GPU rendering** gives 3-30x speedup depending on the operation
- **`PF_OutFlag_NON_PARAM_VARY`** — massive cache improvement if your output is deterministic

## Common Pitfalls

- **Rowbytes != width * pixel_size** — always use `rowbytes` for pointer arithmetic
- **Don't allocate memory in the render loop** — pre-allocate in sequence setup
- **Check every `PF_Err`** — use the `ERR()` macro pattern from the SDK examples
- **Flatten/unflatten sequence data** — if you skip this, your plugin's state won't survive save/load
- **Widgets you draw in the ECW never fire `USER_CHANGED_PARAM`** — they arrive as coordinate hit-tests in `PF_Event_DO_CLICK`, so a "fix" to the param handler can silently miss the button the user actually clicks. See [ecw-drawn-widgets-vs-param-handlers.md](ecw-drawn-widgets-vs-param-handlers.md)

## Compatibility

- **Premiere Pro** can host AE plugins but doesn't support all features (no AEGP, limited custom UI)
- **Older AE versions** may not have newer suites — always check suite acquisition
- **Apple Silicon** requires Universal Binary builds

*This section contains 317 documents covering edge cases, workarounds, and deep technical topics. Use search to find what you need.*

---

*317 documents in this section. Use **Search** (top of page) to find what you need.*
