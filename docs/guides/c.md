# C++

> 9 Q&As · source: AE plugin dev community Discord

### Is there a way to communicate between a CEP panel and a C++ plugin in After Effects?

Direct communication between CEP and C++ plugins is not possible. However, indirect communication can be achieved: a C++ plugin can open a CEP panel by using the ExecuteScript to find the Command ID (menu) by the extension name, then executing that command through the command suite. CEP cannot directly change plugin parameter values, and the plugin cannot directly check if a CEP panel is open or bring it into focus.

*Tags: `aegp`, `c++`, `cep`, `scripting`, `ui`*

---

### Can you integrate a C++ backend with UXP for Premiere Pro, or do you need to use a separate service with IPC for computational tasks?

There is a new hybrid UXP approach that combines HTML and C++ functionality, though details on how it works are still emerging. This hybrid approach is currently in beta and promises to allow C++ backend integration rather than requiring a separate IPC service for computational tasks.

*Tags: `c++`, `premiere`, `ui`, `uxp`*

---

### How can I make a plugin support 8-bit, 16-bit, and 32-bit color depths without duplicating code for each bit depth?

Use C++ templates to create a single implementation that can be instantiated for different data types and range constants. This allows you to share the same core logic while supporting multiple bit depths. You can also manually convert higher bit-depth input to 8-bit within the plugin itself if needed, rather than relying on After Effects' automatic conversion which may introduce unwanted color noise.

*Tags: `bit-depth`, `c++`, `codebase-reuse`, `color-processing`, `sdk`, `templates`*

---

### Why does my map of effect matchnames to install keys return incorrect install keys when applying effects?

The issue was that the map was declared as `std::map<A_char, A_long>`, storing only a single character instead of the full matchname string. The map should be declared with a string type (e.g., `std::map<std::string, A_long>` or `std::map<A_char*, A_long>`) to properly store and look up the complete effect matchname. When accessing the map with `InstallKeys::keys[*myMatchName]`, only the first character of the matchname was being used as the key, causing incorrect lookups.

```cpp
// Incorrect:
std::map<A_char, A_long> keys;
InstallKeys::keys[*nextMatchName] = nextKey;

// Correct:
std::map<std::string, A_long> keys;
InstallKeys::keys[std::string(nextMatchName)] = nextKey;
```

*Tags: `aegp`, `c++`, `debugging`, `plugin-development`*

---

### How do you get the duration of a composition layer in the After Effects SDK for C++?

To get layer duration, use AEGP_GetLayerDuration(). To get composition duration, use AEGP_GetItemFromComp() followed by AEGP_GetItemDuration().

*Tags: `aegp`, `c++`, `reference`, `sdk`*

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

### Should arbitrary parameter data be stored in a single struct for all arb params or separate structs per parameter?

You can use either approach, but a single generalized arb handler that is reusable across multiple arb params is more efficient. Store the data in AE rather than in the handler class itself. During global setup, create and store the handlers in the global data structure, then pass pointers to them during param setup. Keep handlers locked throughout the session using placed new allocation. The key is that except for the 'create new' function, most handler functions remain the same across different arb types.

```cpp
// C++ style approach with reusable handler
// Store handler pointer in arb definition
// Handler class with virtual methods for each arb type
class ArbHandler {
public:
  virtual PF_Err CreateDefault(PF_Handle *arbH) = 0;
  virtual PF_Err Handle(PF_Cmd cmd, PF_Handle arbH) = 0;
};

// During global setup:
ArbHandler *handler = new ArbHandler();
ArbHandler **handlerH = (ArbHandler**)AE_Allocate(sizeof(ArbHandler*));
*handlerH = handler;
// Lock handle and store in global data
```

*Tags: `aegp`, `arb-data`, `c++`, `params`, `plugin-architecture`*

---

### How can I convert Pixel Bender code that samples and transforms pixel coordinates to a native After Effects plugin?

You can accomplish coordinate transformation and pixel sampling using the C++ API. The recommended approach is to pull values from input layers rather than push to arbitrary pixels. Iterate through output samples and pull input values from transformed locations. For direct buffer pixel access in RAM, study the CCU sample project. For iterating through output and pulling from other locations, examine the Shifter sample. Note that subpixel interpolation to 4 adjacent pixels is not supported by the standard suite.

*Tags: `c++`, `effect_api`, `pixel_bender`, `reference`, `sampling`*

---

### How can I retrieve text justification from a text layer using the AEGP C++ API?

There is no direct way to access text justification through the C++ AEGP API. However, you can use AEGP_ExecuteScript to execute JavaScript code that accesses the text justification property. The workaround involves calling AEGP_ExecuteScript with a JavaScript function that retrieves the justification value from the text layer, then parsing the returned result. The justification constants are numeric values (6012, 6013, 6014).

```cpp
#define TEXT_JUSTIFICATION_SCRIPT "function getTextJustificationForLayer(layerId) { \
var comp = app.project.activeItem; \
for (var i = 0; i < comp.numLayers; i++) \
if (i == layerId && comp.layer(i+1) instanceof TextLayer) \
return comp.layer(i+1).property(\"ADBE Text Properties\").property(\"ADBE Text Document\").value.justification; \
} \
getTextJustificationForLayer(%d);"

AEGP_MemHandle resultMemH = NULL;
A_char *resultAC = NULL;
A_char scriptAC[AEGP_MAX_MARKER_URL_SIZE] = {'\0'};
sprintf(scriptAC, TEXT_JUSTIFICATION_SCRIPT, layerIndex);
ERR(suites.UtilitySuite5()->AEGP_ExecuteScript(S_my_id, scriptAC, FALSE, &resultMemH, NULL));
ERR(suites.MemorySuite1()->AEGP_LockMemHandle(resultMemH, reinterpret_cast<void**>(&resultAC)));
int just = atoi(resultAC);
ERR(suites.MemorySuite1()->AEGP_FreeMemHandle(resultMemH));
```

*Tags: `aegp`, `c++`, `scripting`, `text`, `workaround`*

---
