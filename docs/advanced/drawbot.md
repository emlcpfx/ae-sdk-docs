# Drawbot

> 11 Q&As · source: AE plugin dev community Discord

### Why does DrawBot draw colors at ~90% brightness compared to what you specify?

This can be caused by using DrawImage with opacity less than 1.0, which makes the drawing darker while alpha still appears as 1.0 (since AE's UI is behind with solid alpha). It could also be related to AE's UI brightness preferences or the project's color space settings, as AE's drawing suites may compensate for those things.

*Tags: `brightness`, `color-accuracy`, `custom-ui`, `drawbot`*

---

### How does AE's Custom UI / arb params bug in AE 2025 manifest?

In AE 2025 (25.0.1), custom UI and arb params display glitches: one arb param's UI can corrupt other arb UIs, causing flashes or scrambled/duplicated content. The likely culprit is NewImageFromBuffer in DrawBot - every call may be setting pixels of all retained DRAWBOT_ImageRef within the same context. Adobe acknowledged fixing something in arb handling, but it may have introduced this unintended consequence. The issue affects multiple third-party plugins including Trapcode.

*Tags: `ae-2025`, `arb-data`, `bug`, `custom-ui`, `drawbot`, `regression`*

---

### How does the AddArc function work in the Drawbot Path Suite, and why do two consecutive arcs produce unexpected shapes?

When drawing multiple arcs/circles in the same path, you need to call close() on the path after each circle. Without closing, the entire sequence is treated as one continuous path, and the renderer draws a connecting line from the end point of one arc back to the beginning of the next, creating unexpected shapes. You can have multiple closed shapes in one path, or alternatively draw each shape in separate paths with separate drawing calls.

*Tags: `addarc`, `custom-ui`, `drawbot`, `drawing`, `path-suite`*

---

### What causes custom UI artifacts and glitching in After Effects 2025 when using DrawBot with arbitrary data parameters?

The issue is not directly related to arbitrary data (arb) parameters, but rather to how DrawBot's Image buffer compositing works. The problem occurs when the UI composites multiple images per plugin context. The culprit appears to be NewImageFromBuffer, which on every call sets the pixels of all retained DRAWBOT_ImageRef objects within the same context, causing visual artifacts like flashes, scrambled, or duplicated UI elements. The issue can be avoided by either not compositing any image in the UI, or by only compositing a single image per plugin context.

*Tags: `ae-2025`, `arb-data`, `debugging`, `drawbot`, `ui`*

---

### How does the AddArc function work when drawing multiple circles in After Effects UI?

The AddArc function treats all drawing operations as a continuous path. When drawing multiple circles, you must call close() after each AddArc to properly close that shape before starting a new one. Without closing, the path continues from the end point of one circle back to the beginning of the next, creating unexpected intersections. Alternatively, you can draw each shape in separate paths with separate drawing calls. The center argument for AddArc should use absolute position coordinates.

```cpp
DRAWBOT_PointF32 center1 = { drawRectF.left + point.x , drawRectF.top + drawRectF.height - point.y };
suites.PathSuiteCurrent()->AddArc(plotPath, &center1, 4.0f, 0.0f, 360.0f);
suites.PathSuiteCurrent()->Close(plotPath);
DRAWBOT_PointF32 center2 = { drawRectF.left + point.x + 100 , drawRectF.top + drawRectF.height - point.y };
suites.PathSuiteCurrent()->AddArc(plotPath, &center2, 4.0f, 0.0f, 360.0f);
suites.PathSuiteCurrent()->Close(plotPath);
```

*Tags: `debugging`, `drawbot`, `path`, `ui`*

---

### How can I create a custom text input field in the Effect Control Window instead of using a JavaScript popup?

Custom UI elements in the ECW or comp window must be handled exclusively via DrawBot. You cannot add OS-level text fields directly to the UI. Instead, you need to either: (1) translate user input through AE's event system onto an offscreen text controller and copy its image buffer, or (2) create a text editor from scratch using the event system and DrawBot. Alternatively, use a JavaScript window for input and display the result as non-interactive text in the ECW that launches the JavaScript window when clicked.

*Tags: `aegp`, `drawbot`, `scripting`, `ui`*

---

### Where can I find a sample project demonstrating Drawbot usage?

The "CCU" sample project in the After Effects SDK demonstrates the use of Drawbot for custom UI drawing.

*Tags: `drawbot`, `reference`, `sample`, `ui`*

---

### How can I draw gradients in an After Effects effect UI using DrawBot?

DrawBot does not have a direct API function for drawing gradients. The recommended approach is to draw the gradient into a buffer using any available means, then convert that buffer to a DrawBot image using the NewImageFromBuffer() function. This allows you to create dynamic gradient fills like color wheels for custom effect panels.

*Tags: `aegp`, `drawbot`, `reference`, `ui`*

---

### Can you access the comp overlay buffer during a render call using Drawbot?

No, you cannot access the comp overlay buffer during a render call, regardless of whether you're using Drawbot. Instead, you must cache whatever you want to render during a "draw" event call, and then use that cache during the render call. Drawbot's data structures are opaque and don't offer tools for reading data back out. If you need to draw during render, consider using third-party drawing tools or OS-level drawing tools instead, such as OpenGL.

*Tags: `caching`, `drawbot`, `render-loop`, `ui`*

---

### How can I draw anti-aliased shapes and text directly into an After Effects effect's output buffer?

DrawBot is not suitable for rendering into the output buffer—it only operates on AE's internal interface graphics buffers. Instead, use OS-native drawing tools: on macOS, use Quartz (Core Graphics) APIs like CGBitmapContextCreate(), CGContextShowTextAtPoint(), and CGContextDrawPath(); on Windows, use GDI+ or equivalent APIs. Create an OS graphics context in memory allocated via the Memory Suite, draw your shapes and text to that context, then copy the pixel data from the OS buffer to the effect's output buffer by locking the memory handle and iterating through pixels. See the CCU (Custom Color UI) sample code in the SDK for an example of accessing the output buffer directly in RAM.

```cpp
osBufferBaseAddress = suites.HandleSuite1()->host_lock_handle(osBufferMemHandle);
// Copy pixel data from OS context to output buffer
// Then unlock and free the OS graphics context
```

*Tags: `drawbot`, `macos`, `output-rect`, `reference`, `windows`*

---

### What is the difference between DrawBot and OS drawing libraries for After Effects plugin development?

DrawBot is designed exclusively for drawing custom UI overlays on AE's interface in CS5 and later. It provides built-in drawing tools and operates only on AE's internal interface graphics buffers. OS drawing libraries (Quartz on macOS, GDI+ on Windows) are general-purpose graphics APIs used to render into effect output images. If you need to draw shapes, lines, or text into an effect's output buffer rather than a UI overlay, you must use OS-native drawing tools combined with the Memory Suite to manage the graphics context, not DrawBot.

*Tags: `drawbot`, `macos`, `reference`, `ui`, `windows`*

---
