# Popup / Choice Param Values Are 1-Based on the Wire

> Pop-up (choice) parameters report their selected index as 1-based throughout
> the AE API -- in `PF_ADD_POPUP` defaults, in `PF_ParamDef::u.pd.value`, and in
> `AEGP_GetNewStreamValue`. If your enums are 0-based (as they should be),
> convert at exactly one boundary or every choice read is off by one.

## Symptom

A popup shows the user's selection correctly in the UI, but the value the code
consumes is shifted by +1. "Rect" (enum 0) reads as the next entry; "Primitive"
(enum 1) reads as the one after it and falls through to a "not implemented"
branch. The effect behaves as if the user picked the wrong item.

## Root cause

AE pop-up params are **1-based on the wire**. Three places where this collides
with 0-based enums:

1. `PF_ADD_POPUP` takes a **1-based** default index.
2. `PF_ParamDef::u.pd.value` (via `PF_CHECKOUT_PARAM`) is **1-based**.
3. `AEGP_GetNewStreamValue` returns a **1-based** value for a popup.

Your schema enums and the rest of the code are 0-based. A particularly easy place
to leak the raw 1-based value is a generic AEGP stream walker: a popup comes back
as an `AEGP_StreamType_OneD`, so if the walker pushes every `OneD` value through
unchanged, the popup's 1-based value flows into 0-based consumer code and every
choice shifts by +1.

## The fix

Normalize popups at the boundary by querying the param's actual `PF_ParamType` and
subtracting 1 when it is a popup:

```cpp
PF_ParamType ptype = PF_Param_RESERVED;
PF_ParamDefUnion punion;
AEFX_CLR_STRUCT(punion);
bool is_popup = false;
if (effectSuite->AEGP_GetEffectParamUnionByIndex(
        plugin_id, effH, pi, &ptype, &punion) == A_Err_NONE) {
    is_popup = (ptype == PF_Param_POPUP);
}
// when pushing the OneD value downstream:
double v = is_popup ? (sv.val.one_d - 1.0) : sv.val.one_d;
```

`AEGP_GetEffectParamUnionByIndex` lives on `EffectSuite4`, which you likely
already acquire for match-name reads, so there is no extra suite cost.

For your *own* render path, do the same: when you read a popup via
`PF_CHECKOUT_PARAM`, subtract 1 immediately so the rest of the code sees the
0-based enum.

## How to avoid it

Keep schema headers and consumer code 0-based everywhere, and do the AE-side
1-based -> 0-based conversion at exactly **one** boundary (the stream walker, or
your `get_choice` helper). After that single normalization, everything downstream
uses native enum values. When you read another plugin's popup state via AEGP and
it flows back through 0-based enums or default-index arguments, that single
conversion is mandatory.

## See also

- [Walking Effect Streams](stream-walking.md) -- the broader rules for safely
  walking another effect's params.

*Tags: `aegp`, `popup`, `choice`, `param`, `1-based`, `effect-walker`*
