# AEGP Command Hooking: The Commando Example

## Overview

Command hooking is one of the most fundamental AEGP capabilities: it allows your plugin to **add custom menu commands** to After Effects and **intercept any existing command** before or after AE processes it. Every AEGP that adds a menu item uses this mechanism.

The **Commando** SDK example (`Examples/AEGP/Commando/`) is Adobe's minimal "husk" for command-hooking plugins. It demonstrates the complete lifecycle: registering a command, inserting it into a menu, enabling/disabling it based on context, and responding when the user invokes it.

---

## Core Concepts

### What Is a Command?

In AE's architecture, every menu item (File > Save, Edit > Undo, Layer > Pre-compose, etc.) is backed by an `AEGP_Command` -- an opaque integer token. Your plugin can:

1. **Create new commands** with unique tokens and insert them into AE menus.
2. **Listen for any command** (including AE's built-in ones) and react before or after AE handles it.

### The Command Lifecycle

```
Plugin loads (EntryPointFunc)
    |
    v
AEGP_GetUniqueCommand()     -- reserve a command token
    |
    v
AEGP_InsertMenuCommand()    -- place it in a menu
    |
    v
AEGP_RegisterCommandHook()  -- register callback for command execution
AEGP_RegisterUpdateMenuHook() -- register callback for menu state updates
    |
    v
[User opens menu]
    |
    v
UpdateMenuHook fires         -- enable/disable your command based on context
    |
    v
[User clicks your menu item]
    |
    v
CommandHook fires            -- execute your plugin logic
```

---

## AEGP_CommandSuite1

The `AEGP_CommandSuite1` (frozen since AE 5.0) provides all command management functions:

| Function | Purpose |
|----------|---------|
| `AEGP_GetUniqueCommand` | Allocates a unique `AEGP_Command` token for your plugin |
| `AEGP_InsertMenuCommand` | Inserts your command into an AE menu at a specified position |
| `AEGP_SetMenuCommandName` | Changes the display text of a menu item dynamically |
| `AEGP_EnableCommand` | Enables (un-greys) your menu item |
| `AEGP_DisableCommand` | Disables (greys out) your menu item |
| `AEGP_CheckMarkMenuCommand` | Adds/removes a checkmark next to the menu item |

### Menu Locations

When inserting a command, you specify which menu it belongs to:

| Constant | Menu |
|----------|------|
| `AEGP_Menu_FILE` | File menu |
| `AEGP_Menu_EDIT` | Edit menu |
| `AEGP_Menu_COMPOSITION` | Composition menu |
| `AEGP_Menu_LAYER` | Layer menu |
| `AEGP_Menu_EFFECT` | Effect menu |
| `AEGP_Menu_WINDOW` | Window menu |
| `AEGP_Menu_ANIMATION` | Animation menu |
| `AEGP_Menu_IMPORT` | File > Import submenu |
| `AEGP_Menu_EXPORT` | File > Export submenu |
| `AEGP_Menu_PREFS` | Edit > Preferences submenu |
| `AEGP_Menu_PURGE` | Edit > Purge submenu |
| `AEGP_Menu_SAVE_FRAME_AS` | Composition > Save Frame As submenu |
| `AEGP_Menu_NEW` | (AE 12.0+) Layer > New submenu |

### Insertion Position

| Constant | Behavior |
|----------|----------|
| `AEGP_MENU_INSERT_SORTED` (`-2`) | Insert alphabetically among existing items |
| `AEGP_MENU_INSERT_AT_BOTTOM` (`-1`) | Insert at the end of the menu |
| `AEGP_MENU_INSERT_AT_TOP` (`0`) | Insert at the top of the menu |

---

## How the Commando Example Works

### Static Globals

Commando stores three static globals that persist for the plugin's lifetime. As the source comments note, AEGPs never get unloaded, so statics are safe here:

```cpp
static AEGP_Command   S_Commando_cmd  = 0L;   // Our unique command token
static AEGP_PluginID  S_my_id         = 0L;   // Our plugin ID from AE
static SPBasicSuite   *sP             = NULL;  // PICA basic suite pointer
```

### Entry Point: Registration

The `EntryPointFunc` performs all one-time setup:

```cpp
// 1. Get a unique command token
ERR(suites.CommandSuite1()->AEGP_GetUniqueCommand(&S_Commando_cmd));

// 2. Insert into the Window menu, sorted alphabetically
ERR(suites.CommandSuite1()->AEGP_InsertMenuCommand(
    S_Commando_cmd,
    "Commando!",
    AEGP_Menu_WINDOW,
    AEGP_MENU_INSERT_SORTED));

// 3. Register to receive ALL commands (not just ours)
ERR(suites.RegisterSuite5()->AEGP_RegisterCommandHook(
    S_my_id,
    AEGP_HP_BeforeAE,       // Fire before AE processes the command
    AEGP_Command_ALL,       // Listen for every command, not just ours
    CommandHook,
    0));                    // Refcon (user data pointer)

// 4. Register auxiliary hooks
ERR(suites.RegisterSuite5()->AEGP_RegisterDeathHook(S_my_id, DeathHook, NULL));
ERR(suites.RegisterSuite5()->AEGP_RegisterUpdateMenuHook(S_my_id, UpdateMenuHook, NULL));
ERR(suites.RegisterSuite5()->AEGP_RegisterIdleHook(S_my_id, IdleHook, NULL));
```

### UpdateMenuHook: Context-Sensitive Enabling

This callback fires every time the user opens a menu or AE needs to update menu state. Commando enables its command only when the user has a comp or footage item selected:

```cpp
static A_Err UpdateMenuHook(
    AEGP_GlobalRefcon       plugin_refconPV,
    AEGP_UpdateMenuRefcon   refconPV,
    AEGP_WindowType         active_window)
{
    A_Err err = A_Err_NONE, err2 = A_Err_NONE;
    AEGP_ItemH    active_itemH = NULL;
    AEGP_ItemType item_type    = AEGP_ItemType_NONE;

    AEGP_SuiteHandler suites(sP);

    ERR(suites.ItemSuite6()->AEGP_GetActiveItem(&active_itemH));

    if (active_itemH) {
        ERR(suites.ItemSuite6()->AEGP_GetItemType(active_itemH, &item_type));
        if (AEGP_ItemType_COMP == item_type || AEGP_ItemType_FOOTAGE == item_type) {
            ERR(suites.CommandSuite1()->AEGP_EnableCommand(S_Commando_cmd));
        }
    } else {
        ERR2(suites.CommandSuite1()->AEGP_DisableCommand(S_Commando_cmd));
    }
    return err;
}
```

### CommandHook: Execution

The command hook receives **all** commands (because `AEGP_Command_ALL` was registered). Your plugin must check whether the incoming command matches your token:

```cpp
static A_Err CommandHook(
    AEGP_GlobalRefcon   plugin_refconPV,
    AEGP_CommandRefcon  refconPV,
    AEGP_Command        command,           // Which command was triggered
    AEGP_HookPriority   hook_priority,
    A_Boolean           already_handledB,  // Another plugin already handled it?
    A_Boolean           *handledPB)        // Tell AE whether you handled it
{
    A_Err err = A_Err_NONE;

    if (S_Commando_cmd == command) {
        // Your plugin logic here
        *handledPB = TRUE;   // Tell AE we consumed this command
    }
    // If not our command, leave *handledPB as FALSE (its default)

    return err;
}
```

### Hook Priority

| Priority | Meaning |
|----------|---------|
| `AEGP_HP_BeforeAE` (`0x1`) | Your hook fires **before** AE processes the command |
| `AEGP_HP_AfterAE` (`0x2`) | Your hook fires **after** AE processes the command |

Using `AEGP_HP_BeforeAE` lets you intercept and suppress AE's built-in behavior by setting `*handledPB = TRUE`. Using `AEGP_HP_AfterAE` lets you react to what AE already did.

### Additional Hooks Registered by Commando

| Hook | Purpose |
|------|---------|
| `AEGP_RegisterDeathHook` | Called when AE is shutting down -- clean up allocations |
| `AEGP_RegisterIdleHook` | Called during idle time -- for background processing |
| `AEGP_RegisterUpdateMenuHook` | Called when menus need updating -- enable/disable items |

---

## Intercepting Built-In AE Commands

A particularly powerful (and potentially dangerous) capability: because you register for `AEGP_Command_ALL`, you receive notifications for **every** command AE executes, not just your own. This means you can:

- Detect when the user saves, opens, or creates a project
- React to render queue changes
- Monitor undo/redo operations
- Override default behavior of any menu command

The Commando source includes this warning comment:

> "It's legal, but underhanded, to do something in response to AE's commands but not tell AE about it. Rest assured, users will let your support department know about it."

To discover the command IDs for AE's built-in menu items, you can log the `command` parameter in your CommandHook and trigger menu items manually.

---

## PiPL Resource

The Commando PiPL resource declares the plugin as an AEGP:

```
resource 'PiPL' (16000) {
    {
        Kind { AEGP },
        Name { "Commando" },
        Category { "General Plugin" },
        Version { 196608 },
        CodeWin64X86 {"EntryPointFunc"},   // Windows
        CodeMacIntel64 {"EntryPointFunc"}, // macOS Intel
        CodeMacARM64 {"EntryPointFunc"},   // macOS Apple Silicon
    }
};
```

Key points:
- `Kind { AEGP }` marks this as an AEGP (not an effect or AEIO).
- The entry point function name must match exactly between PiPL and code.
- No `AE_Effect_*` flags are needed for AEGPs.

---

## Use Cases

### Custom Menu Items
The most common use: add your own tools, panels, or actions to AE menus.

### Keyboard Shortcuts
After inserting a menu command, users can assign keyboard shortcuts to it through Edit > Keyboard Shortcuts. Your command appears in the shortcut editor automatically.

### Workflow Automation
Combine command hooking with other AEGP suites to build one-click workflows: select layers, apply effects, adjust properties, add to render queue -- all from a single menu command.

### Command Monitoring
Use `AEGP_HP_AfterAE` to log or react to user actions without modifying behavior. Useful for analytics, auto-save triggers, or synchronization with external tools.

### Context-Sensitive Tools
The `UpdateMenuHook` pattern lets you build tools that are only available in specific contexts (certain layer types selected, certain panel active, etc.), keeping your UI clean.

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Forgetting to set `*handledPB = TRUE` | AE will also process the command, causing double-execution or unexpected behavior |
| Setting `*handledPB = TRUE` for commands that are not yours | You will silently suppress other plugins' or AE's commands |
| Caching `AEGP_Command` values for AE's built-in commands | These can change between AE versions; always detect dynamically |
| Expensive work in `UpdateMenuHook` | This fires very frequently (every menu hover); keep it fast |
| Not checking `already_handledB` | Another plugin may have already handled the command; respect this flag |
| Using `AEGP_Menu_NONE` | Results in an invisible command with no menu presence |

---

## Error Handling Pattern

The SDK examples use a dual-error pattern with `ERR()` and `ERR2()`:

```cpp
A_Err err  = A_Err_NONE;  // Primary error -- stops execution on failure
A_Err err2 = A_Err_NONE;  // Secondary error -- for cleanup that must run

ERR(some_important_call());     // If this fails, subsequent ERR() calls skip
ERR2(cleanup_call());           // ERR2 always executes, even if err is set
```

The entry point also checks errors with `if (err)` (not `if (!err)`) to report problems -- this is called out explicitly in the source comments.

---

## Complete Registration Pattern

Here is the minimal boilerplate for any command-hooking AEGP:

```cpp
static AEGP_Command   S_my_cmd = 0;
static AEGP_PluginID  S_my_id  = 0;
static SPBasicSuite   *sP      = NULL;

static A_Err CommandHook(AEGP_GlobalRefcon, AEGP_CommandRefcon,
                         AEGP_Command command, AEGP_HookPriority,
                         A_Boolean already_handledB, A_Boolean *handledPB)
{
    if (command == S_my_cmd && !already_handledB) {
        // Do work here
        *handledPB = TRUE;
    }
    return A_Err_NONE;
}

static A_Err UpdateMenuHook(AEGP_GlobalRefcon, AEGP_UpdateMenuRefcon,
                            AEGP_WindowType active_window)
{
    AEGP_SuiteHandler suites(sP);
    // Determine whether to enable or disable based on context
    suites.CommandSuite1()->AEGP_EnableCommand(S_my_cmd);
    return A_Err_NONE;
}

A_Err EntryPointFunc(struct SPBasicSuite *pica_basicP,
                     A_long major_versionL, A_long minor_versionL,
                     AEGP_PluginID aegp_plugin_id,
                     AEGP_GlobalRefcon *global_refconP)
{
    A_Err err = A_Err_NONE;
    sP     = pica_basicP;
    S_my_id = aegp_plugin_id;

    AEGP_SuiteHandler suites(sP);

    ERR(suites.CommandSuite1()->AEGP_GetUniqueCommand(&S_my_cmd));
    ERR(suites.CommandSuite1()->AEGP_InsertMenuCommand(
        S_my_cmd, "My Plugin", AEGP_Menu_WINDOW, AEGP_MENU_INSERT_SORTED));
    ERR(suites.RegisterSuite5()->AEGP_RegisterCommandHook(
        S_my_id, AEGP_HP_BeforeAE, AEGP_Command_ALL, CommandHook, NULL));
    ERR(suites.RegisterSuite5()->AEGP_RegisterUpdateMenuHook(
        S_my_id, UpdateMenuHook, NULL));

    return err;
}
```

---

## Recipe: Intercept the "Render" button to run pre-render work (auto-bake)

A common need is to run an expensive precompute (bake a cache, build a BVH,
freeze a simulation) **right before** AE renders the Render Queue, then let AE
render normally and consume the freshly baked result. There is no dedicated
"about to render" callback, but a command hook on AE's built-in **Render**
command does the job. (Verified in a shipping plugin's `auto-bake-on-render`.)

### The built-in Render command IDs are not in the SDK headers

AE's menu/button commands are opaque `AEGP_Command` integers, and the built-in
ones are **not** declared in any SDK header -- you discover them empirically by
logging the command in your hook. Observed values (AE 2024/2025-era; treat as
**version-specific** and re-confirm by logging):

| ID | Command |
|----|---------|
| `2303` | Render Queue **Render** button (when the queue is STOPPED) |
| `2302` | **Stop** (when the queue is RENDERING) |
| `2161` | **Add to Render Queue** |

Crucial subtlety: `2302`/`2303` behave like a **single toggle** -- the Render
button and the Stop button flip between the two IDs across sessions/versions.
You disambiguate "a render is starting" from "the user pressed Stop" using the
**Render Queue state**, not the ID alone.

### Register for ALL commands, BeforeAE

```cpp
suites.RegisterSuite5()->AEGP_RegisterCommandHook(
    g_aegpId,
    AEGP_HP_BeforeAE,        // fire BEFORE AE processes the command -> our window to bake
    AEGP_Command_ALL,        // listen to every command, not one custom token
    PlumebusCommandHook,
    reinterpret_cast<AEGP_CommandRefcon>(0));
```

`AEGP_HP_BeforeAE` + `AEGP_Command_ALL` means the hook runs on the **main
thread** before AE dispatches the command, so it is safe to query project /
render-queue state (exactly like any main-thread AEGP call).

### Detect render-start, bake synchronously, do NOT consume

```cpp
static A_Err PlumebusCommandHook(AEGP_GlobalRefcon, AEGP_CommandRefcon,
                                 AEGP_Command command, AEGP_HookPriority,
                                 A_Boolean already_handledB, A_Boolean* handledPB)
{
    AEGP_SuiteHandler suites(g_basicSuite);

    A_long numProj = 0;
    suites.ProjSuite6()->AEGP_GetNumProjects(&numProj);
    if (numProj <= 0) return A_Err_NONE;            // guard: needs an open project

    AEGP_RenderQueueState rqs = AEGP_RenderQueueState_STOPPED;
    suites.RenderQueueSuite1()->AEGP_GetRenderQueueState(&rqs);

    A_long nItems = 0;
    suites.RQItemSuite2()->AEGP_GetNumRQItems(&nItems);

    // A render is STARTING iff a render command fires while the queue is STOPPED
    // with an item queued. STOPPED excludes the Stop press (which fires while
    // RENDERING), so the 2302/2303 toggle ambiguity is resolved by state.
    const bool isRenderPress =
        (command == 2302 || command == 2303) &&
        rqs == AEGP_RenderQueueState_STOPPED && nItems > 0;

    if (isRenderPress && !g_baking.exchange(true)) {
        runBake();                 // synchronous precompute -> disk/cache
        g_baking.store(false);
    }

    // INTENTIONALLY never set *handledPB. We do NOT consume the command, so AE
    // proceeds to render normally and reads the fresh bake. (Consuming it, or
    // calling AEGP_SetRenderQueueState, freezes the UI / hangs the render.)
    (void)handledPB;
    return A_Err_NONE;
}
```

### Why this shape

- **Don't consume the command.** Leaving `*handledPB` untouched lets AE run its
  normal render right after your hook returns -- you are *augmenting* the
  render, not replacing it. Consuming it (or poking `SetRenderQueueState`) leads
  to UI freezes / no-op renders.
- **Re-entrancy guard.** A render command can fire more than once; an atomic
  `g_baking` flag stops a second bake from stacking.
- **Run the bake on the main thread with a real `in_data`.** A command hook has
  no effect `in_data`. If your bake must execute the effect's own code, drive it
  with `AEGP_EffectCallGeneric` -> `EffectMain(PF_Cmd_COMPLETELY_GENERAL)` so it
  runs on the main thread against a real instance, rather than trying to bake
  from the hook directly.
- **`AEGP_GetRQItemByIndex(0)` is accessible here** (BeforeAE, STOPPED) but
  returns null once the render is actually dispatched -- another reason to act
  in the BeforeAE window.

### How to find the IDs on your AE version

Register the ALL/BeforeAE hook above, log `command` (plus the RQ state) for
every invocation, press the Render button, and read the log. The ID that fires
while `rqState == STOPPED` with a queued item is your "Render". Hard-code it but
keep the discovery probe behind an env flag so a future AE that renumbers the
command is a one-line re-confirm.

*Tags: `command-hook`, `render-queue`, `AEGP_RegisterCommandHook`, `auto-bake`, `pre-render`, `AEGP_HP_BeforeAE`, `RenderQueueState`*

---

## Related Suites

| Suite | Relationship |
|-------|-------------|
| `AEGP_RegisterSuite5` | Registers all hook callbacks (command, menu update, idle, death) |
| `AEGP_UtilitySuite3+` | `AEGP_ReportInfo` for displaying error messages to the user |
| `AEGP_ItemSuite6+` | Query active items for context-sensitive menu enabling |
| `AEGP_LayerSuite5+` | Query active layers for layer-specific commands |
