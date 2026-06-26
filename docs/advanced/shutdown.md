# Shutdown

> 4 Q&As · source: AE plugin dev community Discord

### Is 5000+ arb dispose calls on AE shutdown normal?

Yes, this is expected behavior. AE's allocators generally don't dispose until available memory is saturated, thread destruction, or app exit. You can test this by setting AE's available memory artificially low (like 2GB) - you'll see many more destruction calls during normal operation. On exit, AE cleans up everything at once.

*Tags: `arb-data`, `dispose`, `memory-management`, `shutdown`*

---

### What suites should you avoid calling from Death Hook?

You shouldn't rely on any suite being available in the Death Hook. Attempting to use suites like UtilitySuite's ReportInfo from Death Hook can cause unhandled exceptions. By that point in the shutdown process, many suites may already be torn down.

*Tags: `aegp`, `death-hook`, `shutdown`, `suites`*

---

### Is it normal to receive thousands of PF_Arbitrary_DISPOSE_FUNC calls on After Effects shutdown?

Yes, this is normal behavior. The high number of disposal calls during shutdown is likely due to Multi-Frame Rendering (MFR) duplicating arbitrary parameters for each thread, combined with parameter changes triggering dispose/copy cycles.

*Tags: `arb-data`, `memory`, `mfr`, `shutdown`, `threading`*

---

### How can I safely detect if a layer is available for checkout in a userChangedParam callback?

A workaround is to validate layer data by checking width/height and Lrect values—if they are random, over 100,000, or negative, the layer is not properly available for checkout. Note that Param[0].u.ld.data will be empty if checkout hasn't occurred in that thread. However, this is not a fully reliable solution, and the proper approach may require additional checks during effect duplication and shutdown phases.

*Tags: `arb-data`, `layer-checkout`, `params`, `shutdown`, `threading`*

---
