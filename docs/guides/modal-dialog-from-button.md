# Blocking Modal Dialogs from a Button Param (Mocha-style pop-ups)

Some effects need a richer pop-up than the Effect Controls panel can show -- a
multi-select list, a picker, a mini editor. Mocha Pro's **Visible Layers...**
button and Red Giant Universe's text pop-ups do this with a **blocking, custom
native modal window**: a Win32 `HWND` on Windows, a Cocoa `NSPanel` on macOS,
opened synchronously from a button and dismissed with OK/Cancel.

This is **not** a CEP panel or an AEGP docked panel -- those are non-modal and
asynchronous. If the dialog blocks the UI until you dismiss it, it is a native
modal run on the host's main thread.

The whole pattern is four steps:

1. A **button** parameter.
2. The click arrives as `PF_Cmd_USER_CHANGED_PARAM`, **on AE's main UI thread**.
3. You open a **native modal** parented to AE and pump its message loop.
4. On OK you **write the result back into a parameter** (and re-render if needed).

A complete, self-contained implementation of the Windows modal lives in
[emlcpfx/AE_ModalDialog_Example](https://github.com/emlcpfx/AE_ModalDialog_Example);
this page explains the mechanism.

## 1. The button

Raw SDK, in `PF_Cmd_PARAM_SETUP`:

```c
PF_ParamDef def;
AEFX_CLR_STRUCT(&def);
PF_ADD_BUTTON("Materials", "Visible Materials...",
              0, PF_ParamFlag_SUPERVISE, VISIBLE_BTN_DISK_ID);
```

`PF_ParamFlag_SUPERVISE` makes AE send you `PF_Cmd_USER_CHANGED_PARAM` on the
click. (In PluginPort: `p.add_button("pick_visible", "Visible Materials...")` --
buttons are supervised automatically.)

## 2. The click -> your handler, on the main thread

Raw SDK -- handle the command and match the button's param index:

```c
case PF_Cmd_USER_CHANGED_PARAM: {
    PF_UserChangedParamExtra* extra = (PF_UserChangedParamExtra*)extraP;
    if (extra->param_index == VISIBLE_BTN_INDEX)
        err = OpenVisibilityDialog(in_data, out_data, params);
    break;
}
```

PluginPort -- override `param_changed`:

```cpp
void param_changed(const std::string& id) override {
    if (id == "pick_visible") open_visibility_dialog();
}
```

`PF_Cmd_USER_CHANGED_PARAM` is delivered on AE's **main UI thread** with a live
`in_data`. That is exactly the thread AE would block on for its own modal
dialogs, so **running a blocking modal loop here is correct** -- AE simply waits
for you to return. (Do **not** try this from a render thread; see
[Multi-Frame Rendering](../memory-threading/index.md).)

## 3. Open the native modal

### Getting the parent window (the part people get stuck on)

There is **no AE suite that hands you the host's window handle**, and you do not
need one. Use the OS foreground/active window as the owner:

```cpp
HWND owner = GetActiveWindow();
if (!owner) owner = GetForegroundWindow();
```

On macOS the analogue is the frontmost window: `[NSApp keyWindow]` (or
`[NSApp mainWindow]`).

### The simple case: a built-in modal

If a stock dialog does the job, just call it -- these already block and take a
parent `HWND`:

```cpp
// File / folder picker (COM), message box, color picker, font dialog...
IFileOpenDialog* dlg = /* CoCreateInstance(CLSID_FileOpenDialog...) */;
dlg->Show(GetForegroundWindow());   // blocks until the user is done
```

### The custom case: your own window + a modal loop

For a bespoke UI (a checkbox/eye list, a mini editor), create a window and run a
**nested modal message loop**, disabling the owner for the duration:

```cpp
HWND dlg = CreateWindowExW(WS_EX_DLGMODALFRAME, kClass, L"Visible Materials",
                           WS_POPUP | WS_CAPTION | WS_SYSMENU,
                           x, y, w, h, owner, nullptr, hInst, &state);

if (owner) EnableWindow(owner, FALSE);   // modal: block the parent
ShowWindow(dlg, SW_SHOW);
SetForegroundWindow(dlg);

MSG msg;                                  // pump until OK/Cancel/close
while (!state.done && GetMessageW(&msg, nullptr, 0, 0) > 0) {
    if (!IsDialogMessageW(dlg, &msg)) {   // Tab/arrow nav, Enter->OK, Esc->Cancel
        TranslateMessage(&msg);
        DispatchMessageW(&msg);
    }
}

if (owner) EnableWindow(owner, TRUE);     // re-enable BEFORE destroying
DestroyWindow(dlg);
if (owner) { SetActiveWindow(owner); SetForegroundWindow(owner); }
```

Give the OK button control id `IDOK` (1) and Cancel `IDCANCEL` (2): then
`IsDialogMessage` maps **Enter -> OK** (via `BS_DEFPUSHBUTTON`) and **Esc ->
Cancel** for free.

This is the same loop `DialogBoxParam` runs internally; hand-rolling it just lets
the window size and contents be dynamic (e.g. one row per material) without a
`.rc` `DLGTEMPLATE`.

## 4. Write the result back into a param

The dialog's job is to produce a value and store it where render can read it.
For a multi-item toggle (N materials), a scalar param can't hold the set -- pack
it into an **integer bitmask** (or a string/ARB param).

AEGP write-back (raw SDK). Get the effect's stream by index and set it -- **you
must set `value.streamH` before `AEGP_SetStreamValue`** or it silently no-ops:

```c
AEGP_StreamRefH streamH = NULL;
suites.StreamSuite6()->AEGP_GetNewEffectStreamByIndex(pluginID, effectH,
                                                      MASK_PARAM_INDEX, &streamH);
AEGP_StreamValue2 val;
AEFX_CLR_STRUCT(&val);
val.streamH   = streamH;          // <-- required, easy to forget
val.val.one_d = (A_FpLong)newMask;
suites.StreamSuite6()->AEGP_SetStreamValue(pluginID, streamH, &val);
suites.StreamSuite6()->AEGP_DisposeStream(streamH);
```

PluginPort collapses that to one call that resolves the authoritative param
index and triggers a re-render:

```cpp
pluginport::ae::set_effect_param_value(in_data, "visibility_mask", (double)newMask);
```

If the effect's *output* depends on the new value, make sure AE re-pulls the
frame. Setting the value through AEGP usually dirties the render tree; if it does
not (e.g. the effect realises the change on a *different* layer), force it -- see
[NON_PARAM_VARY / force re-render](../parameters/non-param-vary.md).

!!! warning "Bitmask, not a string, if a scene-walk reads it"
    A scalar (int bitmask) is readable by another effect walking this effect's
    streams. A string/ARB blob is **not** reachable by a scalar stream walk --
    fine if only this effect reads it, a trap if a companion plugin does.

## Caveat: dialogs that edit the project

Writing your **own** params from `PF_Cmd_USER_CHANGED_PARAM` is safe and
synchronous. But if the dialog also **creates or reorders layers / footage**,
that is an *undoable AEGP project edit*, and doing it in some command contexts
throws ("modify read-only flag" / invalid parameter). Defer those to
`PF_Event_IDLE` (or an app-level AEGP idle hook). See
[AEGP idle-hook deferral](../aegp/index.md).

## Optional polish

- **Dark title bar** (Win10/11): `DwmSetWindowAttribute(hwnd,
  DWMWA_USE_IMMERSIVE_DARK_MODE, &TRUE, ...)`, plus
  `DWMWA_WINDOW_CORNER_PREFERENCE` (rounded) and `DWMWA_CAPTION_COLOR`. Load it
  with `GetProcAddress("dwmapi.dll","DwmSetWindowAttribute")` so you don't add a
  link dependency; unsupported attributes are ignored on older builds.
- **Crisp icons**: draw eyes/toggles from the **Segoe MDL2 Assets** icon font
  (present on all Win10/11), e.g. the "RedEye" glyph, rather than hand-rolled GDI
  shapes. Owner-draw (`BS_OWNERDRAW` + `WM_DRAWITEM`) the OK/Cancel buttons for a
  dark theme.

!!! danger "Write icon-font glyphs as `\uXXXX` escapes, not literal characters"
    Private-Use-Area glyphs (like the MDL2 eye) are silently stripped or
    corrupted by many editors, terminals, and copy/paste paths, and a raw UTF-8
    glyph can mojibake under MSVC without `/utf-8`. Always write the
    **universal-character-name escape** in source: `L"\uE7B3"`, not the literal
    character. The bytes stay 7-bit ASCII and survive every pipe.

## macOS

The Cocoa equivalent: build an `NSPanel` (dark via
`NSAppearanceNameDarkAqua`), add an eye-toggle `NSButton` per row, and block on
`[NSApp runModalForWindow:panel]` / `[NSApp stopModal]`, parented via
`[NSApp keyWindow]`. Same four-step shape; different windowing API.

## See also

- [Button parameters](../parameters/button-param.md)
- [PF_Cmd_USER_CHANGED_PARAM](../parameters/user-changed-param.md)
- [AEGP stream values](../parameters/aegp-stream.md)
- Full example: [emlcpfx/AE_ModalDialog_Example](https://github.com/emlcpfx/AE_ModalDialog_Example)
