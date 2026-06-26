# Panel

> 1 Q&A · source: AE plugin dev community Discord

### What is UXP for Premiere Pro and how does it relate to CEP panel migration?

UXP (Unified Extensibility Platform) for Premiere Pro entered public beta in December 2024. It is the successor to CEP for building panels and extensions. Initially it is a web-based development environment, but a hybrid UXP+C++ approach is planned. For C++ computational tasks without the hybrid mode, you would need a service with IPC. The Premiere UXP beta is currently limited in functionality but expected to expand. Plugin developers with existing CEP panels should begin evaluating migration. Resources include the Adobe Creative Cloud Developer forums and the Hyperbrew blog (hyperbrew.co/blog/premiere-pro-uxp-beta). Bolt UXP (hyperbrew.co/resources/bolt-uxp/) is a tool to help with UXP development.

*Tags: `beta`, `cep`, `hybrid-cpp`, `migration`, `panel`, `premiere`, `uxp`*

---

## Native AEGP custom panels (C++) — the "current context" rule

> Hard-won field notes from building a fully native (no CEP/HTML) AEGP panel
> that reads and writes an effect's params + keyframes. The single most
> important rule is buried here: **when you may call AEGP suites.**

### How a native panel is wired

An AEGP plugin registers a panel and an idle hook at `EntryPointFunc` time:

```
sp.RegisterSuite5()->AEGP_RegisterIdleHook(plugin_id, &S_IdleHook, nullptr);
// + a panel-creation hook that hands you the platform window:
//   Windows -> HWND, macOS -> NSView*.
```

You typically subclass that window's `WndProc` (Windows) / override `NSView`
event methods (macOS) to get paint + mouse events. To support **drags** on
Windows, `SetCapture(hwnd)` on `WM_LBUTTONDOWN` so you keep receiving
`WM_MOUSEMOVE` / `WM_LBUTTONUP` after the cursor leaves the window;
`ReleaseCapture()` on button-up.

### THE RULE: AEGP suites are only valid inside an AE-initiated callback

AE only sets up a valid AEGP "current context" while it is calling **your AEGP
hooks** — above all the **idle hook** (`AEGP_RegisterIdleHook`). Raw OS window
messages (`WM_PAINT`, `WM_LBUTTONDOWN`, `WM_MOUSEMOVE`, menu/flyout commands)
are pumped by AE's message loop but run **outside** that context.

Calling **any** AEGP suite from an OS-message handler raises AE's modal assert:

```
After Effects error: internal verification failure, sorry! {no current context}
```

This bites BOTH directions and is easy to misdiagnose:

- **Reads in `on_paint` (WM_PAINT):** `AEGP_GetNewStreamValue`, `KeyframeSuite`
  reads, etc. raise it — and because painting repeats, the alert fires in a
  loop. (It looks like the panel is "spamming errors.")
- **Writes in mouse / menu handlers:** `AEGP_SetStreamValue`,
  `AEGP_SetItemCurrentTime`, `AEGP_InsertKeyframe`, even
  `AEGP_GetActiveItem` / `AEGP_GetNewCollectionFromCompSelection` (used to find
  the selected layer) all raise it. Moving a slider, scrubbing the timeline,
  picking a color — anything that touches AE from a click — fails.

The offending calls are the *same suites* that work perfectly from the idle
hook. It is purely about **which callback you are in**, not the call itself.

### Worse than the assert: creating project items from a click HARD-CRASHES

The "no current context" alert is recoverable. Some edits skip the assert and
fault outright. **Creating a comp item from a mouse / menu handler** - e.g. an
AEGP create-solid call (the "new solid + apply effect" path, internally
`CComposition::DoNewSolidItemAndLayer`) - reaches AE's project engine, which
dereferences an undo context that was never set up:

```
... your on_mouse_down / WndProc
... DoNewSolidItemAndLayer -> BEE_CmdNewFolder -> BEE_Project::GetUndoContext
c0000005 (read)  mov rax,[rcx+0x368]   ; rcx = 0  -> null project/undo context
```

