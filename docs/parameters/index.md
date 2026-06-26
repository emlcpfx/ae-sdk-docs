# Parameters

Parameters are how users interact with your plugin — sliders, checkboxes, color pickers, dropdowns.

## Common Parameter Types

| Type | Use For | Notes |
|------|---------|-------|
| `PF_Param_FLOAT_SLIDER` | Continuous values | Most common |
| `PF_Param_CHECKBOX` | On/off toggles | |
| `PF_Param_POPUP` | Dropdown menus | 1-indexed |
| `PF_Param_COLOR` | Color pickers | |
| `PF_Param_POINT` | 2D positions | |
| `PF_Param_ARBITRARY_DATA` | Custom data blobs | For complex state |
| `PF_Param_BUTTON` | Clickable buttons | Triggers `PF_Cmd_USER_CHANGED_PARAM` |

## Important Concepts

- **Parameters are added in `PARAMS_SETUP`** and cannot be added/removed after
- **Dynamic visibility** — hide/show params at runtime with `PF_UpdateParamUI`
- **Arbitrary data** — store complex state (structs, arrays) that survives save/load via flatten/unflatten
- **Param checkout** — read parameter values at specific times with `PF_CHECKOUT_PARAM`
- **`USER_CHANGED_PARAM`** — called when the user changes a parameter interactively

---

*51 documents in this section. Use **Search** (top of page) to find what you need.*
