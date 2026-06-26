# Testing

> 2 Q&As · source: AE plugin dev community Discord

### How do you check AE plugin compatibility with older versions of After Effects?

You can get installers for AE 2019 and older from the ProDesignTools website (https://prodesigntools.com/adobe-direct-download-links.html). For all newer versions, contact Adobe support for an offline installer. They typically respond quickly and provide installers within minutes.

*Tags: `compatibility`, `installer`, `older-versions`, `testing`*

---

### How can you test whether memory disposal is working correctly in an After Effects plugin?

One practical approach is stress-testing by rendering a long video (e.g., 5 hours) to identify memory leaks. If the plugin runs without crashing and memory usage remains stable throughout the render, it indicates that necessary resources have been properly disposed. This method relies on observation rather than formal memory profiling tools.

*Tags: `debugging`, `disposal`, `memory`, `testing`*

---
