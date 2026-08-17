# Arb Param Value Is Always NULL: The 16-Bit `disk_id` and the Discarded `params[]` Write

> Two independent traps make an arbitrary-data parameter's value read back
> `NULL` forever, with **no error, no warning, and no failed call to inspect**.
> One is a silent integer truncation in `PF_ArbParamsExtra::id`; the other is AE
> quietly discarding a value you assign into `params[]`. Both look like correct
> code.

## Symptom

An arbitrary-data (ARB) param behaves as if it has no value:

- `params[i]->u.arb_d.value` is `NULL`,
- the AEGP stream value's `val.arbH` is `NULL`,
- data written to it never persists, never reaches the render thread, and never
  appears on the undo stack.

Every call involved returns `A_Err_NONE`. Nothing fails. The param simply never
has a value, so any feature built on it silently does nothing.

## Trap 1: `PF_ArbParamsExtra::id` is an `A_short`

AE tells your `PF_Cmd_ARBITRARY_CALLBACK` handler *which* ARB param it is asking
about via `PF_ArbParamsExtra::id`. In `AE_Effect.h` that field is declared:

```c
typedef struct {
    PF_FunctionSelector  which_function;
    A_short              id;          /* <-- 16-bit, NOT A_long */
    ...
} PF_ArbParamsExtra;
```

The `disk_id` you pass to `PF_ADD_ARBITRARY2` is an `A_long`. If it does not fit
in 16 bits, the value arriving in `extra->id` is **truncated**, so the usual
dispatch never matches:

```c
// Registered with disk_id 900006 -- a perfectly ordinary "fresh, high,
// non-colliding" id for a newly appended param.
PF_ADD_ARBITRARY2("My Data", 1, 1, PF_ParamFlag_CANNOT_TIME_VARY,
                  PF_PUI_NO_ECW_UI, def.u.arb_d.dephault, 900006, NULL);

// In PF_Cmd_ARBITRARY_CALLBACK:
if (extra->id == 900006) { ... }   // NEVER TRUE
// 900006 truncated to 16 bits arrives as -17498.
```

Because the `COPY`/`NEW`/`UNFLATTEN` branches never run, `*dst_arbPH` is never
written, and **AE can never construct a value for the param**. The handle stays
`NULL` for the lifetime of the effect.

Note this is a *distinct* failure from the more familiar arb-dispatch mistake of
comparing `extra->id` against the `params[]` **array index** instead of the
**disk ID**. Both produce the same NULL handle, and fixing that one does not fix
this one: `extra->id` genuinely is the disk ID, but if that disk ID does not fit
in 16 bits, comparing against it *correctly* still never matches.

This matters more than it first appears, because `PF_Arbitrary_NEW_FUNC` is not
called in post-CS6 AE: instances are created by AE sending
`PF_Arbitrary_COPY_FUNC` with your ParamSetup default. Miss that selector and
there is no other path to a valid value.

**Rule:** an ARB param's `disk_id` must be `<= 32767`. Assert it at compile time:

```c
#define MY_ARB_DISK_ID  1471
static_assert(MY_ARB_DISK_ID > 0 && MY_ARB_DISK_ID <= 32767,
              "ARB disk_ids must fit in A_short (PF_ArbParamsExtra::id)");
```

**Plain (non-arb) params are exempt.** They never reach arb dispatch, so large
`disk_id` values are fine for sliders, checkboxes and popups. That asymmetry is
what makes this trap so easy to walk into: a project can append several params
with ids in the 900000s, all working perfectly, and then the first ARB param
added the same way is dead on arrival.

Two ARB params whose ids truncate to the *same* `A_short` is the other failure
mode of this — they will dispatch to each other's handlers.

## Trap 2: assigning `u.arb_d.value` during an event is discarded

For plain params, writing through the `params[]` array during a `PF_Cmd_EVENT`
and flagging the change is the normal, working idiom:

```c
params[MY_SLIDER]->u.fs_d.value = 42.0;
params[MY_SLIDER]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;   // works
```

The equivalent for an ARB param does **not** work. AE does not adopt a handle
you assign into the `PF_ParamDef`:

```c
params[MY_ARB]->u.arb_d.value = my_new_handle;                       // discarded
params[MY_ARB]->uu.change_flags |= PF_ChangeFlag_CHANGED_VALUE;
```

The write appears to succeed. On the very next event the slot reads back `NULL`
(or the previous value), and nothing was placed on the undo stack.

### Write ARB params with `AEGP_SetStreamValue`

Fetch the stream value, modify the handle it hands you, and set it back:

