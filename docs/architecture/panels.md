# AEGP Panels: Dockable UI Panels in After Effects

AEGP panels allow plugins to create custom dockable UI panels that integrate seamlessly into After Effects' workspace system. These panels appear in the Window menu, can be docked alongside native panels like Timeline and Composition, participate in workspace presets, and persist across sessions. The `Panelator` SDK example demonstrates the complete lifecycle and implementation pattern.

---

## Overview

An AEGP panel plugin is an AEGP plugin (not an effect plugin) that:

1. Registers a menu command in the Window menu
2. Registers a panel creation hook via `AEGP_PanelSuite1`
3. Creates platform-native UI inside a container window provided by AE
4. Handles flyout menus, snap sizes, and panel lifecycle

The architecture uses a callback-based design: AE provides a container window, and the plugin fills it with native platform controls and custom drawing.

---

## Required Headers and Suites

```cpp
#include "AE_GeneralPlug.h"         // AEGP base types
#include "AE_GeneralPlugPanels.h"    // Panel-specific types and suites
#include "AEGP_SuiteHandler.h"       // Suite access wrapper
#include "SuiteHelper.h"             // Template-based suite helper
```

### AEGP_PanelSuite1

Defined in `AE_GeneralPlugPanels.h`, this suite provides all panel operations:

```cpp
#define kAEGPPanelSuite           "AEGP Workspace Panel Suite"
#define kAEGPPanelSuiteVersion1   1  // Frozen in AE 8.0

typedef struct {
    SPAPI A_Err (*AEGP_RegisterCreatePanelHook)(
        AEGP_PluginID          in_plugin_id,
        const A_u_char*        in_utf8_match_nameZ,   // Do not localize
        AEGP_CreatePanelHook   in_create_panel_hook,
        AEGP_CreatePanelRefcon in_refconP,
        A_Boolean              in_paint_backgroundB);

    SPAPI A_Err (*AEGP_UnRegisterCreatePanelHook)(
        const A_u_char*        in_utf8_match_nameZ);

    SPAPI A_Err (*AEGP_SetTitle)(
        AEGP_PanelH            in_panelH,
        const A_u_char*        in_utf8_nameZ);

    SPAPI A_Err (*AEGP_ToggleVisibility)(
        const A_u_char*        in_utf8_match_nameZ);

    SPAPI A_Err (*AEGP_IsShown)(
        const A_u_char*        in_utf8_match_nameZ,
        A_Boolean*             out_tab_is_shownB,
        A_Boolean*             out_panel_is_frontmostB);
} AEGP_PanelSuite1;
```

| Function | Purpose |
|---|---|
| `AEGP_RegisterCreatePanelHook` | Register the panel factory function |
| `AEGP_UnRegisterCreatePanelHook` | Remove a registered panel |
| `AEGP_SetTitle` | Change the panel tab title at runtime |
| `AEGP_ToggleVisibility` | Toggle panel visibility (show/hide/bring to front) |
| `AEGP_IsShown` | Query whether the panel is visible and/or frontmost |

---

## Platform View Reference

The panel container is a native platform window:

```cpp
// macOS (64-bit):
typedef NSView *AEGP_PlatformViewRef;

// Windows:
typedef HWND AEGP_PlatformViewRef;
```

Your plugin receives this view in the `CreatePanelHook` callback and is responsible for creating child controls and handling paint/resize messages within it.

---

## The Panelator Example Architecture

The Panelator example uses a three-class design:

```
Panelator (AEGP plugin core)
    |
    |-- Registers menu command, panel hook
    |-- Handles command hook (toggle visibility)
    |-- Handles menu update hook (check mark)
    |
    +-- Creates PanelatorUI_Plat per panel instance
            |
            |-- PanelatorUI (cross-platform base)
            |       |-- Manages flyout menu
            |       |-- Manages snap sizes
            |       |-- Tracks color state
            |
            +-- PanelatorUI_Plat (platform-specific)
                    |-- Windows: HWND subclassing, WM_PAINT
                    |-- macOS: NSView-based (minimal in example)
```

---

## Plugin Registration and Entry Point

