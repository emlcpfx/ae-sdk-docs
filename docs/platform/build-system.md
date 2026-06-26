# Build System

> 2 Q&As · source: AE plugin dev community Discord

### Are there CMake-based build systems for AE plugins?

Yes, there are several examples on GitHub: Vulkanator (https://github.com/Wunkolo/Vulkanator) uses CMake, and after_effects_cmake (https://github.com/mobile-bungalow/after_effects_cmake) is based on Vulkanator's setup. There's also a Rust-based project (https://github.com/virtualritz/after-effects) with its own PiPL compiler written in Rust. On Mac, you can use the Rez tool for .rsrc files and potentially use Ninja for faster incremental compilation.

*Tags: `build-system`, `cmake`, `cross-platform`, `ninja`, `xcode`*

---

### How do you keep PiPL out_flags in sync with GlobalSetup flags?

Define the flags as macros in your plugin header file (e.g., #define FX_OUT_FLAGS (...)) and use those same macros in both your PiPL .r file and GlobalSetup code. The PiPL should include the header with all bit flags defined so it auto-updates when compiling. However, note that this approach may fail with higher bits/newest flags due to overflow in Adobe's PiPL compiler. An online tool for calculating flag bitmasks is available at https://reduxfx.com/aeoutflags.htm.

```cpp
#define FX_OUT_FLAGS (PF_OutFlag_FORCE_RERENDER + \
                       PF_OutFlag_USE_OUTPUT_EXTENT + \
                       PF_OutFlag_SEND_UPDATE_PARAMS_UI + \
                       PF_OutFlag_DEEP_COLOR_AWARE + \
                       PF_OutFlag_CUSTOM_UI + \
                       PF_OutFlag_I_DO_DIALOG)

// In PiPL:
AE_Effect_Global_OutFlags { FX_OUT_FLAGS },
AE_Effect_Global_OutFlags_2 { FX_OUT_FLAGS2 },
```

*Tags: `build-system`, `macros`, `out-flags`, `pipl`*

---
