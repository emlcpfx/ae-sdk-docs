# Pixel Iteration

> 2 Q&As · source: AE plugin dev community Discord

### How can I use multithreading for pixel iteration in Premiere, since the AE Iterate8Suite doesn't work correctly?

The AE Iterate8Suite may not work correctly in Premiere and can cause rendering issues (e.g., sequence thumbnail updates but program window doesn't). For 8-bit BGRA, Premiere has its own dedicated iteration suite. For 32-bit float, you need to write your own iteration function. Use std::parallel or equivalent for multithreading. The SDK Noise example demonstrates BGRA and VUYA colorspace handling. The iterate suite should work in 8-bit ARGB (Suitev2 since 2022), but for BGRA you need the special Premiere suite.

*Tags: `bgra`, `colorspace`, `iterate-suite`, `multithreading`, `pixel-iteration`, `premiere`*

---

### How can I implement a custom pixel iteration function similar to PF_Iterate8Suite1::iterate?

To implement a custom iteration function, start by examining the 'shifter' sample which implements one of the iteration functions. For direct pixel data access without the iteration suite, consult the 'CCU' sample, particularly its render function. You can also use the iterate_generic() function to obtain threading services without additional functionality. For optimal performance, thread rows rather than individual pixels to reduce CPU overhead per pixel.

*Tags: `mfr`, `pixel-iteration`, `reference`, `render-loop`, `threading`*

---
