# Stream

> 1 Q&A · source: AE plugin dev community Discord

### When should I set expressions on parameters in a plugin to ensure proper undo behavior?

While UpdateParameterUI is a common location, consider using USER_CHANGED_PARAM instead if UpdateParameterUI causes undo stack issues. USER_CHANGED_PARAM may provide better control over when expressions are applied and how they integrate with After Effects' undo system.

```cpp
ERR(suites.StreamSuite4()->AEGP_SetExpression(globP->my_id, aegp_get_position_streamH, ("transform.position")));
```

*Tags: `aegp`, `expression`, `params`, `stream`*

---
