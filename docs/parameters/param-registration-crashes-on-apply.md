# Parameter registration crashes / errors on apply

Two `PF_Cmd_PARAMS_SETUP` mistakes that pass compilation and only fail when the
effect is applied to a layer. Both verified on After Effects 25.3 (2602). Both
are easy to misdiagnose because the failure surfaces deep inside AE's host
code, not your plugin.

## 1. An ARB param with `PF_ParamFlag_CANNOT_TIME_VARY` crashes AE (`FATAL_APP_EXIT`)

### Symptom

Applying the effect crashes AE immediately, during `AddParams`. The crash dump
shows `FATAL_APP_EXIT (0x40000015)` bucketed under `sentry.dll` (Adobe's crash
reporter — ignore it). The real frames:

```
... EffectMain -> ParamsSetup -> AddParams -> (PF_ADD_PARAM)
  -> FLT_MakeStreamFromParamDef ... -> _CxxThrowException        (throw A)
  -> FLT_ParamArbDataStream::~FLT_ParamArbDataStream
       -> DeleteArbDataItem -> _CxxThrowException                (throw B, during unwind)
         -> std::terminate -> abort
```

A C++ throw during the unwind of another throw = `std::terminate`. Your
`try/catch` in `EffectMain` cannot catch it — `terminate` fires inside AE's
frames first.

### Cause

A `PF_Param_ARBITRARY_DATA` is an **animatable stream**; AE builds an
arb-data-stream object for it. `PF_ParamFlag_CANNOT_TIME_VARY` declares "no
stream," contradicting what AE is constructing, so AE throws (and its
destructor double-throws). It fails synchronously inside `PF_ADD_PARAM`, on the
first arb param, **before any of your arbitrary callbacks run** — proving it is
the param *def* AE rejects, not your `NEW_FUNC`/`COPY_FUNC`/`FLATTEN`.

### Fix

Register ARB params with `flags = 0` (`PF_ParamFlag_NONE`), as in the ColorGrid
sample and the `PF_ADD_ARBITRARY2` macro. Do not attempt to make an ARB
non-keyframeable via param flags. `PF_PUI_NO_ECW_UI` is fine for a hidden ARB
whose value is set programmatically (e.g. by a sibling Browse button) —
`AE_Effect.h` lists `PF_PUI_NO_ECW` as a valid ARB UI flag; the UI flag was
never the problem.

```cpp
AEFX_CLR_STRUCT(def);
def.param_type       = PF_Param_ARBITRARY_DATA;
def.flags            = PF_ParamFlag_NONE;     // 0 — NEVER CANNOT_TIME_VARY on an ARB
def.ui_flags         = PF_PUI_NO_ECW_UI;
def.uu.id            = MY_DISK_ID;
def.u.arb_d.id       = (A_short)MY_DISK_ID;   // mirror uu.id (PF_ADD_ARBITRARY2)
def.u.arb_d.refconPV = nullptr;
def.u.arb_d.value    = nullptr;
ERR(CreateDefaultArb(in_data, &def.u.arb_d.dephault));
ERR(PF_ADD_PARAM(in_data, -1, &def));
```

## 2. `Duplicate matchname found during FillInStreamsFromCanonicalLayout (29::0)`

### Symptom

On a **fresh** apply into a new comp:

```
After Effects error: internal verification failure, sorry!
{Duplicate matchname found during FillInStreamsFromCanonicalLayout}
( 29 :: 0 )
```

It can lurk: an already-applied / project-saved instance keeps working, so it
only bites a clean apply.

### Cause

AE derives a stream **matchname from each param's `def.uu.id`** and enforces
uniqueness across the WHOLE param tree — **including `PF_Param_GROUP_END`
markers**. Two registered params (any type, group starts and group ends
included) resolving to one `uu.id` is rejected. A common offender is reusing a
group's start id for its end:

```cpp
PF_ADD_TOPIC("Lighting", LIGHTING_GROUP_ID);   // GROUP_START, uu.id = LIGHTING_GROUP_ID
PF_END_TOPIC(LIGHTING_GROUP_ID);               // GROUP_END,  uu.id = LIGHTING_GROUP_ID  <-- duplicate
```

The SDK samples always give the end a distinct id.

### Fix

Give every GROUP_END a unique disk id (e.g. `start_id + 50000`, or any offset
that clears your real ids). Group ends carry no value, so saved projects lose
nothing.

### Finding the duplicate

Static scans of a disk-id enum are unreliable here — they miss hand-built
`def.uu.id = X` params, **computed** per-slot ids (`base + slot*stride + off`),
and group-end markers. Enumerate the *actual* `uu.id`s you register. A robust
trick: a thin wrapper around `PF_ADD_PARAM` during `AddParams` that records
`(uu.id, param_type, name)` and reports any repeat — it names the colliding
pair directly. A build-time uniqueness check over all registered ids prevents
the whole class.

## See also

- `parameters/param-flags.md` — `PF_ParamFlag` reference (`CANNOT_TIME_VARY`).
- `parameters/arb-data.md` — arbitrary-data param handlers.
- `aegp/match-name.md` — match names.
