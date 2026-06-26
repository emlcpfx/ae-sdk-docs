# Rust

> 10 Q&As · source: AE plugin dev community Discord

### How do you debug AE plugins built with Rust on macOS?

Use VSCode with the CodeLLDB extension. Create a launch.json that launches AE directly and specifies 'sourceLanguages': ['rust']. The sourceLanguages line is important - without it, launching AE from LLDB can cause crashes. Use a preLaunchTask to build the plugin before launching.

```cpp
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "(macOS) lldb: After Effects",
      "program": "/Applications/Adobe After Effects 2024/Adobe After Effects 2024.app",
      "preLaunchTask": "just: build",
      "sourceLanguages": ["rust"]
    }
  ]
}
```

*Tags: `debugging`, `lldb`, `macos`, `rust`, `vscode`*

---

### Are there any good C++ wrapper libraries for the After Effects Filter GUI API?

No well-known general-purpose wrapper exists. Most wrappers that have been built end up being very complex themselves if you try to handle all use cases -- you end up with a non-standard system that is almost as complex as the original, just with more C++ features. Copy-pasting C chunks from the SDK examples works well in practice. That said, even a C wrapper with automatic memory management and state stored in structs would be an improvement over the current state machine. There is also a Rust binding project (https://github.com/virtualritz/after-effects/) that significantly reduces boilerplate compared to C/C++.

*Tags: `boilerplate`, `cpp`, `gui-api`, `rust`, `sdk`, `wrapper`*

---

### Is there a Rust alternative for AE/Premiere plugin development?

Yes, there is a Rust bindings project at https://github.com/virtualritz/after-effects/ with documentation at https://docs.rs/after-effects/ and https://docs.rs/premiere/. It was originally created by Moritz Moeller and significantly refactored by Adrian Eddy (author of Gyroflow). The Gyroflow AE/Premiere plugin uses these bindings. The examples show significantly less boilerplate compared to C/C++ versions, are mostly 100% safe Rust, and include a PiPL helper crate. The crates also build on Linux for development purposes (though you need Windows/macOS to run the plugin).

*Tags: `alternative-language`, `bindings`, `gyroflow`, `pipl`, `rust`*

---

### How do you get a stacktrace with symbols when using the Rust bindings for After Effects on macOS?

Use VSCode with the CodeLLDB extension and configure a launch.json file to attach the debugger to the After Effects application. Set up a launch configuration that points to the After Effects executable and runs a build task before launching.

```cpp
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "(macOS) lldb: After Effects",
      "program": "/Applications/Adobe After Effects 2024/Adobe After Effects 2024.app",
      "preLaunchTask": "just: build"
    }
  ]
}
```

*Tags: `debugging`, `lldb`, `macos`, `rust`*

---

### What pattern should be used to convert traditional After Effects C API functions to Rust?

Convert functions that return PF_Err with output parameters to Rust Result types. For example, a C function like `PF_Err GetThing(Thing* output);` becomes `fn get_thing() -> Result<Thing, Error>;` in the Rust crate.

```cpp
// C API
PF_Err GetThing(Thing* output);

// Rust
fn get_thing() -> Result<Thing, Error>;
```

*Tags: `error-handling`, `rust`, `scripting`*

---

### What is an alternative cross-platform build system for After Effects plugins with a custom PiPL compiler?

The virtualritz/after-effects project at https://github.com/virtualritz/after-effects provides its own cross-platform build system with a custom PiPL compiler written in Rust, offering an alternative to traditional build approaches.

*Tags: `build`, `cross-platform`, `open-source`, `pipl`, `reference`, `rust`*

---

### Is there an open-source Rust crate for After Effects plugin development?

The virtualritz/after-effects repository (https://github.com/virtualritz/after-effects) is an open-source Rust crate for AE plugin development that includes f32 support. According to the repository owner, it was developed based on SDK examples, After Effects SDK mailing list discussions, and reverse-engineering from the AtomKraft for AE plugin. The crate was initially published after the author completed a year-long Rust plugin project integrating a 3D renderer, where f32 support proved essential.

*Tags: `build`, `open-source`, `reference`, `rust`*

---

### Is there an open-source Rust binding for After Effects plugin development?

The virtualritz/after-effects repository on GitHub provides Rust bindings for After Effects plugin development. This project enables developers to write AE plugins using Rust instead of C++.

*Tags: `build`, `cross-platform`, `open-source`, `reference`, `rust`*

---

### How do you debug Rust bindings for After Effects on macOS with symbols?

Use VSCode with the CodeLLDB extension. Configure a launch.json with the lldb debugger type pointing to the After Effects application bundle, and set up a preLaunchTask to build the plugin before launching. The configuration should specify the full path to the After Effects app (e.g., /Applications/Adobe After Effects 2024/Adobe After Effects 2024.app).

```cpp
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "lldb",
      "request": "launch",
      "name": "(macOS) lldb: After Effects",
      "program": "/Applications/Adobe After Effects 2024/Adobe After Effects 2024.app",
      "preLaunchTask": "just: build"
    }
  ]
}
```

*Tags: `debugging`, `macos`, `open-source`, `rust`*

---

### Is there an open-source Rust binding library for After Effects plugin development?

The virtualritz/after-effects repository on GitHub provides Rust bindings for After Effects plugin development. It enables developers to write AE plugins in Rust with support for debugging on macOS using standard debuggers.

*Tags: `macos`, `open-source`, `reference`, `rust`*

---
