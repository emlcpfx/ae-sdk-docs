# Process Injection

> 1 Q&A · source: AE plugin dev community Discord

### How do you use RenderDoc to debug GPU plugins in After Effects?

Use RenderDoc's in-application API with process injection. Steps: (1) Enable 'Enable process injection' in RenderDoc's Tools > Settings. (2) Compile your plugin using RenderDoc's API - in GlobalSetup, check for renderdoc.dll with GetModuleHandleA and get the API pointer. (3) Launch AE first, then inject RenderDoc via File > Inject into Process. (4) RenderDoc must be injected before applying your plugin. (5) Use rdoc_api in your render function to manually begin/end frame capture.

```cpp
// In PF_Cmd_GLOBAL_SETUP:
RENDERDOC_API_1_1_2 *rdoc_api = NULL;
if(HMODULE mod = GetModuleHandleA("renderdoc.dll")) {
    pRENDERDOC_GetAPI RENDERDOC_GetAPI =
        (pRENDERDOC_GetAPI)GetProcAddress(mod, "RENDERDOC_GetAPI");
    int ret = RENDERDOC_GetAPI(eRENDERDOC_API_Version_1_1_2, (void **)&rdoc_api);
    assert(ret == 1);
}
// Then use rdoc_api->StartFrameCapture() / EndFrameCapture() in render
```

*Tags: `debugging`, `gpu`, `opengl`, `process-injection`, `renderdoc`*

---
