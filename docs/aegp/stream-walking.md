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

1. **Out-of-range index.** The loop bound must come from
   `AEGP_GetEffectNumParamStreams`, never a guess. Calling out of range does not
   return an error -- AE asserts via the verification dialog. Read the assert
   text literally: it enforces `[1 .. num_streams-1]`. Index 0 is the effect's
   input layer and trips `5027:150` the moment you feed it to
   `GetNewEffectStreamByIndex`, so start param walks at **1** (as the fix loop
   below does) and reach the input layer through the layer APIs, not this call.

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

## A related trap: two different index spaces

`AEGP_GetNewEffectStreamByIndex` indexes the effect's **flat param list** (`0` =
input layer, then one entry per `PF_ADD_*` param in declaration order, *including*
group start / end markers). That is **not** the same as the child index you get
from walking a group with `AEGP_DynamicStreamSuite`'s `AEGP_GetNewStreamRefByIndex`
on the effect's stream group. The two agree only when the effect has no nested
structure; once there are groups they diverge, and feeding a DynamicStream child
index into `GetNewEffectStreamByIndex` runs off the end -> `5027:150`.

So:

- If you already hold an `AEGP_StreamRefH` from a DynamicStream walk, read or
  write it **directly** (`AEGP_GetNewStreamValue` / `AEGP_SetStreamValue` on that
  ref) -- do not convert it back to an index.
- If you only know a param by name, resolve its real *flat* index from your own
  param table (the mapping you built at `PARAMS_SETUP`) rather than reusing a
  walk position from a different API.

This bit a "set my own param by name" helper: it found the param via a
DynamicStream match, returned that child index, then passed it to
`GetNewEffectStreamByIndex` -- which asserted `5027:150` the moment a group sat
before the target param. A variant of the same helper skipped the walk entirely
and just hard-coded index `0` to grab "the group", which asserts on its own (see
contract violation 1 above) even before the index-space mismatch matters.

These wrong helpers tend to recur in clusters: one bad by-name resolver often
backs several buttons on the same effect family -- a part picker, a material
loader, a "read my sibling effect's source layer" scan. When you find one,
grep every other call site that reuses a walk position (or a literal `0`) as a
`GetNewEffectStreamByIndex` index and fix them together, or the next button
reproduces the identical `5027:150`.

## See also

- [Choice / Popup Params Are 1-Based](choice-popup-params.md) -- the other
  cross-effect stream-walking gotcha: popup values come back 1-based.
- [Runtime-Populated Dropdowns](runtime-dropdown-popup.md) -- writes a picked
  index back via AEGP; a prime place to hit the two-index-spaces trap above.

*Tags: `aegp`, `stream`, `effect-walker`, `verification`, `no-data`*
