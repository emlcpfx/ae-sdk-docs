# Aegp_Api

> 1 Q&A · source: AE plugin dev community Discord

### How can you detect when a user sets parenting on a layer in After Effects?

There is no direct event to detect parenting changes. However, you can use several workarounds: (1) Keep a hidden slider that tracks the parent layer's index and detect changes via re-render calls when the index changes, (2) Register a command hook with the 'ALL COMMANDS' flag to monitor all commands sent by AE and look for parenting-related commands, or (3) Use expressions to recursively detect parenting changes through the layer chain. A slider with expressions is a practical approach for tracking parent changes.

*Tags: `aegp_api`, `debugging`, `params`, `ui`*

---
