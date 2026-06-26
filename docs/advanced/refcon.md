# Refcon

> 2 Q&As · source: AE plugin dev community Discord

### How do I work with multiple arbitrary (ARB) parameters in an AE effect?

Each ARB param is its own thing with its own routines. Use ARB_REFCON to route handling: it's pointer-sized and can point to a function or class instance that handles that specific param. This eliminates the need for switch statements. Each ARB needs a unique disk ID and a separately allocated default handle. PF_CustomUIInfo registration only needs to be done once per PARAM_SETUP call (not per param). For a scalable approach, create a base class for generic ARB handling and override specific methods per param type. The refconPV content is entirely up to you - AE is indifferent to it.

```cpp
switch(param_index) {
    case ARB_1_INDEX:
        ArbHandler1();
        break;
    case ARB_2_INDEX:
        ArbHandler2();
        break;
}
```

*Tags: `arb-param`, `class-design`, `custom-ui`, `multiple-params`, `param-routing`, `refcon`*

---

### Why does data passed via AEGP_IdleRefcon become corrupted when retrieved in the idle hook?

The issue occurs when you save the address of a locally-defined variable in the entry point function and cast it to AEGP_IdleRefcon. Once the function scope ends, that stack memory becomes invalid and may be reused by other code, causing the data to appear corrupted when accessed later in the idle hook. Solution: Use dynamic allocation (new/delete or preferably AE's memory suite) to allocate the data structure on the heap, which remains valid until explicitly freed. Always deallocate the memory in global setdown.

```cpp
// Wrong - local variable on stack:
my_global_data myStruct;
myStruct.my_int = 5;
my_global_dataP globP = &myStruct;  // invalid after function ends

// Correct - heap allocation:
my_global_dataP globP = new my_global_data();
globP->my_int = 5;
AEGP_IdleRefcon idleRefcon = reinterpret_cast<AEGP_IdleRefcon>(globP);
// Later in global setdown:
delete globP;
```

*Tags: `aegp`, `debugging`, `memory`, `refcon`*

---