### PiPL

The panel is an AEGP plugin, so its PiPL uses `Kind { AEGP }`:

```c
resource 'PiPL' (16000) {
    {
        Kind { AEGP },
        Name { "Panelator" },
        Category { "General Plugin" },
        Version { 196608 },
#ifdef AE_OS_WIN
    #ifdef AE_PROC_INTELx64
        CodeWin64X86 {"EntryPointFunc"},
    #endif
#endif
    }
};
```

### Entry Point

```cpp
A_Err EntryPointFunc(
    struct SPBasicSuite   *pica_basicP,
    A_long                 major_versionL,
    A_long                 minor_versionL,
    AEGP_PluginID          aegp_plugin_id,
    AEGP_GlobalRefcon      *global_refconP)
{
    *global_refconP = (AEGP_GlobalRefcon) new Panelator(pica_basicP, aegp_plugin_id);
    return A_Err_NONE;
}
```

The `Panelator` object is stored as the global refcon, making it available to all subsequent callbacks.

---

## Panel Registration

The `Panelator` constructor performs all registration:

```cpp
Panelator(SPBasicSuite *pica_basicP, AEGP_PluginID pluginID)
    : i_pica_basicP(pica_basicP),
      i_pluginID(pluginID),
      i_sp(pica_basicP),
      i_ps(pica_basicP),
      i_match_nameZ((A_u_char*)STR(StrID_Name))
{
    // 1. Get a unique command ID
    PT_ETX(i_sp.CommandSuite1()->AEGP_GetUniqueCommand(&i_command));

    // 2. Insert into the Window menu
    PT_ETX(i_sp.CommandSuite1()->AEGP_InsertMenuCommand(
        i_command, STR(StrID_Name),
        AEGP_Menu_WINDOW, AEGP_MENU_INSERT_SORTED));

    // 3. Register command hook (for menu clicks)
    PT_ETX(i_sp.RegisterSuite5()->AEGP_RegisterCommandHook(
        i_pluginID, AEGP_HP_BeforeAE, i_command,
        &Panelator::S_CommandHook, (AEGP_CommandRefcon)(this)));

    // 4. Register menu update hook (for check marks)
    PT_ETX(i_sp.RegisterSuite5()->AEGP_RegisterUpdateMenuHook(
        i_pluginID, &Panelator::S_UpdateMenuHook, NULL));

    // 5. Register the panel creation hook
    PT_ETX(i_ps->AEGP_RegisterCreatePanelHook(
        i_pluginID, i_match_nameZ,
        &Panelator::S_CreatePanelHook,
        (AEGP_CreatePanelRefcon)this,
        true));  // true = AE paints the background
}
```

### The Match Name

The match name (`i_match_nameZ`) is the unique identifier for this panel. It must be a UTF-8 string, must not be localized, and must remain stable across versions. AE uses it to:

- Associate the panel with saved workspace layouts
- Identify the panel for `ToggleVisibility` and `IsShown` calls
- Persist the panel's docking state across sessions

---

## The Panel Creation Hook

When AE needs to create the panel (either at startup for a saved workspace, or when the user selects the menu command), it calls the registered hook:

```cpp
typedef A_Err (*AEGP_CreatePanelHook)(
    AEGP_GlobalRefcon      plugin_refconP,
    AEGP_CreatePanelRefcon refconP,
    AEGP_PlatformViewRef   container,         // The native window/view to fill
    AEGP_PanelH            panelH,            // Handle for suite calls
    AEGP_PanelFunctions1*  outFunctionTable,  // You fill this in
    AEGP_PanelRefcon*      outRefcon);         // Per-panel refcon
```

The Panelator implementation:

```cpp
void CreatePanelHook(
    AEGP_PlatformViewRef container,
    AEGP_PanelH          panelH,
    AEGP_PanelFunctions1* outFunctionTable,
    AEGP_PanelRefcon*    outRefcon)
{
    *outRefcon = reinterpret_cast<AEGP_PanelRefcon>(
        new PanelatorUI_Plat(i_pica_basicP, panelH,
                             container, outFunctionTable));
}
```

