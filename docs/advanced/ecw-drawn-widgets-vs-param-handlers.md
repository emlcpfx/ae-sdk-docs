# ECW-drawn widgets never fire `USER_CHANGED_PARAM`

> Battle-tested on SuperClone (2026): three consecutive "fixes" to a delete handler changed nothing, because the button the user was clicking did not route through the handler being fixed. The bug was found only after instrumenting the handler and observing **zero** log lines from a real click.

## Symptoms (what you'll see)

| Symptom | What it actually means |
|---------|------------------------|
| A fix to your param-change handler has **no effect at all** — not a wrong value, *no* change | The user's click never reaches that handler |
| Behavior differs between the "same" action done two ways (header button vs right-click menu vs external panel) | Each entry point runs its own copy of the logic |
| A `PF_ADD_BUTTON` handler looks correct and is never observed running | The visible control is drawn by your ECW, not by AE |
| State updates partially: your internal blob changes, but params/UI don't | One path mutates internal state only, another does the full job |

If a change appears to do *nothing*, suspect the code path before you suspect SDK semantics. "The write is being reverted" and "the code never ran" look identical from the UI.

---

## Root cause: two independent dispatch paths

An effect with a custom ECW usually has **both** kinds of controls, and they arrive through completely different selectors:

| Control | How it's created | How a click arrives |
|---------|------------------|---------------------|
| **Real param** (`PF_ADD_BUTTON`, checkbox, slider…) | `ParamsSetup` | `PF_Cmd_USER_CHANGED_PARAM` → your `HandleParamChange`, with `extra->param_index` |
| **Widget you drew yourself** in the ECW | Your `PF_Event_DRAW` handler (Drawbot) | `PF_Cmd_EVENT` / `PF_Event_DO_CLICK` → your click handler, which **hit-tests coordinates manually** |

A drawn widget is just pixels. AE has no idea it is a button, so it can never generate `USER_CHANGED_PARAM` for it. The only thing that makes it "a button" is your own rectangle hit-test in the `DO_CLICK` handler:

```cpp
// Your ECW click handler - this IS the button, as far as the user is concerned
if (click.h >= del_btn_x && click.h <= del_btn_x + kDeleteBtnWidth &&
    click.v >= del_btn_y && click.v <= del_btn_y + kDeleteBtnHeight) {
    // ... whatever you do here is the entire behavior of that button
}
```

### The sharpened trap: a hidden param that mirrors the drawn one

It gets worse when a **real** button param exists *alongside* the drawn one — typically kept for scripting/extension triggering — and is hidden:

```cpp
AEFX_CLR_STRUCT(def);
def.flags    = PF_ParamFlag_SUPERVISE;
def.ui_flags = PF_PUI_INVISIBLE;          // hidden from the user entirely
PF_ADD_BUTTON("Delete Selected", "Delete", 0,
              PF_ParamFlag_SUPERVISE, SC_DISK(PARAM_DELETE_SELECTED));
```

Now `HandleParamChange`'s `param_index == PARAM_DELETE_SELECTED` branch *looks* like the delete handler. It is well-commented, it sets the right out-flags, and it is the natural place to "fix delete." But a user clicking the visible [Del] never triggers it — that param is invisible and only an external script can set it. The real behavior lives in the coordinate hit-test, potentially with a **duplicate, divergent** implementation.

---

## The failure mode: duplicated mutation logic

Both paths "delete strokes," but only one was complete:

```cpp
// Path A - drawn [Del] button, in the ECW click handler (what users hit)
for (int i = n - 1; i >= 0; i--)
    if (IsSelected(&seq->strokes, i))
        seq->strokes.RemoveStroke(i);       // internal blob ONLY
ClearSelection(&seq->strokes);

// Path B - hidden button param + right-click menu + external panel
StrokeManager::DeleteSelectedStrokes(in_data, out_data, params, &seq->strokes);
// ^ tombstones the blob AND clears the slot's per-item params, scrubs keyframes, etc.
```

Path A skipped every param-level side effect. Symptom: deleting from the ECW left the item's parameters (checkbox still checked, point param still at its old position, so AE kept drawing its crosshair in the comp) fully alive in the Effect Controls panel.

---

## The rule

**One mutation authority per operation. Every entry point calls it — none re-implements it.**

```cpp
// Drawn widget hit-test
if (hit_delete_button) {
    SequenceData* seq = GetSequenceDataLocked(in_data);
    if (seq && seq->IsValid()) {
        StrokeManager::DeleteSelectedStrokes(in_data, out_data, params, &seq->strokes);
        // out-flags / invalidate / GPU cache drop stay here (per-entry-point concerns)
    }
    if (seq) UnlockSequenceData(in_data);
}
```

Enumerate your entry points explicitly when auditing an operation. For a typical custom-UI effect that is:

1. Drawn widget in the ECW (`DO_CLICK` hit-test)
2. Right-click / context menu built by your own UI code
3. Real (possibly hidden) param → `USER_CHANGED_PARAM`
4. Keyboard handler (`PF_Event_KEYDOWN`)
5. External AEGP panel / script writing through an arb bridge

Grep for the *lowest-level* mutator (here `RemoveStroke(`) rather than the friendly wrapper — the wrapper is what you already know about; the raw call is where a bypass hides.

---

## Diagnostic: prove the handler runs before theorizing

This class of bug is cheap to find and expensive to reason about. Before forming any theory about *why* a write didn't stick, prove the code executed:

1. Add one log line at the top of the handler, dumping the identifying values (`slot/index`, whether `params[]` entries are non-null, the before value).
2. Log again after the write, dumping the after value and `change_flags`.
3. Perform the action **once** in AE.
4. Read the log.

The result is binary and unambiguous:

| Log output | Conclusion |
|------------|------------|
| **No lines at all** | Wrong code path — find the real entry point. Stop debugging semantics. |
| Lines present, before == after | The write itself failed (wrong index, null param, illegal context) |
| Lines present, value changed, UI stale | Genuinely a refresh/persistence problem — *now* the SDK theories apply |

Use a **file** logger, not `OutputDebugString` (which floods DebugView and is unsafe in hot paths under MFR). A header-only, always-compiled logger writing to a fixed per-plugin path means you can inspect it yourself instead of asking a user to reproduce with a debugger attached.

> Note the failure this prevents: "the params[] write is silently discarded because the property is keyframed" is a *plausible* SDK-level explanation, consistent with every symptom, and completely wrong here. Only the empty log distinguished it from "the function never ran."

---

## Related

- [ecw.md](ecw.md) — ECW basics, event routing
- [custom-ecw-ui.md](custom-ecw-ui.md) — drawing custom UI in the ECW
- [../guides/custom-ui-drawing.md](../guides/custom-ui-drawing.md) — Drawbot drawing patterns, cursor handling
- [../parameters/arb-callback-disk-id-and-checkout-context.md](../parameters/arb-callback-disk-id-and-checkout-context.md) — the other "handler silently never runs" trap (disk ID vs array index)
- [../parameters/point-parameters.md](../parameters/point-parameters.md) — point-param crosshairs are drawn by AE and cannot be suppressed

*Tags: `ecw`, `custom-ui`, `do-click`, `user-changed-param`, `event`, `debugging`, `params`*
