# Gabriel Grenier

**1 contributions** to AE SDK community knowledge.

Top topics: `wrapper`, `gui-api`, `cpp`, `rust`, `boilerplate`, `sdk`

---

## Are there any good C++ wrapper libraries for the After Effects Filter GUI API?

No well-known general-purpose wrapper exists. Most wrappers that have been built end up being very complex themselves if you try to handle all use cases -- you end up with a non-standard system that is almost as complex as the original, just with more C++ features. Copy-pasting C chunks from the SDK examples works well in practice. That said, even a C wrapper with automatic memory management and state stored in structs would be an improvement over the current state machine. There is also a Rust binding project (https://github.com/virtualritz/after-effects/) that significantly reduces boilerplate compared to C/C++.

*Source: aescripts discord · 2024-09-27 · Tags: `wrapper`, `gui-api`, `cpp`, `rust`, `boilerplate`, `sdk` · [View in Q&A](../qa/wrapper/)*

---
