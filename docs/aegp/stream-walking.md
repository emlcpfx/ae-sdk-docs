# Walking Effect Streams: Clamp to the Stream Count and Check the Type First

> `AEGP_GetNewEffectStreamByIndex` does not return an error for an out-of-range
> index, and `AEGP_GetNewStreamValue` does not handle NO_DATA streams gracefully.
> Both trip AE's internal verification asserts instead. Query
> `AEGP_GetEffectNumParamStreams` for the loop bound and `AEGP_GetStreamType`
> before fetching a value.

## Symptom

An AEGP that walks the parameters of an effect (often across many layers in a
comp) hits AE's internal verification dialog with errors like:

- `5027:247` -- *"Cannot get AEGP_StreamValue2 for AEGP_StreamType_NO_DATA
  streams"*
- `5027:150` -- *"effect param_index out of range ([1..num_streams-1])"*

These are not error returns you can branch on -- they are assertion dialogs, so
the walker that "looked fine" trips them in sequence.

## Root cause

A common but wrong pattern uses a hard-coded upper bound and assumes
out-of-range indices return an error:

```cpp
for (PF_ParamIndex pi = 1; pi < 1024; ++pi) {
    AEGP_StreamRefH sref = nullptr;
    A_Err se = AEGP_GetNewEffectStreamByIndex(plugin_id, effH, pi, &sref);
    if (se != A_Err_NONE || !sref) break;   // never happens for out-of-range pi
    ...
}
```

Two contract violations:

1. **Out-of-range index.** The SDK header documents `param_index` as valid in
   `[0 .. AEGP_GetEffectNumParamStreams - 1]`, where 0 is the effect's input
   layer. Calling past that range does not return an error -- AE asserts via the
   verification dialog.

2. **NO_DATA streams.** Groups, group-ends, buttons, and arbitrary-data params
   are `AEGP_StreamType_NO_DATA`. Calling `AEGP_GetNewStreamValue` on them trips
   a different verification assert. You must check the stream type first and skip
   the value fetch for NO_DATA.

## The fix

Fetch the count for the loop bound, and check the type before fetching the value.
This mirrors the canonical SDK pattern (see
`Examples/AEGP/ProjDumper/ProjDumper.cpp`):

```cpp
A_long num_streams = 0;
StreamSuite->AEGP_GetEffectNumParamStreams(effH, &num_streams);
for (PF_ParamIndex pi = 1; pi < num_streams; ++pi) {   // 0 = input layer, skip
    AEGP_StreamRefH sref = nullptr;
    if (StreamSuite->AEGP_GetNewEffectStreamByIndex(plugin_id, effH, pi, &sref)
            != A_Err_NONE || !sref)
        break;

    AEGP_StreamType st = AEGP_StreamType_NO_DATA;
    StreamSuite->AEGP_GetStreamType(sref, &st);

    if (st == AEGP_StreamType_OneD || /* TwoD, ThreeD, COLOR, LAYER_ID, ... */) {
        AEGP_StreamValue2 sv = {};
        if (StreamSuite->AEGP_GetNewStreamValue(/* ... */) == A_Err_NONE) {
            // consume sv based on st
            StreamSuite->AEGP_DisposeStreamValue(&sv);
        }
    }
    StreamSuite->AEGP_DisposeStream(sref);
}
```

## How to avoid it

When walking AEGP streams or params on someone else's effect, never use a
hard-coded loop upper bound -- always fetch `AEGP_GetEffectNumParamStreams` and
clamp to it. Likewise, query the stream type before fetching its value, because
NO_DATA streams (groups, group-ends, buttons, arbitrary) trip a separate
verification assert when you blindly call `GetNewStreamValue`.

The general lesson: AEGP does not validate-and-return for these cases -- it
asserts via the internal verification dialog. Treat the `5027:NNN` numbers as
documentation of the exact contract you violated, then read the SDK header for
the precondition you missed.

## See also

- [Choice / Popup Params Are 1-Based](choice-popup-params.md) -- the other
  cross-effect stream-walking gotcha: popup values come back 1-based.

*Tags: `aegp`, `stream`, `effect-walker`, `verification`, `no-data`*
