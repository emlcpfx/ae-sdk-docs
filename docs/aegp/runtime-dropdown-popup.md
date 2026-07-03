# Runtime-Populated Dropdowns: Use a Button + a Native Popup Menu

> Source: PluginPort / Lux 3D + BlendPort (2026-07). AE choice (popup) params are
> fixed at `PF_Cmd_PARAMS_SETUP`, so you cannot show a list whose entries are only
> known at render time (cameras in a `.blend`, materials in a `.glb`, layers in
> the comp). The idiom is a **button** that opens a native OS popup menu built
> from current data and writes the chosen **index** into a plain param.

## The problem

`PF_ADD_POPUP` fixes its entries once, at `PARAMS_SETUP`. There is no API to
repopulate a choice param's items after the effect is created, so any list that
depends on external state cannot be a dropdown:

- the cameras inside the `.blend` a BlendPort effect points at,
- the material names inside the `.glb` a Lux 3D mesh uses,
- the light / null names discovered when a scene loads.

Falling back to a raw **index** ("Camera Index = 3", "Material Slot = 2") works
but is guess-and-check for the user.

## The pattern

Pair a **button** param with a plain int (or popup) param that stores the result.
The button's click handler -- a `PF_Cmd_USER_CHANGED_PARAM` on the button, or
your framework's `param_changed` -- opens a **native popup menu** populated at
click time from whatever data you have, and writes the picked index back with
AEGP.

On Windows that is `TrackPopupMenu`; on macOS the equivalent is an `NSMenu` shown
with `-popUpMenuPositioningItem:atLocation:inView:` (or `+[NSMenu popUpContextMenu:...]`).

```cpp
// param_changed / USER_CHANGED_PARAM handler for the "Pick..." button.
void pick_popup(PF_InData* in_data) {
    std::vector<std::string> names = current_list();     // from your runtime data
    int current = read_index_param(in_data, "target_index");

    HMENU menu = CreatePopupMenu();
    for (size_t i = 0; i < names.size() && i < 64; ++i) {
        UINT flags = MF_STRING | ((int)i == current ? MF_CHECKED : 0u);
        AppendMenuA(menu, flags, (UINT_PTR)(i + 1), names[i].c_str());  // id 0 = "cancelled"
    }

    HWND owner = GetActiveWindow();
    if (!owner) owner = GetForegroundWindow();
    POINT pt; GetCursorPos(&pt);
    if (owner) SetForegroundWindow(owner);   // documented TrackPopupMenu nudge
    UINT sel = TrackPopupMenu(menu, TPM_RETURNCMD | TPM_NONOTIFY,
                              pt.x, pt.y, 0, owner, nullptr);
    DestroyMenu(menu);

    if (sel > 0)                              // 0 = user dismissed
        set_index_param(in_data, "target_index", sel - 1);   // menu id (i+1) -> index i
}
```

## Gotchas

- **Open the menu synchronously from the click handler.** Deferring the
  `TrackPopupMenu` to `PF_Event_IDLE` to avoid its modal loop re-entering AE
  sounds safer, but the effect's idle event does not fire reliably enough and the
  menu never appears. The AEGP param write *after* the pick is valid from the
  `USER_CHANGED_PARAM` command context, and the modal pump is a brief,
  user-driven window, so open it where it actually works.

- **Write the result through the correct AEGP index.** Resolve the target param's
  real *flat* AE param index (from your own param table, or by matching the
  match-name), then `AEGP_SetStreamValue`. Do **not** reuse a `DynamicStreamSuite`
  child-walk index with `AEGP_GetNewEffectStreamByIndex` -- that is a different
  index space and asserts `5027:150`. See
  [Walking Effect Streams](stream-walking.md).

- **The stored value is a plain index, not the name.** The menu is display-only;
  what persists in the project is the index. If items can reorder between sessions,
  prefer a stable key (a name written into an arbitrary-data param) and resolve it
  at render.

- **Populate lazily.** Build the menu from whatever data you have *at click time*
  (a cached file parse, a fresh AEGP walk). If nothing is loaded yet, append a
  single greyed `(render a frame first)` item so the button still opens and
  explains itself.

- **Windows-only, so far.** `TrackPopupMenu` is Win32; a Mac build needs the
  `NSMenu` equivalent. Gate the Win32 path behind `#ifdef _WIN32` and no-op
  elsewhere until the Cocoa path lands.

## How to avoid the guess-and-check UI

Any time a plugin param would ideally be "pick one of a list AE can't know until
runtime," reach for this button + native-menu pattern instead of a raw numeric
index. It keeps the persisted value a simple, stable scalar (good for saved
projects and cross-effect reads) while giving the user a real by-name picker.

## See also

- [Walking Effect Streams](stream-walking.md) -- resolve the write index
  correctly; the `5027` asserts you hit if you do not.
- [Choice / Popup Params Are 1-Based](choice-popup-params.md) -- if you store the
  result in an actual popup param, remember it is 1-based on the wire.

*Tags: `aegp`, `params`, `popup`, `custom-ui`, `win32`, `trackpopupmenu`*
