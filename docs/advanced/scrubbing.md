# Scrubbing

> 1 Q&A · source: AE plugin dev community Discord

### Is there an equivalent to PF_ABORT() in Premiere for cancelling long renders?

PF_ABORT(in_data) does not work in Premiere -- it never sets err to PF_Interrupt_CANCEL even when scrubbing the timeline with frames taking over half a second to render. There may be a realtime flag that hints whether the effect should be treated as realtime or not, but no confirmed workaround for render cancellation in Premiere was identified.

*Tags: `pf-abort`, `premiere`, `realtime`, `render-cancel`, `scrubbing`*

---