This creates a new `PanelatorUI_Plat` instance for each panel. Multiple instances can exist simultaneously (e.g., if the user opens the panel in multiple workspaces or docks).

---

## Panel Function Table

The `AEGP_PanelFunctions1` structure contains three function pointers that your plugin must fill in:

```cpp
typedef struct {
    A_Err (*GetSnapSizes)(AEGP_PanelRefcon refcon,
                          A_LPoint* snapSizes,
                          A_long* numSizesP);

    A_Err (*PopulateFlyout)(AEGP_PanelRefcon refcon,
                            AEGP_FlyoutMenuItem* itemsP,
                            A_long* in_out_numItemsP);

    A_Err (*DoFlyoutCommand)(AEGP_PanelRefcon refcon,
                             AEGP_FlyoutMenuCmdID commandID);
} AEGP_PanelFunctions1;
```

The Panelator sets these in the `PanelatorUI` constructor:

```cpp
PanelatorUI::PanelatorUI(SPBasicSuite* spbP, AEGP_PanelH panelH,
                         AEGP_PlatformViewRef platformWindowRef,
                         AEGP_PanelFunctions1* outFunctionTable)
{
    outFunctionTable->DoFlyoutCommand = S_DoFlyoutCommand;
    outFunctionTable->GetSnapSizes    = S_GetSnapSizes;
    outFunctionTable->PopulateFlyout  = S_PopulateFlyout;
}
```

---

## Snap Sizes

Snap sizes define preferred panel dimensions. When the user resizes a docked panel, AE "snaps" to these sizes:

```cpp
void PanelatorUI::GetSnapSizes(A_LPoint* snapSizes, A_long* numSizesP)
{
    snapSizes[0].x = 100;
    snapSizes[0].y = 100;
    snapSizes[1].x = 200;
    snapSizes[1].y = 400;
    *numSizesP = 2;
}
```

> **Constraint**: No more than 5 snap sizes. The SDK comment in `AE_GeneralPlugPanels.h` states: "no more than 5 snap sizes. Our algo can't really cope and it confuses the user."

---

## Flyout Menus

The flyout menu is the small hamburger/arrow menu in the panel's tab. It is declared as a flat array of `AEGP_FlyoutMenuItem` structures:

```cpp
typedef struct {
    AEGP_FlyoutMenuIndent   indent;     // 1 = top-level, 2 = submenu, etc.
    AEGP_FlyoutMenuMarkType type;       // Normal, Checked, Radio, Separator
    A_Boolean               enabledB;   // Enabled state
    AEGP_FlyoutMenuCmdID    cmdID;      // Command ID (your custom enum)
    const A_u_char*         utf8NameZ;  // Menu item text (UTF-8)
} AEGP_FlyoutMenuItem;
```

### Mark Types

```cpp
enum {
    AEGP_FlyoutMenuMarkType_NORMAL,
    AEGP_FlyoutMenuMarkType_CHECKED,
    AEGP_FlyoutMenuMarkType_RADIO_BULLET,
    AEGP_FlyoutMenuMarkType_SEPARATOR
};
```

### The Panelator's Flyout Menu

```cpp
AEGP_FlyoutMenuItem myMenu[] = {
    // indent type                         enabled cmdID                 text
    {1, AEGP_FlyoutMenuMarkType_NORMAL,    FALSE,  AEGP_FlyoutMenuCmdID_NONE, "Hi!"},
    {1, AEGP_FlyoutMenuMarkType_SEPARATOR, TRUE,   AEGP_FlyoutMenuCmdID_NONE, NULL},
    {1, AEGP_FlyoutMenuMarkType_NORMAL,    TRUE,   AEGP_FlyoutMenuCmdID_NONE, "Set BG Color"},
    {2, AEGP_FlyoutMenuMarkType_NORMAL,    TRUE,   PT_MenuCmd_RED,            "Red"},
    {2, AEGP_FlyoutMenuMarkType_NORMAL,    TRUE,   PT_MenuCmd_GREEN,          "Green"},
    {2, AEGP_FlyoutMenuMarkType_NORMAL,    TRUE,   PT_MenuCmd_BLUE,           "Blue"},
    {1, AEGP_FlyoutMenuMarkType_NORMAL,    TRUE,   PT_MenuCmd_STANDARD,       "Normal Fill Color"},
    {1, AEGP_FlyoutMenuMarkType_NORMAL,    TRUE,   AEGP_FlyoutMenuCmdID_NONE, "Set Title"},
    {2, AEGP_FlyoutMenuMarkType_NORMAL,    TRUE,   PT_MenuCmd_TITLE_LONGER,   "Longer"},
    {2, AEGP_FlyoutMenuMarkType_NORMAL,    TRUE,   PT_MenuCmd_TITLE_SHORTER,  "Shorter"},
};
```