```c
AEGP_EffectRefH  effect = NULL;
AEGP_StreamRefH  stream = NULL;
suites.PFInterfaceSuite1()->AEGP_GetNewEffectForEffect(my_id, in_data->effect_ref, &effect);
// Effect stream index == the params[] index of the param.
suites.StreamSuite6()->AEGP_GetNewEffectStreamByIndex(my_id, effect, MY_ARB_PARAM_INDEX, &stream);

AEGP_StreamValue2 v = {0};
A_Time t = {0, 1};
if (suites.StreamSuite6()->AEGP_GetNewStreamValue(
        my_id, stream, AEGP_LTimeMode_LayerTime, &t, FALSE, &v) == A_Err_NONE) {

    PF_Handle arb = reinterpret_cast<PF_Handle>(v.val.arbH);
    if (arb) {
        // Resize AE's OWN handle if your payload is variable-size, then
        // overwrite in place. Do not swap in a handle you allocated --
        // ownership stays with AE.
        suites.HandleSuite1()->host_resize_handle(new_size, &arb);
        v.val.arbH = reinterpret_cast<PF_ArbitraryH>(arb);
        void* p = suites.HandleSuite1()->host_lock_handle(arb);
        memcpy(p, my_bytes, new_size);
        suites.HandleSuite1()->host_unlock_handle(arb);

        // Wrap to get ONE undo entry for the change.
        suites.UtilitySuite6()->AEGP_StartUndoGroup("My Edit");
        suites.StreamSuite6()->AEGP_SetStreamValue(my_id, stream, &v);
        suites.UtilitySuite6()->AEGP_EndUndoGroup();
    }
    suites.StreamSuite6()->AEGP_DisposeStreamValue(&v);
}
suites.StreamSuite6()->AEGP_DisposeStream(stream);
suites.EffectSuite4()->AEGP_DisposeEffect(effect);
```

`v` must come from `AEGP_GetNewStreamValue` — `AEGP_SetStreamValue` on an ARB
silently no-ops if `v.streamH` is not populated, which `GetNewStreamValue` does
for you. That is a *third* silent-failure mode in the same area.

## Why this combination is worth knowing

These two traps produce an identical symptom, so fixing one while the other is
still present looks like the fix did nothing. Diagnosing them in the wrong order
wastes a full debug cycle: the `params[]` write can be corrected and the value
will *still* be `NULL`, because the param never had a handle to begin with.

If the ARB param is backing plugin state that should be undoable, there is a
fourth reason it can appear broken — `sequence_data` is not undoable at all, and
AE gives no notification when it services an Undo. See
[Undo/Redo with sequence data and arb data](../advanced/undo-redo.md).

## How to diagnose

When an ARB param's value is `NULL`, resolve it in this order — the first step
takes one log line and eliminates the most likely cause:

1. **Log `extra->id` at the top of `PF_Cmd_ARBITRARY_CALLBACK`**, next to the
   `disk_id` you registered. If they differ, you have Trap 1. Do not reason about
   it — a truncated value looks arbitrary (`900006` becomes `-17498`) and is
   obvious in a log and invisible in source.
2. **Confirm `PF_Arbitrary_COPY_FUNC` fires and writes `*dst_arbPH`.** If it
   never fires, you are still on Trap 1. If it fires and the value is still
   `NULL`, your ParamSetup `dephault` handle was never created.
3. **Check how you write the value.** Any assignment to `u.arb_d.value` is
   Trap 2; move it to `AEGP_SetStreamValue`.

Add an "unknown arb id" fallback log to the dispatcher so a mismatched id is
loud rather than a silent no-op:

```c
if (extra->id == MY_ARB_DISK_ID)       { ... }
else if (extra->id == OTHER_DISK_ID)   { ... }
else { LOG("arb callback for unknown id %d", (int)extra->id); }
```

## How to avoid

- Keep every ARB `disk_id` `<= 32767` and `static_assert` it.
- Never assign `u.arb_d.value` directly; write ARB values via
  `AEGP_SetStreamValue`, wrapped in an undo group when the change should be
  undoable.
- Modify the handle AE gives you (resizing if needed) rather than substituting
  your own, so ownership semantics stay AE's.
- Give the dispatcher an unknown-id branch that logs, so this class of bug
  reports itself the next time.

## See also

- [Arbitrary (Arb) Data](arb-data.md) — lifecycle, flatten/unflatten, and the
  `NEW_FUNC`-is-never-called behavior.
- [Arb Param](arb-param.md) — param flags, multiple ARB params, and refcon
  routing.
- [Undo/Redo](../advanced/undo-redo.md) — why `sequence_data` cannot be undone
  and the arb-vs-sequence state-identifier pattern.

*Tags: `arb-data`, `arb-param`, `params`, `disk-id`, `aegp`, `stream-value`, `undo`, `silent-failure`, `debugging`*