(Creating a solid implicitly creates/finds the "Solids" folder, hence the
`BEE_CmdNewFolder`.) Same fix: the click only **records intent** (which object
to create) into a member; the actual create runs on the next **idle** tick. The
`Layer > New` menu commands escape this only because AE establishes a valid
context before invoking a command hook - a raw OS-message handler never does.

### The pattern that works: idle is the only place you touch AE

1. **Reads → idle, cached.** On the idle hook (valid context) read every value
   the paint path needs (param values, colors, keyframe times) into plain
   member caches. `on_paint` draws **only** from those caches and never calls a
   single AEGP suite.
2. **Writes → enqueue from the handler, flush on idle.** Mouse / menu handlers
   push a pure-data command (param name, value, op) onto a queue and touch no
   AEGP. At the top of the idle hook, drain the queue and perform the real
   `AEGP_SetStreamValue` / `AEGP_SetItemCurrentTime` / keyframe edits there.
3. **Optimistically update the local cache in the handler** (a plain map write,
   no AEGP) so the slider / swatch / playhead moves instantly, before the next
   idle flush re-reads the authoritative value.
4. **Coalesce a drag** (keep only the latest value per target in the queue) so a
   slider drag becomes one `SetStreamValue` + one undo group per flush instead
   of dozens.

Flush the write-queue **every** idle tick (don't gate it behind your ~10 Hz
repaint throttle) so edits apply promptly.

### Diagnosing it

Log each handler's context tag and each AEGP call's `A_Err` to a file (open +
close per line so the last line survives the modal alert). In a healthy run
**every** call inside the idle flush returns `err=0`; if you ever see the alert,
the last logged line names the exact call + the context it ran in. Mirror to
`OutputDebugStringA` to watch live in DebugView.

### Other native-panel gotchas

- **`AEGP_GetLayerEffectByIndex` needs your real `AEGP_PluginID` as its first
  arg.** Passing literal `0` makes AE walk an uninitialized table and panic
  inside `PF_SetOverrideFunc` (seen as an idle-hook crash). Use the panel host's
  registered id.
- **`AEGP_GetNewStreamValue` dereferences its `A_Time*`** — never pass `nullptr`;
  pass `&t` (e.g. `A_Time t{0,1};`) or it crashes.
- **`AEGP_GetNewStreamValue` raises an internal-verification failure on a
  `NO_DATA` stream** — group headers, buttons, separators, *and* any stale or
  group-aligned param index resolve to `AEGP_StreamType_NO_DATA`, which carries
  no value:
  `{Cannot get AEGP_StreamValue2 for AEGP_StreamType_NO_DATA streams} (5027)`.
  Call `AEGP_GetStreamType` first and skip `NO_DATA` so a drifted index degrades
  to a default instead of throwing on every idle read. (Combine with resolving
  params by NAME, below, so the index doesn't drift in the first place.)
- **Reading the comp playhead:** `AEGP_GetItemCurrentTime` in `AEGP_ItemSuite`
  (not CompSuite). Set it with `AEGP_SetItemCurrentTime` to scrub.
- **Find the selected layer** via `AEGP_GetActiveItem` -> check
  `AEGP_ItemType_COMP` -> `AEGP_GetCompFromItem` ->
  `AEGP_GetNewCollectionFromCompSelection` -> walk for the first
  `AEGP_CollectionItemType_LAYER`. (All of this must run in idle context per the
  rule above.)
- **Resolve effect params by display NAME, cached**, not by hard-coded index —
  large effects have 1000+ streams and indices shift as the layout evolves.
  Re-verify the cached index still names the param before trusting it.

*Tags: `aegp`, `panel`, `idle-hook`, `current-context`, `custom-ui`, `streams`, `keyframes`, `threading`, `undo`, `crash`*

---