The `indent` field controls submenu nesting. Items with indent 1 are top-level; items with indent 2 appear as children of the preceding indent-1 item.

Items with `AEGP_FlyoutMenuCmdID_NONE` are non-actionable (headers or disabled items). Items with a custom command ID trigger `DoFlyoutCommand`.

### Handling Flyout Commands

```cpp
void PanelatorUI::DoFlyoutCommand(AEGP_FlyoutMenuCmdID commandID)
{
    switch (commandID) {
        case PT_MenuCmd_RED:
            i_use_bg = false;
            red = PF_MAX_CHAN8; green = blue = 0;
            break;
        case PT_MenuCmd_GREEN:
            i_use_bg = false;
            green = PF_MAX_CHAN8; red = blue = 0;
            break;
        case PT_MenuCmd_BLUE:
            i_use_bg = false;
            blue = PF_MAX_CHAN8; red = green = 0;
            break;
        case PT_MenuCmd_STANDARD:
            i_use_bg = true;
            break;
        case PT_MenuCmd_TITLE_LONGER:
            i_panelSuite->AEGP_SetTitle(i_panelH,
                reinterpret_cast<const A_u_char*>("This is a longer name"));
            break;
        case PT_MenuCmd_TITLE_SHORTER:
            i_panelSuite->AEGP_SetTitle(i_panelH,
                reinterpret_cast<const A_u_char*>("P!"));
            break;
    }
    InvalidateAll();  // Trigger repaint
}
```

Note the use of `AEGP_SetTitle` to dynamically change the panel's tab name.

---

## Command Hook: Window Menu Toggle

When the user clicks the panel's entry in the Window menu:

```cpp
void CommandHook(AEGP_Command command,
                 AEGP_HookPriority hook_priority,
                 A_Boolean already_handledB,
                 A_Boolean *handledPB)
{
    if (command == i_command) {
        PT_ETX(i_ps->AEGP_ToggleVisibility(i_match_nameZ));
    }
}
```

`AEGP_ToggleVisibility` implements the standard Window menu behavior:
- If the panel is not in the workspace, it creates it
- If the panel exists but is not the frontmost tab, it brings it to front
- If the panel is visible and frontmost, it closes it

### Menu Check Mark

The update menu hook maintains the check mark:

```cpp
void UpdateMenuHook(AEGP_WindowType active_window)
{
    PT_ETX(i_sp.CommandSuite1()->AEGP_EnableCommand(i_command));

    A_Boolean out_thumb_is_shownB = FALSE, out_panel_is_frontmostB = FALSE;
    PT_ETX(i_ps->AEGP_IsShown(i_match_nameZ,
                               &out_thumb_is_shownB,
                               &out_panel_is_frontmostB));
    PT_ETX(i_sp.CommandSuite1()->AEGP_CheckMarkMenuCommand(
        i_command,
        out_thumb_is_shownB && out_panel_is_frontmostB));
}
```

The check mark appears only when the panel is both shown and frontmost in its tab group.

---

## Platform-Specific Drawing (Windows)

The Windows `PanelatorUI_Plat` class subclasses the container HWND:

