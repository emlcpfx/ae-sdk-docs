# Q&A: workaround

**8 entries** tagged with `workaround`.

---

## How do you handle Custom UI mouse-leave events to un-hover buttons?

AE has a known bug where no PF_Event is sent when the mouse leaves the Custom UI area, causing hover states to persist. A workaround is to use a debounce pattern: create a timer callback that triggers after a delay (e.g., 2 seconds). On every mouse-over event, reset the timer. When the timer fires (meaning no mouse-over received recently), trigger an UpdateUI on the main thread to force a Custom UI re-render that resets the hover state.

```cpp
template <typename Func, typename... Args>
std::function<void(Args...)> debounce(Func func, std::chrono::milliseconds delay) {
    std::shared_ptr<std::chrono::steady_clock::time_point> lastCallTime =
        std::make_shared<std::chrono::steady_clock::time_point>();
    return [=](Args... args) {
        *lastCallTime = std::chrono::steady_clock::now();
        std::thread([=]() {
            std::this_thread::sleep_for(delay);
            if (std::chrono::steady_clock::now() - *lastCallTime >= delay) {
                func(args...);
            }
        }).detach();
    };
}
```

*Contributors: [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/) · Source: adobe-plugin-devs · 2025-07-25 · Tags: `custom-ui`, `mouse-events`, `hover`, `debounce`, `workaround`*

---

## How can I trigger a plugin refresh in Premiere when CEP updates hidden parameters, since Premiere doesn't treat script interactions as user actions like AE does?

There is no known workaround for this. Premiere does not consider script interactions as user actions, so PF_UserChangedParam won't be triggered from CEP. A practical workaround is to use external dependencies watching for file changes (though it's not very reactive) or add a button parameter to force a manual refresh.

*Contributors: [**gabgren**](../contributors/gabgren/) · Source: adobe-plugin-devs · 2022-05-31 · Tags: `premiere`, `cep`, `script-interaction`, `parameter-update`, `workaround`*

---

## Why does Premiere give a NULL input_world in the Render call when compiled with fast optimizations (-Ofast), but works without optimizations?

This is a known issue that occurs in Premiere but not AE when using -Ofast compiler optimizations. A workaround is to add __attribute__((optnone)) before the render function to disable optimization for that specific function while keeping the rest of the plugin optimized. Another suggestion is to check if the input world is null and return PF_Err_OUT_OF_MEMORY (not bad param), which may cause Premiere to retry the render. Also verify your colorspace setup in GlobalSetup and consider trying BGRA instead of ARGB.

```cpp
__attribute__ ((optnone))
static PF_Err Render(...) {
    // render code here
}
```

*Contributors: [**tlafo**](../contributors/tlafo/), [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/) · Source: adobe-plugin-devs · 2023-09-07 · Tags: `premiere`, `null-input-world`, `optimization`, `ofast`, `compiler`, `render`, `workaround`*

---

## How can I store custom AEGP data persistently in a comp or project?

There's no facility designed for persistent AEGP data storage, but several workarounds exist: (1) Use layer and comp comments - rarely displayed in UI and almost never used by users. (2) Add a locked null layer named descriptively (e.g., 'my stuff, do not delete') and use its comments field or marker comments. (3) Create a folder in the AE project with sub-folders whose names encode comp IDs and data. This creates minimal project clutter.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2023-03-01 · Tags: `aegp`, `persistent-data`, `project-storage`, `comments`, `workaround`*

---

## Can you disable the crosshair/visual representation of point parameters in the AE comp window?

No, the crosshairs are the visual representation of point parameters and cannot be disabled. As a workaround, you could use a different parameter type like an int or float param pair for x and y values instead of a point parameter.

*Contributors: [**Tobias Fleischer (reduxFX)**](../contributors/tobias-fleischer-reduxfx/) · Source: aescripts discord · 2025-12-29 · Tags: `point-parameters`, `crosshair`, `comp-window`, `ui`, `workaround`*

---

## What is a method to pass text data from a render into an expression in After Effects?

One semi-reliable method is to encode and render dead pixels in your render output, then use a sampleImage expression to retrieve those pixels, which can then set the source text. This technique works with both regular layers and Null objects, though it has a few caveats.

*Tags: `scripting`, `expressions`, `render-loop`, `workaround`*

---

## How can I set keyframe velocity and influence values using the AEGP API?

The AEGP_SetKeyframeTemporalEase function is broken and does not properly set keyframe velocity and influence values—these settings simply do not stick. You can set keyframe interpolation types (linear, hold, etc.) but not velocity. A workaround is to use AEGP_ExecuteScript() to execute JavaScript that sets the interpolations instead, though this approach is slower when dealing with many properties and layers.

*Tags: `aegp`, `keyframe`, `velocity`, `api`, `workaround`*

---

## How can I retrieve text justification from a text layer using the AEGP C++ API?

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

*Tags: `aegp`, `scripting`, `text`, `c++`, `workaround`*

---
