# Param Routing

> 1 Q&A · source: AE plugin dev community Discord

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
