# Mylenium

**1 contributions** to AE SDK community knowledge.

Top topics: `sequential-frames`, `render-suite`, `face-detection`, `pre-processing`, `background-processing`

---

## How can I get decoded video frames in sequential order for an AE plugin that needs sequential processing (e.g., face detection)?

Use AEGP_RenderAndCheckoutFrame (from AEGP_RenderSuite4) to request any project item's image at any time in any order. AE's work scheme revolves around random frame access, so sequential processing works against the user experience. However, plugins like AE's camera tracker handle this by doing sequential pre-processing in the background while showing progress without blocking the UI. Consider the tradeoff: (1) Fast sequential pre-processing at the cost of usage intuitiveness, or (2) Slower random-access processing with the benefit of not waiting for the whole pre-process to finish.

*Source: adobe-forum-sdk · 2023-03-01 · Tags: `sequential-frames`, `render-suite`, `face-detection`, `pre-processing`, `background-processing` · [View in Q&A](../qa/sequential-frames/)*

---
