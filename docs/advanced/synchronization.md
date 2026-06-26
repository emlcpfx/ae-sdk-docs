# Synchronization

> 1 Q&A · source: AE plugin dev community Discord

### Can I call AEGP functions from an external thread, and what is the correct pattern for using idle hooks with external threads?

You can call any AE function from an external thread, but only when AE is ready for it. The correct pattern is: (1) register the idle hook and launch the external thread from the entry point function, (2) have your external thread store its data in a global scope structure, (3) in the idle hook function, check if the global structure is filled and ready; if so, execute processing code; if not, check again next time, (4) set a skip idle hook flag to true when done and kill the external thread. Alternatively, make a synchronized call to your server from within the idle hook and avoid the external thread complexity.

*Tags: `aegp`, `idle-hook`, `plugin-architecture`, `synchronization`, `threading`*

---
