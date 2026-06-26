# Outputdebugstring

> 1 Q&A · source: AE plugin dev community Discord

### How do you debug/log variables from a C++ AE plugin on Windows?

Use OutputDebugString to send debug output that can be viewed with tools like DebugView or Visual Studio's Output window.

```cpp
std::string log1 = "variable: " + std::to_string(variable) + "\n"; OutputDebugString(log1.c_str());
```

*Tags: `cpp`, `debugging`, `logging`, `outputdebugstring`, `windows`*

---
