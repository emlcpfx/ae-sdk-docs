# Update

> 1 Q&A · source: AE plugin dev community Discord

### How can I update plugin parameters when the effect is first applied to a layer?

Parameter value changes made directly during UPDATE_PARAMS_UI are ignored by After Effects. Instead, use SetStreamValue to modify parameter values during initialization, which will persist the changes. While not considered best practice, this is the reliable method to initialize parameters on first application of the effect.

```cpp
params[EM_TOPLEFT]->u.td.x_value = FLOAT2FIX(globP->x0);
params[EM_TOPLEFT]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
```

*Tags: `aegp`, `params`, `sdk`, `update`*

---
