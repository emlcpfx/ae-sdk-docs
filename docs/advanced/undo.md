# Undo

> 7 Q&As · source: AE plugin dev community Discord

### What causes AEGP_StartUndoGroup(null) to crash in AE 2025?

Starting from AE 2025 beta, passing null/NULL to AEGP_StartUndoGroup causes a crash. Use an empty string "" instead. AEGP_StartUndoGroup("") works correctly and behaves as expected (no entry in the undo stack). This regression has occurred before in earlier AE betas but was corrected.

```cpp
// CRASHES in AE 2025:
suites.UtilitySuite5()->AEGP_StartUndoGroup(null);

// WORKS:
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Tags: `ae-2025`, `aegp`, `crash`, `regression`, `undo`*

---

### What is the correct way to call AEGP_StartUndoGroup in After Effects 2025?

In After Effects 2025 beta and later, AEGP_StartUndoGroup must be called with an empty string "" rather than null. Passing null will cause a crash, while passing an empty string works as expected and does not add an entry to the Undo stack.

```cpp
suites.UtilitySuite5()->AEGP_StartUndoGroup("");
```

*Tags: `aegp`, `debugging`, `macos`, `undo`, `windows`*

---

### How can I group multiple plugin operations into a single undo step in After Effects?

Use StartUndoGroup and EndUndoGroup to bunch multiple operations together into one undo step. This prevents the undo stack from appearing limited when multiple actions are performed.

*Tags: `aegp`, `sdk`, `undo`*

---

### Why are undo groups not working when called from PF_Cmd_USER_CHANGED_PARAM?

There are two common issues: (1) When creating the action string, ensure it is properly null-terminated by using string literal syntax like `const A_char action[] = "Text";` instead of character array syntax. (2) AEGP_StartUndoGroup() cannot accept NULL as a parameter in newer After Effects versions—you must pass an empty string (two consecutive quotes "") instead. Also note that undo groups may only work properly with AEGP-style callbacks, and you should verify that parameter value changes are working in your context.

```cpp
const A_char action[] = "Text";
suites.UtilitySuite5()->AEGP_StartUndoGroup(action);
params[TEXTBOX_GRAD_OFFSETSTARTX]->u.fs_d.value = 50;
params[TEXTBOX_GRAD_OFFSETSTARTX]->uu.change_flags = PF_ChangeFlag_CHANGED_VALUE;
suites.UtilitySuite5()->AEGP_EndUndoGroup();
```

*Tags: `aegp`, `params`, `sdk`, `undo`*

---

### How can I make PF_Topic rename changes non-undoable without cluttering the undo history?

Start and end an undo group with an empty string as the name (two quotes, not NULL). Wrap your AEGP_SetStreamName() call within this undo group. The operation won't appear in the undo queue, but it remains technically undoable—just not listed in the user-visible undo history.

```cpp
// Start undo group with empty name
A_Err err = AEGP_StartUndoGroup("");
// Perform your name change
AEGP_SetStreamName(cur_streamH, new_name);
// End undo group
AEGP_EndUndoGroup();
```

*Tags: `aegp`, `c++`, `params`, `plugin`, `undo`*

---

### How can I detect when the user performs undo or redo actions in my After Effects effect plugin so I can update my custom UI?

One effective approach is to save a random number or state identifier with your data, then poll the effect data (stored in a data parameter) once or twice per second to check if the state index differs from what your UI window expects. If there's a mismatch, the data has changed via undo/redo or project load. To optimize performance, you can save the random number to a separate parameter like a slider. When saving data, use StartUndoGroup() and EndUndoGroup() to bundle operations into a single undo entry. This technique works well for effects with custom external UIs that manage internal parameters in a structure, which is then serialized to an AEGP data parameter using AEGP_SetStreamValue().

*Tags: `aegp`, `debugging`, `params`, `redo`, `ui`, `undo`*

---

### How can I force a render on a custom effect from AEGP without adding the action to the undo/redo stack?

Create an undo group with a NULL instead of a name. Anything that happens in that group will NOT be added to the undo stack. Alternatively, you can use AEGP_SetStreamValue() to change an invisible parameter's value, which will trigger a render without affecting the result, but this approach will add the action to the undo/redo stack unless wrapped in a NULL undo group.

*Tags: `aegp`, `params`, `render-loop`, `undo`*

---
