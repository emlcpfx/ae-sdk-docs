# Deprecated

> 4 Q&As · source: AE plugin dev community Discord

### Does the old GLator sample have memory leaks in its GL rendering code?

Yes, the GLator sample has significant memory leaks. The GL rendering is wrapped in a try/catch, and in the download texture function suites.IterateFloatSuite1()->iterate can throw a PF_Interrupt_CANCEL, which means bufferH is never deallocated. This results in an entire output buffer of 32bpc pixels being leaked each time from the CPU, plus all the GPU FBOs.

*Tags: `deprecated`, `gpu`, `memory`, `opengl`, `reference`*

---

### Are there alternatives to PiPL for describing plugin entrypoints in After Effects plugins?

Yes, PiPL is considered somewhat deprecated. There are alternative ways to describe plugin entrypoints, though the conversation does not specify the exact modern replacement method. Developers should investigate newer plugin descriptor formats beyond the traditional PiPL approach.

*Tags: `aegp`, `deprecated`, `pipl`, `reference`*

---

### What historical CPU architecture values were stored in the Gestalt API for After Effects plugins?

The Gestalt API values held gestalt values for CPU and FPU on 68x and PowerPC architectures. While the Gestalt API was deprecated on macOS since 10.8, Apple added values for Intel and Silicon ARM CPUs (values 10 and 20). However, Adobe no longer passes these values through to plugins.

*Tags: `apple-silicon`, `architecture`, `deprecated`, `macos`, `reference`*

---

### Should obsolete CPU architecture fields have been removed from in_data when After Effects went 64-bit only?

Yes, the fields in in_data related to old CPU architecture detection (such as Gestalt values for 68x and PowerPC) should have been removed from the plugin API when Adobe transitioned to 64-bit only support around CS5, rather than keeping legacy fields that are no longer used or meaningful.

*Tags: `aegp`, `backwards-compatibility`, `deprecated`, `macos`, `windows`*

---
