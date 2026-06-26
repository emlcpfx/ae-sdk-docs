# Alternative Language

> 1 Q&A · source: AE plugin dev community Discord

### Is there a Rust alternative for AE/Premiere plugin development?

Yes, there is a Rust bindings project at https://github.com/virtualritz/after-effects/ with documentation at https://docs.rs/after-effects/ and https://docs.rs/premiere/. It was originally created by Moritz Moeller and significantly refactored by Adrian Eddy (author of Gyroflow). The Gyroflow AE/Premiere plugin uses these bindings. The examples show significantly less boilerplate compared to C/C++ versions, are mostly 100% safe Rust, and include a PiPL helper crate. The crates also build on Linux for development purposes (though you need Windows/macOS to run the plugin).

*Tags: `alternative-language`, `bindings`, `gyroflow`, `pipl`, `rust`*

---
