# Callback

> 2 Q&As · source: AE plugin dev community Discord

### How can I move a layer from one composition to another or create a precomposition from a layer using AEGP calls?

You can trigger the precompose command just like any other After Effects command through AEGP. However, if you attempt to execute this during a PF_Cmd_USER_CHANGED_PARAM callback, it will crash because precomposing invalidates the effect from which you're currently operating. To avoid this crash, you need to call the precompose command from an AEGP idle hook instead, which executes the operation outside of the current effect's execution context.

*Tags: `aegp`, `callback`, `idle-hook`, `plugin-crash`, `precomp`*

---

### How can you determine which arbitrary parameter is being referenced in callback handlers that are shared across multiple arb params?

Use the C style approach with a switch statement on the disk_id that AE transfers with each PF_Cmd_ARBITRARY_CALLBACK call, or use the C++ style approach where you place a pointer to an arb handling class in the arb definition parameter. This pointer is sent back with every callback, allowing you to use virtual methods to handle different arb types. The C++ approach with virtual methods is cleaner and easier to maintain than switch statements.

```cpp
// C style with disk_id switch
case PF_Cmd_ARBITRARY_CALLBACK:
  switch(params->arb_d.disk_id) {
    case ARB_PARAM_1:
      return Handle_Arb1(arbH);
    case ARB_PARAM_2:
      return Handle_Arb2(arbH);
  }

// C++ style - pointer passed back in callback
// Arb definition includes: (void*)arbHandlerPtr
// In callback, cast back and call virtual methods
```

*Tags: `aegp`, `arb-data`, `callback`, `params`*

---