```cpp
PanelatorUI_Plat::PanelatorUI_Plat(SPBasicSuite* spbP,
    AEGP_PanelH panelH,
    AEGP_PlatformViewRef platformWindowRef,
    AEGP_PanelFunctions1* outFunctionTable)
    : PanelatorUI(spbP, panelH, platformWindowRef, outFunctionTable),
      i_prevWindowProc(NULL)
{
    // Subclass the window
    i_prevWindowProc = (WindowProc)GetWindowLongPtr(
        platformWindowRef, GWLP_WNDPROC);
    SetWindowLongPtrA(platformWindowRef, GWLP_WNDPROC,
        (LONG_PTR)PanelatorUI_Plat::StaticOSWindowWndProc);
    ::SetProp(platformWindowRef, OSWndObjectProperty, (HANDLE)this);

    // Create a child button control
    HWND btn = CreateWindow("BUTTON", "Click Me",
        WS_CHILD | WS_VISIBLE | BS_CENTER | BS_VCENTER | BS_TEXT,
        10, 200, 100, 30,
        platformWindowRef, (HMENU)BtnID, NULL, NULL);
}
```

### Window Procedure

The subclassed window procedure handles painting, resizing, and button clicks:

```cpp
LRESULT PanelatorUI_Plat::OSWindowWndProc(HWND hWnd, UINT message,
                                           WPARAM wParam, LPARAM lParam)
{
    bool handledB = false;
    switch (message) {
        case WM_PAINT: {
            RECT clientArea;
            GetClientRect(hWnd, &clientArea);
            HBRUSH brush = NULL;

            if (i_use_bg) {
                // Use AE's theme color
                brush = CreateBrushFromSelector(
                    i_appSuite.get(), PF_App_Color_PANEL_BACKGROUND);
            } else {
                // Use custom color
                brush = CreateSolidBrush(RGB(red, green, blue));
            }

            HDC hdc = GetDC(hWnd);
            FillRect(hdc, &clientArea, brush);
            DeleteObject(brush);

            // Draw AE theme color swatches
            // ... (draws several colored bands using PF_App_Color_* selectors)

            // Display panel dimensions
            sprintf_s(textToDraw, "Size: %d, %d",
                      clientArea.right - clientArea.left,
                      clientArea.bottom - clientArea.top);
            DrawText(hdc, textToDraw, strlen(textToDraw),
                     &clientArea, DT_SINGLELINE | DT_LEFT);

            // Display click count
            sprintf_s(textToDraw, "NumClicks: %d", i_numClicks);
            DrawText(hdc, ...);

            handledB = true;
        }

        case WM_SIZING:
            InvalidateAll();
            break;

        case WM_COMMAND:
            if (HIWORD(wParam) == BN_CLICKED && LOWORD(wParam) == BtnID) {
                i_numClicks++;
                InvalidateAll();
                handledB = true;
            }
            break;
    }

    if (i_prevWindowProc && !handledB) {
        return CallWindowProc(i_prevWindowProc, hWnd, message, wParam, lParam);
    } else {
        return DefWindowProc(hWnd, message, wParam, lParam);
    }
}
```

### Theme Colors via PFAppSuite

The example reads AE's current theme colors using `PFAppSuite4`:

```cpp
HBRUSH CreateBrushFromSelector(PFAppSuite4* appSuiteP, PF_App_ColorType sel)
{
    PF_App_Color appColor = {0};
    appSuiteP->PF_AppGetColor(sel, &appColor);
    return CreateSolidBrush(
        RGB(appColor.red/255, appColor.green/255, appColor.blue/255));
}
```

Common color selectors used by Panelator:

| Selector | Usage |
|---|---|
| `PF_App_Color_PANEL_BACKGROUND` | Panel background color |
| `PF_App_Color_LIST_BOX_FILL` | List/table background |
| `PF_App_Color_TEXT_ON_LIGHTER_BG` | Text color for light backgrounds |
| `PF_App_Color_DARK_CAPTION_FILL` | Dark caption area fill |
| `PF_App_Color_DARK_CAPTION_TEXT` | Dark caption text color |

These adapt automatically to AE's brightness/theme settings.

### Invalidation

The `InvalidateAll` method triggers a repaint of the entire panel:

