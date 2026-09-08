# Growing an Arbitrary-Data Blob: 25::124 After the Keyframe Flush

> You appended a field to an arbitrary-data (ARB) struct, re-registered the
> param with the bigger size, and After Effects put up **"After Effects can't
> continue: internal structure inconsistency (UI context) (25::124)"** a second
> after your keyframes were written. Nothing in your code crashed, no dump was
> produced, and the plugin log ends with a storm of
> `PF_Cmd_ARBITRARY_CALLBACK` on the UI thread. This page is what that is.

## Symptom

- Analyze / bake finishes, keyframes are written (AEGP `SetKeyframe` with your
  ARB handles), your log says "complete".
- AE then fires hundreds of `PF_Cmd_ARBITRARY_CALLBACK` (COMPARE, FLATTEN,
  INTERP, COPY) on the UI thread. Every one returns `PF_Err_NONE`.
- One to thirty seconds later: `25::124`. No exception in your module, so a
  vectored exception handler sees nothing.
- It happens on a layer that ALREADY had keyframes of that ARB param from an
  older build - or on a project saved by an older build.

## Cause

`PF_ArbParamsExtra` hands you a **refcon** (the size you passed to
`PF_ADD_ARBITRARY`) in every callback's param block, and it is tempting to
treat that as *the* blob size:

```c
case PF_Arbitrary_COMPARE_FUNC:
    size = (size_t)extra->u.compare_func_params.refconPV;   // the DECLARED size
    memcmp(PF_LOCK_HANDLE(a), PF_LOCK_HANDLE(b), size);     // <- overread
```

But the handles AE passes are whatever size they were **created at**:

- keyframes restored from a project file are the size the older build
  flattened (`PF_Arbitrary_UNFLATTEN_FUNC` gave you `buf_sizeLu`);
- keyframes the older build wrote are still on the layer next to the new,
  larger ones you just wrote.

So after the blob grows, COMPARE `memcmp`s, FLATTEN `memcpy`s and INTERP reads
`declared - old` bytes past the end of AE's own handles - a heap overread of
host memory on every callback. AE's heap checks catch it later, on the UI
thread, as 25::124. The flush itself succeeds, which is why it looks like a
post-analysis mystery.

## Fix: two invariants in the ARB callback

1. **Every handle you CREATE is the declared size**, zero-filled beyond what
   the source carried. `NEW` already is; make `COPY` and `UNFLATTEN` allocate
   `max(source_size, declared)` and `memset` the tail. A stale small handle
   then never propagates.
2. **Every handle you READ is read no further than its own size**
   (`PF_GET_HANDLE_SIZE`). FLATTEN copies `min(handle, declared, buf_size)`
   and zero-pads to what `FLAT_SIZE` promised; COMPARE and INTERP copy each
   side into declared-size scratch (zero beyond the handle) before touching
   it; INTERP re-allocates an undersized output handle.

```c
static PF_Handle arb_new_sized(PF_InData* in_data, const void* src,
                               size_t src_n, size_t declared) {
    PF_Handle h = PF_NEW_HANDLE((A_HandleSize)declared);
    if (!h) return NULL;
    void* d = PF_LOCK_HANDLE(h);
    if (d) {
        memset(d, 0, declared);
        if (src && src_n) memcpy(d, src, src_n < declared ? src_n : declared);
        PF_UNLOCK_HANDLE(h);
    }
    return h;
}
```

With both invariants, **appending fields to an ARB struct is safe**: an old
keyframe simply reads its new fields as zero. Your plugin-side readers still
have to tolerate the old size (accept `>= legacy_bytes`, zero-fill), and your
interpolator should only blend the new fields when both sides carry them.

## What does NOT fix it

- Bumping a magic number or adding a version field: AE never sees it; the
  overread happens in your callback before you look at the payload.
- Only reading tolerantly on the plugin side: your readers are fine, the
  overread is in the callbacks AE drives.
- Waiting for a crash dump: there is none. Read the plugin log and look at
  what ran right before the error.

## Triage checklist for a 25::124 right after keyframes are written

1. Does the plugin log show the flush completing? (Then it is not the flush.)
2. Is the tail of the log `PF_Cmd_ARBITRARY_CALLBACK` (cmd 22) over and over?
3. Did an ARB struct's size change in this build? Did the layer or project
   have keyframes of that param from before?
4. If yes to all: it is the callbacks reading the declared size. Apply the
   two invariants above.

## Related

- [Arbitrary Data](arbitrary-data.md) - the callback set and when AE calls each.
- [Arb Param Value Is Always NULL](arb-param-null-value-traps.md) - the other
  silent ARB trap (16-bit `id`, discarded `params[]` writes).
- [Arb Callback disk_id and Checkout Context](arb-callback-disk-id-and-checkout-context.md).

*Tags: `arb-data`, `arb-param`, `keyframes`, `25-124`, `backward-compat`, `heap-overread`, `debugging`*
