# Dll

> 3 Q&As · source: AE plugin dev community Discord

### What is plugplug.DLL and how can you use it from C++ plugins?

plugplug.DLL is the gateway to communicate with CEP (Common Extensibility Platform) from C++. It allows sending events between C++ plugins and CEP panels. You can reverse-engineer the DLL's exports using depends.exe and reference the Illustrator SDK sample as a guide (it's not 1:1 but close enough). A GitHub repo with a working implementation is available: https://github.com/Trentonom0r3/AE-SDK-CEP-UTILS

*Tags: `cep`, `dll`, `inter-process-communication`, `plugplug`*

---

### Should .aex plugins be code signed on Windows?

.aex plugins are regular DLLs (just renamed), so they can be codesigned. Microsoft currently allows unsigned DLLs to be loaded from a signed process, but they have hinted this might change in the future (was supposed to happen with Win11). On macOS, a plugin is a bundle (renamed folder), and you don't need to sign the actual plugin binary inside it as long as the bundle itself is signed.

*Tags: `aex`, `code-signing`, `dll`, `macos`, `security`, `windows`*

---

### Can PreRender and SmartRender functions be placed in a separate file from the main plugin code?

Yes, you can place functions in whatever file you want. Just make sure the file is referenced in your project and the implementation is done only once (the compiler will warn you about duplicate definitions). Declare functions in a separate header file and include it in your main header. You don't have to worry about other plugins calling your functions -- each plugin is its own DLL with its own symbol scope.

*Tags: `code-organization`, `cpp`, `dll`, `prerender`, `smartfx`, `smartrender`*

---
