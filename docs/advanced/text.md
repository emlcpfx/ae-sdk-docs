# Text

> 3 Q&As · source: AE plugin dev community Discord

### How can you set text on a text layer from an effect on a frame-by-frame basis?

You need to communicate with an AEGP plugin that can set the text of a layer for each frame, since invoking scripts via the render loop does not work for this purpose.

*Tags: `aegp`, `render-loop`, `scripting`, `text`*

---

### Is there an API to retrieve the rotation value of individual characters when a text animator is applied?

No, there is no such API available in After Effects. However, you can get the text in its current transformation as shape vertices and attempt to deduce the rotation from the vertex data, though this approach is highly problematic and not reliable.

*Tags: `aegp`, `api`, `scripting`, `text`*

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
