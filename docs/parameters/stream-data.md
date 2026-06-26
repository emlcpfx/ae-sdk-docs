# Stream Data

> 1 Q&A · source: AE plugin dev community Discord

### What is the correct way to extract arbitrary data from a stream value in After Effects plugins?

The correct approach is to cast the stream value's arbH handle to a PF_Handle, then use the HandleSuite1 to lock the handle and access the arbitrary data. However, it's critical to ensure you either complete all data operations before releasing the stream value, or copy the data before release. Otherwise, you may be left with an invalidated pointer that could cause crashes or undefined behavior. The stream value should be disposed with AEGP_DisposeStreamValue after you're done with it.

```cpp
PF_Handle arbH = reinterpret_cast<PF_Handle>(streamVal.val.arbH);
arbData *arbP = reinterpret_cast<arbDataP>(suites.HandleSuite1()->host_lock_handle(arbH));
// Use arbP data here before disposing
suites.StreamSuite5()->AEGP_DisposeStreamValue(&streamVal);
```

*Tags: `aegp`, `arb-data`, `memory`, `stream-data`*

---
