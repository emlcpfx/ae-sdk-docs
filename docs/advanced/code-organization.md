# Code Organization

> 1 Q&A · source: AE plugin dev community Discord

### Can PreRender and SmartRender functions be placed in a separate file from the main plugin code?

Yes, you can place functions in whatever file you want. Just make sure the file is referenced in your project and the implementation is done only once (the compiler will warn you about duplicate definitions). Declare functions in a separate header file and include it in your main header. You don't have to worry about other plugins calling your functions -- each plugin is its own DLL with its own symbol scope.

*Tags: `code-organization`, `cpp`, `dll`, `prerender`, `smartfx`, `smartrender`*

---
