# Interactive

> 2 Q&As · source: AE plugin dev community Discord

### How can you implement live drawing and UI interactions like rotoscope in After Effects plugins?

Live drawing and UI interactions such as rotoscope functionality in After Effects plugins are implemented through the PF_CMD_EVENT command, which allows plugins to handle real-time user input and interactive drawing on the canvas.

*Tags: `aegp`, `drawing`, `events`, `interactive`, `ui`*

---

### What is the best way to create interactive UI elements like draggable masks in After Effects plugins?

Use a custom UI overlay rather than rendering the mask as part of the image. Rendering the mask in the image has several drawbacks: it gets affected by subsequent effects and display channels, is confined to the layer's size, and forces re-renders when toggling visibility. A custom UI is necessary for interactivity since After Effects won't report click events otherwise. The CCU (Custom Composition UI) sample in the After Effects SDK provides an excellent starting point for creating interactive interfaces in the composition window.

*Tags: `custom-ui`, `interactive`, `reference`, `sdk`, `ui`*

---