```cpp
void PanelatorUI_Plat::InvalidateAll()
{
    RECT clientArea;
    GetClientRect(i_refH, &clientArea);
    InvalidateRect(i_refH, &clientArea, FALSE);
}
```

---

## Error Handling: PT_Err.h

The Panelator example defines its own error handling macros in `PT_Err.h`:

```cpp
#define PT_XTE_START  { A_Err _err = A_Err_NONE; try {
#define PT_XTE_CATCH  } catch (long _tmp_err) { _err = _tmp_err; } \
                        catch(std::bad_alloc&) { _err = A_Err_ALLOC; } \
                        if (_err) {
#define PT_ENDTRY     } if (_err) throw ((long) _err); }

#define PT_ETX(EXPR)  { A_Err _res = (EXPR); if(_res != A_Err_NONE) throw _res; }

#define PT_XTE_CATCH_RETURN_ERR  /* catches and returns error instead of throwing */
```

| Macro | Purpose |
|---|---|
| `PT_ETX(expr)` | Evaluate expression; throw on error |
| `PT_XTE_START / PT_XTE_CATCH` | Try/catch block |
| `PT_XTE_CATCH_RETURN_ERR` | Catch and return error (for static callbacks) |

The static callback wrappers use `PT_XTE_CATCH_RETURN_ERR` to convert C++ exceptions into return values at the AE boundary:

```cpp
static A_Err S_DoFlyoutCommand(AEGP_PanelRefcon refcon,
                                AEGP_FlyoutMenuCmdID commandID)
{
    PT_XTE_START {
        reinterpret_cast<PanelatorUI*>(refcon)->DoFlyoutCommand(commandID);
    } PT_XTE_CATCH_RETURN_ERR;
}
```

---

## Common Pitfalls

### 1. Not chaining the previous window procedure

On Windows, the container HWND already has a window procedure installed by AE. If you replace it without chaining to the original, AE's internal handling breaks. Always store and call the previous procedure:

```cpp
if (i_prevWindowProc && !handledB) {
    return CallWindowProc(i_prevWindowProc, hWnd, message, wParam, lParam);
}
```

### 2. Forgetting that multiple instances can exist

A panel can be instantiated multiple times if the user has multiple workspaces. Each `CreatePanelHook` call must create an independent instance. Never use global state for panel UI.

### 3. Thread safety

Panel callbacks are called on the main UI thread. Do not perform long-running operations in `WM_PAINT` or flyout command handlers. If heavy work is needed, dispatch it to a background thread and invalidate the panel when complete.

### 4. Match name stability

The match name passed to `AEGP_RegisterCreatePanelHook` must never change across plugin versions. AE uses it to restore workspace layouts. Changing it causes the panel to disappear from saved workspaces.

### 5. Missing DllExport on entry point

The entry point must be exported:

```cpp
extern "C" DllExport AEGP_PluginInitFuncPrototype EntryPointFunc;
```

Without `DllExport`, the function is not visible to AE's plugin loader.

---

## Source Files

| File | Purpose |
|---|---|
| `Panelator.cpp` | AEGP entry point, Panelator class (registration, hooks) |
| `Panelator.h` | Includes, string IDs, entry point declaration |
| `PanelatorUI.cpp` | Cross-platform base class (flyout menu, snap sizes) |
| `PanelatorUI.h` | Base class declaration with virtual `InvalidateAll()` |
| `Win/PanelatorUI_Plat.cpp` | Windows implementation (HWND subclassing, WM_PAINT) |
| `Win/PanelatorUI_Plat.h` | Windows platform class declaration |
| `Mac/PanelatorUI_Plat.cpp` | macOS implementation (stub in SDK example) |
| `Mac/PanelatorUI_Plat.h` | macOS platform class declaration |
| `PT_Err.h` | Error handling macros |
| `Panelator_Strings.cpp` | String table |
| `Panelator_PiPL.r` | PiPL resource |

---

## See Also

- `AE_GeneralPlugPanels.h` for the complete `AEGP_PanelSuite1` definition
- `PFAppSuite` for reading AE theme colors
- `CommandSuite` for menu command registration
- `RegisterSuite` for hook registration
